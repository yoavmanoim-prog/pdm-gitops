# Database split plan (prod / staging / dev)

## Why
Production, staging, and dev all share **one RDS database** (`pdm`, user `pdm`).
On 2026-06-14 this caused a production outage: staging/dev had migrated the shared
DB to Alembic revision `0004`, but production's pinned image predated `0004`, so a
pod roll failed `Can't locate revision identified by '0004'`. It also commingles
all three environments' data. Splitting gives each env its own schema, migration
state, and data.

## Target topology
Same RDS instance, **separate databases + users** (cheapest; full schema/data
isolation at no extra infra cost — separate instances are overkill here):

| Env | Database | User | Password secret |
|-----|----------|------|-----------------|
| production | `pdm` (existing — keep, preserves data) | `pdm` (RDS master) | `pdm/backend/db-password` |
| staging | `pdm_staging` (new, empty) | `pdm_staging` | `pdm/staging/db-password` |
| dev | `pdm_dev` (new, empty) | `pdm_dev` | `pdm/dev/db-password` |

**Caveat:** the current `pdm` DB holds commingled prod+staging+dev rows (no env
column). Production keeps `pdm` as-is (least risk). Staging/dev start fresh. If you
want production's DB free of test data, that's a separate data cleanup, out of scope.

## Steps

### 1. Create databases + users (one-time)
RDS isn't public — run psql from inside the cluster:
```bash
kubectl run pg --rm -it -n pdm-production --image=postgres:16-alpine --restart=Never \
  --env=PGPASSWORD="$(aws secretsmanager get-secret-value --secret-id pdm/backend/db-password \
     --region us-east-1 --query SecretString --output text)" -- \
  psql -h pdm-prod-eks-postgres.ci3ceikmuv20.us-east-1.rds.amazonaws.com -U pdm -d postgres
```
```sql
CREATE DATABASE pdm_staging;
CREATE DATABASE pdm_dev;
CREATE USER pdm_staging WITH PASSWORD '<staging-pw>';
CREATE USER pdm_dev     WITH PASSWORD '<dev-pw>';
GRANT ALL ON DATABASE pdm_staging TO pdm_staging;
GRANT ALL ON DATABASE pdm_dev     TO pdm_dev;
\c pdm_staging
GRANT ALL ON SCHEMA public TO pdm_staging;   -- needed on Postgres 15+
\c pdm_dev
GRANT ALL ON SCHEMA public TO pdm_dev;
```

### 2. Store the new passwords in Secrets Manager
```bash
aws secretsmanager create-secret --name pdm/staging/db-password --secret-string '<staging-pw>' --region us-east-1
aws secretsmanager create-secret --name pdm/dev/db-password     --secret-string '<dev-pw>'     --region us-east-1
```

### 3. Point staging at its own DB (gitops `staging/backend/values.yaml`)
```yaml
postgres:
  user: "pdm_staging"
  database: "pdm_staging"
externalSecrets:
  enabled: true
  passwordSecretName: pdm/staging/db-password
```
ArgoCD syncs → ESO builds `DATABASE_URL` for `pdm_staging` → the migrate initContainer
runs `alembic upgrade head` against the empty DB → schema created → staging isolated.
Verify the pod is healthy and `externalsecret`/secret are `SecretSynced`.

### 4. Fix node capacity (prerequisite for re-enabling dev)
The cluster sits at the 2× t3.medium pod ceiling; the 3rd node keeps getting removed
by the CPU-based ASG scale-in policy (it can't see pod-count pressure). Before
re-enabling dev, make a 3rd node stick: raise the managed node group `min_size` to 3
in terraform (and/or replace the SimpleScaling scale-in alarm with Cluster
Autoscaler / Karpenter). Otherwise dev and Elasticsearch will fight for the last slot.

### 5. Point dev at its own DB and re-enable
```yaml
replicaCount: 1   # currently 0 (turned off after the incident)
postgres:
  user: "pdm_dev"
  database: "pdm_dev"
externalSecrets:
  enabled: true
  passwordSecretName: pdm/dev/db-password
```

### 6. Production cutover to ESO (the original goal — now safe)
Once staging/dev are off the shared DB, nothing else migrates `pdm`, so production
is stable on it. The risky prep is already done (the manual AWS-cred drift env was
removed and `maxSurge=1/maxUnavailable=0` is set), so this is now the boring,
staging-style cutover — edit `production/backend/values.yaml`:
```yaml
postgres:
  user: "pdm"
  database: "pdm"
# remove: password: "..."
externalSecrets:
  enabled: true
```
Watch the ExternalSecret sync and a fresh pod come up. This removes the committed
password from production's gitops values (the secret now comes from Secrets Manager).

### 7. Rotate the production password
```bash
aws secretsmanager put-secret-value --secret-id pdm/backend/db-password \
  --secret-string '<new-strong-password>' --region us-east-1
cd pdm-infra/environments/production && terraform apply -var-file=vars/production.tfvars
# force ESO to resync, then roll:
kubectl annotate externalsecret pdm-backend-secret -n pdm-production force-sync="$(date +%s)" --overwrite
kubectl rollout restart deploy/pdm-backend -n pdm-production
```
`terraform apply` rotates the RDS **master** (`pdm`) password = production's user.
Staging/dev users are separate — rotate them with `ALTER USER ... PASSWORD` + update
their SM secrets (they are not managed by terraform).

## Order summary
1. Create DBs + users + SM secrets.
2. Staging → own DB (verify).
3. Make a 3rd node stick (capacity).
4. Dev → own DB, re-enable.
5. Production cutover to ESO.
6. Rotate production (and separately staging/dev users).

## Status (2026-06-14) — DONE for prod + staging
- ✅ `pdm_staging` / `pdm_dev` databases + users created; passwords in Secrets Manager.
- ✅ **Staging** isolated onto `pdm_staging` — healthy.
- ✅ **Production** cut over to ESO and **password rotated**. The old committed
  password is now a **dead credential** (RDS rejects it), so the git-history exposure
  is neutralized. Production runs on `pdm` with the new password, which lives only in
  Secrets Manager.
- ⏳ **Remaining:** make a 3rd node stick (capacity), then point **dev** at `pdm_dev`
  and re-enable it (currently `replicaCount: 0`). Dev's values already use ESO and
  carry no password.

No DB password remains in any config file. The only copies of the old password are in
git history, and it no longer authenticates.
