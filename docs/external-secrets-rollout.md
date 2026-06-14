# DB password → AWS Secrets Manager via External Secrets Operator

How the backend's database password is sourced, and the exact order to roll it
out without an outage. The password must never live in git again.

## How it works after rollout

```
AWS Secrets Manager (pdm/backend/db-password)
   │
   ├── terraform → RDS master password           (pdm-infra)
   │
   └── External Secrets Operator                 (installed by pdm-infra)
          │  reads via the ClusterSecretStore "aws-secrets-manager"
          │  (auth: ESO controller IRSA role pdm-external-secrets-<env>)
          ▼
       ExternalSecret (rendered by the backend Helm chart, per namespace)
          ▼
       Secret  pdm-backend-secret / database-url
          ▼
       backend Deployment + migrate job + init container
```

The password lives in exactly one place: the `pdm/backend/db-password` secret in
AWS Secrets Manager. Terraform reads it for the RDS master password; ESO syncs the
same value into each namespace's `pdm-backend-secret`.

## The four PRs

| Repo | PR | What | Safe to merge alone? |
|------|----|------|----------------------|
| pdm-infra | #3 | RDS password from Secrets Manager; ESO install + IRSA role | Yes (after the secret exists) |
| pdm-app | #83 | Chart `ExternalSecret` template; `secret.yaml` becomes the no-ESO fallback (`externalSecrets.enabled`, default off) | Yes (no-op by default) |
| pdm-gitops | #1 | `ClusterSecretStore` (this PR) | Yes (after ESO CRDs exist) |
| pdm-gitops | #2 | **Cutover**: remove `postgres.password`, set `externalSecrets.enabled=true` | **No — merge last** |

## Rollout order

### 1. Create the secret (do this first — it's a precondition for terraform)

```bash
aws secretsmanager create-secret \
  --name pdm/backend/db-password \
  --secret-string '<a-new-strong-password>' \
  --region us-east-1
```

Use a **new** value here, not `Yoavmanoim70` — the old one is burned (it was in
git history). Setting a new value here is also the rotation: step 2 applies it to RDS.

### 2. Apply pdm-infra #3

```bash
cd pdm-infra/environments/production
terraform plan    # expect: RDS password update, ESO helm release, 2 IAM resources
terraform apply
```

This installs ESO (CRDs + controller with the IRSA annotation) and sets the RDS
master password to the Secrets Manager value.

Confirm ESO is up:

```bash
kubectl -n external-secrets get pods
kubectl get crd | grep external-secrets.io   # clustersecretstores, externalsecrets, ...
```

### 3. Merge pdm-gitops #1 (ClusterSecretStore)

ArgoCD creates the store. Confirm it validates against AWS:

```bash
kubectl get clustersecretstore aws-secrets-manager -o jsonpath='{.status.conditions}'
# want: type=Ready, status=True
```

### 4. Merge pdm-app #83 (chart support)

No behavior change by itself (`externalSecrets.enabled` defaults to false). Make
sure it lands on the chart revision each env's ArgoCD app tracks:
**production → `HEAD`, staging/dev → `develop`** (see each `argocd-app.yaml`).

### 5. Merge pdm-gitops #2 (the cutover) — LAST

Mark the draft ready and merge. ArgoCD re-renders the backend with
`externalSecrets.enabled=true`: the chart now emits an `ExternalSecret` instead of
the plaintext `Secret`, ESO fills `pdm-backend-secret`, and pods roll with the new
password. Verify before walking away:

```bash
kubectl -n pdm-production get externalsecret pdm-backend-secret \
  -o jsonpath='{.status.conditions}'          # want: Ready / SecretSynced
kubectl -n pdm-production get secret pdm-backend-secret -o jsonpath='{.data.database-url}' \
  | base64 -d                                  # sanity-check host/db, password present
kubectl -n pdm-production rollout status deploy/pdm-backend
```

Repeat the spot check for `pdm-staging` and `pdm-dev`.

## Rotating later

1. Update the `pdm/backend/db-password` secret value in Secrets Manager.
2. `terraform apply` in pdm-infra (updates the RDS master password).
3. ESO re-syncs within `refreshInterval` (1h); to apply immediately, annotate the
   ExternalSecret to force a refresh or delete `pdm-backend-secret` (ESO recreates it),
   then `kubectl rollout restart deploy/pdm-backend` in each namespace.

## Rollback

Before the cutover (#2) merges, reverting any single PR is safe. After the cutover,
to fall back to a templated secret: revert pdm-gitops #2 (restores
`postgres.password` + `externalSecrets.enabled=false`). Note this puts a password
back in git — only a break-glass measure; prefer fixing the ESO/store path.
