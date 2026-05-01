# Analytics Stack Setup

## Scope
- `analytics` namespace
- `Redash` web / worker / scheduler
- `Redis`
- `dbt` execution is handled by Airflow, not by this repo directly

## Required Kubernetes Secret
Create `redash-app-secret` in namespace `analytics`.

Required keys:
- `REDASH_DATABASE_URL`
- `REDASH_COOKIE_SECRET`
- `REDASH_SECRET_KEY`

Example:

```bash
kubectl -n analytics create secret generic redash-app-secret \
  --from-literal=REDASH_DATABASE_URL='postgresql://USER:PASSWORD@HOST:5432/redash' \
  --from-literal=REDASH_COOKIE_SECRET='replace-with-random-cookie-secret' \
  --from-literal=REDASH_SECRET_KEY='replace-with-random-secret-key'
```

## Redash Access
- Internal hostname: `redash.int.selfronny.com`

## Metadata DB
Redash metadata DB should be separate from application tables if possible.

Recommended:
- DB name: `redash`
- Separate DB user for Redash metadata

## Analytics Data Source
Inside Redash, add the SolSQLD PostgreSQL database as a data source with a read-only account.

Recommended access:
- Read-only user
- Limit to analytics-facing schemas such as `raw`, `stg`, `mart`

## dbt Execution Direction
`dbt` project code should live in the private Airflow repo.

Recommended flow:
1. Application data accumulates in PostgreSQL
2. Airflow runs `dbt run` and `dbt test`
3. `dbt` writes analytics models into `mart`
4. Redash reads `mart`

## Recommended Schema Split
- `raw`
- `stg`
- `mart`

## Notes
- `dbt` is not deployed here as an always-on service
- `dbt` should run as Airflow-managed jobs or pods
- If needed later, add RBAC for Airflow to create pods in `analytics`
