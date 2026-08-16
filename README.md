# Print-Shop-AI-Chatbot

Stack de comunicaciones para el chatbot de WhatsApp de la imprenta: Chatwoot (atención) + n8n (automatización/bot), con Postgres (pgvector) y Redis. Pensado para deployar en un VPS con Dokploy detrás de Traefik + Cloudflare.

## Servicios

| Servicio | Imagen | Rol |
|---|---|---|
| `rails` | `chatwoot/chatwoot:v4.16.2-ce` | App web de Chatwoot (puerto interno 3000) |
| `sidekiq` | `chatwoot/chatwoot:v4.16.2-ce` | Worker de background de Chatwoot |
| `postgres` | `pgvector/pgvector:pg16` | Base de datos (pgvector para el RAG del bot) |
| `redis` | `redis:7.4-alpine` | Cache y colas |
| `n8n` | `docker.n8n.io/n8nio/n8n:2.34.6` | Motor del bot (puerto interno 5678) |

Versiones fijadas a propósito: `latest` rompe en redeploy y Chatwoot corre migraciones de DB entre versiones.

## Deploy (Dokploy)

1. Crear un servicio tipo **Compose** apuntando a este repo (GitHub).
2. **Domains:** `rails` → puerto 3000, `n8n` → puerto 5678. Traefik termina el TLS.
3. **Environment:** cargar todas las variables de `.env.example` con valores reales. Dokploy escribe el `.env` que consume el compose.

El routing entra por Traefik; ningún servicio publica puertos al host. Postgres y Redis solo son alcanzables por la red interna de Docker (nombre de servicio: `postgres:5432`, `redis:6379`).

## Variables de entorno

Ver `.env.example` para la lista completa. Notas:

- Los valores reales **nunca** van al repo — se cargan en la UI de Dokploy (cifrados). `.env` está en `.gitignore`.
- `POSTGRES_PASSWORD` y `REDIS_PASSWORD` los usan tanto el contenedor de la DB como Chatwoot: mismo valor en ambos lados.
- `N8N_ENCRYPTION_KEY` cifra las credenciales de los workflows de n8n. Generala una vez y guardala aparte: sin ella no se pueden recuperar las credenciales ni migrar el n8n.
- `SECRET_KEY_BASE` (Chatwoot): 64+ caracteres.
- `N8N_HOST` / `N8N_WEBHOOK_URL` / `FRONTEND_URL`: el dominio público real, o los webhooks de Meta no validan.

## Datos

Los datos viven en volúmenes Docker, no en el repo: `postgres_data` (Chatwoot + pgvector), `n8n_data` (workflows y credenciales de n8n), `storage_data` (adjuntos de Chatwoot), `redis_data`. Respaldar por `pg_dump` + copia off-site; un volumen no es backup.
