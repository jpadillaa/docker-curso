![Banner del proyecto](/modulos/assets/Banner.png)

# Tutorial 03 - Persistencia, configuración y diagnóstico en aplicaciones compuestas

> **Objetivo:** Configurar una aplicación multicontenedor con segmentación de redes, persistencia explícita con volúmenes y bind mounts, inyección de configuración por variables de entorno, y aplicar una estrategia sistemática de diagnóstico ante fallos inducidos deliberadamente.

Este tutorial asume que el estudiante completó el Tutorial 02 y tiene experiencia básica operando aplicaciones con Docker Compose. Se requiere Docker y Docker Compose instalados y operativos.

## Contexto

El Tutorial 02 presentó una aplicación multicontenedor funcional con una red implícita y un volumen nombrado. Este tutorial profundiza en tres aspectos que determinan la robustez operativa de una aplicación compuesta:

- **Redes segmentadas**: no todos los servicios deben poder comunicarse entre sí.
- **Persistencia diferenciada**: el código de desarrollo y los datos de negocio requieren estrategias distintas.
- **Diagnóstico**: la capacidad de identificar y resolver fallos es tan importante como la de definir la aplicación.

La aplicación que se construirá es un registro de tareas (tasks) con tres servicios y dos redes:

```plaintext
                    ┌─────────────┐
    Host :80 ──────►│   web       │
                    │  (Nginx)    │
                    └──────┬──────┘
               red:        │        
              frontend     │ proxy_pass → api:5000
                           │
                    ┌──────▼──────┐
                    │   api       │
                    │  (Flask)    │
                    └──────┬──────┘
               red:        │
              backend      │ db:5432
                           │
                    ┌──────▼──────┐
                    │   db        │
                    │ (PostgreSQL)│
                    └─────────────┘
```

`web` y `api` comparten la red `frontend`. `api` y `db` comparten la red `backend`. `web` no tiene acceso directo a `db`.

## 1. Crear la estructura del proyecto

```bash
mkdir -p persist-tutorial/api persist-tutorial/nginx
cd persist-tutorial
```

Estructura final:

```plaintext
persist-tutorial/
├── api/
│   ├── main.py
│   └── requirements.txt
├── nginx/
│   └── default.conf
├── Dockerfile.api
├── compose.yml
├── .env
└── .env.example
```

## 2. Escribir el código de la aplicación

### 2.1. Dependencias

```bash
cat > api/requirements.txt << 'EOF'
flask==3.1.*
psycopg2-binary==2.9.*
EOF
```

### 2.2. Código de la API

```bash
cat > api/main.py << 'PYEOF'
import os
import psycopg2
from flask import Flask, request, jsonify

app = Flask(__name__)

def get_conn():
    return psycopg2.connect(os.environ["DATABASE_URL"])

def init_db():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id SERIAL PRIMARY KEY,
            descripcion VARCHAR(500) NOT NULL,
            completada BOOLEAN DEFAULT FALSE,
            creado TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    cur.close()
    conn.close()

@app.route("/api/health")
def health():
    return jsonify({"status": "ok"})

@app.route("/api/tasks", methods=["GET"])
def listar():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, descripcion, completada, creado FROM tasks ORDER BY id DESC")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify([
        {"id": r[0], "descripcion": r[1], "completada": r[2], "creado": str(r[3])}
        for r in rows
    ])

@app.route("/api/tasks", methods=["POST"])
def crear():
    data = request.get_json(force=True)
    descripcion = data.get("descripcion", "")
    if not descripcion:
        return jsonify({"error": "campo 'descripcion' requerido"}), 400
    conn = get_conn()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO tasks (descripcion) VALUES (%s) RETURNING id",
        (descripcion,)
    )
    new_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"id": new_id}), 201

if __name__ == "__main__":
    init_db()
    debug_mode = os.environ.get("FLASK_DEBUG", "0") == "1"
    app.run(host="0.0.0.0", port=5000, debug=debug_mode)
PYEOF
```

> [!NOTE]
> La variable `FLASK_DEBUG` controla si Flask activa la recarga automática al detectar cambios en el código. Esto será relevante cuando se use un bind mount para desarrollo.

