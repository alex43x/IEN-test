# IEN — Repositorio de referencia (dev local + ops)

Plataforma IEN (Inteligencia Emocional).

Este repositorio es un **umbrella** que fusiona los dos proyectos mediante submódulos git:

| Submódulo | Repo | Tecnología |
|-----------|------|-----------|
| `back` | [ien-back](https://github.com/alex43x/ien-back) | API Node.js/Express + MongoDB |
| `front` | [ien-front](https://github.com/alex43x/ien-front) | React + Vite + Tailwind |

> ⚠️ **Este repo NO es fuente de despliegue.** Northflank no inicializa submódulos git, así que
> un build context `/back` o `/front` sobre este repo queda vacío. El despliegue se hace desde los
> repos independientes `ien-back` y `ien-front`. Ver **[DEPLOY-NORTHFLANK.md](./DEPLOY-NORTHFLANK.md)**.

## Para qué sirve este repo

- **Desarrollo local** del stack completo con `docker-compose.yml` (mongo + back + front + seeder).
- **Docs centralizadas**: `DEPLOY-NORTHFLANK.md`, `PLAN_AUDITORIA.md`, `NOTAS-FRONTEND.md`, `.env.example`.
- **Apuntadores**: cada submódulo apunta a un commit concreto (rama `main`).

## Clonar con submódulos

```bash
git clone --recurse-submodules https://github.com/alex43x/IEN-test.git
```

O si ya lo clonaste sin submódulos:

```bash
git submodule update --init --recursive
```

## Testing local

```bash
cp .env.example .env
# Completar valores reales en .env
docker compose up --build
```

- Frontend: `http://localhost:80`
- Backend: `http://localhost:3000/api`
- MongoDB: `mongodb://localhost:27017/ien`
- Swagger: `http://localhost:3000/api-docs`

El seeder se ejecuta una vez al arrancar el stack (servicio `seeder`). Credenciales por defecto:
`admin@ien.test` / `admin123`.

## Estructura

```
IEN-test/
├── back/                  # Submódulo → ien-back (API)
├── front/                 # Submódulo → ien-front (SPA)
├── docker-compose.yml     # Stack local completo
├── DEPLOY-NORTHFLANK.md   # Guía de despliegue (repos separados)
├── PLAN_AUDITORIA.md      # Notas de auditoría
├── NOTAS-FRONTEND.md      # Notas del frontend
└── .env.example           # Variables de entorno de referencia
```