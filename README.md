# ci-config

Reusable GitHub Actions workflows shared across a fleet of small services: build an image, and
check that nothing private got committed.

Public so that private repositories can call it without the per-repo "Accessible from
repositories owned by …" setting, and so the workflows can be pinned to a tag rather than
tracking a moving branch.

Nothing here identifies its owner. That is deliberate and load-bearing: the identity gate below
used to hardcode an email, a personal domain and NAS path prefixes, in the workflow whose entire
job was keeping exactly those strings out of source. That made it unpublishable. It now takes the
pattern as a secret from the caller.

## `reusable-build.yml`

Builds a Docker image and pushes it to GHCR as `:latest` and `:<sha>`. It does **not** deploy, so
callers hold no deploy secrets.

```yaml
jobs:
  build:
    uses: <owner>/ci-config/.github/workflows/reusable-build.yml@main
    with:
      image-name: ghcr.io/<owner>/<service>
      context: .          # optional, some repos keep the Dockerfile under ./api
    secrets: inherit
```

Fetches submodules first when the repo has a `.gitmodules`. All shared config packages are
public, so this needs no token.

## `secret-scan.yml`

Two independent jobs.

**gitleaks** runs against the caller's own `.gitleaks.toml`, so each repo can allowlist its own
false positives. Vendor that file into the repo root.

**identity-gate** greps the source for strings that should never be committed — an owner's email,
a personal domain, internal hostnames or filesystem paths. These are not secrets, so gitleaks
will not catch them, but they are exactly what makes a repository unpublishable later.

```yaml
jobs:
  scan:
    uses: <owner>/ci-config/.github/workflows/secret-scan.yml@main
    secrets: inherit
```

`secrets: inherit` passes `IDENTITY_PATTERN`, an extended regex of those strings, set per
repository. Optionally set `identity-include` to widen the file glob, which defaults to `*.go`.

**The gate fails closed.** With no pattern, `grep -E ''` matches every line, so the alternative to
erroring is a check that reports green while examining nothing.

## Why the pattern is a secret and not an input

A workflow input would be simpler, and would put the identity into every calling repository's
YAML — and its git history, permanently. Since the reason to care about any of this is being able
to open-source a repo later, and history is the part that is hard to clean, that would defeat the
purpose. As a secret it lives once per repository and in no file at all.