## 3. Escribir la configuración de Nginx

Nginx actúa como punto de entrada público y redirige las peticiones `/api/*` al servicio `api`:

```bash
cat > nginx/default.conf << 'EOF'
server {
    listen 80;

    location / {
        return 200 'Servicio activo\n';
        add_header Content-Type text/plain;
    }

    location /api/ {
        proxy_pass http://api:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

Observe que `proxy_pass` usa `http://api:5000`. Nginx resuelve `api` a la IP del contenedor correspondiente mediante el DNS interno de Docker.

## 4. Escribir el Dockerfile

```bash
cat > Dockerfile.api << 'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY api/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY api/ .

EXPOSE 5000

CMD ["python", "main.py"]
EOF
```

## 5. Crear los archivos de configuración

### 5.1. Archivo `.env`

```bash
cat > .env << 'EOF'
POSTGRES_USER=tasks_user
POSTGRES_PASSWORD=persist_pwd_2026
POSTGRES_DB=tasksdb
FLASK_DEBUG=1
EOF
```

### 5.2. Archivo `.env.example`

Este archivo sirve como plantilla para otros desarrolladores. No contiene valores reales:

```bash
cat > .env.example << 'EOF'
# Copie este archivo como .env y complete los valores
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
FLASK_DEBUG=0
EOF
```

> [!TIP]
> En un proyecto real, versione `.env.example` en el repositorio y agregue `.env` a `.gitignore`. Esto documenta qué variables se necesitan sin exponer credenciales.

## 6. Definir la aplicación con Docker Compose

```bash
cat > compose.yml << 'EOF'
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    networks:
      - frontend

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      - FLASK_DEBUG=${FLASK_DEBUG}
    volumes:
      - ./api:/app
    depends_on:
      db:
        condition: service_healthy
    networks:
      - frontend
      - backend

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  pgdata:

networks:
  frontend:
  backend:
EOF
```

### Análisis de las decisiones de diseño

#### Dos tipos de almacenamiento en la misma aplicación

| Servicio | Tipo de montaje | Ruta | Propósito |
|----------|----------------|------|-----------|
| `web` | Bind mount (solo lectura) | `./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro` | Configuración local del proxy |
| `api` | Bind mount | `./api:/app` | Código fuente para desarrollo con recarga en vivo |
| `db` | Volumen nombrado | `pgdata:/var/lib/postgresql/data` | Datos persistentes de negocio |

El bind mount de `api` permite editar `main.py` en el host y ver los cambios reflejados sin reconstruir la imagen (gracias a `FLASK_DEBUG=1`). El volumen nombrado de `db` garantiza que los datos sobrevivan a la recreación de contenedores.

#### Dos redes explícitas

- **`frontend`**: conecta `web` y `api`. Permite que Nginx alcance a la API.
- **`backend`**: conecta `api` y `db`. Permite que la API alcance a PostgreSQL.

`web` no está en la red `backend`, por lo tanto no puede comunicarse directamente con `db`. Esta segmentación limita la superficie de comunicación de cada servicio.

#### Publicación selectiva de puertos

Solo `web` publica el puerto 80. Ni `api` ni `db` exponen puertos al host. El tráfico externo entra por Nginx y este lo redirige internamente.

> [!IMPORTANT]
> Valide la configuración antes de levantar. Este paso detecta errores de sintaxis YAML y problemas de interpolación de variables:
> ```bash
> docker compose config
> ```

## 7. Levantar la aplicación

```bash
docker compose up -d
```

Verifique el estado:

```bash
docker compose ps
```

Resultado esperado: tres servicios en estado `running`, con `db` reportando `healthy`.

## 8. Validar la funcionalidad

### 8.1. Endpoint de salud

```bash
curl http://localhost/api/health
```

```json
{"status":"ok"}
```

### 8.2. Crear tareas

```bash
curl -X POST http://localhost/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"descripcion": "Leer la lectura 10"}'

curl -X POST http://localhost/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"descripcion": "Completar el tutorial 03"}'
```

