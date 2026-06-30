# 🚀 Guía de Despliegue — Postgres MCP Server

Esta guía documenta **todos los pasos necesarios** para desplegar el servidor MCP de PostgreSQL
en un entorno de producción, incluyendo los requisitos ocultos que no están en el README oficial.

---

## 📋 Prerrequisitos

### Del lado del servidor (Linux)

| Componente | Mínimo | Verificación |
|-----------|--------|-------------|
| Docker | 20.10+ | `docker --version` |
| Docker Compose | v2+ | `docker compose version` |
| PostgreSQL | 15-18 | `psql -c "SELECT version()"` |
| Nginx | 1.18+ | `nginx -v` |
| Certbot | 2.x | `certbot --version` |
| Git | 2.x | `git --version` |
| make + gcc | cualquiera | `which make gcc` |

### Del lado de la base de datos

Estas extensiones **deben instalarse manualmente** — el MCP no puede crearlas porque
requieren permisos de superusuario y/o instalación a nivel sistema operativo.

| Extensión | ¿Para qué? | Tools afectadas |
|-----------|-----------|-----------------|
| `pg_stat_statements` | Estadísticas de queries ejecutadas | `get_top_queries`, `analyze_workload_indexes`, `analyze_query_indexes` |
| `hypopg` | Índices hipotéticos (simulación) | `analyze_workload_indexes`, `analyze_query_indexes`, `explain_query` (modo hypothetical) |

> ⚠️ **Sin pg_stat_statements**: 3 de 9 tools no funcionan.
> ⚠️ **Sin hypopg**: el index tuning devuelve error pidiendo la instalación.

---

## 🔧 Paso 1 — Preparar la base de datos

### 1.1 Activar pg_stat_statements

Editar `postgresql.conf` y añadir a `shared_preload_libraries`:

```
shared_preload_libraries = 'pg_stat_statements'
```

Reiniciar PostgreSQL:

```bash
# Con aaPanel:
sudo -u postgres /www/server/pgsql/bin/pg_ctl -D /www/server/pgsql/data restart -l /www/server/pgsql/logs/pg.log

# Con systemd:
sudo systemctl restart postgresql
```

> ⚠️ **Problema conocido**: Si PostgreSQL no reinicia con error `pre-existing shared memory block`,
> limpiar los segmentos de shared memory:
> ```bash
> sudo ipcrm -M <KEY>    # la key aparece en el log de PostgreSQL
> # o limpiar todos los segmentos del usuario postgres:
> sudo ipcs -m | grep postgres | awk '{print $2}' | xargs -r sudo ipcrm -m
> ```

### 1.2 Crear extensiones

Conectarse como superusuario y ejecutar:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### 1.3 Instalar hypopg

**Si existe paquete para tu versión de PostgreSQL:**

```bash
sudo apt install postgresql-XX-hypopg   # XX = versión (16, 17...)
```

**Si NO existe paquete** (ej: PostgreSQL 18 con aaPanel):

```bash
cd /tmp
git clone --depth 1 https://github.com/HypoPG/hypopg.git
cd hypopg

# Compilar contra la instalación de PostgreSQL target
# Ajustar PATH al binario pg_config de tu PostgreSQL
sudo env PATH=/www/server/pgsql/bin:$PATH make
sudo env PATH=/www/server/pgsql/bin:$PATH make install
```

> 📌 La ruta `/www/server/pgsql/bin` es donde aaPanel instala PostgreSQL.
> Para otras instalaciones, usar `pg_config --bindir` para obtener la ruta.

Luego crear la extensión:

```sql
CREATE EXTENSION IF NOT EXISTS hypopg;
```

### 1.4 Permisos para el usuario MCP

El usuario que usará el MCP necesita permisos sobre los objetos de la BD:

```sql
GRANT CONNECT ON DATABASE wz_innov TO wz_innov;
GRANT USAGE ON SCHEMA public TO wz_innov;
GRANT ALL ON ALL TABLES IN SCHEMA public TO wz_innov;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO wz_innov;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO wz_innov;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO wz_innov;
```

### 1.5 Actualizar estadísticas

Después de instalar `pg_stat_statements`, ejecutar `ANALYZE` para poblar las
estadísticas iniciales:

```sql
ANALYZE;
```

> ⚠️ **Sin ANALYZE**: las tools de index tuning devuelven "statistics are not up-to-date".

---

## 🐳 Paso 2 — Configurar y levantar el contenedor

### 2.1 Clonar el repositorio

```bash
git clone https://github.com/luislopezsanchez/postgres-mcp-server.git
cd postgres-mcp-server
```

### 2.2 Crear el archivo `.env`

```bash
cp .env.example .env
nano .env
```

Contenido mínimo:

```env
POSTGRES_HOST=172.17.0.1
POSTGRES_PORT=5432
POSTGRES_DB=wz_innov
POSTGRES_USER=wz_innov
POSTGRES_PASSWORD=***
ACCESS_MODE=unrestricted
MCP_PORT=8000
```

> ⚠️ **IMPORTANTE sobre POSTGRES_HOST**:
>
> | Valor | ¿Funciona? | Nota |
> |-------|:----------:|------|
> | `localhost` | ✅ | El entrypoint lo remappea automáticamente |
> | `host.docker.internal` | ❌ | Solo funciona en Docker Desktop (Mac/Windows), **no en Linux** |
> | `172.17.0.1` | ✅ | Docker bridge gateway — funciona en Linux |
> | IP pública | ❌ | Causa error de shared memory en PostgreSQL |
>
> **Recomendación para Linux**: usar `localhost` (el entrypoint lo convierte) o `172.17.0.1`.

