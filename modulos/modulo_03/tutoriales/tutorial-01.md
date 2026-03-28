![Banner del proyecto](/modulos/assets/Banner.png)

# Tutorial 01 - Aplicación multicontenedor con Docker Compose

> **Objetivo:** Construir, levantar y operar una aplicación compuesta por tres servicios (aplicación web, base de datos y herramienta de administración) usando Docker Compose. Al finalizar, el estudiante habrá definido un archivo `docker-compose.yml`, verificado la comunicación entre servicios por nombre, inspeccionado la topología de red y validado el ciclo de vida completo de la aplicación.

Este tutorial asume que Docker y Docker Compose están instalados y operativos. Si no ha completado la instalación, consulte el Tutorial 01.

## Contexto

Hasta este punto del curso, los contenedores se han ejecutado de forma individual con `docker run`. Este tutorial introduce el paso hacia aplicaciones compuestas por múltiples servicios que colaboran entre sí, definidos y operados de forma declarativa con Docker Compose.

La aplicación que se construirá registra y consulta marcadores (bookmarks) a través de una API REST. Su arquitectura involucra tres servicios:

| Servicio | Imagen / Build | Función |
|----------|---------------|---------|
| `api` | Imagen construida desde Dockerfile (Flask) | Recibe peticiones HTTP y opera sobre la base de datos |
| `db` | `postgres:16` | Almacena los datos de la aplicación |
| `adminer` | `adminer` | Interfaz web para inspeccionar la base de datos |

## 1. Crear la estructura del proyecto

Cree el directorio del proyecto y los archivos necesarios:

```bash
mkdir -p compose-tutorial/api
cd compose-tutorial
```

La estructura final será:

```plaintext
compose-tutorial/
├── api/
│   ├── main.py
│   └── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env
```

## 2. Escribir el código de la aplicación

### 2.1. Archivo de dependencias

Cree `api/requirements.txt` con las dependencias de Python:

```bash
cat > api/requirements.txt << 'EOF'
flask==3.1.*
psycopg2-binary==2.9.*
EOF
```

### 2.2. Código de la API

Cree `api/main.py`. Esta aplicación Flask expone dos endpoints: uno para crear marcadores y otro para listarlos.

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
        CREATE TABLE IF NOT EXISTS bookmarks (
            id SERIAL PRIMARY KEY,
            url VARCHAR(500) NOT NULL,
            titulo VARCHAR(200) DEFAULT '',
            creado TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    cur.close()
    conn.close()

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

@app.route("/bookmarks", methods=["GET"])
def listar():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, url, titulo, creado FROM bookmarks ORDER BY id DESC LIMIT 20")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify([
        {"id": r[0], "url": r[1], "titulo": r[2], "creado": str(r[3])}
        for r in rows
    ])

@app.route("/bookmarks", methods=["POST"])
def crear():
    data = request.get_json(force=True)
    url = data.get("url", "")
    titulo = data.get("titulo", "")
    if not url:
        return jsonify({"error": "campo 'url' requerido"}), 400
    conn = get_conn()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO bookmarks (url, titulo) VALUES (%s, %s) RETURNING id",
        (url, titulo)
    )
    new_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"id": new_id}), 201

if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=5000)
PYEOF
```

> [!NOTE]
> La cadena de conexión a la base de datos proviene de la variable de entorno `DATABASE_URL`. No está hardcodeada en el código. Esto permite que la misma imagen funcione en diferentes ambientes con distintas configuraciones.

## 3. Escribir el Dockerfile

El Dockerfile define cómo se construye la imagen del servicio `api`:

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY api/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY api/ .

EXPOSE 5000

CMD ["python", "main.py"]
EOF
```

Cada instrucción tiene un propósito:

| Instrucción | Función |
|-------------|---------|
| `FROM python:3.12-slim` | Imagen base ligera con Python 3.12 |
| `WORKDIR /app` | Establece el directorio de trabajo |
| `COPY ... requirements.txt` + `RUN pip install` | Instala dependencias antes de copiar el código (aprovecha la caché de capas) |
| `COPY api/ .` | Copia el código de la aplicación |
| `EXPOSE 5000` | Documenta el puerto que usa la aplicación |
| `CMD` | Define el proceso principal del contenedor |

## 4. Crear el archivo de variables de entorno

El archivo `.env` centraliza la configuración de credenciales y parámetros que se interpolan en `docker-compose.yml`:

```bash
cat > .env << 'EOF'
POSTGRES_USER=bookmarks_user
POSTGRES_PASSWORD=tutorial_pwd_2026
POSTGRES_DB=bookmarksdb
EOF
```

> [!CAUTION]
> En un proyecto real, el archivo `.env` debe incluirse en `.gitignore` para evitar que las credenciales se publiquen en el repositorio. En este tutorial se crea directamente con fines didácticos.

## 5. Definir la aplicación con Docker Compose

