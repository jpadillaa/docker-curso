![Banner del proyecto](/modulos/assets/Banner.png)

# Lectura 12 - Operación y troubleshooting con Docker Compose

## Prerrequisitos

Esta lectura asume que el estudiante:

- ha definido y operado aplicaciones multicontenedor con Docker Compose
- comprende el modelo de red de Docker, incluyendo comunicación por nombre de servicio y segmentación con redes explícitas
- ha trabajado con volúmenes nombrados, bind mounts y variables de entorno en Compose
- tiene familiaridad con la estructura de un archivo `docker-compose.yml` y el ciclo de vida de contenedores
- maneja la terminal en un entorno Linux o equivalente

## Introducción

Las lecturas anteriores mostraron cómo definir una aplicación multicontenedor con Docker Compose, cómo conectar sus servicios mediante redes Docker y cómo gestionar su persistencia y configuración. En todos los casos, los ejemplos asumieron que la aplicación se iniciaba correctamente y que los servicios se comunicaban sin inconvenientes.

En la práctica, esto no siempre sucede. Un contenedor puede iniciar y finalizar de inmediato. Un servicio puede reportar estado `running` sin aceptar conexiones. Una variable de entorno puede estar ausente sin que el contenedor lo advierta al arrancar. Una base de datos puede no estar lista cuando la aplicación intenta conectarse por primera vez.

Esta lectura cierra el módulo de aplicaciones multicontenedor al abordar dos competencias complementarias, la **operación ordenada** de una aplicación Compose y el **diagnóstico sistemático** de fallos. El objetivo no es memorizar recetas, sino desarrollar la capacidad de observar, interpretar señales y localizar de forma metódica la causa raíz de un problema.

## Motivación

### Levantar no es operar

Ejecutar `docker compose up -d` y ver tres servicios en estado `running` no garantiza que la aplicación esté funcionando. Puede ocurrir que:

* la aplicación web responda con un error 500 porque no logra conectarse a la base de datos
* la base de datos esté en ejecución, pero aún no haya completado su inicialización
* un servicio entre en un ciclo de reinicio debido a una variable de entorno mal configurada
* los puertos publicados entren en conflicto con otro proceso del host

Cada uno de estos escenarios da lugar a una aplicación que **parece** estar levantada, pero **no funciona**. La diferencia entre un contenedor en ejecución y un servicio realmente operativo es una distinción fundamental que esta lectura busca hacer explícita.

### Los errores en aplicaciones compuestas son multicausales

En una aplicación con un solo contenedor, el espacio de diagnóstico es relativamente acotado. En cambio, en una aplicación con tres o más servicios, el problema puede originarse en distintos niveles:

* la configuración de un servicio individual
* la comunicación entre dos servicios
* el orden de arranque
* la persistencia de datos
* la topología de red
* la configuración del host


```mermaid
flowchart TB
    subgraph Compose["Aplicación Compose"]
        Web["web :80"]
        API["api :5000"]
        DB["db :5432"]
    end
    
    Host["Host"] -->|"puerto publicado"| Web
    Web -->|"red interna"| API
    API -->|"red interna"| DB
    
    F1["⚠ Puerto ocupado\nen el host"] -.-> Host
    F2["⚠ Nombre de servicio\nincorrecto"] -.-> Web
    F3["⚠ Variable de entorno\nausente"] -.-> API
    F4["⚠ Base de datos\naún no lista"] -.-> DB
    F5["⚠ Volumen en ruta\nincorrecta"] -.-> DB
    
    style Compose fill:#e8f4fd,color:#000
    style F1 fill:#dc3545,color:#fff
    style F2 fill:#dc3545,color:#fff
    style F3 fill:#dc3545,color:#fff
    style F4 fill:#dc3545,color:#fff
    style F5 fill:#dc3545,color:#fff
```

Sin una estrategia ordenada, el diagnóstico se convierte en un proceso errático de prueba y error que consume tiempo y provoca frustración.

