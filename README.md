# flixflox-manifests

Raw Kubernetes manifests for the FlixFlox stack.

## Layout

```
flixflox-api/            Deployment, Service, Ingress, PVC
flixflox-ui/             Deployment, Service, Ingress, ConfigMap
flixflox-telegram-bot/   Telegram bot + local bot API server
mongodb/                 Single-node StatefulSet + headless Service
```

Files are named `<kind>-<name>.yaml`.

## Namespace

No manifest hardcodes a namespace — pick one when you apply:

```sh
export NS=apps
kubectl create namespace "$NS"
kubectl apply -n "$NS" -f mongodb
kubectl apply -n "$NS" -f flixflox-api
kubectl apply -n "$NS" -f flixflox-telegram-bot
kubectl apply -n "$NS" -f flixflox-ui
```

All services use short in-namespace DNS names (`mongodb`, `flixflox`,
`flixflox-telegram-api`), so they keep working in any namespace.

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
kubectl -n "$NS" create secret generic flixflox-secret \
  --from-literal=JWT_SECRET_KEY="$(openssl rand -base64 48)" \
  --from-literal=MONGO_CONNECTION_STRING='mongodb://<user>:<pass>@mongodb:27017/flixflox?authSource=admin'

# Telegram bot API server
kubectl -n "$NS" create secret generic flixflox-telegram-api \
  --from-literal=TELEGRAM_API_ID='<id>' \
  --from-literal=TELEGRAM_API_HASH='<hash>' \
  --from-literal=TELEGRAM_LOCAL='true'

# Telegram bot
kubectl -n "$NS" create secret generic flixflox-telegram-bot \
  --from-literal=FLIXFLOX_BOT_TOKEN='<token>' \
  --from-literal=FLIXFLOX_ALLOWED_USER_IDS='<user_id>' \
  --from-literal=FLIXFLOX_DOWNLOAD_DIR='/app/videos' \
  --from-literal=FLIXFLOX_TELEGRAM_DATA_DIR='/var/lib/telegram-bot-api' \
  --from-literal=FLIXFLOX_TELEGRAM_SERVER_WORK_DIR='/var/lib/telegram-bot-api' \
  --from-literal=FLIXFLOX_TELEGRAM_API_ENDPOINT='http://flixflox-telegram-api/bot%s/%s' \
  --from-literal=FLIXFLOX_TELEGRAM_FILE_ENDPOINT='http://flixflox-telegram-api/file/bot%s/%s' \
  --from-literal=FLIXFLOX_API_URL='http://flixflox'
```

Alternatively copy a template and apply it:

```sh
cp flixflox-api/secret-flixflox.example.yaml flixflox-api/secret-flixflox.yaml
$EDITOR flixflox-api/secret-flixflox.yaml
kubectl apply -n "$NS" -f flixflox-api/secret-flixflox.yaml
```

The `MONGO_CONNECTION_STRING` username/password **must match**
`mongodb-secret`, since both are read by different workloads.

## Apply order

1. `mongodb` — API needs it at startup.
2. `flixflox-api`
3. `flixflox-telegram-bot` (needs both the API and the bot API server)
4. `flixflox-ui`

## Runs on

- Storage class `local-path` for `videos-pvc` and `telegram-data`.
- An ingress controller (NGINX) for `flixflox.lan` and `api.flixflox.lan`.