### 8.3. Listar tareas

```bash
curl http://localhost/api/tasks
```

Confirme que ambas tareas aparecen en la respuesta.

## 9. Verificar la segmentación de redes

### 9.1. Listar redes del proyecto

```bash
docker network ls | grep persist-tutorial
```

Deben aparecer `persist-tutorial_frontend` y `persist-tutorial_backend`.

### 9.2. Verificar qué contenedores están en cada red

```bash
docker network inspect persist-tutorial_frontend \
  --format '{{range .Containers}}{{.Name}} {{end}}'
```

Resultado esperado: `persist-tutorial-web-1 persist-tutorial-api-1`

```bash
docker network inspect persist-tutorial_backend \
  --format '{{range .Containers}}{{.Name}} {{end}}'
```

Resultado esperado: `persist-tutorial-api-1 persist-tutorial-db-1`

### 9.3. Confirmar aislamiento

Intente resolver `db` desde el contenedor `web`:

```bash
docker compose exec web sh -c "ping -c 1 -W 2 db 2>&1 || echo 'No se puede resolver db (esperado)'"
```

La resolución debe fallar, confirmando que `web` no tiene acceso a `db` porque no comparten red.

Ahora verifique que `api` sí puede resolver `db`:

```bash
docker compose exec api python -c "import socket; print('db resuelve a:', socket.gethostbyname('db'))"
```

> [!NOTE]
> Este aislamiento es un efecto directo de la topología de redes definida en `compose.yml`. No requiere reglas de firewall adicionales: basta con no compartir la red.

## 10. Verificar la persistencia

### 10.1. Confirmar que los datos sobreviven a `down`

```bash
docker compose down
docker compose up -d
curl http://localhost/api/tasks
```

Las tareas creadas anteriormente deben seguir presentes.

### 10.2. Inspeccionar el volumen

```bash
docker volume ls | grep pgdata
docker volume inspect persist-tutorial_pgdata
```

Examine el campo `Mountpoint` en la salida: indica la ruta física del volumen en el host.

### 10.3. Confirmar que `down -v` destruye los datos

```bash
docker compose down -v
docker compose up -d
curl http://localhost/api/tasks
```

La respuesta debe ser un arreglo vacío. El volumen fue eliminado y PostgreSQL reinició con una base de datos limpia.

> [!CAUTION]
> Ejecute `down -v` solo con la intención deliberada de destruir los datos persistentes. Este paso no es reversible.

Recree datos de prueba para los pasos siguientes:

```bash
curl -X POST http://localhost/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"descripcion": "Tarea de prueba para diagnostico"}'
```

## 11. Verificar el bind mount para desarrollo

### 11.1. Confirmar que el código del host se refleja en el contenedor

```bash
docker compose exec api cat main.py | head -5
```

Compare con el archivo `api/main.py` local. Deben coincidir.

### 11.2. Modificar el código y verificar la recarga

Edite `api/main.py` en el host. Agregue un nuevo endpoint al final del archivo, antes de `if __name__`:

```bash
cat >> api/main.py << 'PYEOF'

@app.route("/api/version")
def version():
    return jsonify({"version": "0.2.0"})
PYEOF
```

Espere unos segundos (Flask detecta el cambio automáticamente con debug habilitado) y pruebe:

```bash
curl http://localhost/api/version
```

```json
{"version":"0.2.0"}
```

El cambio se reflejó sin reconstruir la imagen ni reiniciar el contenedor.

> [!TIP]
> Este comportamiento funciona porque el bind mount `./api:/app` sobrescribe el contenido copiado por el Dockerfile. En un entorno de producción, se usaría la imagen construida (sin bind mount) para que contenga el código final.

## 12. Verificar variables de entorno

### 12.1. Desde el contenedor

```bash
docker compose exec api env | grep -E "DATABASE_URL|FLASK_DEBUG"
```

Confirme que `DATABASE_URL` usa `@db:5432` y que `FLASK_DEBUG` es `1`.

### 12.2. Verificar la configuración resuelta de Compose