> [!IMPORTANT]
> La capacidad de diagnosticar fallos en aplicaciones multicontenedor es tan importante como la capacidad de definirlas. Contar con un archivo `docker-compose.yml` correcto no elimina la necesidad de saber operar y depurar la aplicación.

## Operación básica con Docker Compose

### Ciclo de vida operativo

Una aplicación Compose atraviesa un ciclo de vida que incluye la creación de recursos como redes, volúmenes y contenedores, la ejecución de los servicios y, finalmente, su detención y eliminación. Los comandos principales de Compose corresponden a las distintas transiciones dentro de ese ciclo.

```mermaid
flowchart LR
    A["docker-compose.yml"] -->|"up"| B["Contenedores\ncreados y\nen ejecución"]
    B -->|"stop"| C["Contenedores\ndetenidos"]
    C -->|"start"| B
    B -->|"restart"| B
    C -->|"down"| D["Contenedores\ny redes\neliminados"]
    B -->|"down"| D
    D -->|"up"| B
    
    style A fill:#2496ED,color:#fff
    style B fill:#28a745,color:#fff
    style C fill:#ffc107,color:#000
    style D fill:#dc3545,color:#fff
```

### Comandos operativos

| Comando | Acción | Contenedores | Redes | Volúmenes |
|---------|--------|--------------|-------|-----------|
| `docker compose up` | Crea y levanta todos los servicios en modo attached | Creados e iniciados | Creadas | Creados si no existen |
| `docker compose up -d` | Igual que `up`, en modo detached (segundo plano) | Creados e iniciados | Creadas | Creados si no existen |
| `docker compose up --build` | Reconstruye imágenes antes de levantar | Recreados | Creadas | Sin cambio |
| `docker compose ps` | Lista servicios y su estado actual | Sin cambio | Sin cambio | Sin cambio |
| `docker compose stop` | Detiene los contenedores sin eliminarlos | Detenidos | Sin cambio | Sin cambio |
| `docker compose start` | Inicia contenedores previamente detenidos | Iniciados | Sin cambio | Sin cambio |
| `docker compose restart` | Detiene e inicia los contenedores | Reiniciados | Sin cambio | Sin cambio |
| `docker compose down` | Detiene y elimina contenedores y redes | Eliminados | Eliminadas | Sin cambio |
| `docker compose down -v` | Igual que `down`, además elimina volúmenes nombrados | Eliminados | Eliminadas | Eliminados |

> [!WARNING]
> `docker compose restart` no vuelve a leer el archivo `docker-compose.yml` ni reconstruye las imágenes. Si modificó el archivo Compose o el `Dockerfile`, debe ejecutar `docker compose up --build` o `docker compose up -d` para aplicar los cambios.

### Diferencia entre `stop` y `down`

`stop` preserva los contenedores en estado detenido. Estos pueden reiniciarse con `start` sin necesidad de recrearlos. En cambio, `down` elimina los contenedores y las redes del proyecto. El siguiente `up` los vuelve a crear desde cero.

En la práctica, la secuencia `down` seguida de `up` es la más habitual cuando se realizan cambios en la configuración o en las imágenes. Por su parte, `stop` y `start` resultan útiles cuando se quiere pausar temporalmente la aplicación sin eliminar el estado de los contenedores.

## Inspección del estado de los servicios

### Lectura de `docker compose ps`

El comando `docker compose ps` muestra el estado de cada servicio del proyecto:

```bash
$ docker compose ps
NAME                  SERVICE   STATUS          PORTS
proyecto-web-1        web       running         0.0.0.0:80->80/tcp
proyecto-api-1        api       running         
proyecto-db-1         db        running (healthy)
```

Los campos relevantes son:

| Campo | Significado |
|-------|-------------|
| `NAME` | Nombre del contenedor, derivado del proyecto y servicio |
| `SERVICE` | Nombre del servicio en `docker-compose.yml` |
| `STATUS` | Estado actual del contenedor |
| `PORTS` | Puertos publicados hacia el host |

### Estados del contenedor

