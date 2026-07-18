# Testing these charts locally

This walks through a real smoke test on a local cluster (kind, but minikube
or Docker Desktop's k8s work the same way — just skip the `kind load`
steps). Steps 1–2 need no cluster at all — do those first.

## 0. Prereqs

- `helm` (v3), `kubectl`, and `kind` (or another local cluster)
- `docker` if you're using kind

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
values even for a dry run:

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
kind create cluster --name testfleet
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

## 5. Build and load the images (if you're not pulling from GHCR)

```bash
docker build -t test-fleet/control-server:local -f dockerfile .
kind load docker-image test-fleet/control-server:local --name testfleet

docker build -t test-fleet/test-runner:local -f dockerfile .   # from the test-runner repo
kind load docker-image test-fleet/test-runner:local --name testfleet
```

Then set `image.repository`/`image.tag` accordingly in your values (or
`--set image.repository=test-fleet/control-server --set image.tag=local`),
and `--set image.pullPolicy=Never` so kubelet uses the loaded image instead
of trying to pull it.

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

## 8. Grab the admin token and register a runner

Open `http://localhost:3000` in a browser and log in with the
`BOOTSTRAP_ADMIN_EMAIL` account from step 6 (Google OAuth). Once logged in, pull
your JWT out of local storage — open devtools console and run:

```js
localStorage.getItem('token')
```

```bash
export ADMIN_TOKEN="<paste the token>"

curl -X POST http://localhost:3000/api/v1/runners/register \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"name": "local-runner-01"}'
# => save the returned apiKey/apiSecret
```

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

curl http://localhost:3000/api/v1/runners \
  -H "Authorization: Bearer $ADMIN_TOKEN"
# look for local-runner-01 with a recent lastSeen
```

## Teardown

```bash
kind delete cluster --name testfleet
```
