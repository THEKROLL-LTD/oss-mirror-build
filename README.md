# oss-mirror-build

A GitHub Actions template for building, scanning, and hardening OSS container images from upstream source. Produces audit-ready artifacts — SBOM, scan reports, diff-review — on every upstream bump, with a pull request as the human review gate.

Designed for teams that need to defensibly self-host third-party OSS containers but don't want to pay Chainguard or Docker Hardened Images pricing. Rebases onto [Google's distroless](https://github.com/GoogleContainerTools/distroless) by default; any minimal base works.

For the "why and when" argument behind this pipeline, see the companion post: *[Stop Pulling Random Docker Images](https://medium.thekroll.ltd/stop-pulling-random-docker-images-c19e94559cc6)*.

## What it does

Each night (03:00 UTC by default), the pipeline in your fork:

1. Checks your configured upstream repository for a new release tag
2. Clones the upstream source at that tag
3. Applies your Dockerfile override if present — this is how you rebase onto distroless or any hardened base
4. Generates a diff-review artifact: commit log and full patch between the last built tag and the new one
4a. **(override only)** Diffs the upstream Dockerfile path(s) — configurable via `UPSTREAM_DOCKERFILE_PATHS` — between the last reviewed tag and the new one. If anything changed, files an `override-review` issue and blocks the build until a human closes it. Catches silent drift between your `Dockerfile.override` and a restructured upstream Dockerfile that Trivy can't detect (new apt packages, renamed COPY paths, restructured build stages).
5. Runs a [Trivy](https://github.com/aquasecurity/trivy) **filesystem scan** of the cloned source, reading every language-layer lockfile (`mix.lock`, `package-lock.json`, `requirements.txt`, `Gemfile.lock`, `Cargo.lock`, `go.sum`, …)
6. Builds the container image once, loaded locally
7. Runs a Trivy **image scan**, producing a SARIF report for the Security tab and a CycloneDX SBOM as a build artifact
8. If either scan finds a `CRITICAL` or `HIGH` vulnerability with an available fix: blocks the push, opens a single issue covering both scans, retains all artifacts
9. If clean: pushes to your GHCR namespace and opens a pull request with the new digest pinned in `image-pin.yml`

Why two scans: an image scan only sees what survives the build. For Go and Rust binaries that's enough — every dep is in the binary's BuildInfo. For Elixir, Python, Ruby, Node and similar runtimes the lockfiles are build-stage-only and the runtime image is just compiled bytecode, so the language-layer CVE landscape is invisible to an image-only scan. The fs scan closes that gap.

On merge of the PR, your GitOps system (Flux, ArgoCD, or anything that pins `image-pin.yml`) rolls out the new digest.

## Issues and PRs the workflow opens

Everything the maintainer ever has to act on shows up in one of four places: a pull request or one of three issue types. The bot owns the lifecycle — it opens, updates, and closes them automatically. You only act when an issue is in front of you.

| Surface | When it appears | Label(s) | What you do | Auto-close |
|---|---|---|---|---|
| **Pull request** on `mirror-build/<tag>` | Successful build: scan clean, image pushed to GHCR | `mirror-build` | Review the diff (upstream changes between last and new tag), merge to roll out the digest. | n/a — closed by your merge |
| **Issue: `CVE found: …`** | Build-time scan found `CRITICAL`/`HIGH` with available fix; image was **not pushed** | `security`, `cve`, `blocked` | Decide: wait for upstream, add a documented `.trivyignore` waiver, or roll a Dockerfile-override patch. The build stays blocked until the next clean scan. | Closes itself when a later run scans clean for the same tag (`block-resolved`) |
| **Issue: `New CVE detected in deployed image …`** | Nightly rescan of the previously-published image found a new finding (CVE database advanced after the build) | `security`, `cve`, `rescan` | Information only — the image is unchanged. Decide whether to wait for upstream or trigger an override+rebuild via `workflow_dispatch`. | Closes itself when the next rescan is clean (`rescan-resolved`) |
| **Issue: `Override review required: …`** | Upstream changed one of the Dockerfile paths in `UPSTREAM_DOCKERFILE_PATHS` since you last reviewed; build is **blocked** | `override-review`, `blocked` | Read the diff in the issue, decide whether `dockerfiles/Dockerfile.override` needs updating. **Closing this issue IS your approval** — the build is unblocked on the next `workflow_dispatch`. | n/a — you close it manually; closing is the unblock |

A few non-obvious lifecycle properties:

- **All issues are idempotent per tag.** Re-running the same workflow against the same tag never spams duplicates. The bot finds the existing issue by hidden marker, updates body if findings shifted, posts a diff comment, and otherwise stays silent.
- **Closing a CVE issue does NOT clear the underlying CVE.** It's treated as triage ("I've seen this finding-set"). If you close one and tomorrow's scan surfaces *different* CVEs against the same tag, the issue is **automatically reopened** with a comment listing what was added/cleared. If the next scan is bit-for-bit identical, the close is honored — no nagging.
- **The `override-review` issue is the only one where closing is meaningful by itself.** Closing it tells the bot "the override still matches the new upstream, proceed." There is no automated reopen for override-reviews — re-flagging requires another upstream Dockerfile change.
- **First run on a fresh fork** files no diff (no prior tag exists), and the rescan job skips silently (no `.last-built-tag` yet). From the second successful build onward, both kick in.

If you only do one thing: **subscribe to `cve` and `override-review` labels in your fork's notification settings.** Those two cover every case where the build is blocked and waiting on you.

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

### Daily distro-package refresh (`APT_REFRESH`)

The workflow forwards a `APT_REFRESH=<UTC-date>` build-arg into every
build. The intent is to invalidate the apt/apk install layer once per
day so Debian (or Alpine) security advisories actually land in the
nightly image.

Why this is needed: BuildKit hashes a `RUN` layer on `(parent layer +
command string)`. The state of the upstream package mirror is **not**
part of that hash. With `cache-from: type=gha` and an unchanged
Dockerfile, BuildKit cache-hits on the apt layer indefinitely — even
when Debian has shipped a new chromium/openssl/etc. The image gets
re-tagged but the userland is frozen at the cache moment, and Trivy
keeps flagging CVEs that have a fix available upstream.

Threading `APT_REFRESH` (today's UTC date) into the package-install
command bumps the layer hash exactly once per UTC day. Within the same
day, cache reuse is unaffected; across day boundaries, the apt/apk
layer is re-resolved against the live mirror state. This matches
Debian's security cadence, which publishes most updates day-of.

To use it in an override, declare the arg and reference it in the same
`RUN` that calls `apt-get update`/`apk upgrade`:

```dockerfile
ARG APT_REFRESH=unset

RUN echo "apt-refresh=${APT_REFRESH}" > /etc/.apt-refresh && \
    apt-get update && \
    apt-get install --no-install-recommends -y …
```

A few gotchas:

- The default of `unset` keeps local `docker build` invocations working
  without the arg (no daily cache-bust then, but no breakage either).
- Threading it into the **first** apt/apk-using `RUN` is enough — every
  later layer is downstream of that hash, so the cascade rebuilds them
  too.
- The name is `APT_REFRESH` for historical reasons (apt was the
  original target). Alpine-based overrides reuse the same arg with
  `apk` — the mechanism is identical.
- Overrides that don't declare `ARG APT_REFRESH` ignore the build-arg
  silently. Pure-distroless images (Gokapi-style) don't need it.

## Artifacts every run produces

Whether or not the build path ends with a push, the same evidence bundle is retained for 90 days as workflow artifacts so audits can reconstruct what was built and what was scanned:

- CycloneDX SBOMs (image + filesystem)
- Trivy SARIF reports (image + fs) — also uploaded to the repository's Security tab under categories `trivy-image`, `trivy-fs`, and `trivy-rescan`
- Trivy JSON reports (used by the bot to file issues; same data, different format)
- `diff-review.md` and `diff-full.patch` — git diff between the last built tag and the new one

On a successful build the image is also pushed to `ghcr.io/<your-org>/<image>:<tag>` and `ghcr.io/<your-org>/<image>@sha256:<digest>`. The PR body contains: digest, Dockerfile source (`override` or `upstream`), scan summary, links to artifacts.

## How the rescan job works

The build-time scan only sees what existed when the image was built. If a new CVE is published tomorrow against a dependency in yesterday's image, the main job won't notice until the next upstream release bumps the affected dependency. The `rescan-last-build` job closes that gap — it runs on the nightly schedule (parallel to the main build), pulls the image recorded in `.last-built-tag`, rescans it against the current Trivy database, and files a `rescan`-labeled issue if anything new shows up.

The image itself is never modified by this job — this is information, not remediation. The decision *"wait for upstream"* vs. *"patch via Dockerfile override and rebuild"* stays human. The `rescan` issue auto-closes the moment a later rescan comes back clean (typically when upstream ships the fix).

If your GHCR package is private, link the mirror repository explicitly in the package settings, otherwise the rescan job will fail with a 403 on image pull.

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

- **Blog post:** *[Stop Pulling Random Docker Images](https://medium.thekroll.ltd/stop-pulling-random-docker-images-c19e94559cc6)* — the argument and context this template implements
- **Live example:** [`THEKROLL-LTD/mirror-gokapi`](https://github.com/THEKROLL-LTD/mirror-gokapi) — a working fork that builds and scans `Forceu/Gokapi` nightly. Pipeline runs, PRs get merged, images land in `ghcr.io/thekroll-ltd/gokapi`. No SLA; use for reference or pull at your own risk

## Maintained by

[THEKROLL](https://thekroll.ltd). Issues and pull requests welcome — especially Dockerfile overrides for upstreams not yet covered in `examples/`.