| Estado | Significado |
|--------|-------------|
| `created` | El contenedor fue creado pero no se ha iniciado |
| `running` | El contenedor está en ejecución |
| `running (healthy)` | En ejecución y el healthcheck reporta éxito |
| `running (unhealthy)` | En ejecución pero el healthcheck falla |
| `restarting` | El contenedor se está reiniciando (posible bucle) |
| `exited (0)` | Finalizó exitosamente |
| `exited (1)` | Finalizó con error |
| `exited (137)` | Fue terminado forzadamente (SIGKILL) |

> [!NOTE]
> Un contenedor en estado `running` no garantiza que el servicio interno esté operativo. PostgreSQL puede estar en estado `running` y seguir ejecutando su proceso de inicialización. Del mismo modo, una aplicación web puede estar en estado `running` y aun así lanzar excepciones en cada petición.

### Códigos de salida frecuentes

| Código | Causa habitual |
|--------|----------------|
| `0` | El proceso finalizó normalmente |
| `1` | Error general de la aplicación (excepción no capturada, fallo de configuración) |
| `2` | Uso incorrecto de un comando del shell |
| `126` | Permiso denegado al intentar ejecutar el comando |
| `127` | Comando no encontrado dentro del contenedor |
| `137` | El contenedor fue terminado con SIGKILL (por Docker, por el OOM killer o manualmente) |
| `143` | El contenedor recibió SIGTERM (detención solicitada) |

## Logs y análisis inicial

### Consulta de logs

Los logs de los contenedores son la primera fuente de información cuando algo no funciona como se espera:

```bash
# Logs de todos los servicios
$ docker compose logs

# Logs de un servicio específico
$ docker compose logs api

# Últimas 50 líneas de un servicio
$ docker compose logs --tail 50 api

# Seguimiento en tiempo real
$ docker compose logs -f api

# Seguimiento de todos los servicios con timestamps
$ docker compose logs -f -t
```

### Cómo leer logs sin perder contexto

Cuando se inspeccionan simultáneamente los logs de varios servicios, cada línea aparece precedida por el nombre del servicio. Esto permite correlacionar eventos entre ellos:

```plaintext
db-1   | 2026-03-26 14:00:01 LOG:  database system is ready to accept connections
api-1  | 2026-03-26 14:00:02 Connecting to database...
api-1  | 2026-03-26 14:00:02 Connection established
web-1  | 2026-03-26 14:00:03 Upstream api:5000 is reachable
```

En un escenario de error, la secuencia temporal de los logs permite determinar qué servicio falló primero y cuál fue la consecuencia en los demás:

```plaintext
db-1   | 2026-03-26 14:00:01 LOG:  database system is ready to accept connections
api-1  | 2026-03-26 14:00:02 Connecting to database at localhost:5432...
api-1  | 2026-03-26 14:00:02 psycopg2.OperationalError: connection refused
api-1  | 2026-03-26 14:00:02 Exiting with code 1
web-1  | 2026-03-26 14:00:03 connect() failed: Connection refused (api:5000)
```

En este caso, el log de `api` revela que usa `localhost:5432` para la base de datos (incorrecto) en lugar del nombre de servicio `db:5432`. El error en `web` es una consecuencia del fallo de `api`, no la causa raíz.

### Patrones frecuentes de error en logs

| Patrón en el log                  | Causa probable                                                                                                           |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `Connection refused`              | El servicio de destino no está escuchando en el puerto esperado o el hostname es incorrecto                              |
| `Name does not resolve`           | El nombre del servicio no existe en la red Docker, ya sea por un error tipográfico o porque el servicio está en otra red |
| `password authentication failed`  | Las credenciales no coinciden entre la aplicación y la base de datos                                                     |
| `relation "tabla" does not exist` | La base de datos no ha sido inicializada o la conexión se realizó sobre una base incorrecta                              |
| `address already in use`          | El puerto ya está en uso, ya sea dentro del contenedor o en el host                                                      |
| `permission denied`               | El proceso no tiene permisos para acceder a un archivo o directorio                                                      |
| `no such file or directory`       | Un archivo esperado no existe en la ruta indicada debido a una ruta de montaje incorrecta                                |

