# Promotion Plan

This is the target flow after the clusters are working.

## Desired Flow

1. A code change is pushed.
2. CI runs validation.
3. If validation passes, deploy to `dev`.
4. If dev is healthy, promote to `staging`.
5. If staging is healthy, request approval for `prod`.
6. After approval, promote to `prod`.

## Recommended GitOps Model

Use Git as the promotion gate:

- Dev tracks the newest approved change.
- Staging tracks the dev version after validation.
- Prod changes require manual approval.

For this lab, the practical first step is:

- Keep Argo CD deploying from this config repo.
- Use CI to validate every pull request and push.
- Use GitHub protected environments for the prod approval gate.

Later, when the app image is built in CI, each environment overlay should pin the image tag explicitly. Promotion then means copying the tested image tag forward:

```text
dev image tag -> staging image tag -> prod image tag
```

That avoids accidentally deploying a different image to prod than the one tested in staging.

## Validation Checks

Minimum checks before promotion:

- Render all Kustomize overlays.
- Render Argo CD platform manifests.
- Confirm each overlay uses the expected namespace.
- Confirm prod has more than one replica.
- Confirm the container image tag is explicit.

## Approval Gate

Use a GitHub Environment named `prod` with required reviewers. A GitHub Actions job that targets `environment: prod` will pause and ask for approval before continuing.

