# TestFleet Helm Charts

TestFleet is a distributed API testing and monitoring tool.

- **Scenes** are cron-scheduled sequences of HTTP requests, with assertions and variables passed between them.
- Each scene broadcasts over Redis to every registered, headless [runner](https://github.com/test-fleet/test-runner).
- Results are aggregated across runners against a configurable pass threshold.
- The [control-server](https://github.com/test-fleet/control-server) hosts the API, scheduler, and web UI.

This repo holds the two Helm charts that deploy it:

- **`charts/control-server`**: API/scheduler/UI backend. Always 1 replica.
- **`charts/test-runner`**: one execution worker per Helm release, tied to one registered runner identity.

## Dependencies

Neither chart installs these. Have them ready before you start:

- **A Kubernetes cluster**, with `kubectl`/`helm` pointed at it.
- **MongoDB**, reachable from the cluster. control-server's primary datastore.
- **Redis**, reachable from the cluster. control-server publishes scene jobs on a pub/sub channel and every test-runner subscribes to it; the channel name itself is hardcoded (`testfleet:jobs`) on both sides, not something you configure.
- **An OAuth app**, registered with Google, GitHub, Microsoft, or Okta. There is no local email/password login. Every account authenticates through this provider, so you need its client ID/secret and a callback URL before deploying control-server.

## Deploying

Both charts are published as OCI artifacts to GHCR. This is the preferred way to install, no need to clone this repo. `image.tag` defaults to the chart's `appVersion`, an immutable pinned release tag.

### 1. Create the namespace

```bash
kubectl create namespace testfleet
```

### 2. Create the control server's secret

These are the credential-shaped values. Everything else is plain config, set in step 3. See the [control-server env var reference](#control-server-env-vars) below for what each one is.

`MASTER_KEY` has a hard format requirement, not just "make it long": it must decode to exactly 32 bytes as hex (64 hex characters), or the app refuses to boot. Generate it with `openssl rand -hex 32`. `JWT_SECRET` has no such constraint, but a random hex key beats a typed-in passphrase; `openssl rand -hex 16` works well.

```bash
kubectl -n testfleet create secret generic control-server-secrets \
  --from-literal=MONGODB_URI='mongodb://user:pass@host:27017/testfleet' \
  --from-literal=REDIS_URL='redis://:password@host:6379' \
  --from-literal=JWT_SECRET='<32 hex char random key>' \
  --from-literal=MASTER_KEY='<AES-256 key as 64 hex chars>' \
  --from-literal=OAUTH_CLIENT_ID='<from your OAuth provider>' \
  --from-literal=OAUTH_CLIENT_SECRET='<from your OAuth provider>'
```

### 3. Deploy the control server

Every required `config` var goes on the command as a `--set` flag, same idea as the secret above (and just as easy to source from CI secrets/variables in a pipeline). See the [env var reference](#control-server-env-vars) for the full list, including the optional ones with defaults.

```bash
helm upgrade --install control-server oci://ghcr.io/test-fleet/charts/control-server \
  --version 0.1.4 -n testfleet \
  --set existingSecret=control-server-secrets \
  --set config.OAUTH_PROVIDER=google \
  --set config.OAUTH_REDIRECT_URL=https://control-server.example.com/auth/callback \
  --set config.BOOTSTRAP_ADMIN_EMAIL=you@example.com \
  --set config.ALLOWED_DOMAINS=example.com
```

### 4. Log in and register a runner

Log in via OAuth as `BOOTSTRAP_ADMIN_EMAIL`, then in the UI go to **Runners → Register Runner**, give it a name, and copy the `apiKey`/`apiSecret` it shows you. It's shown once, so save it immediately.

### 5. Create that runner's secret

```bash
kubectl -n testfleet create secret generic runner-01-creds \
  --from-literal=API_KEY='<returned apiKey>' \
  --from-literal=API_SECRET='<returned apiSecret>'
```

### 6. Deploy the runner

`runnerName` must match the name you registered in step 4. `CONTROL_SERVER_URL` and a Redis URL (either `config.REDIS_URL` or `sharedExistingSecret`) are hard-required: the runner binary has no fallback for either and will crash-loop without them (the chart refuses to install without at least one Redis option set). See the [test-runner env var reference](#test-runner-env-vars) for the rest.

```bash
helm upgrade --install runner-01 oci://ghcr.io/test-fleet/charts/test-runner \
  --version 0.1.3 -n testfleet \
  --set runnerName=prod-runner-01 \
  --set existingSecret=runner-01-creds \
  --set config.CONTROL_SERVER_URL=http://control-server.testfleet.svc.cluster.local \
  --set config.REDIS_URL=redis://:password@host:6379
```

Need another runner? Repeat steps 4 through 6 with a new name and release name (`runner-02`, ...); don't scale `runner-01` instead (see [Singleton by design](#singleton-by-design)).

Working from a clone of this repo instead (e.g. testing unreleased chart changes, or an `edge` app build, see TESTING.md)? Swap the `oci://ghcr.io/test-fleet/charts/<chart>` + `--version` in any command above for the local path, e.g. `charts/control-server`.

## Env var reference

Every row below was checked against what each app's source actually reads (`process.env.*` in control-server, `config.go` in test-runner), not just the `.env.example` files, which include a few dev-only/unused vars that don't apply to a Helm deployment at all. "Secret" means it's a credential and belongs in `existingSecret`; "config" means it's plain and goes under `config.KEY` in your values file.

### control-server env vars

| Var | Kind | Required? | Notes |
|---|---|---|---|
| `MONGODB_URI` | secret | yes | |
| `REDIS_URL` | secret | yes | |
| `JWT_SECRET` | secret | yes | signs session JWTs |
| `MASTER_KEY` | secret | yes | AES-256-GCM key encrypting every runner's API_SECRET at rest. Must be exactly 64 hex characters (32 bytes); the app refuses to boot otherwise. `openssl rand -hex 32` |
| `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` | secret | yes | from your registered OAuth app |
| `OAUTH_PROVIDER` | config | yes | one of `google`/`github`/`microsoft`/`okta`; there is no local-auth fallback |
| `OAUTH_REDIRECT_URL` | config | yes | must match the callback URL registered with your OAuth app |
| `BOOTSTRAP_ADMIN_EMAIL` | config | yes (first install) | the only way to get an initial admin: accounts only exist via OAuth login, there's no other path to admin |
| `ALLOWED_DOMAINS` | config | required in practice | comma-separated email domains allowed to be invited. `inviteUser()` reads this with no fallback/try-catch, so an unset value throws an uncaught error (not a clean 400) the first time anyone tries to invite a user |
| `OKTA_DOMAIN` | config | only if `OAUTH_PROVIDER=okta` | |
| `ENV` / `NODE_ENV` | config | no (default `production`) | |
| `PORT` | config | no (default `3000`) | must match `service.targetPort` if changed |
| `JWT_EXPIRES_IN` | config | no (default `24h`) | |
| `HEARTBEAT_INTERVAL` | config | no | informational only. Returned by `GET /api/v1/config` for the frontend to display; app defaults to `30000` (ms) if unset. Unrelated to the test-runner chart's own `HEARTBEAT_INTERVAL` |

Not wired into this chart at all, and not needed for a Helm deployment: `API_KEY_A`/`API_SECRET_A`/`API_KEY_B`/`API_SECRET_B` (a dev-only bootstrap that only runs when `ENV=dev`, which the chart never sets), and `MONGO_INITDB_ROOT_USERNAME`/`MONGO_INITDB_ROOT_PASSWORD`/`MONGO_INITDB_DATABASE` (those configure a self-hosted MongoDB container's own bootstrap, not the control server app; this chart doesn't deploy MongoDB). `SERVER_URL`, `LOG_LEVEL`, and `ORGANIZATION_NAME` used to be chart values but were removed in chart `0.1.1`; none of the three were ever read anywhere in server code. `REDIS_CHANNEL` used to be a chart value too but was removed in chart `0.1.2`: both apps now hardcode the same pub/sub channel name (`testfleet:jobs`) rather than reading it from the environment, since it's a shared protocol constant, not a pointer to a distinct resource. `FRONTEND_URL` was removed in chart `0.1.3` along with the app code that read it: it only ever supported redirecting to a frontend hosted on a separate origin from this API, which isn't a real deployment mode without CORS support that was never built, and isn't a goal of this project anyway (single control-server deployment, embedded frontend, one origin, always).

### test-runner env vars

| Var | Kind | Required? | Notes |
|---|---|---|---|
| `API_KEY` / `API_SECRET` | secret | yes | from registering this runner in step 4 above |
| `CONTROL_SERVER_URL` | config | yes | no fallback in the runner binary; chart refuses to install without it (as of `0.1.1`) |
| `REDIS_URL` | config or secret | yes | no fallback in the runner binary; chart refuses to install unless this or `sharedExistingSecret` is set (as of `0.1.1`). Plain `config.REDIS_URL` if it has no embedded credential, otherwise put it in `sharedExistingSecret` instead and leave `config.REDIS_URL` blank |
| `RUNNER_NAME` | n/a | yes (chart-enforced) | not part of the step 5 secret. Set via the top-level `runnerName` value on the `helm install` in step 6. The Go binary itself would fall back to `"unnamed-runner"` if this were blank, but the chart's `fail` guard doesn't allow that; you always want a real distinguishing name |
| `MAX_WORKERS` | config | no (default `3`) | worker pool size. Raise if tests queue up faster than they run |
| `HEARTBEAT_INTERVAL` | config | no (default `15`) | seconds between heartbeats to the control server |

`REDIS_CHANNEL` used to be a chart value here too but was removed in chart `0.1.2`: the runner binary now hardcodes the same pub/sub channel name control-server does (`testfleet:jobs`), since both sides just need to agree on a string, not point at a distinct resource.

## Using these charts as a dependency

Reference them from another chart's `Chart.yaml`:

```yaml
dependencies:
  - name: control-server
    version: "0.1.4"
    repository: "oci://ghcr.io/test-fleet/charts"
  - name: test-runner
    version: "0.1.3"
    repository: "oci://ghcr.io/test-fleet/charts"
```

Then `helm dependency update` as usual. Or pull one standalone:

```bash
helm pull oci://ghcr.io/test-fleet/charts/control-server --version 0.1.4
```

The publish workflow (`.github/workflows/publish-charts.yml`) skips a chart if that exact version is already in GHCR (OCI tags here are meant to be immutable, same as the app images); bump `version` in `Chart.yaml` to publish a new one. This `version` is the chart's own packaging version, independent of `appVersion`/the app's release tag.

**First-time setup:** GHCR publishes OCI Helm charts as their own package, which doesn't always inherit the repo's public visibility automatically. After the first push, check `ghcr.io/test-fleet` in GitHub's Packages UI and flip `control-server`/`test-runner` (the chart packages, not the image ones) to public if they land as private, otherwise consumers outside the org will get pull-access errors.

## Singleton by design

- **control-server**: `replicas: 1` is hardcoded in the template, not a value. It runs a cron scheduler and a startup bootstrap routine that aren't safe to run twice.
- **test-runner**: `replicaCount` is exposed (default `1`) but scaling it means multiple pods sharing one API key/secret, which the control server UI flags as a corrupting anti-pattern. More capacity = another registered runner + another release, not a bigger number here.
- **Only one control-server release, period**: Helm won't stop a second `helm install control-server-2 ...` from coexisting; nothing about the chart or namespace prevents it. For v1 this is enforced by process, not tooling: keep control-server deploys behind one CI pipeline with a fixed release name, and optionally have it check `kubectl get deployments -l app.kubernetes.io/name=control-server` before installing. A pre-install Helm hook that enforces this automatically is a reasonable future addition, just not built for v1.

## Secrets

Both charts take an `existingSecret` name rather than any real secret values. Helm just wires env vars to it via `secretKeyRef`/`envFrom`. Nothing sensitive ever lands in a values.yaml, `helm get values`, or `helm history`; rotating a credential is a `kubectl` operation, not a release.

**From GitHub Actions:** have the workflow materialize the Secret right before deploying, then let Helm reference it by name. Two separate steps.

`--dry-run=client -o yaml | kubectl apply -f -` makes this step idempotent, safe to re-run on every deploy. When you create the `MASTER_KEY` GitHub Secret itself, generate it with `openssl rand -hex 32`; anything that isn't exactly 64 hex characters makes the app refuse to boot:

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
      --version 0.1.4 -n testfleet \
      --set existingSecret=control-server-secrets --set image.tag=${{ github.sha }} \
      -f values.prod.yaml
```

Runner credentials are one-time-generated by the API, so they don't fit this flow the same way: register runners and create their Secrets once (manually or via a small setup script), and let CI just redeploy the Deployment that references it.

If secret sprawl or rotation ever becomes a real burden, look at **External Secrets Operator** (syncs from AWS/GCP/Azure/Vault into the same `existingSecret` shape) or **Sealed Secrets** (encrypted secrets safe to commit to git). Neither is necessary for v1.