> [!TIP]
> Cuando enfrente un error, busque la **primera** línea que lo reporta en los logs, no la última. Los errores en cascada generan múltiples mensajes, pero la causa raíz suele aparecer en el primer fallo.


## Ejecución de comandos dentro de un contenedor

### `docker compose exec`

El comando `exec` permite ejecutar un proceso dentro de un contenedor que **ya está en ejecución**:

```bash
# Abrir un shell interactivo
$ docker compose exec api sh

# Ejecutar un comando específico
$ docker compose exec db psql -U app -d midb

# Verificar variables de entorno
$ docker compose exec api env | grep DATABASE_URL

# Verificar conectividad de red
$ docker compose exec api python -c "import socket; print(socket.gethostbyname('db'))"

# Verificar si un puerto responde
$ docker compose exec api python -c "
import socket
s = socket.socket()
s.settimeout(3)
try:
    s.connect(('db', 5432))
    print('Puerto 5432 en db: accesible')
except Exception as e:
    print(f'Puerto 5432 en db: {e}')
finally:
    s.close()
"
```

### Diferencia entre `exec` y `run`

| Comando | Contenedor | Uso |
|---------|------------|-----|
| `docker compose exec` | Entra a un contenedor **ya en ejecución** | Inspección, diagnóstico, comandos administrativos |
| `docker compose run` | Crea un **nuevo contenedor** a partir de un servicio | Ejecutar tareas únicas (migraciones, scripts, tests) |

```bash
# exec: entra al contenedor db existente
$ docker compose exec db psql -U app -d midb

# run: crea un nuevo contenedor del servicio api para ejecutar un script
$ docker compose run --rm api python migrate.py
```
> [!NOTE]
> `docker compose run` crea un contenedor adicional y no reemplaza al que está en ejecución. Use `--rm` para que se elimine automáticamente al finalizar.

### Valor operativo de entrar a un contenedor

Ingresar a un contenedor permite verificar hipótesis de manera directa:

* ¿La variable `DATABASE_URL` tiene el valor esperado? → `env | grep DATABASE_URL`
* ¿El archivo de configuración está montado correctamente? → `cat /etc/nginx/conf.d/default.conf`
* ¿El contenedor puede resolver el nombre del servicio de base de datos? → `getent hosts db`
* ¿La base de datos acepta conexiones? → intentar conectarse con el cliente correspondiente
* ¿El directorio de datos tiene los permisos correctos? → `ls -la /var/lib/postgresql/data`

Esta inspección directa es más confiable que inferir el estado de la aplicación desde fuera del contenedor.

## Dependencias entre servicios

### El problema de la disponibilidad

Cuando una aplicación depende de una base de datos, existe un intervalo entre el momento en que el contenedor de la base de datos se inicia y el momento en que el motor queda listo para aceptar conexiones. Si la aplicación intenta conectarse durante ese intervalo, la conexión será rechazada.

### `depends_on` y sus límites

La directiva `depends_on` controla el **orden de arranque** de los contenedores, pero no verifica la **disponibilidad** del servicio interno:

```yaml
services:
  api:
    depends_on:
      - db    # el contenedor db se inicia antes que api
```

Con esta configuración, Docker garantiza que el contenedor de `db` se haya iniciado antes de crear el contenedor de `api`. Sin embargo, PostgreSQL puede requerir varios segundos para completar su inicialización. Si `api` intenta conectarse durante ese intervalo, la conexión fallará.

### Healthchecks como mejora progresiva

