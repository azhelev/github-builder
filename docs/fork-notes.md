# Fork notes (`azhelev/github-builder`)

Notes about how this fork diverges from upstream `docker/github-builder`,
why, and the landmines to watch for.

## Why the fork exists

To call the `build.yml` / `bake.yml` reusable workflows from caller repos
that push to **Amazon ECR via OIDC** (no static registry credentials), and
to make signing/caching behave sensibly when the workflow runs from a
fork instead of `docker/github-builder`.

The branch holding these changes is `oidc-ecr`. Callers reference the
workflow as e.g.
`azhelev/github-builder/.github/workflows/build.yml@oidc-ecr`.

## What's different from upstream

1. **ECR OIDC login** — new inputs (`aws-ecr-login`, `aws-region`,
   `aws-role-to-assume`, optional `aws-ecr-registries`,
   `aws-ecr-registry-type`) that run
   `aws-actions/configure-aws-credentials` + `aws-actions/amazon-ecr-login`
   before push/sign/manifest steps. `docker-login: false` lets the caller
   skip the upstream docker-login step entirely.

2. **Cosign cert-identity hardcoded to the fork's path.** Upstream
   hardcodes `docker/github-builder` in the `certificateIdentityRegexp`
   and `subjectAlternativeName` strings that verify signatures the
   workflow itself just produced. In a fork the signing cert's SAN
   points at the fork's repo, so the upstream regex never matches and
   verify fails. The fork hardcodes `azhelev/github-builder` instead.

   Locations:
   - `.github/workflows/build.yml` — three spots (buildkit cache verify
     policy, attestation-manifest verify, local-artifact verify).
   - `.github/workflows/bake.yml` — three matching spots.

   Upstream's *dependency-verification* regex (verifying `BUILDKIT_IMAGE`
   / `SBOM_IMAGE` / `BINFMT_IMAGE`, which are signed by
   `docker/github-builder` upstream) is left alone — that one really
   should still point at `docker/github-builder`.

3. **GHA cache signing disabled.** `prepare.outputs.ghaCacheSign` is
   hardcoded to `'false'` (instead of `inpActionsIdTokenSet ? 'true' :
   'false'`). This disables both `[cache.gha.sign]` and
   `[cache.gha.verify]` in the inline buildkit config, so the GHA cache
   import/export works without Sigstore round-trips. Suitable for a
   single-org fork; the original feature was for cross-project cache
   sharing under docker's own infrastructure.

## Landmines

- **If you rename or move this fork** (new owner, new repo name), the
  hardcoded `azhelev/github-builder` strings (six total: three in
  `build.yml`, three in `bake.yml`) need updating, or signing will start
  failing again with `expected SAN value to match regex ...`.

- **`github.repository` in a reusable workflow = the caller**, not the
  reusable workflow itself. Don't try to parameterize the cert-identity
  with it.

- **`github.workflow_ref` is also the *entry* workflow's ref**, not the
  reusable workflow's. Same trap. There is no Actions context variable
  for "the currently-executing reusable workflow's repo/ref" — that's
  why upstream hardcoded the path and why we hardcode it too.

- **Pulling from upstream `docker/github-builder`** will conflict on the
  hardcoded paths and the `ghaCacheSign` line. Re-apply the changes
  after merging.

## Caller invocation example

Minimum YAML for a caller pushing a multi-arch image to ECR via OIDC,
with no docker-login and with cache enabled:

```yaml
jobs:
  build-and-push:
    uses: azhelev/github-builder/.github/workflows/build.yml@oidc-ecr
    permissions:
      contents: read
      id-token: write          # required for ECR OIDC and cosign
    with:
      file: deployment/Dockerfile
      context: .
      output: image
      platforms: linux/amd64,linux/arm64
      push: ${{ github.event_name != 'pull_request' }}
      cache: true
      cache-mode: max
      meta-images: <aws-account-id>.dkr.ecr.<region>.amazonaws.com/<ecr-repo>
      meta-tags: |
        type=sha
      docker-login: false
      aws-ecr-login: true
      aws-region: <region>
      aws-role-to-assume: arn:aws:iam::<aws-account-id>:role/<role-name>
```

The caller needs an IAM role with a trust policy that permits
`token.actions.githubusercontent.com` to assume it from
`repo:<caller-org>/<caller-repo>:*`, plus an attached policy granting
the ECR push permissions on the target repository.
