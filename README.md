# floxflix-manifests

Raw Kubernetes manifests for the FloxFlix stack.

## Layout

```
floxflix-api/            Deployment, Service, Ingress, PVC
floxflix-ui/             Deployment, Service, Ingress, ConfigMap
floxflix-telegram-bot/   Telegram bot + local bot API server
mongodb/                 Single-node StatefulSet + headless Service
```

Files are named `<kind>-<name>.yaml`.

## Namespace

No manifest hardcodes a namespace — pick one when you apply:

```sh
export NS=apps
kubectl create namespace "$NS"
kubectl apply -n "$NS" -f mongodb
kubectl apply -n "$NS" -f floxflix-api
kubectl apply -n "$NS" -f floxflix-telegram-bot
kubectl apply -n "$NS" -f floxflix-ui
```

All services use short in-namespace DNS names (`mongodb`, `floxflix`,
`floxflix-telegram-api`), so they keep working in any namespace.

## Secrets

Do **not** commit real secrets. Templates live in `*.example.yaml`, and
`secret-*.yaml` is git-ignored. Create the secrets with `kubectl`:

```sh
export NS=apps

# MongoDB credentials (also consumed by the StatefulSet)
kubectl -n "$NS" create secret generic mongodb-secret \
  --from-literal=mongo-root-username='<user>' \
  --from-literal=mongo-root-password='<pass>'

# API — generate a fresh JWT key, do not reuse the old one
kubectl -n "$NS" create secret generic floxflix-secret \
  --from-literal=JWT_SECRET_KEY="$(openssl rand -base64 48)" \
  --from-literal=MONGO_CONNECTION_STRING='mongodb://<user>:<pass>@mongodb:27017/floxflix?authSource=admin'

# Telegram bot API server
kubectl -n "$NS" create secret generic floxflix-telegram-api \
  --from-literal=TELEGRAM_API_ID='<id>' \
  --from-literal=TELEGRAM_API_HASH='<hash>' \
  --from-literal=TELEGRAM_LOCAL='true'

# Telegram bot
kubectl -n "$NS" create secret generic floxflix-telegram-bot \
  --from-literal=FLOXFLIX_BOT_TOKEN='<token>' \
  --from-literal=FLOXFLIX_ALLOWED_USER_IDS='<user_id>' \
  --from-literal=FLOXFLIX_DOWNLOAD_DIR='/app/videos' \
  --from-literal=FLOXFLIX_TELEGRAM_DATA_DIR='/var/lib/telegram-bot-api' \
  --from-literal=FLOXFLIX_TELEGRAM_SERVER_WORK_DIR='/var/lib/telegram-bot-api' \
  --from-literal=FLOXFLIX_TELEGRAM_API_ENDPOINT='http://floxflix-telegram-api/bot%s/%s' \
  --from-literal=FLOXFLIX_TELEGRAM_FILE_ENDPOINT='http://floxflix-telegram-api/file/bot%s/%s' \
  --from-literal=FLOXFLIX_API_URL='http://floxflix'
```

Alternatively copy a template and apply it:

```sh
cp floxflix-api/secret-floxflix.example.yaml floxflix-api/secret-floxflix.yaml
$EDITOR floxflix-api/secret-floxflix.yaml
kubectl apply -n "$NS" -f floxflix-api/secret-floxflix.yaml
```

The `MONGO_CONNECTION_STRING` username/password **must match**
`mongodb-secret`, since both are read by different workloads.

## Apply order

1. `mongodb` — API needs it at startup.
2. `floxflix-api`
3. `floxflix-telegram-bot` (needs both the API and the bot API server)
4. `floxflix-ui`

## Runs on

- Storage class `local-path` for `videos-pvc` and `telegram-data`.
- An ingress controller (NGINX) for `floxflix.lan` and `api.floxflix.lan`.
