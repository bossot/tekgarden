# Plantilles Docker Compose

Exemples de desplegaments Docker Compose anonimitzats per a entorns d'homelab i producció lleugera.

## Exemple: Servei web + Base de dades + Redis

Plantilla completa per un servei web amb PostgreSQL i Redis, amb xarxes separades, volums persistents i variables d'entorn.

```yaml
# docker-compose.yml
# Servei web amb PostgreSQL i Redis
# Ús: copia com a docker-compose.yml, crea un .env i executa `docker compose up -d`
---
version: "3.9"

networks:
  frontend:
    name: [STACK_NAME]_frontend
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
  backend:
    name: [STACK_NAME]_backend
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.0.1.0/24

volumes:
  postgres_data:
    name: [STACK_NAME]_postgres_data
    driver: local
  redis_data:
    name: [STACK_NAME]_redis_data
    driver: local

services:
  # --- Servei web principal ---
  web:
    image: [IMAGE_NAME]:[IMAGE_TAG]
    container_name: [STACK_NAME]_web
    restart: unless-stopped
    networks:
      - frontend
      - backend
    ports:
      - "[HOST_PORT]:[CONTAINER_PORT]"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=${POSTGRES_DB:?Cal definir POSTGRES_DB}
      - DB_USER=${POSTGRES_USER:?Cal definir POSTGRES_USER}
      - DB_PASSWORD=${POSTGRES_PASSWORD:?Cal definir POSTGRES_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SECRET_KEY=${SECRET_KEY:?Cal definir SECRET_KEY}
      - LOG_LEVEL=info
    volumes:
      - "[LOCAL_CONFIG_PATH]:/app/config:ro"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:[CONTAINER_PORT]/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    labels:
      - "app=[STACK_NAME]"
      - "stack=[STACK_NAME]"
      - "traefik.enable=true"
      - "traefik.http.routers.[STACK_NAME].rule=Host(`[DOMAIN_NAME]`)"
      - "traefik.http.routers.[STACK_NAME].entrypoints=websecure"
      - "traefik.http.services.[STACK_NAME].loadbalancer.server.port=[CONTAINER_PORT]"

  # --- Base de dades PostgreSQL ---
  postgres:
    image: postgres:16-alpine
    container_name: [STACK_NAME]_postgres
    restart: unless-stopped
    networks:
      - backend
    environment:
      - POSTGRES_DB=${POSTGRES_DB:?Cal definir POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER:?Cal definir POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:?Cal definir POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - "[LOCAL_BACKUP_SCRIPT]:/etc/periodic/daily/backup.sh"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # --- Caché Redis ---
  redis:
    image: redis:7-alpine
    container_name: [STACK_NAME]_redis
    restart: unless-stopped
    networks:
      - backend
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "${REDIS_PASSWORD:?Cal definir REDIS_PASSWORD}"]
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
```

### Fitxer .env d'exemple

```bash
# .env — Copia a .env i omple els valors
# No commitegeuis aquest fitxer al repositori!

STACK_NAME=myapp
DOMAIN_NAME=myapp.example.com
IMAGE_NAME=myapp/web
IMAGE_TAG=latest
HOST_PORT=8080
CONTAINER_PORT=3000

POSTGRES_DB=myapp
POSTGRES_USER=myapp
POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD
REDIS_PASSWORD=CHANGE_ME_ANOTHER_PASSWORD
SECRET_KEY=CHANGE_ME_SECRET_KEY

LOCAL_CONFIG_PATH=./config
LOCAL_BACKUP_SCRIPT=./scripts/backup.sh
```

## Estàndards

- **Xarxes separades**: `frontend` (pública, amb accés al món exterior) i `backend` (privada, només entre serveis)
- **Volums amb nom explícit**: Facilita la identificació amb `docker volume ls`
- **Variables d'entorn**: Via fitxer `.env` — mai valors en clar al docker-compose.yml
- **Healthchecks**: Tots els serveis en tenen, condició `service_healthy` als depends_on
- **Restart policy**: `unless-stopped` per defecte (no reinicia si l'atura l'usuari)
- **Labels per Traefik**: Si s'usa reverse proxy, labels preparades per auto-descoberta
- **Contenidors Alpine**: Preferir variants Alpine per consum mínim de RAM i disc

## Més exemples

### Stack de monitoratge lleuger

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
    networks:
      - monitoring
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.example.com`)"
```

### Base de dades amb backups automàtics

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./scripts/backup.sh:/usr/local/bin/backup.sh
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    networks:
      - storage_net

  backup:
    image: alpine:3.19
    entrypoint: |
      sh -c '
      apk add --no-cache postgresql-client && (
        echo "0 3 * * * /scripts/backup.sh" > /var/spool/cron/crontabs/root
        crond -f
      )'
    volumes:
      - backup_data:/backups
      - ./scripts/backup.sh:/scripts/backup.sh:ro
    environment:
      - DB_HOST=db
      - DB_USER=myuser
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      - db
    networks:
      - storage_net
```

---

*Fet amb 💚 des de Mallorca*
