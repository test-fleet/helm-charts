# Testing these charts locally

This walks through a real smoke test on a local cluster (examples below show
kind and k3d side by side; minikube or Docker Desktop's k8s work the same
way as kind — just skip the image-load steps). Steps 1–2 need no cluster at
all — do those first.

## 0. Prereqs

- `helm` (v3), `kubectl`, and one of `kind` / `k3d` (or another local cluster)
- `docker`

Everything below uses the local chart source (`charts/control-server`, `charts/test-runner`) — the right thing when you're iterating on the templates themselves. If you instead want to sanity-check the chart as actually published to GHCR (e.g. right after `.github/workflows/publish-charts.yml` runs, before anyone else pulls it), see [Testing the published GHCR chart](#testing-the-published-ghcr-chart) near the end — steps 1–5 don't apply there, but 6 onward are the same with one flag swapped.

## 1. Lint first — no cluster needed

Catches template syntax errors before you touch a cluster:

```bash
helm lint charts/control-server
helm lint charts/test-runner
```

## 2. Dry-run render — no cluster needed

This is the most useful check: it renders the actual manifests Helm would
apply, so you can eyeball them for anything obviously wrong. Both charts
`fail` fast if required values are missing, so you'll need placeholder
values even for a dry run.

First create your own local overrides file for each chart — these are
gitignored, so put whatever you're iterating on in them (locally-built
image repo/tag, resource tweaks, etc.); it's fine to start empty:

```bash
touch charts/control-server/values.local.yaml
touch charts/test-runner/values.local.yaml
```

```bash
helm template control-server charts/control-server \
  --set existingSecret=control-server-secrets \
  -f charts/control-server/values.local.yaml

helm template test-runner charts/test-runner \
  --set runnerName=local-runner-01 \
  --set existingSecret=runner-01-creds \
  -f charts/test-runner/values.local.yaml
```

Read through the output — check the env vars, the probe paths, the image
tag, before moving on.

## 3. Spin up a cluster and namespace

```bash
# kind
kind create cluster --name testfleet

# k3d
k3d cluster create testfleet

kubectl create namespace testfleet
```

## 4. Get Mongo + Redis running