```bash
docker compose config | grep DATABASE_URL
```

Este comando muestra el valor después de la interpolación desde `.env`.

## 13. Ejercicios de diagnóstico

Esta sección presenta fallos deliberados para practicar el proceso de diagnóstico. En cada caso: introduzca el error, observe el síntoma, diagnostique y corrija.

### Fallo 1: uso incorrecto de `localhost`

**Introducir el error**: modifique temporalmente la variable `DATABASE_URL` en `compose.yml` para usar `localhost` en lugar de `db`:

```bash
sed -i 's/@db:5432/@localhost:5432/' compose.yml
docker compose up -d
```

**Observar el síntoma**:

```bash
docker compose ps
```

El servicio `api` probablemente aparezca como `exited (1)` o `restarting`.

**Diagnosticar**:

```bash
docker compose logs --tail 10 api
```

Busque un mensaje como `Connection refused` apuntando a `localhost:5432`.

**Comprender la causa**: dentro del contenedor `api`, `localhost` refiere al propio contenedor, donde no hay ningún proceso PostgreSQL escuchando. El servicio de base de datos está en otro contenedor, accesible únicamente por el nombre de servicio `db`.

**Corregir**:

```bash
sed -i 's/@localhost:5432/@db:5432/' compose.yml
docker compose up -d
```

Verifique:

```bash
curl http://localhost/api/health
```

### Fallo 2: archivo `.env` ausente

**Introducir el error**:

```bash
mv .env .env.backup
docker compose down
docker compose up -d
```

**Observar el síntoma**:

```bash
docker compose ps
docker compose logs api
```

La aplicación puede fallar porque `DATABASE_URL` contiene los literales `${POSTGRES_USER}` en lugar de valores reales.

**Diagnosticar**:

```bash
docker compose config 2>&1 | head -20
```

Observe si las variables aparecen vacías o sin resolver.

**Corregir**:

```bash
mv .env.backup .env
docker compose up -d
```

### Fallo 3: volumen montado en ruta incorrecta

**Introducir el error**: modifique temporalmente la ruta del volumen de `db`:

```bash
sed -i 's|pgdata:/var/lib/postgresql/data|pgdata:/var/lib/postgres/data|' compose.yml
docker compose down -v
docker compose up -d
```

**Observar el síntoma**: la aplicación funciona, pero los datos no persisten entre recreaciones.

```bash
curl -X POST http://localhost/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"descripcion": "Dato que deberia persistir"}'

docker compose down
docker compose up -d
curl http://localhost/api/tasks
```

La respuesta será un arreglo vacío: los datos se perdieron.

**Diagnosticar**: verifique la ruta de montaje real:

```bash
docker inspect persist-tutorial-db-1 --format '{{json .Mounts}}' | python3 -m json.tool
```

La ruta montada es `/var/lib/postgres/data`, pero PostgreSQL almacena sus datos en `/var/lib/postgresql/data`. El volumen existe pero no contiene los datos reales.

**Corregir**:

```bash
sed -i 's|pgdata:/var/lib/postgres/data|pgdata:/var/lib/postgresql/data|' compose.yml
docker compose down -v
docker compose up -d
```

> [!WARNING]
> Los errores de ruta en volúmenes no producen mensajes de error al arrancar. El contenedor funciona normalmente, pero los datos se escriben en la capa de escritura efímera en lugar del volumen. Este tipo de fallo solo se detecta al recrear el contenedor y observar que los datos desaparecieron.

### Fallo 4: servicios que no comparten red

**Introducir el error**: retire `api` de la red `backend`:

```bash
cat > /tmp/patch.py << 'PYEOF'
import re
with open("compose.yml") as f:
    content = f.read()
# Remove backend from api's networks
content = content.replace(
    "    networks:\n      - frontend\n      - backend\n",
    "    networks:\n      - frontend\n",
    1  # only first occurrence (api)
)
with open("compose.yml", "w") as f:
    f.write(content)
PYEOF
python3 /tmp/patch.py
docker compose down
docker compose up -d
```

**Observar el síntoma**:

```bash
docker compose ps
docker compose logs --tail 5 api
```