### 2.3 Transporte Streamable HTTP

El docker-compose usa `--transport=sse` por defecto. Para **Streamable HTTP** (recomendado):

```yaml
command:
  - "--transport=streamable-http"
  - "--access-mode=${ACCESS_MODE:-restricted}"
```

### 2.4 Renombrar docker-compose

```bash
mv docker-compose-1bd.yml docker-compose.yml
```

### 2.5 Levantar

```bash
docker compose up -d --build
```

### 2.6 Verificar

```bash
curl http://localhost:8000/health
# {"status":"ok","database":"connected","access_mode":"unrestricted"}
```

---

## 🌐 Paso 3 — Configurar Nginx Reverse Proxy

### 3.1 Obtener certificado SSL

```bash
# Crear sitio temporal solo HTTP para el challenge de Let's Encrypt
sudo tee /etc/nginx/sites-available/postgresmcp-temp << 'EOF'
server {
    listen 80;
    server_name postgresmcp.innovanet.uy;
    root /www/wwwroot/postgresmcp.innovanet.uy;
    location /.well-known/acme-challenge/ {
        root /www/wwwroot/postgresmcp.innovanet.uy;
    }
    location / {
        return 200 "provisioning";
    }
}
EOF

sudo mkdir -p /www/wwwroot/postgresmcp.innovanet.uy
sudo ln -s /etc/nginx/sites-available/postgresmcp-temp /etc/nginx/sites-enabled/
sudo nginx -s reload

# Emitir certificado
sudo certbot certonly --webroot \
  -w /www/wwwroot/postgresmcp.innovanet.uy \
  -d postgresmcp.innovanet.uy \
  --non-interactive --agree-tos \
  --email admin@innovanet.uy
```

### 3.2 Configurar el proxy con SSL + X-Api-Key

```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name postgresmcp.innovanet.uy;

    ssl_certificate     /etc/letsencrypt/live/postgresmcp.innovanet.uy/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/postgresmcp.innovanet.uy/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # HTTP → HTTPS
    if ($server_port !~ 443) {
        rewrite ^(/.*)$ https://$host$1 permanent;
    }

    location / {
        # X-Api-Key authentication
        if ($http_x_api_key != "Azseginfo18") { return 401; }

        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;

        # SSE / Streamable HTTP
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
    }
}
```

### 3.3 Recargar

```bash
sudo nginx -t && sudo nginx -s reload
```

### 3.4 Verificar

```bash
# Debe devolver 401 sin API key
curl -sk https://postgresmcp.innovanet.uy/health

# Debe devolver 200 con API key
curl -sk -H "X-Api-Key: Azseginfo18" https://postgresmcp.innovanet.uy/health
```

---

## 🔌 Paso 4 — Conectar clientes MCP

### Hermes Agent

Añadir a `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  Postgres-MCP:
    url: https://postgresmcp.innovanet.uy/mcp
    headers:
      X-Api-Key: Azseginfo18
    timeout: 180
```

Reiniciar Hermes para que detecte el nuevo servidor.

### N8N (MCP Client Tool)

| Campo | Valor |
|-------|-------|
| Server Transport | `HTTP Streamable` |
| Endpoint URL | `https://postgresmcp.innovanet.uy/mcp` |
| Authentication | `Header Auth` |
| Header Name | `X-Api-Key` |
| Header Value | `Azseginfo18` |

---

## 🛠️ Tools disponibles (9)

| Tool | Función | Requisito |
|------|---------|-----------|
| `list_schemas` | Lista esquemas | — |
| `list_objects` | Lista tablas/vistas/secuencias | — |
| `get_object_details` | Detalle de columnas, constraints, índices | — |
| `execute_sql` | Ejecuta SQL (solo SELECT en restricted) | — |
| `explain_query` | Plan de ejecución | hypopg (opcional, para hypothetical indexes) |
| `get_top_queries` | Queries más lentas | pg_stat_statements |
| `analyze_workload_indexes` | Recomienda índices para la carga de trabajo | pg_stat_statements + hypopg |
| `analyze_query_indexes` | Recomienda índices para queries específicas | pg_stat_statements + hypopg |
| `analyze_db_health` | Salud integral de la BD (buffers, conexiones, índices, vacuum...) | Permisos en todos los objetos |

---

## 🔍 Troubleshooting

### "No index recommendations found" en workload indexes
Normal al principio. La BD necesita generar tráfico para que `pg_stat_statements` acumule datos.

### "Query execution timed out" en restricted mode
Cambiar `ACCESS_MODE=unrestricted` en `.env` y `docker compose up -d`.

### "Statistics are not up-to-date"
Ejecutar `ANALYZE;` como superusuario en la BD después de instalar pg_stat_statements.

### "permission denied for sequence/table"
El usuario MCP no tiene permisos. Ejecutar los GRANT del paso 1.4.

### "could not open shared memory segment" al reiniciar PostgreSQL
Limpiar shared memory: `sudo ipcs -m | grep postgres | awk '{print $2}' | xargs -r sudo ipcrm -m`

### Error 400 "invalid choice: 'http'" al levantar el contenedor
El transporte está mal escrito. Debe ser `--transport=streamable-http` (no `--transport=http`).

### "host.docker.internal" no resuelve
Usar `172.17.0.1` o `localhost`. `host.docker.internal` solo existe en Docker Desktop (Mac/Windows).

### Container no conecta a PostgreSQL
Verificar que PostgreSQL escucha en `listen_addresses = '*'` y que `pg_hba.conf` permite conexiones desde `172.17.0.0/16` con `md5`.

---

## 📄 Licencia

MIT — Ver [LICENSE](LICENSE)