Para resolver este problema, Compose admite la forma extendida de `depends_on` con verificación de salud:

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 5
```

Con esta configuración, Compose espera a que el *healthcheck* de `db` sea exitoso antes de iniciar `api`. El comando `pg_isready` verifica que PostgreSQL esté aceptando conexiones y no solo que el contenedor exista.

> [!TIP]
> Si no implementa *healthchecks*, considere al menos incorporar lógica de reintentos en su aplicación. Un bucle simple que intente conectarse a la base de datos con una espera entre intentos es más robusto que asumir disponibilidad inmediata.

### Ejemplo de falla por tiempo de inicialización

```plaintext
db-1   | PostgreSQL init process complete; ready for start up.
db-1   | 2026-03-26 LOG:  starting PostgreSQL 16.2
api-1  | Attempting database connection...
api-1  | psycopg2.OperationalError: could not connect to server: Connection refused
api-1  | Exiting with code 1
db-1   | 2026-03-26 LOG:  database system is ready to accept connections
```

Observe la secuencia: `api` intenta conectarse antes de que PostgreSQL termine de iniciar. Con healthcheck, `api` no se habría iniciado hasta después de la línea `database system is ready`.

## Ejemplo práctico guiado

Esta sección presenta un ejemplo completo con tres servicios, simula fallos realistas y demuestra el proceso de diagnóstico.

### Estructura del proyecto

```plaintext
ops-demo/
├── api/
│   ├── main.py
│   └── requirements.txt
├── nginx/
│   └── default.conf
├── Dockerfile.api
├── docker-compose.yml
└── .env
```

### Archivos del proyecto

#### `api/requirements.txt`

```plaintext
flask==3.1.*
psycopg2-binary==2.9.*
```

#### `api/main.py`

```python
import os
import psycopg2
from flask import Flask, jsonify

app = Flask(__name__)

def get_connection():
    return psycopg2.connect(os.environ["DATABASE_URL"])

def init_db():
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS items (
            id SERIAL PRIMARY KEY,
            nombre VARCHAR(255) DEFAULT 'item',
            creado TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    cur.close()
    conn.close()

@app.route("/api/health")
def health():
    return jsonify({"status": "ok"})

@app.route("/api/items")
def listar():
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("SELECT id, nombre, creado FROM items ORDER BY id DESC LIMIT 10")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify([{"id": r[0], "nombre": r[1], "creado": str(r[2])} for r in rows])

@app.route("/api/items", methods=["POST"])
def crear():
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("INSERT INTO items DEFAULT VALUES RETURNING id")
    new_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"id": new_id}), 201

if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=5000)
```

#### `Dockerfile.api`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY api/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY api/ .

EXPOSE 5000

CMD ["python", "main.py"]
```

#### `nginx/default.conf`

```nginx
server {
    listen 80;

    location / {
        return 200 'Servidor operativo\n';
        add_header Content-Type text/plain;
    }

    location /api/ {
        proxy_pass http://api:5000;
        proxy_set_header Host $host;
    }
}
```

#### `.env`

```dotenv
POSTGRES_USER=app
POSTGRES_PASSWORD=secreto123
POSTGRES_DB=opsdb
```

#### `docker-compose.yml`

```yaml
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
```

### Operación normal

```bash
# Levantar
$ docker compose up -d

# Verificar estado
$ docker compose ps
NAME              SERVICE   STATUS            PORTS
ops-demo-web-1    web       running           0.0.0.0:80->80/tcp
ops-demo-api-1    api       running           
ops-demo-db-1     db        running (healthy)

# Probar
$ curl http://localhost/api/health
{"status":"ok"}

$ curl -X POST http://localhost/api/items
{"id":1}

$ curl http://localhost/api/items
[{"id":1,"nombre":"item","creado":"2026-03-26 ..."}]
```

### Escenario de fallo 1: puerto del host ocupado

Suponga que otro proceso ya ocupa el puerto 80 en el host.

```bash
$ docker compose up -d
...
Error response from daemon: driver failed programming external connectivity:
Bind for 0.0.0.0:80 failed: port is already allocated
```

**Diagnóstico**:

```bash
# Identificar qué proceso ocupa el puerto
$ lsof -i :80
# o
$ ss -tlnp | grep :80
```

**Corrección**: detenga el proceso que ocupa el puerto, o modifique `docker-compose.yml`:

