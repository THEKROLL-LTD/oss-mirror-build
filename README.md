# oss-mirror-build

A GitHub Actions template for building, scanning, and hardening OSS container images from upstream source. Produces audit-ready artifacts — SBOM, scan reports, diff-review — on every upstream bump, with a pull request as the human review gate.

Designed for teams that need to defensibly self-host third-party OSS containers but don't want to pay Chainguard or Docker Hardened Images pricing. Rebases onto [Google's distroless](https://github.com/GoogleContainerTools/distroless) by default; any minimal base works.

For the "why and when" argument behind this pipeline, see the companion post: *[Stop Pulling Random Docker Images](https://medium.com/@sventhekroll/stop-pulling-random-docker-images-c19e94559cc6)*.

## What it does

Each night (03:00 UTC by default), the pipeline in your fork:

1. Checks your configured upstream repository for a new release tag
2. Clones the upstream source at that tag
3. Applies your Dockerfile override if present — this is how you rebase onto distroless or any hardened base
4. Generates a diff-review artifact: commit log and full patch between the last built tag and the new one
5. Builds the container image once, loaded locally
6. Scans the built image with [Trivy](https://github.com/aquasecurity/trivy), producing a SARIF report for the Security tab and a CycloneDX SBOM as a build artifact
7. If any `CRITICAL` or `HIGH` vulnerability with an available fix is found: blocks the push, opens an issue, retains all artifacts
8. If clean: pushes to your GHCR namespace and opens a pull request with the new digest pinned in `image-pin.yml`

On merge of the PR, your GitOps system (Flux, ArgoCD, or anything that pins `image-pin.yml`) rolls out the new digest.

## Quickstart

Five steps. Assumes you've worked with a statically-compiled upstream like Go or Rust before; more complex runtimes (Python with native deps, C with runtime-loaded libs) take longer for the first override.

**1. Fork via the "Use this template" button.**

**2. Enable Actions in your new repository.** GitHub disables them in template-derived repos by default. Settings → Actions → General → "Allow all actions and reusable workflows".

**3. Configure the two required variables** in `.github/workflows/mirror-build.yml`:

```yaml
env:
  UPSTREAM_REPO: "Forceu/Gokapi"
  IMAGE_NAME:    "ghcr.io/your-org/gokapi"
```

`IMAGE_NAME` must be lowercase. Container registries reject uppercase in the repository name portion. Until both values are set (and lowercase), the pipeline fails fast with a clear error.

**4. (Recommended) Add a Dockerfile override** for a hardened base. Copy the reference from `examples/gokapi/Dockerfile.override` to `dockerfiles/Dockerfile.override` and adjust for your upstream:

```dockerfile
FROM golang:1.25-alpine AS build
WORKDIR /src
RUN apk add --no-cache git
COPY . .
RUN go generate ./... && \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" \
      -o /out/gokapi github.com/forceu/gokapi/cmd/gokapi

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/gokapi /gokapi
USER nonroot:nonroot
EXPOSE 53842
ENTRYPOINT ["/gokapi"]
```

Without an override, the pipeline builds the upstream Dockerfile unchanged. That still gives you an SBOM, scan reports, and a controlled registry — but Trivy will surface whatever CVEs live in the upstream's base layer. With a `distroless/static` override on a static Go binary: typically zero OS-layer findings, only application-layer CVEs remain.

**5. Trigger the workflow manually once** (Actions tab → OSS Mirror Build → Run workflow). If the first run goes green and opens a PR, the nightly schedule takes over from there.

## The Dockerfile override mechanism

If `dockerfiles/Dockerfile.override` exists in your repository, the pipeline copies it into the cloned upstream source, replacing the upstream `Dockerfile` entirely. The build then runs against your override.

Deliberately a full-file replacement, not a patch. Reasons:

- Robust against upstream whitespace or layout changes
- One place to reconcile when upstream bumps language versions
- No `patch --reject` drama in CI

You still track upstream source — the pipeline clones it fresh every run. Only the container recipe is yours.

### Module-path patching (Go projects that ship v2+ without `/v2` in go.mod)

Some Go projects publish v2.x.x or higher release tags but keep
`module foo/bar` in `go.mod` instead of the required `module foo/bar/v2`.
Go treats every such tag as invalid and stamps the binary's
`runtime/debug.BuildInfo.Main.Version` with a pseudo-version like
`v0.0.0-<timestamp>-<sha>` instead of the tag. Vulnerability scanners
(Trivy, Grype) then cannot semver-match against CVE fix versions and
report false positives for every known CVE against the main module —
not because anything is actually vulnerable, but because they can't
confirm the fix is present.

If your upstream hits this, patch the module path inside your
`Dockerfile.override` before `go build`:

1. `sed` on `go.mod` to append `/v2` (or `/vN`) to the module directive
2. `find | xargs sed` on all `.go` files to rewrite in-repo self-imports
3. `git commit` the patch and `git tag -f <upstream-tag>` onto the
   resulting commit — without a clean tree Go falls back to `(devel)`,
   without the retag it emits a pseudo-version

The workflow passes the upstream tag as `--build-arg VERSION=<tag>` to
every build, so your override can reference it directly. See
`examples/gokapi/Dockerfile.override` for a worked example.

Rule of thumb: if `head -1 go.mod` says `module foo/bar` and the upstream
ships v2.x.x+ tags, you need this patch. If `go.mod` already ends in
`/vN` matching the tag major, a plain override works without patching.

## What each run produces

**Success path:**

- Image pushed to `ghcr.io/<your-org>/<image>:<tag>` and `ghcr.io/<your-org>/<image>@sha256:<digest>`
- Pull request opened on branch `mirror-build/<tag>` against `main`
- PR body contains: digest, Dockerfile source (`override` or `upstream`), scan summary, links to artifacts
- CycloneDX SBOM, Trivy SARIF, diff-review markdown, full diff patch — all retained 90 days as workflow artifacts
- Trivy findings uploaded to the repository's Security tab

**Blocked path (CVE found):**

- No image pushed
- Issue opened with labels `security`, `cve`, `blocked`, linking the workflow run and Security tab
- Same artifacts retained for review
- Pipeline fails, blocking the next nightly until resolved (either a fixed upstream version appears or you add a documented waiver to `.trivyignore`)

## Detecting CVEs published after release

The main build job only scans images at build time — if a new CVE is
published tomorrow against a dependency in yesterday's image, the main
job won't notice until the next upstream release bumps the affected
dependency.

A second scheduled job, `rescan-last-build`, closes this gap. It runs
on the same nightly schedule, pulls the last image recorded in
`.last-built-tag`, rescans it with Trivy against the current database,
and auto-files an issue (label `rescan`) if new findings appear. The
image itself is never modified — this is information, not remediation.
The decision *"wait for upstream"* vs. *"patch via Dockerfile override
and rebuild"* stays human.

If your GHCR package is private, make sure the package settings link
the mirror repository explicitly, otherwise the rescan job will fail
with a 403 on image pull.

## Configuration reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `UPSTREAM_REPO` | yes | empty | `owner/repo` on github.com |
| `IMAGE_NAME` | yes | empty | Full image reference including registry (lowercase), e.g. `ghcr.io/my-org/name` |
| `TRIVY_SEVERITY` | no | `CRITICAL,HIGH` | Comma-separated Trivy severity levels that block the push |

The schedule is hardcoded to `0 3 * * *` UTC in the workflow file. Change it if you need a different cadence.

## FAQ

**Upstream moved the Dockerfile or renamed the main package.**
Update your `Dockerfile.override` accordingly. The clone step always pulls the full tree, so paths inside `upstream-src/` reflect whatever the upstream has now.

**Upstream bumped Go (or Python, or Node) to a version my override can't build against.**
The override declares its own build-stage base image; upstream's runtime version is irrelevant. Update the `FROM <lang>:<version>` line in your override. For static Go binaries this is usually a one-line change.

**Trivy is flagging a CVE I can't act on.**
Add an entry to `.trivyignore` with a comment explaining why:

```
# CVE-2025-12345 — libcurl in upstream base. Not reachable from our deployment
# (container runs behind mTLS ingress, no outbound HTTP). Revisit 2026-08.
CVE-2025-12345
```

Trivy reads both the CVE ID and the comment, and the Git history of this file becomes part of your audit trail.

**Branch protection on `main` prevents me from merging my own PRs as the only maintainer.**
Either grant yourself admin override, or relax the "require approvals" rule on this repository specifically. `mirror-build/*` branches are bot-authored, so "require review from someone other than the author" technically doesn't block you — but many teams configure it more strictly.

**Upstream has no GitHub releases and no git tags.**
The pipeline requires at least git tags. Projects that ship only via rolling `main`-branch releases are out of scope for this template — you'd be mirroring a moving target without a clear review unit.

**The first successful run shows `Initial build of <tag>` with no diff content.**
Expected. There's no prior tag to diff against. From the next successful build onward, the diff-review artifact shows what changed between versions.

**I want to debug a running pod but there's no shell in my distroless image.**
Use `kubectl debug <pod> --image=busybox -it -- sh` with an ephemeral container, or build a `:debug` variant from `gcr.io/distroless/static:debug` for non-production. Distroless deliberately ships no shell to reduce the runtime attack surface.

## Pinning action dependencies

The shipped workflow references actions by major tag (`@v4`, `@v6`) and Trivy by pinned release. For production use, pin every action to a commit SHA. [Renovate](https://docs.renovatebot.com/) is the better fit for this template than Dependabot — it can group Action updates into a single weekly PR, avoiding collisions with the `mirror-build/*` PRs this pipeline generates. A reference config ships as `renovate.json.example`; copy to `renovate.json` after fork. Dependabot works too but requires more tuning.

## License of mirrored images

The Apache-2.0 license above covers this template's own contents — workflow
YAML, documentation, and reference overrides. It does **not** govern the
container images you build with it.

Mirror images inherit the license of the upstream they contain. Examples:

- If you mirror an **AGPL-3.0** upstream (e.g. Gokapi), your published
  image is AGPL-3.0. Operators of that image are bound by AGPL §13
  (Network Use) — they must provide source access to network users.
- If you mirror a **GPL-2.0 / GPL-3.0** upstream, the published image
  inherits those terms correspondingly.
- If you mirror an **MIT / Apache-2.0 / BSD** upstream (e.g. root-gg/plik
  is MIT), the image follows those permissive terms.

Before publishing mirrored images to a public registry, verify the
upstream's `LICENSE` file and add a `NOTICE.md` to your fork that declares
the image license and points at the upstream source. See
[`THEKROLL-LTD/mirror-Gokapi`](https://github.com/THEKROLL-LTD/mirror-Gokapi)
`NOTICE.md` for a worked example.

## License

Apache-2.0. Fork it, adapt it, ship it.

## Related

- **Blog post:** *[Stop Pulling Random Docker Images](https://medium.com/@sventhekroll/stop-pulling-random-docker-images-c19e94559cc6)* — the argument and context this template implements
- **Live example:** [`THEKROLL-LTD/mirror-gokapi`](https://github.com/THEKROLL-LTD/mirror-gokapi) — a working fork that builds and scans `Forceu/Gokapi` nightly. Pipeline runs, PRs get merged, images land in `ghcr.io/thekroll-ltd/gokapi`. No SLA; use for reference or pull at your own risk

## Maintained by

[THEKROLL](https://thekroll.ltd). Issues and pull requests welcome — especially Dockerfile overrides for upstreams not yet covered in `examples/`.
