# TestFleet Helm Charts

- **`charts/control-server`** — API/scheduler/UI backend. Always 1 replica.
- **`charts/test-runner`** — one execution worker per Helm release, tied to one registered runner identity.

## Dependencies

Neither chart installs these — have them ready before you start:

- **A Kubernetes cluster**, with `kubectl`/`helm` pointed at it.
- **MongoDB**, reachable from the cluster. control-server's primary datastore.
- **Redis**, reachable from the cluster. Pub/sub channel control-server and every test-runner use to dispatch and pick up jobs — the `REDIS_CHANNEL` value must match on both sides.
- **An OAuth app**, registered with Google, GitHub, Microsoft, or Okta. There is no local email/password login — every account authenticates through this provider, so you need its client ID/secret and a callback URL before deploying control-server.

## Deploying

Both charts are published as OCI artifacts to GHCR — this is the preferred way to install, no need to clone this repo. `image.tag` defaults to the chart's `appVersion`, an immutable pinned release tag.

### 1. Create the namespace

```bash
kubectl create namespace testfleet
```

### 2. Create the control server's secret

These are the credential-shaped values — everything else is plain config, set in step 3. See the [control-server env var reference](#control-server-env-vars) below for what each one is.

```bash
kubectl -n testfleet create secret generic control-server-secrets \
  --from-literal=MONGODB_URI='mongodb://user:pass@host:27017/testfleet' \
  --from-literal=REDIS_URL='redis://:password@host:6379' \
  --from-literal=JWT_SECRET='<generate a long random string>' \
  --from-literal=MASTER_KEY='<generate a long random string>' \
  --from-literal=OAUTH_CLIENT_ID='<from your OAuth provider>' \
  --from-literal=OAUTH_CLIENT_SECRET='<from your OAuth provider>'
```

### 3. Deploy the control server

`OAUTH_PROVIDER`/`OAUTH_REDIRECT_URL` and `BOOTSTRAP_ADMIN_EMAIL` are required, not optional — without a bootstrap admin email, there's no account with permission to do anything (see the [env var reference](#control-server-env-vars) for the rest).

```bash
helm upgrade --install control-server oci://ghcr.io/test-fleet/charts/control-server \
  --version 0.1.0 -n testfleet \
  --set existingSecret=control-server-secrets \
  --set config.OAUTH_PROVIDER=google \
  --set config.OAUTH_REDIRECT_URL=https://control-server.example.com/auth/callback \
  --set config.BOOTSTRAP_ADMIN_EMAIL=you@example.com \
  -f my-control-server-values.yaml
```

### 4. Log in and register a runner

Log in via OAuth as `BOOTSTRAP_ADMIN_EMAIL`, then in the UI go to **Runners → Register Runner**, give it a name, and copy the `apiKey`/`apiSecret` it shows you — shown once, so save it immediately.

### 5. Create that runner's secret

```bash
kubectl -n testfleet create secret generic runner-01-creds \
  --from-literal=API_KEY='<returned apiKey>' \
  --from-literal=API_SECRET='<returned apiSecret>'
```

### 6. Deploy the runner

`runnerName` must match the name you registered in step 4. See the [test-runner env var reference](#test-runner-env-vars) for the rest of `config`.

```bash
helm upgrade --install runner-01 oci://ghcr.io/test-fleet/charts/test-runner \
  --version 0.1.0 -n testfleet \
  --set runnerName=prod-runner-01 \
  --set existingSecret=runner-01-creds \
  --set config.CONTROL_SERVER_URL=http://control-server.testfleet.svc.cluster.local \
  -f my-test-runner-values.yaml
```

Need another runner? Repeat steps 4–6 with a new name and release name (`runner-02`, ...) — don't scale `runner-01` instead (see [Singleton by design](#singleton-by-design)).

Working from a clone of this repo instead (e.g. testing unreleased chart changes, or an `edge` app build — see TESTING.md)? Swap the `oci://ghcr.io/test-fleet/charts/<chart>` + `--version` in any command above for the local path, e.g. `charts/control-server`.

## Env var reference

Every var below comes straight from each app's `.env.example` (`control-server/.env.example`, `test-runner/.env.example`). "Secret" means it's a credential and belongs in `existingSecret`; "config" means it's plain and goes under `--set config.KEY=...` or in your values file.

### control-server env vars

| Var | Kind | Required? | Notes |
|---|---|---|---|
| `MONGODB_URI` | secret | yes | |
| `REDIS_URL` | secret | yes | |
| `JWT_SECRET` | secret | yes | signs session JWTs |
| `MASTER_KEY` | secret | yes | |
| `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` | secret | yes | from your registered OAuth app |
| `OAUTH_PROVIDER` | config | yes | one of `google`/`github`/`microsoft`/`okta` |
| `OAUTH_REDIRECT_URL` | config | yes | must match the callback URL registered with your OAuth app |
| `BOOTSTRAP_ADMIN_EMAIL` | config | yes (first install) | the only way to get an initial admin — accounts only exist via OAuth login, there's no other path to admin |
| `OKTA_DOMAIN` | config | only if `OAUTH_PROVIDER=okta` | |
| `ALLOWED_DOMAINS` | config | no, but effectively required to invite anyone | comma-separated email domains allowed to be invited |
| `REDIS_CHANNEL` | config | no (default `testfleet:jobs`) | must match the same var on every test-runner |
| `SERVER_URL` | config | no | defaults to the in-cluster Service DNS name; set explicitly if fronted by an Ingress/LB — used as the callback base and in any outbound links |
| `ENV` / `NODE_ENV` | config | no (default `production`) | |
| `PORT` | config | no (default `3000`) | must match `service.targetPort` if changed |
| `JWT_EXPIRES_IN` | config | no (default `24h`) | |
| `LOG_LEVEL` | config | no (default `info`) | |
| `ORGANIZATION_NAME` | config | no | not currently read anywhere in server code as of this writing — safe to leave blank |

Not applicable to this chart: `MONGO_INITDB_ROOT_USERNAME`/`MONGO_INITDB_ROOT_PASSWORD`/`MONGO_INITDB_DATABASE` — those configure a self-hosted MongoDB container's own bootstrap, not the control server app, and this chart doesn't deploy MongoDB.

### test-runner env vars

| Var | Kind | Required? | Notes |
|---|---|---|---|
| `API_KEY` / `API_SECRET` | secret | yes | from registering this runner in step 4 above |
| `CONTROL_SERVER_URL` | config | yes | |
| `REDIS_URL` | config or secret | yes | plain `config.REDIS_URL` if it has no embedded credential, otherwise put it in `sharedExistingSecret` instead and leave `config.REDIS_URL` blank |
| `REDIS_CHANNEL` | config | no (default `testfleet:jobs`) | must match control-server's |
| `RUNNER_NAME` | — | yes | not part of the step 5 secret — set via the top-level `runnerName` value on the `helm install` in step 6 |
| `MAX_WORKERS` | config | no (default `3`) | worker pool size — raise if tests queue up faster than they run |
| `HEARTBEAT_INTERVAL` | config | no (default `15`) | seconds between heartbeats to the control server |

## Using these charts as a dependency

Reference them from another chart's `Chart.yaml`:

```yaml
dependencies:
  - name: control-server
    version: "0.1.0"
    repository: "oci://ghcr.io/test-fleet/charts"
  - name: test-runner
    version: "0.1.0"
    repository: "oci://ghcr.io/test-fleet/charts"
```

Then `helm dependency update` as usual. Or pull one standalone:

```bash
helm pull oci://ghcr.io/test-fleet/charts/control-server --version 0.1.0
```

The publish workflow (`.github/workflows/publish-charts.yml`) skips a chart if that exact version is already in GHCR (OCI tags here are meant to be immutable, same as the app images) — bump `version` in `Chart.yaml` to publish a new one. This `version` is the chart's own packaging version, independent of `appVersion`/the app's release tag.

**First-time setup:** GHCR publishes OCI Helm charts as their own package, which doesn't always inherit the repo's public visibility automatically. After the first push, check `ghcr.io/test-fleet` in GitHub's Packages UI and flip `control-server`/`test-runner` (the chart packages, not the image ones) to public if they land as private — otherwise consumers outside the org will get pull-access errors.

## Singleton by design

- **control-server**: `replicas: 1` is hardcoded in the template, not a value — it runs a cron scheduler and a startup bootstrap routine that aren't safe to run twice.
- **test-runner**: `replicaCount` is exposed (default `1`) but scaling it means multiple pods sharing one API key/secret, which the control server UI flags as a corrupting anti-pattern. More capacity = another registered runner + another release, not a bigger number here.
- **Only one control-server release, period**: Helm won't stop a second `helm install control-server-2 ...` from coexisting — nothing about the chart or namespace prevents it. For v1 this is enforced by process, not tooling: keep control-server deploys behind one CI pipeline with a fixed release name, and optionally have it check `kubectl get deployments -l app.kubernetes.io/name=control-server` before installing. A pre-install Helm hook that enforces this automatically is a reasonable future addition, just not built for v1.

## Secrets

Both charts take an `existingSecret` name rather than any real secret values — Helm just wires env vars to it via `secretKeyRef`/`envFrom`. Nothing sensitive ever lands in a values.yaml, `helm get values`, or `helm history`; rotating a credential is a `kubectl` operation, not a release.

**From GitHub Actions:** have the workflow materialize the Secret right before deploying, then let Helm reference it by name — two separate steps.

`--dry-run=client -o yaml | kubectl apply -f -` makes this step idempotent, safe to re-run on every deploy:

```yaml
- name: Sync control-server secret
  run: |
    kubectl -n testfleet create secret generic control-server-secrets \
      --from-literal=MONGODB_URI='${{ secrets.MONGODB_URI }}' \
      --from-literal=REDIS_URL='${{ secrets.REDIS_URL }}' \
      --from-literal=JWT_SECRET='${{ secrets.JWT_SECRET }}' \
      --from-literal=MASTER_KEY='${{ secrets.MASTER_KEY }}' \
      --from-literal=OAUTH_CLIENT_ID='${{ secrets.OAUTH_CLIENT_ID }}' \
      --from-literal=OAUTH_CLIENT_SECRET='${{ secrets.OAUTH_CLIENT_SECRET }}' \
      --dry-run=client -o yaml | kubectl apply -f -
```

Then deploy, referencing that Secret by name:

```yaml
- name: Deploy
  run: |
    helm upgrade --install control-server oci://ghcr.io/test-fleet/charts/control-server \
      --version 0.1.0 -n testfleet \
      --set existingSecret=control-server-secrets --set image.tag=${{ github.sha }} \
      -f values.prod.yaml
```

Runner credentials are one-time-generated by the API, so they don't fit this flow the same way: register runners and create their Secrets once (manually or via a small setup script), and let CI just redeploy the Deployment that references it.

If secret sprawl or rotation ever becomes a real burden, look at **External Secrets Operator** (syncs from AWS/GCP/Azure/Vault into the same `existingSecret` shape) or **Sealed Secrets** (encrypted secrets safe to commit to git). Neither is necessary for v1.