```yaml
ports:
  - "8080:80"    # usar puerto 8080 en el host
```

### Escenario de fallo 2: variable de entorno incorrecta

Suponga que `.env` contiene una contraseña que no coincide con la que PostgreSQL espera, o que `DATABASE_URL` referencia `localhost` en lugar de `db`.

```bash
$ docker compose ps
NAME              SERVICE   STATUS      PORTS
ops-demo-web-1    web       running     0.0.0.0:80->80/tcp
ops-demo-api-1    api       exited (1)
ops-demo-db-1     db        running (healthy)
```

**Diagnóstico**:

```bash
# Paso 1: revisar logs de api
$ docker compose logs api
api-1  | psycopg2.OperationalError: connection to server at "localhost" (127.0.0.1),
api-1  |   port 5432 failed: Connection refused

# Paso 2: verificar la variable de entorno
$ docker compose config | grep DATABASE_URL
      - DATABASE_URL=postgresql://app:secreto123@localhost:5432/opsdb
```

El log revela que la cadena de conexión usa `localhost` en lugar de `db`.

**Corrección**: corregir el valor de `DATABASE_URL` en `docker-compose.yml` o `.env` para que use `@db:5432`.

```bash
# Después de corregir, recrear
$ docker compose up -d
```

### Escenario de fallo 3: contenedor en bucle de reinicio

Suponga que `api` tiene `restart: unless-stopped` y falla al iniciar. El contenedor se reinicia repetidamente.

```bash
$ docker compose ps
NAME              SERVICE   STATUS       PORTS
ops-demo-api-1    api       restarting
```

**Diagnóstico**:

```bash
# Revisar logs para ver el error que se repite
$ docker compose logs --tail 20 api
api-1  | KeyError: 'DATABASE_URL'
api-1  | KeyError: 'DATABASE_URL'
api-1  | KeyError: 'DATABASE_URL'
```

La aplicación espera la variable `DATABASE_URL`, que no está definida.

**Corrección**: verificar que la sección `environment` del servicio `api` incluya la variable, y que `.env` contenga los valores necesarios. Después de corregir, recrear el servicio:

```bash
$ docker compose up -d api
```

### Escenario de fallo 4: volumen en ruta incorrecta

Suponga que el volumen de PostgreSQL se monta en `/var/lib/postgres/data` (sin `ql`):

```yaml
volumes:
  - pgdata:/var/lib/postgres/data    # ruta incorrecta
```

**Síntoma**: PostgreSQL inicia con una base de datos vacía en cada recreación, y los datos anteriores no persisten.

**Diagnóstico**:

```bash
# Verificar la ruta de montaje
$ docker compose exec db mount | grep pgdata
# o
$ docker inspect ops-demo-db-1 --format '{{json .Mounts}}' | python -m json.tool
```

Si la ruta montada no coincide con `/var/lib/postgresql/data`, que es la ruta estándar de PostgreSQL, los datos se escriben en la capa de escritura del contenedor y se pierden al recrearlo.

**Corrección**. Ajustar la ruta en `docker-compose.yml` a `/var/lib/postgresql/data`.

> [!CAUTION]
> Los errores de ruta en volúmenes no generan mensajes al arrancar. El contenedor funciona con normalidad, pero los datos no se persisten en la ubicación esperada. Este tipo de fallo suele detectarse solo al recrear el contenedor y comprobar que los datos desaparecieron.

## Troubleshooting sistemático

### Metodología paso a paso

Cuando una aplicación multicontenedor no funciona como se espera, seguir un proceso ordenado reduce el tiempo de diagnóstico y evita modificaciones innecesarias:

1. **Verificar si los contenedores levantaron**
   ```bash
   $ docker compose ps
   ```
   Identifique qué servicios están en estado `running`, cuáles en `exited` y cuáles en `restarting`.

2. **Revisar logs del servicio que falla**
   ```bash
   $ docker compose logs <servicio>
   ```
   Busque la primera línea de error. Los errores en cascada comienzan en un punto específico.