Neither chart deploys these — for a quick smoke test, throwaway pods are
enough (don't reuse this for anything real):

```bash
kubectl -n testfleet run mongo --image=mongo:5.0 --port=27017
kubectl -n testfleet run redis --image=redis:7-alpine --port=6379
kubectl -n testfleet expose pod mongo --port=27017
kubectl -n testfleet expose pod redis --port=6379
```

**These are bare pods, not Deployments** — nothing restarts them if they get
killed (node pressure, cluster restart, manual eviction, etc.). The `mongo`
and `redis` Services stick around either way, which can make it look like
they're still fine when they're actually not — if `/ready` starts failing
in step 7, check `kubectl -n testfleet get pods` (and
`kubectl -n testfleet get endpoints mongo redis`) for missing pods before
debugging anything else, and just re-run the two `kubectl run` commands
above to bring them back.

## 5. Build and load the images (if you're not pulling from GHCR)

```bash
docker build -t test-fleet/control-server:local -f dockerfile .
docker build -t test-fleet/test-runner:local -f dockerfile .   # from the test-runner repo

# kind
kind load docker-image test-fleet/control-server:local --name testfleet
kind load docker-image test-fleet/test-runner:local --name testfleet

# k3d
k3d image import test-fleet/control-server:local -c testfleet
k3d image import test-fleet/test-runner:local -c testfleet
```

Then set `image.repository`/`image.tag` accordingly in your values (or
`--set image.repository=test-fleet/control-server --set image.tag=local`),
and `--set image.pullPolicy=Never` so kubelet uses the loaded image instead
of trying to pull it.

**If you're pulling `:edge` from GHCR instead of building locally**, note
that `pullPolicy: IfNotPresent` (the chart default) means a node that's
already cached an image for the `:edge` tag will keep using that cached
copy — pushing a new build to `:edge` and restarting the pod is **not**
enough to pick it up, since the tag name hasn't changed and kubelet only
checks "do I have something by this name," not "is it current." If your
change doesn't seem to have landed after a redeploy, evict the stale image
and force a fresh pull:

```bash
# kind
docker exec <kind-node-name> crictl rmi ghcr.io/test-fleet/control-server:edge
# k3d
docker exec k3d-testfleet-server-0 crictl rmi ghcr.io/test-fleet/control-server:edge

kubectl -n testfleet rollout restart deployment/control-server
```

Or sidestep this entirely for local iteration by setting
`--set image.pullPolicy=Always`, which re-checks the registry on every pod
start.

## 6. Create the control-server secret and install

There's no static/bootstrap admin token anymore — the control server only grants
admin access via OAuth login to a pre-invited email. That means you need a real
OAuth app (e.g. a Google OAuth client) even for local testing; there's no
"unused-for-this-test" shortcut for `OAUTH_CLIENT_ID`/`OAUTH_CLIENT_SECRET` here.

```bash
kubectl -n testfleet create secret generic control-server-secrets \
  --from-literal=MONGODB_URI='mongodb://mongo.testfleet.svc.cluster.local:27017/testfleet' \
  --from-literal=REDIS_URL='redis://redis.testfleet.svc.cluster.local:6379' \
  --from-literal=JWT_SECRET='local-test-secret-change-me' \
  --from-literal=MASTER_KEY='local-test-master-key' \
  --from-literal=OAUTH_CLIENT_ID='<your Google OAuth client id>' \
  --from-literal=OAUTH_CLIENT_SECRET='<your Google OAuth client secret>'

helm upgrade --install control-server charts/control-server -n testfleet \
  --set existingSecret=control-server-secrets \
  --set config.OAUTH_PROVIDER=google \
  --set config.OAUTH_REDIRECT_URL=http://localhost:3000/api/v1/auth/callback \
  --set config.BOOTSTRAP_ADMIN_EMAIL=<your-email> \
  -f charts/control-server/values.local.yaml
```

`BOOTSTRAP_ADMIN_EMAIL` is the only account invited as admin on first boot —
you'll log in with this exact email in step 8.

## 7. Confirm it's healthy

```bash
kubectl -n testfleet get pods
kubectl -n testfleet port-forward svc/control-server 3000:80
curl http://localhost:3000/health   # liveness — cheap, just "is the process up"
curl http://localhost:3000/ready    # readiness — checks Mongo + Redis connectivity
```

## 8. Log in and register a runner

Open `http://localhost:3000` in a browser and log in with the
`BOOTSTRAP_ADMIN_EMAIL` account from step 6 (Google OAuth). Once logged in,
go to **Runners → Register Runner**, give it a name (e.g. `local-runner-01`),
and submit. The resulting modal shows the `apiKey`/`apiSecret` pair with copy
buttons — save both immediately, the secret isn't shown again.

## 9. Create the runner's secret and install it

```bash
kubectl -n testfleet create secret generic runner-01-creds \
  --from-literal=API_KEY='<returned apiKey>' \
  --from-literal=API_SECRET='<returned apiSecret>'

helm upgrade --install runner-01 charts/test-runner -n testfleet \
  --set runnerName=local-runner-01 \
  --set existingSecret=runner-01-creds \
  -f charts/test-runner/values.local.yaml
```

## 10. Confirm the runner connected

```bash
kubectl -n testfleet logs deploy/runner-01-test-runner
```

Then check the **Runners** page in the UI — `local-runner-01` should show up
with a recent "last seen" heartbeat.

## Testing the published GHCR chart

Once `publish-charts.yml` has pushed a version, it's worth confirming that exact OCI artifact installs cleanly before anyone depends on it — this replaces steps 1–5 above; pick back up at step 6, swapping the local chart path for the OCI reference.

Pull it down and confirm the version/contents are what you expect:

```bash
helm pull oci://ghcr.io/test-fleet/charts/control-server --version 0.1.0 --untar
helm lint ./control-server
```

Then install straight from GHCR instead of the local path — same install commands as steps 6 and 9, just replace `charts/control-server` / `charts/test-runner` with the OCI reference and add `--version`:

```bash
helm upgrade --install control-server oci://ghcr.io/test-fleet/charts/control-server \
  --version 0.1.0 -n testfleet \
  --set existingSecret=control-server-secrets \
  --set config.OAUTH_PROVIDER=google \
  --set config.OAUTH_REDIRECT_URL=http://localhost:3000/api/v1/auth/callback \
  --set config.BOOTSTRAP_ADMIN_EMAIL=<your-email> \
  -f charts/control-server/values.local.yaml

helm upgrade --install runner-01 oci://ghcr.io/test-fleet/charts/test-runner \
  --version 0.1.0 -n testfleet \
  --set runnerName=local-runner-01 \
  --set existingSecret=runner-01-creds \
  -f charts/test-runner/values.local.yaml
```

If either package was pushed as private (see README's note on GHCR visibility), you'll need `helm registry login ghcr.io` with a token that has `read:packages` before either command above will pull.

## Teardown

```bash
# kind
kind delete cluster --name testfleet

# k3d
k3d cluster delete testfleet
```