El servicio `api` falla con un error de resolución DNS o conexión rechazada hacia `db`.

**Diagnosticar**:

```bash
docker compose exec api python -c "import socket; socket.gethostbyname('db')" 2>&1
```

La resolución falla porque `api` y `db` ya no comparten una red. Sin red compartida, el DNS interno de Docker no resuelve el nombre.

**Corregir**: restaure la red `backend` en la sección de `api` dentro de `compose.yml` y recree:

```bash
# Restaurar compose.yml original (reescriba la sección networks de api)
# Asegúrese de que api tenga:
#   networks:
#     - frontend
#     - backend
```

Recree el archivo `compose.yml` desde el paso 6 si es necesario, y levante nuevamente:

```bash
docker compose up -d
curl http://localhost/api/health
```

## 14. Comandos de diagnóstico de referencia

| Qué verificar | Comando |
|----------------|---------|
| Estado de los servicios | `docker compose ps` |
| Logs de un servicio | `docker compose logs <servicio>` |
| Logs en tiempo real | `docker compose logs -f <servicio>` |
| Variables de entorno | `docker compose exec <servicio> env` |
| Resolución DNS | `docker compose exec <servicio> python -c "import socket; print(socket.gethostbyname('<destino>'))"` |
| Conectividad TCP | `docker compose exec <servicio> python -c "import socket; s=socket.socket(); s.settimeout(3); s.connect(('<destino>', <puerto>)); print('OK'); s.close()"` |
| Archivos montados | `docker compose exec <servicio> ls -la <ruta>` |
| Redes del proyecto | `docker network ls \| grep <proyecto>` |
| Contenedores en una red | `docker network inspect <red> --format '{{range .Containers}}{{.Name}} {{end}}'` |
| Montajes de un contenedor | `docker inspect <contenedor> --format '{{json .Mounts}}'` |
| Configuración resuelta | `docker compose config` |
| Volúmenes existentes | `docker volume ls` |

## 15. Limpieza final

```bash
docker compose down -v
cd ..
```

Si desea eliminar el directorio del proyecto:

```bash
rm -rf persist-tutorial
```

## Solución de problemas frecuentes

### `502 Bad Gateway` al acceder desde el navegador

Nginx está operativo pero no puede alcanzar a `api`. Verifique:

1. Que `api` esté `running`: `docker compose ps`
2. Que `web` y `api` compartan la red `frontend`
3. Los logs de `api` para detectar errores de arranque

### La API responde pero la base de datos aparece vacía después de recrear

Verifique la ruta de montaje del volumen. La ruta correcta para PostgreSQL es `/var/lib/postgresql/data`. Use `docker inspect` para confirmar la ruta real.

### Cambios en el código no se reflejan

Verifique que:

1. `FLASK_DEBUG=1` esté definido en `.env`
2. El bind mount `./api:/app` esté presente en `compose.yml`
3. El archivo editado sea exactamente `api/main.py` (la ruta relativa al directorio del proyecto)

### `docker compose config` muestra variables vacías

El archivo `.env` no existe o no está en el mismo directorio que `compose.yml`. Verifique:

```bash
ls -la .env
cat .env
```

## Referencias

| Recurso | Enlace |
|---------|--------|
| Docker Compose overview | [docs.docker.com/compose/](https://docs.docker.com/compose/) |
| Networking in Compose | [docs.docker.com/compose/how-tos/networking/](https://docs.docker.com/compose/how-tos/networking/) |
| Volumes in Compose | [docs.docker.com/compose/how-tos/use-volumes/](https://docs.docker.com/compose/how-tos/use-volumes/) |
| Environment variables in Compose | [docs.docker.com/compose/how-tos/environment-variables/](https://docs.docker.com/compose/how-tos/environment-variables/) |
| Bridge network driver | [docs.docker.com/engine/network/drivers/bridge/](https://docs.docker.com/engine/network/drivers/bridge/) |
| Compose CLI reference | [docs.docker.com/reference/cli/docker/compose/](https://docs.docker.com/reference/cli/docker/compose/) |