3. **Validar configuración resuelta**
   ```bash
   $ docker compose config
   ```
   Verifique que las variables de entorno se interpolaron correctamente y que las rutas de volúmenes son las esperadas.

4. **Verificar variables de entorno dentro del contenedor**
   ```bash
   $ docker compose exec <servicio> env
   ```

5. **Verificar conectividad de red**
   ```bash
   $ docker compose exec <servicio> getent hosts <servicio-destino>
   ```
   Si la resolución falla, los servicios probablemente no comparten red.

6. **Verificar disponibilidad del puerto destino**
   ```bash
   $ docker compose exec <servicio> python -c "
   import socket; s = socket.socket(); s.settimeout(3)
   try: s.connect(('<destino>', <puerto>)); print('OK')
   except Exception as e: print(e)
   finally: s.close()"
   ```

7. **Verificar volúmenes y archivos montados**
   ```bash
   $ docker compose exec <servicio> ls -la <ruta>
   ```

8. **Aislar el problema**: si el paso anterior no revela la causa, detenga servicios no relacionados y pruebe los servicios de forma individual para determinar cuál falla.

> [!IMPORTANT]
> Evite cambiar múltiples variables al mismo tiempo durante el diagnóstico. Modifique un elemento, recree y verifique. Si el problema no se resuelve, revierta el cambio antes de intentar otro. Modificar varias cosas simultáneamente impide determinar cuál fue la corrección efectiva.

## Errores comunes

La siguiente tabla resume los errores más frecuentes y las acciones de diagnóstico recomendadas:

| Síntoma                                                | Causa probable                                                                                                  | Acción de diagnóstico                                               |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Puerto ya asignado al iniciar                          | Otro proceso está usando el puerto del host                                                                     | `lsof -i :<puerto>` o `ss -tlnp \| grep :<puerto>`                  |
| Contenedor en bucle de reinicio                        | Fallo de la aplicación durante el arranque, como una variable ausente o un error de código                      | `docker compose logs <servicio>`                                    |
| Contenedor que finaliza inmediatamente (`exited 0`)    | El proceso principal terminó porque la imagen no define un proceso persistente                                  | Verificar `CMD` en el `Dockerfile` o el comando definido en Compose |
| Contenedor que finaliza con `exited 1`                 | Error de la aplicación, como una excepción o un fallo de configuración                                          | `docker compose logs <servicio>`                                    |
| Contenedor que finaliza con `exited 127`               | El comando no existe dentro del contenedor                                                                      | Verificar `CMD`, `ENTRYPOINT` y la imagen base                      |
| `Connection refused` entre servicios                   | Hostname incorrecto, como `localhost` en lugar del nombre del servicio, o servicio de destino aún no disponible | Verificar la cadena de conexión y los logs del servicio de destino  |
| `Name does not resolve`                                | Los servicios están en redes distintas o el nombre está mal escrito                                             | `docker network inspect` y revisar la sección `networks`            |
| Credenciales rechazadas por la base de datos           | Las variables de entorno no coinciden entre la aplicación y la base de datos                                    | `docker compose exec <servicio> env` en ambos servicios             |
| Datos perdidos al recrear                              | El volumen no fue declarado o la ruta de montaje es incorrecta                                                  | `docker volume ls` y `docker inspect <contenedor>`                  |
| Error de parseo al iniciar                             | Sintaxis YAML incorrecta                                                                                        | `docker compose config` para identificar el error de parseo         |
| Servicio en estado `running`, pero funcionalmente roto | El fallo se encuentra en la aplicación y no en el contenedor                                                    | `curl` al endpoint y revisión de los logs del servicio              |

### Servicio correcto a nivel de contenedor, pero incorrecto a nivel funcional

Este caso merece atención especial. Un contenedor puede estar en estado `running` y el servicio interno puede estar aceptando conexiones, pero la aplicación aun así puede responder con errores:

```bash
$ docker compose ps
NAME              SERVICE   STATUS    PORTS
ops-demo-api-1    api       running

$ curl http://localhost/api/items
{"error": "relation \"items\" does not exist"}
```