Cree el archivo `docker-compose.yml`. Este archivo describe los tres servicios, sus relaciones, la red y el volumen persistente:

```bash
cat > docker-compose.yml << 'EOF'
services:
  api:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

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
    restart: unless-stopped

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db
    restart: unless-stopped

volumes:
  pgdata:
EOF
```

Antes de levantar, valide que la configuración se parsea correctamente y que las variables se interpolan:

```bash
docker compose config
```

Revise en la salida que `DATABASE_URL` contenga los valores reales de `.env`, no los literales `${...}`.

### Análisis del archivo Compose

Observe los siguientes aspectos:

- **`api`** construye su imagen con `build: .` y publica el puerto 5000. Su variable `DATABASE_URL` usa `@db:5432`, donde `db` es el nombre del servicio de base de datos, no una IP.
- **`db`** usa la imagen oficial de PostgreSQL. Incluye un `healthcheck` que ejecuta `pg_isready` para verificar que el motor esté operativo antes de que `api` intente conectarse.
- **`adminer`** publica el puerto 8080 para acceso desde el navegador.
- **`pgdata`** es un volumen nombrado que persiste los datos de PostgreSQL.
- No se declara una sección `networks`. Compose crea automáticamente una red bridge para el proyecto.

> [!IMPORTANT]
> La directiva `depends_on` con `condition: service_healthy` garantiza que `api` no se inicie hasta que PostgreSQL esté realmente aceptando conexiones. Sin esta condición, `api` podría fallar al intentar conectarse durante la inicialización de la base de datos.

## 6. Levantar la aplicación

```bash
docker compose up -d
```

Este comando:

1. Lee `docker-compose.yml` y `.env`
2. Construye la imagen de `api` a partir del `Dockerfile`
3. Descarga `postgres:16` y `adminer` si no están disponibles localmente
4. Crea la red del proyecto
5. Crea el volumen `pgdata`
6. Inicia `db` y espera a que su healthcheck sea exitoso
7. Inicia `api` y `adminer`

## 7. Verificar el estado de los servicios

```bash
docker compose ps
```

Verifique que los tres servicios aparezcan con estado `running` (y `db` con `healthy`):

```plaintext
NAME                        SERVICE   STATUS            PORTS
compose-tutorial-api-1      api       running           0.0.0.0:5000->5000/tcp
compose-tutorial-db-1       db        running (healthy)
compose-tutorial-adminer-1  adminer   running           0.0.0.0:8080->8080/tcp
```

> [!WARNING]
> Si algún servicio aparece como `exited` o `restarting`, consulte los logs antes de continuar: `docker compose logs <servicio>`.

## 8. Probar la aplicación

### 8.1. Verificar el endpoint de salud

```bash
curl http://localhost:5000/health
```

Resultado esperado:

```json
{"status":"ok"}
```

### 8.2. Crear un marcador

```bash
curl -X POST http://localhost:5000/bookmarks \
  -H "Content-Type: application/json" \
  -d '{"url": "https://docs.docker.com", "titulo": "Documentación Docker"}'
```

Resultado esperado:

```json
{"id":1}
```

### 8.3. Listar marcadores

```bash
curl http://localhost:5000/bookmarks
```

### 8.4. Crear un segundo marcador y volver a listar

```bash
curl -X POST http://localhost:5000/bookmarks \
  -H "Content-Type: application/json" \
  -d '{"url": "https://flask.palletsprojects.com", "titulo": "Flask"}'

curl http://localhost:5000/bookmarks
```

Verifique que la respuesta incluya ambos marcadores.

### 8.5. Inspeccionar desde Adminer

Abra `http://localhost:8080` en un navegador. Complete los campos de conexión:

| Campo | Valor |
|-------|-------|
| Sistema | PostgreSQL |
| Servidor | `db` |
| Usuario | `bookmarks_user` |
| Contraseña | `tutorial_pwd_2026` |
| Base de datos | `bookmarksdb` |

> [!NOTE]
> En Adminer, el servidor es `db` (el nombre del servicio), no `localhost`. Adminer se ejecuta dentro de la red Docker y alcanza a PostgreSQL por nombre de servicio.

Navegue hasta la tabla `bookmarks` y confirme que los registros creados con `curl` aparecen en la base de datos.

## 9. Inspeccionar la red del proyecto

### 9.1. Listar redes

```bash
docker network ls | grep compose-tutorial
```

Observe la red creada automáticamente por Compose (nombrada `compose-tutorial_default`).

### 9.2. Inspeccionar la red

```bash
docker network inspect compose-tutorial_default
```

En la sección `Containers` de la salida, identifique los tres contenedores conectados. Cada uno tiene una IP asignada dentro del rango de la red.

### 9.3. Verificar resolución DNS entre servicios

Desde el contenedor de `api`, confirme que el nombre `db` resuelve a una dirección IP:

```bash
docker compose exec api python -c "import socket; print(socket.gethostbyname('db'))"
```

El resultado debe ser una IP interna (por ejemplo, `172.18.0.2`). Esta resolución la provee el DNS interno de Docker, que asocia automáticamente el nombre del servicio con la IP del contenedor correspondiente.

### 9.4. Verificar que la comunicación funciona por nombre

```bash
docker compose exec api python -c "
import socket
s = socket.socket()
s.settimeout(3)
s.connect(('db', 5432))
print('Conexión a db:5432 exitosa')
s.close()
"
```

### 9.5. Verificar que la base de datos no está expuesta al host

Observe que `db` no tiene la directiva `ports` en `docker-compose.yml`. Confirme que no es accesible directamente desde el host:

```bash
curl -s --max-time 3 http://localhost:5432 || echo "No accesible desde el host (esperado)"
```

Este comportamiento es correcto: la base de datos solo es accesible desde la red interna de Docker.

## 10. Inspeccionar logs

### 10.1. Logs de todos los servicios

```bash
docker compose logs --tail 10
```

### 10.2. Logs de un servicio específico

```bash
docker compose logs api
```

### 10.3. Seguimiento en tiempo real

En una terminal separada, ejecute:

```bash
docker compose logs -f api
```

Luego, desde otra terminal, envíe una petición:

```bash
curl http://localhost:5000/bookmarks
```

Observe cómo el log de `api` registra la petición en tiempo real. Cierre el seguimiento con `Ctrl+C`.

## 11. Ejecutar comandos dentro de un contenedor

### 11.1. Consultar la base de datos directamente

```bash
docker compose exec db psql -U bookmarks_user -d bookmarksdb -c "SELECT * FROM bookmarks;"
```

### 11.2. Verificar variables de entorno del servicio api

```bash
docker compose exec api env | grep DATABASE_URL
```

Confirme que el valor contiene `@db:5432` y las credenciales correctas.

## 12. Detener y recrear la aplicación

### 12.1. Detener sin eliminar volúmenes

```bash
docker compose down
```

### 12.2. Verificar que el volumen persiste

```bash
docker volume ls | grep pgdata
```

El volumen `compose-tutorial_pgdata` debe seguir listado.

### 12.3. Levantar nuevamente y verificar datos

```bash
docker compose up -d
curl http://localhost:5000/bookmarks
```

Los marcadores creados anteriormente deben seguir presentes. Esto confirma que el volumen nombrado preserva los datos entre ciclos de `down` y `up`.

## 13. Limpieza

Cuando desee eliminar completamente la aplicación, incluyendo los datos persistentes:

```bash
docker compose down -v
```

Verifique:

```bash
docker volume ls | grep pgdata
```

El volumen ya no debe aparecer.

> [!CAUTION]
> El flag `-v` elimina los volúmenes nombrados del proyecto. Los datos almacenados en PostgreSQL se pierden de forma irrecuperable. Use este comando únicamente cuando desee un reinicio completo del estado.

## Solución de problemas frecuentes

### El servicio `api` aparece como `exited (1)`

Revise los logs:

```bash
docker compose logs api
```

Las causas más comunes son:

- `DATABASE_URL` no definida o mal formada. Verifique `.env` y ejecute `docker compose config`.
- La cadena de conexión usa `localhost` en lugar de `db`. Corrija a `@db:5432`.
- Las credenciales de `.env` no coinciden entre `api` y `db`.

### Adminer no puede conectarse a la base de datos

Verifique que en el campo **Servidor** escribió `db`, no `localhost`. Adminer opera dentro de la red Docker y debe usar el nombre del servicio.

### Puerto 5000 o 8080 ya ocupado

```bash
lsof -i :5000
lsof -i :8080
```

Detenga el proceso que ocupa el puerto, o modifique la sección `ports` en `docker-compose.yml` para usar puertos alternativos del host:

```yaml
ports:
  - "5001:5000"    # cambia el puerto del host a 5001
```

### Error de sintaxis en `docker-compose.yml`

```bash
docker compose config
```

Si hay errores de parseo, Compose los reporta con la línea y columna del problema. Verifique que la indentación use espacios (no tabulaciones) y que la estructura YAML sea válida.

## Referencias

| Recurso | Enlace |
|---------|--------|
| Docker Compose overview | [docs.docker.com/compose/](https://docs.docker.com/compose/) |
| Compose file reference | [docs.docker.com/reference/compose-file/](https://docs.docker.com/reference/compose-file/) |
| Networking in Compose | [docs.docker.com/compose/how-tos/networking/](https://docs.docker.com/compose/how-tos/networking/) |
| Imagen oficial de PostgreSQL | [hub.docker.com/_/postgres](https://hub.docker.com/_/postgres) |
| Imagen oficial de Adminer | [hub.docker.com/_/adminer](https://hub.docker.com/_/adminer) |
