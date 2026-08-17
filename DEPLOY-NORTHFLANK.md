# Despliegue en Northflank — Repos separados

> **IMPORTANTE**: Este proyecto despliega desde los repos **independientes** `ien-back` y `ien-front`.
> El repo `IEN-test` (este) **NO es fuente de despliegue**: Northflank no inicializa submódulos git,
> por lo que un build context `/back` o `/front` sobre este repo quedaría vacío.
> Usa `IEN-test` solo para desarrollo local (docker-compose), docs y apuntadores.

## Arquitectura

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Frontend (Static)│────▶│  Backend (Docker) │────▶│  Northflank      │
│  ien-front        │     │  ien-back         │     │  Addon (MongoDB) │
│  Northflank       │     │  Northflank       │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

| Servicio   | Repo           | Tipo            | Puerto / Publish |
|------------|----------------|-----------------|------------------|
| Frontend   | `ien-front`    | Docker (nginx)  | `80`             |
| Backend    | `ien-back`     | Docker          | `3000`           |
| MongoDB    | —              | Addon           | inyecta `MONGO_URI` |
| Cron jobs  | —              | 2 jobs curl     | —                |

## Paso 1 — Base de datos (Addon MongoDB)

1. Proyecto Northflank → **Add-ons** → **Add Add-on** → **MongoDB**.
2. Una vez creado, vincula el addon al servicio backend para que inyecte `MONGO_URI` automáticamente.

## Paso 2 — Backend (servicio Docker)

Crear servicio → **Deploy from GitHub** → tipo **Docker**:

| Campo | Valor |
|-------|-------|
| Repository | `ien-back` |
| Branch | `main` |
| Build type | `Dockerfile` |
| Build context | `/` (raíz) |
| Dockerfile location | `/Dockerfile` |
| Port | `3000` (público) |
| Health check path | `/health` |

Variables de entorno:

| Variable | Valor |
|----------|-------|
| `NODE_ENV` | `production` |
| `PORT` | `3000` |
| `MONGO_URI` | inyectada por el addon (no configurar manualmente) |
| `JWT_SECRET` | `openssl rand -hex 32` |
| `CRON_API_KEY` | `openssl rand -hex 32` (misma en los cron jobs) |
| `RESEND_API_KEY` | API key de resend.com |
| `EMAIL_FROM` | email de envío |
| `FRONTEND_URL` | URL exacta del frontend (sin slash final). Se setea **después** de desplegar el frontend (CORS usa `origin: FRONTEND_URL`). |

## Paso 3 — Frontend (servicio Docker con Nginx)

> **Nota**: Northflank **no tiene** un producto de "Static Site" (ni en la UI, ni en sus docs, ni en la
> guía oficial de Vite). La vía correcta para servir el SPA es el `Dockerfile` + `nginx.conf` ya
> incluidos en `ien-front`, que además resuelven el SPA fallback nativamente.

Crear servicio → **Deploy from GitHub** → tipo **Docker**:

| Campo | Valor |
|-------|-------|
| Repository | `ien-front` |
| Branch | `main` |
| Build type | `Dockerfile` |
| Build context | `/` (raíz) |
| Dockerfile location | `/Dockerfile` |
| Port | `80` (público) |

> **Orden**: primero pushear el fix del `nginx.conf` (commit `ade2785`) y **después** crear el servicio,
> para que el build use la plantilla corregida. Sin él, el proxy caía al default `backend:3000` y nginx
> crasheaba con `host not found in upstream "backend"`.

Variables de entorno (runtime, las lee nginx en el entrypoint):

| Variable | Valor |
|----------|-------|
| `BACKEND_SCHEME` | `https` |
| `BACKEND_HOST` | hostname del backend sin esquema: `<BACKEND>.northflank.app` (ej. `p01--ien-back--...code.run`) |
| `BACKEND_PORT` | `443` |

> `VITE_API_URL` no hace falta: el `Dockerfile` usa `/api` por defecto y nginx proxea `/api/*` al backend.
> Con este proxy, el navegador solo habla con el dominio del frontend (mismo origen → sin CORS).

## Paso 4 — CORS (`FRONTEND_URL`)

1. Volver al backend → variables de entorno.
2. `FRONTEND_URL` = `https://<FRONTEND>.northflank.app` (sin slash final).
3. Redeploy/restart del backend.

## Paso 5 — Seeder (una sola vez)

1. Backend → pestaña **Terminal**.
2. Ejecutar: `node src/seed.js`
3. Verificar que termine con los logs de creación. Credenciales por defecto:
   - `admin@ien.test` / `admin123` (admin general)
   - `admin_negocio@ien.test` / `admin123`
   - `moderador@ien.test` / `admin123`
4. No volver a ejecutarlo (borra y reseedea).

## Paso 6 — Cron Jobs (2)

En el proyecto → **Cron Jobs** → source **External image** → `docker.io/curlimages/curl:latest`.

| Job | Horario (UTC) | Endpoint |
|-----|:-------------:|----------|
| `ien-reminders` | `*/30 * * * *` | `POST /api/jobs/send-reminders` |
| `ien-daily` | `0 3 * * *` | `POST /api/jobs/run-daily` |

Variables (ambos): `CRON_API_KEY` (la del backend) y `BACKEND_URL` (`https://<BACKEND>.northflank.app`).

Command override:

```sh
sh -c 'curl -sS -X POST -H "X-API-KEY: $CRON_API_KEY" $BACKEND_URL/api/jobs/send-reminders'
sh -c 'curl -sS -X POST -H "X-API-KEY: $CRON_API_KEY" $BACKEND_URL/api/jobs/run-daily'
```

Los endpoints validan el header `X-API-KEY` contra `CRON_API_KEY`. Otros endpoints disponibles:
`reset-streaks`, `send-activation-nudge`, `send-recovery`.

## Paso 7 — Checklist final

- [ ] `https://<BACKEND>.northflank.app/health` → `{"status":"ok"}`
- [ ] `https://<BACKEND>.northflank.app/api-docs` → Swagger
- [ ] Login `admin@ien.test` / `admin123` en el frontend → panel admin
- [ ] Refrescar `/login` y otras rutas → sin 404 (SPA fallback)
- [ ] Run manual de ambos cron jobs → 200
- [ ] Logs del backend limpios (Mongo conectado, sin crash loops)