El contenedor está operativo, pero la base de datos no fue inicializada y la tabla no existe. Este tipo de fallo no se detecta con `docker compose ps` ni con verificaciones de conectividad. Se detecta **probando la funcionalidad** de la aplicación y revisando los logs del servicio.

**Diagnóstico**. Revisar si la lógica de inicialización de la base de datos se ejecutó correctamente, si la aplicación se conectó a la base de datos adecuada y si el volumen contiene datos de una inicialización previa incompatible.

## Buenas prácticas operativas

* **Diagnosticar primero, modificar después**. Antes de cambiar el `docker-compose.yml`, el código o las variables de entorno, identifique la causa probable del error. Modificar sin diagnóstico introduce variables adicionales que dificultan el análisis.
* **Empezar por lo observable**. `docker compose ps` y `docker compose logs` son los dos primeros comandos ante cualquier problema. Proporcionan gran parte de la información necesaria para orientar el diagnóstico.
* **Revisar estado, logs y configuración antes de reconstruir todo**. `docker compose up --build` y `docker compose down -v` son acciones costosas. Úselas cuando el diagnóstico las justifique y no como primer recurso.
* **Documentar los comandos operativos mínimos**. Incluya en el `README` del proyecto los comandos para levantar, detener, ver logs y acceder a la base de datos. Esto reduce el tiempo de incorporación y evita errores por desconocimiento.
* **Introducir healthchecks cuando el caso lo requiera**. No todos los servicios necesitan *healthcheck*. Priorícelo en servicios de los que dependen otros y que tienen un proceso de inicialización no trivial, como bases de datos o servicios de mensajería.
* **Evitar depender de supuestos sobre el orden de arranque**. `depends_on` sin condición no garantiza disponibilidad. Si la aplicación no tolera que la base de datos esté temporalmente inaccesible, use *healthchecks* o reintentos.
* **Mantener archivos Compose legibles y consistentes**. Use nombres de servicio descriptivos, variables interpoladas desde `.env` y puertos y volúmenes claramente documentados. Un archivo desordenado dificulta el diagnóstico cuando algo falla.
* **Tratar los errores como señales observables**. Un error no es un evento aislado que se resuelve reiniciando. Es una señal que indica una discrepancia entre lo esperado y lo real. Leer esa señal con atención es la base de un diagnóstico efectivo.

## Preguntas de autoevaluación

1. Después de modificar el `Dockerfile` de un servicio, un desarrollador ejecuta `docker compose restart api`. Los cambios no se reflejan. ¿Por qué, y qué comando debería usar?

2. El servicio `web` muestra `502 Bad Gateway` al acceder desde el navegador. El servicio `api` está en estado `running`. ¿Qué hipótesis consideraría y cómo las verificaría?

3. ¿Por qué `depends_on` sin `condition: service_healthy` no garantiza que la base de datos esté lista para recibir conexiones? ¿Qué alternativas existen?

4. Describa una estrategia de diagnóstico de tres pasos que aplicaría cuando una aplicación multicontenedor no responde como se espera, sin saber de antemano cuál es el problema.

## Referencias

- Docker Inc. *Docker Compose CLI reference*. https://docs.docker.com/reference/cli/docker/compose/
- Docker Inc. *View container logs*. https://docs.docker.com/engine/logging/
- Docker Inc. *Compose file reference: depends_on*. https://docs.docker.com/reference/compose-file/services/#depends_on
- Docker Inc. *Compose file reference: healthcheck*. https://docs.docker.com/reference/compose-file/services/#healthcheck
- Docker Inc. *Networking in Compose*. https://docs.docker.com/compose/how-tos/networking/
- Nickoloff, J., & Kuenzli, S. *Docker in Action* (2nd ed.). Manning Publications.
- Stoneman, E. *Learn Docker in a Month of Lunches*. Manning Publications.
- McKendrick, R. *Mastering Docker* (4th ed.). Packt Publishing.
