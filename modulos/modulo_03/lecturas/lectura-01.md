![Banner del proyecto](/modulos/assets/Banner.png)

# Lectura 01 - Docker Compose y aplicaciones multicontenedor

## Prerrequisitos
Esta lectura asume que el estudiante:

- comprende los conceptos de imagen y contenedor en Docker
- ha construido imágenes con `Dockerfile`
- ha ejecutado contenedores con `docker run`, incluyendo opciones como `-d`, `-p`, `-v`, `-e` y `--name`
- tiene familiaridad básica con redes y volúmenes en Docker
- maneja la terminal en un entorno Linux o equivalente

## Introducción
A esta altura del curso, el trabajo con contenedores se ha abordado principalmente desde una lógica individual. Cada contenedor ha ejecutado un único proceso a partir de una imagen específica y mediante instrucciones directas como `docker run`. Este enfoque es adecuado para ejercicios introductorios, pruebas controladas y servicios aislados. Sin embargo, resulta insuficiente para representar la estructura y las exigencias de una aplicación en un ambiente de desarrollo real o de producción.

En escenarios reales, incluso una aplicación de complejidad moderada está compuesta por múltiples servicios que interactúan entre sí. Entre ellos pueden encontrarse un servidor web, una base de datos, un sistema de caché o un servicio de colas. Cada componente posee su propio ciclo de vida, requiere configuraciones particulares y mantiene dependencias que deben ser gestionadas de forma coherente con el resto del sistema.

En este contexto, esta lectura introduce Docker Compose como una herramienta orientada a definir, desplegar y operar aplicaciones compuestas por varios servicios de manera declarativa y reproducible, con especial relevancia en entornos de desarrollo.

## Motivación: los límites de `docker run`

### Una aplicación rara vez es un solo contenedor

Considere una aplicación web que almacena datos en una base de datos relacional. En un entorno sin contenedores, ambos procesos coexisten en la misma máquina. Al migrar a Docker, cada componente debería ejecutarse en su propio contenedor, siguiendo el principio de **un proceso principal por contenedor**:

- Un contenedor para la aplicación web
- Un contenedor para la base de datos

A esto podría agregarse un tercer contenedor para una herramienta de administración de la base de datos, un servicio de caché o cualquier otro componente auxiliar.

### Gestión manual con `docker run`

Cuando la aplicación involucra dos o tres servicios, levantarlos manualmente implica ejecutar múltiples comandos, cada uno con sus propias opciones:

```bash
# Crear la red
$ docker network create mi-red

# Levantar la base de datos
$ docker run -d \
  --name postgres-db \
  --network mi-red \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD=secreto123 \
  -e POSTGRES_DB=miapp \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Levantar la aplicación web
$ docker run -d \
  --name webapp \
  --network mi-red \
  -p 8000:8000 \
  -e DATABASE_URL=postgresql://app:secreto123@postgres-db:5432/miapp \
  mi-webapp:latest

# Levantar Adminer para inspección
$ docker run -d \
  --name adminer \
  --network mi-red \
  -p 8080:8080 \
  adminer
```

Este enfoque presenta varias limitaciones:

- **No es reproducible**: los comandos residen en la memoria del desarrollador, en notas sueltas o en scripts frágiles
- **Es propenso a errores**: un flag olvidado, un nombre de red mal escrito o una variable de entorno ausente pueden causar fallos difíciles de diagnosticar
- **No documenta la arquitectura**: al leer los comandos de forma aislada, no se percibe la relación entre los servicios
- **Requiere gestión manual del orden**: hay que recordar qué servicio debe levantarse primero
- **Detener y limpiar requiere múltiples pasos**: cada contenedor, red y volumen debe gestionarse por separado

### El valor del enfoque declarativo

Docker Compose permite superar estas limitaciones al ofrecer un mecanismo para expresar la definición completa de la aplicación en un único archivo de configuración, legible y susceptible de versionamiento. En este archivo se describen de manera explícita los servicios, las redes, los volúmenes y los parámetros de configuración necesarios para su operación.

| Aspecto | `docker run` manual | Docker Compose |
|---------|---------------------|----------------|
| Definición | Comandos imperativos | Archivo declarativo (`docker-compose.yml`) |
| Reproducibilidad | Depende de scripts o documentación externa | El archivo es la documentación ejecutable |
| Gestión de redes | Creación y asignación manual | Red por defecto creada automáticamente |
| Variables de entorno | Flags `-e` en cada comando | Centralizadas en el archivo o en `.env` |
| Ciclo de vida | Cada contenedor se gestiona individualmente | Un comando opera todos los servicios |
| Versionamiento | Difícil de rastrear en control de versiones | El archivo `docker-compose.yml` se versiona con el proyecto |

> [!IMPORTANT]
> Docker Compose no reemplaza a Docker ni introduce un motor diferente. Compose es una capa declarativa que **traduce** la definición del archivo en llamadas al Docker Engine.

## Qué es Docker Compose

Docker Compose es una herramienta que permite **definir y ejecutar aplicaciones multicontenedor** mediante un archivo de configuración en formato YAML.

### Propósito

El propósito central de Compose es **modelar una aplicación como un conjunto de servicios relacionados**, donde cada servicio corresponde a uno o más contenedores. Compose gestiona la creación de redes, volúmenes y contenedores según la definición provista, y ofrece un conjunto de comandos para operar la aplicación como una unidad.

### Relación con Docker Engine

Compose no es un motor de contenedores independiente. Funciona como un **cliente** que se comunica con el Docker Engine existente en la máquina. Cuando se ejecuta `docker compose up`, Compose lee el archivo de configuración, interpreta la definición y emite las instrucciones necesarias al Engine para crear redes, volúmenes y contenedores.

```mermaid
flowchart TB
    A["docker-compose.yml"] -->|"docker compose up"| B["Docker Compose CLI"]
    B --> C["Docker Engine"]
    C --> D["Red: mi-app_default"]
    C --> E["Contenedor: web"]
    C --> F["Contenedor: db"]
    C --> G["Contenedor: adminer"]
    C --> H["Volumen: pgdata"]
    
    E -.- D
    F -.- D
    G -.- D
    F -.- H
    
    style A fill:#2496ED,color:#fff
    style B fill:#066da5,color:#fff
    style C fill:#066da5,color:#fff
    style D fill:#28a745,color:#fff
    style E fill:#ffc107,color:#000
    style F fill:#ffc107,color:#000
    style G fill:#ffc107,color:#000
    style H fill:#6f42c1,color:#fff
```

> [!NOTE]
> En versiones recientes de Docker Desktop y Docker Engine, Compose viene integrado como un subcomando: `docker compose` (sin guion). La versión anterior, `docker-compose` (con guion), es un binario independiente que se considera legado. Esta lectura utiliza la sintaxis moderna `docker compose`.

### La idea de "aplicación"

En el modelo de Compose, una **aplicación** (o *project*) es el conjunto de todos los servicios definidos en un archivo `docker-compose.yml` que comparten un contexto operativo común: la misma red por defecto, el mismo ciclo de vida y el mismo espacio de nombres. El nombre del proyecto se deriva, por defecto, del nombre del directorio que contiene el archivo Compose.

## Conceptos fundamentales

Antes de examinar la estructura de un archivo Compose, es necesario establecer los conceptos que articula.

### Services

Un **servicio** representa una unidad lógica de la aplicación dentro del archivo `docker-compose.yml`. Su definición establece cómo debe ejecutarse uno o varios contenedores a partir de una misma configuración, por ejemplo imagen, comando, variables de entorno, redes, volúmenes y políticas de reinicio. En escenarios introductorios, suele asumirse que un servicio corresponde a un único contenedor. Sin embargo, conceptualmente un servicio puede escalar a múltiples instancias homogéneas cuando la arquitectura lo requiere.

### Networks

Docker Compose crea, por defecto, una **red de aplicación** para el proyecto. Los servicios se conectan a esta red y pueden comunicarse entre sí sin necesidad de exponer sus puertos al host. Esta red recibe un nombre derivado del nombre del proyecto, lo que favorece el aislamiento entre aplicaciones distintas que se ejecutan en el mismo entorno. Además, es posible definir redes adicionales para introducir segmentación, controlar el alcance de la conectividad o separar dominios funcionales dentro de la solución.

### Volumes

Los **volúmenes** permiten desacoplar los datos del ciclo de vida de los contenedores. Esta capacidad es especialmente importante en componentes con estado, como bases de datos, colas persistentes o servicios que almacenan archivos. Si un contenedor es eliminado y posteriormente recreado, el volumen permite conservar la información previamente almacenada. En Compose, los volúmenes también facilitan la reutilización y administración explícita del almacenamiento persistente entre servicios.

### Variables de entorno

Compose admite varias estrategias para gestionar configuración mediante **variables de entorno**, pero conviene distinguir sus propósitos. La clave `environment` permite definir variables que serán inyectadas al contenedor en tiempo de ejecución. La clave `env_file` permite cargar variables desde uno o varios archivos externos. Por su parte, el archivo `.env` se utiliza principalmente para la **interpolación de variables dentro del propio archivo Compose**, lo que permite parametrizar la configuración sin duplicar valores sensibles o dependientes del entorno.

> [!NOTE]
> Aunque en la práctica `.env` y `env_file` suelen mencionarse conjuntamente, no cumplen exactamente la misma función. El primero participa en la resolución de variables del archivo Compose, mientras que el segundo inyecta variables al entorno del contenedor.

### Puertos publicados

La directiva `ports` permite **publicar** puertos de un contenedor hacia el host, de modo que el servicio pueda ser consumido desde fuera de la red interna de Compose. La forma más habitual es `"PUERTO_HOST:PUERTO_CONTENEDOR"`. Esta publicación resulta necesaria cuando se desea acceder al servicio desde el sistema anfitrión, un navegador, una herramienta externa o cualquier cliente ubicado fuera de la red del proyecto.

> [!TIP]
> La comunicación entre servicios dentro de la misma red de Compose no requiere la directiva `ports`. Un servicio puede conectarse a otro por red interna aunque ninguno publique puertos hacia el host.

### Resolución DNS interna

Dentro de la red creada por Compose, cada servicio puede ser localizado por los demás mediante su **nombre de servicio**, el cual funciona como identificador DNS interno. Por ejemplo, si un servicio se define con el nombre `db`, otro servicio de la misma red puede establecer conexión usando `db` como hostname. Este mecanismo elimina la necesidad de fijar direcciones IP internas y favorece una configuración más estable, portable y mantenible.

> [!TIP]
> El identificador que normalmente debe utilizarse para la comunicación entre servicios es el **nombre del servicio** definido en la sección `services`, no el valor de `container_name`. Basar la conectividad en el nombre del servicio hace que la configuración sea más robusta y coherente con el modelo de Compose.

### El archivo `compose.yml`

El archivo `compose.yml` (también aceptado como `docker-compose.yml`) es un documento YAML que describe la aplicación. Su ubicación habitual es la raíz del proyecto. Compose lo busca automáticamente en el directorio de trabajo actual al ejecutar cualquier comando.

## Anatomía de un archivo Compose

Esta sección presenta la estructura de un archivo `docker-compose.yml` a partir de un ejemplo compuesto por tres servicios: una aplicación web desarrollada en Python con Flask, una base de datos PostgreSQL y Adminer como herramienta de administración e inspección de la base de datos.

### Estructura del proyecto

```plaintext
mi-proyecto/
├── app/
│   ├── main.py
│   └── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env
```

### Archivo `docker-compose.yml`

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
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
```

### Explicación por bloques

#### Bloque `services`

Define los componentes de la aplicación. Cada clave bajo `services` es un nombre de servicio.

| Clave | Descripción |
|-------|-------------|
| `build` | Indica que la imagen se construye a partir de un `Dockerfile` en la ruta especificada (`.` = directorio actual) |
| `image` | Especifica una imagen preexistente del registry (p. ej., `postgres:16`) |
| `ports` | Mapea puertos entre el host y el contenedor (`"HOST:CONTENEDOR"`) |
| `environment` | Define variables de entorno para el contenedor |
| `volumes` | Monta volúmenes en el contenedor |
| `depends_on` | Declara que un servicio depende de otro (ver nota abajo) |
| `restart` | Política de reinicio del contenedor (p. ej., `unless-stopped`) |

#### Diferencia entre `build` e `image`

La clave `build` indica a Compose **cómo construir una imagen a partir del código fuente**. En su forma más simple, `build: .` señala que debe utilizarse el directorio actual como contexto de construcción y que allí se buscará el `Dockerfile`. Esta opción se emplea típicamente en servicios desarrollados como parte del propio proyecto.

La clave `image`, en cambio, indica **qué imagen debe ejecutar el servicio**. Esta imagen puede provenir de un registry, como en `postgres:16`, o puede corresponder al nombre y etiqueta asignados a una imagen construida localmente.

Ambas claves pueden coexistir en un mismo servicio, pero su interacción no debe simplificarse en exceso. Cuando se definen `build` e `image` simultáneamente, Compose sigue las reglas establecidas por `pull_policy`. En ausencia de una política explícita, Compose intenta primero obtener la imagen y, si no la encuentra, procede a construirla desde el código fuente. Además, si se construye la imagen y se ha definido `image`, el resultado puede etiquetarse con ese nombre.

> [!NOTE]
> En términos prácticos, `build` describe el **proceso de construcción**, mientras que `image` identifica la **imagen resultante o la imagen a consumir**.

#### Bloque `volumes` en el nivel superior

Al declarar `pgdata:` en el nivel superior, se define un **volumen nombrado** gestionado por Docker. Este volumen existe de manera independiente al contenedor y puede reutilizarse mientras no sea eliminado explícitamente. La referencia `pgdata:/var/lib/postgresql/data` dentro del servicio `db` monta ese volumen en la ruta donde PostgreSQL persiste sus datos.

Este patrón resulta especialmente importante en servicios con estado, ya que desacopla la persistencia de la vida útil del contenedor. En otras palabras, permite recrear el contenedor sin perder automáticamente la información almacenada por la base de datos.

> [!CAUTION]
> Si no se monta un volumen persistente, los datos quedan asociados a la capa escribible del contenedor. En ese escenario, al eliminar y recrear el contenedor, dichos datos dejan de estar disponibles. Por el contrario, un volumen nombrado no se elimina con `docker compose down` salvo que se solicite explícitamente, por ejemplo con la opción `-v`.

#### `depends_on` y sus límites

La directiva `depends_on` permite expresar **dependencias de arranque y apagado** entre servicios. En un caso simple, indica que Compose debe crear o iniciar primero el servicio del cual otro depende. Sin embargo, este mecanismo no implica, por sí solo, que el servicio dependiente esté listo para operar a nivel funcional.

Este matiz es fundamental en componentes como PostgreSQL. Un contenedor puede encontrarse en estado de ejecución y, aun así, la base de datos no estar preparada para aceptar conexiones porque continúa en su proceso de inicialización.

Por esta razón, `depends_on` no debe interpretarse como una garantía de disponibilidad lógica del servicio, sino únicamente como un mecanismo de **ordenamiento del ciclo de vida de los contenedores**. Cuando se requiere esperar a que un servicio esté realmente listo, conviene complementar esta directiva con un `healthcheck` y con condiciones como `service_healthy`.

> [!TIP]
> En aplicaciones multicontenedor, asumir que "contenedor iniciado" equivale a "servicio disponible" suele producir fallos intermitentes difíciles de diagnosticar, especialmente durante el arranque de la aplicación.

Para escenarios donde esta garantía es necesaria, Compose admite la forma extendida de `depends_on` con verificación de salud:

```yaml
services:
  web:
    build: .
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

En esta configuración, el servicio `web` solo se iniciará cuando el `healthcheck` asociado a `db` informe un estado saludable. En este punto, los *healthchecks* se introducen únicamente como un mecanismo puntual para coordinar de forma más segura el arranque entre servicios. Su estudio detallado se abordará más adelante, dado que excede el alcance de esta lectura.

> [!WARNING]
> No conviene asumir que un `depends_on` básico garantiza conectividad efectiva entre servicios. Si la aplicación depende de que la base de datos esté realmente disponible antes de procesar solicitudes, es recomendable complementar esta configuración con `healthchecks` o con lógica de reintentos desde la propia aplicación.

#### Redes implícitas

Observe que el archivo no declara una sección `networks`. En este caso, Compose crea automáticamente una **red por defecto** para el proyecto y conecta a ella todos los servicios definidos en el archivo. Usualmente, esta red recibe un nombre con el patrón `<nombre-del-proyecto>_default`, donde el nombre del proyecto suele derivarse del directorio de trabajo, aunque puede configurarse explícitamente.

Gracias a esta red, los tres servicios pueden comunicarse entre sí utilizando el **nombre del servicio** como identificador dentro de la red interna. Este comportamiento simplifica la conectividad entre componentes y evita la necesidad de gestionar direcciones IP manualmente.

Si la aplicación requiere un mayor nivel de aislamiento o una topología de red más controlada, es posible declarar redes explícitas. Esto resulta útil, por ejemplo, cuando se desea que `adminer` tenga acceso a `db`, pero no necesariamente al servicio `web`. En ese tipo de escenarios, la definición manual de redes permite segmentar la comunicación entre servicios de forma más precisa.

```yaml
services:
  web:
    networks:
      - frontend
      - backend
  db:
    networks:
      - backend
  adminer:
    networks:
      - backend

networks:
  frontend:
  backend:
```

## Ejemplo 

Esta sección presenta un ejemplo. Se construirá una aplicación Flask que registra visitas en una base de datos PostgreSQL, con Adminer disponible para inspeccionar los datos.

### Estructura del proyecto

```plaintext
compose-demo/
├── app/
│   ├── main.py
│   └── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env
```

### Archivos del proyecto

#### `app/requirements.txt`

```plaintext
flask==3.1.*
psycopg2-binary==2.9.*
```

#### `app/main.py`

```python
import os
import datetime
import psycopg2
from flask import Flask

app = Flask(__name__)

def get_connection():
    return psycopg2.connect(os.environ["DATABASE_URL"])

def init_db():
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS visitas (
            id SERIAL PRIMARY KEY,
            fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    cur.close()
    conn.close()

@app.route("/")
def index():
    conn = get_connection()
    cur = conn.cursor()
    cur.execute("INSERT INTO visitas DEFAULT VALUES")
    conn.commit()
    cur.execute("SELECT COUNT(*) FROM visitas")
    count = cur.fetchone()[0]
    cur.close()
    conn.close()
    return f"Visitas registradas: {count}\n"

if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=8000)
```

#### `Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 8000

CMD ["python", "main.py"]
```

#### `.env`

```bash
POSTGRES_USER=app
POSTGRES_PASSWORD=secreto123
POSTGRES_DB=visitasdb
```

> [!CAUTION]
> El archivo `.env` contiene credenciales. **No debe incluirse en control de versiones**. Agréguelo a `.gitignore` en proyectos reales. En esta lectura se incluye con fines didácticos.

#### `docker-compose.yml`

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
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
```

### Levantar la aplicación

Desde el directorio `compose-demo/`, ejecute:

```bash
$ docker compose up -d
```

Al ejecutar este comando, Compose realiza, en términos generales, las siguientes acciones:

1. Lee la definición del proyecto a partir del archivo `docker-compose.yml` y procesa las variables disponibles para su interpolación, incluidas aquellas provenientes de `.env`, cuando corresponda.
2. Construye la imagen del servicio `web` a partir del `Dockerfile`, de acuerdo con la configuración declarada en el bloque `build`.
3. Obtiene las imágenes requeridas para servicios externos, como `postgres:16` y `adminer`, si no se encuentran disponibles localmente.
4. Crea la red por defecto del proyecto, normalmente con un nombre derivado del nombre efectivo del proyecto.
5. Crea los volúmenes declarados, como `pgdata`, en caso de que aún no existan.
6. Crea e inicia los contenedores conforme a la configuración definida y respetando las relaciones de dependencia declaradas.
7. Si la configuración incorpora `healthchecks` junto con condiciones de dependencia apropiadas, puede posponer el inicio de un servicio como `web` hasta que `db` reporte un estado saludable.

La opción `-d` ejecuta los contenedores en modo desacoplado, es decir, en segundo plano.

> [!NOTE]
> El nombre exacto de la red no necesariamente será `compose-demo_default`. Este valor depende del nombre efectivo del proyecto en Compose, que por lo general coincide con el directorio de trabajo, aunque puede definirse explícitamente.> 

### Verificar el estado

```bash
$ docker compose ps
```

La salida deberá mostrar los tres servicios en ejecución, normalmente con estado `running` o `Up`, junto con los puertos que hayan sido publicados hacia el host.

### Probar la aplicación

```bash
$ curl http://localhost:8000
Visitas registradas: 1

$ curl http://localhost:8000
Visitas registradas: 2
```

Para acceder a Adminer, abra `http://localhost:8080` en el navegador. Los datos de conexión son los siguientes:

| Campo | Valor |
|-------|-------|
| Sistema | PostgreSQL |
| Servidor | `db` |
| Usuario | `app` |
| Contraseña | `secreto123` |
| Base de datos | `visitasdb` |

> [!NOTE]
> Observe que, en Adminer, el servidor debe indicarse como `db` y no como `localhost`. Esto se debe a que Adminer se ejecuta dentro de la red interna de Compose y se conecta a PostgreSQL mediante el nombre del servicio definido en el archivo de configuración.

### Inspeccionar logs

```bash
# Logs de todos los servicios
$ docker compose logs

# Logs de un servicio específico
$ docker compose logs web

# Seguimiento en tiempo real
$ docker compose logs -f web
```

### Ejecutar un comando dentro de un servicio

```bash
$ docker compose exec db psql -U app -d visitasdb -c "SELECT * FROM visitas;"
```

Este comando ejecuta `psql` dentro del contenedor correspondiente al servicio `db` que ya se encuentra en ejecución, de forma análoga a lo que ocurre con `docker exec`, aunque utilizando la abstracción definida por Compose.

### Detener la aplicación

```bash
# Detener y eliminar contenedores y la red
$ docker compose down

# Detener, eliminar contenedores, red Y volúmenes
$ docker compose down -v
```

> [!WARNING]
> El flag `-v` en `docker compose down` elimina los volúmenes nombrados declarados en la configuración de Compose, así como los volúmenes anónimos asociados a los contenedores. En este ejemplo, su uso provoca la eliminación de `pgdata` y, en consecuencia, la pérdida permanente de los datos persistidos por PostgreSQL.

> [!WARNING]
> La eliminación de volúmenes mediante `docker compose down -v` es una operación destructiva. Si el volumen contiene datos relevantes, estos no estarán disponibles una vez eliminado.

> [!NOTE]
> Los volúmenes externos no son eliminados por `docker compose down -v`.

Utilice este flag únicamente cuando desee realizar un reinicio completo del estado de la aplicación.

## Conectividad entre servicios

La comunicación entre contenedores en Docker Compose suele ser uno de los aspectos que más dudas genera en las primeras etapas de aprendizaje. Por esta razón, esta sección presenta el modelo de conectividad interna de manera explícita.

### Comunicación por nombre de servicio

Dentro de una red de Compose, cada servicio puede ser localizado mediante el **nombre con el que fue definido** en la sección `services`. Esto significa que, cuando una aplicación necesita conectarse a otro componente de la arquitectura, no debe utilizar direcciones IP internas ni, en la mayoría de los casos, nombres de contenedor fijados manualmente, sino el nombre lógico del servicio.

En el caso de una aplicación Flask que requiere conectarse a PostgreSQL, la cadena de conexión debe utilizar `db` como hostname, siempre que `db` sea el nombre del servicio definido en el archivo `docker-compose.yml`.

> [!IMPORTANT]
> Este mecanismo de resolución funciona entre servicios que comparten una misma red de Compose. En la configuración por defecto, todos los servicios del proyecto se conectan automáticamente a esa red común.

> [!TIP]
> En la comunicación interna entre servicios debe utilizarse el puerto del contenedor expuesto por la aplicación dentro de la red, no el puerto publicado hacia el host. La publicación de puertos resulta necesaria para el acceso externo desde la máquina anfitriona, pero no para la comunicación entre servicios dentro del proyecto.

```
postgresql://app:secreto123@db:5432/visitasdb
```

Docker resuelve `db` internamente a la dirección IP del contenedor correspondiente. Este mecanismo se basa en el **servidor DNS integrado** de Docker para redes definidas por el usuario.

### Acceso interno vs. acceso desde el host

```mermaid
flowchart LR
    subgraph Host["Máquina host"]
        Browser["Navegador"]
        Terminal["Terminal"]
    end
    
    subgraph DockerNet["Red Docker: compose-demo_default"]
        Web["web :8000"]
        DB["db :5432"]
        Adm["adminer :8080"]
    end
    
    Browser -->|"localhost:8000"| Web
    Browser -->|"localhost:8080"| Adm
    Terminal -->|"localhost:8000"| Web
    Web -->|"db:5432"| DB
    Adm -->|"db:5432"| DB
    
    style Host fill:#f5f5f5,color:#000
    style DockerNet fill:#e8f4fd,color:#000
    style Web fill:#ffc107,color:#000
    style DB fill:#28a745,color:#fff
    style Adm fill:#ffc107,color:#000
```

Existen dos planos de comunicación claramente diferenciados dentro de una aplicación ejecutada con Docker Compose:

- **Desde el host**. El acceso a un servicio ocurre a través de los **puertos publicados** hacia la máquina anfitriona, por ejemplo `localhost:8000` o `localhost:8080`. En consecuencia, solo los servicios que declaran la directiva `ports` pueden ser consumidos directamente desde fuera de la red interna de Docker.
- **Entre servicios**. La comunicación ocurre dentro de la red del proyecto utilizando el **nombre del servicio** como hostname y el **puerto interno** sobre el que escucha la aplicación dentro del contenedor. Para esta comunicación interna no es necesario publicar puertos hacia el host.

En el ejemplo, PostgreSQL no publica el puerto `5432` hacia la máquina anfitriona. Aun así, los servicios `web` y `adminer` pueden conectarse a la base de datos a través de la red interna de Compose usando `db:5432`, siempre que `db` sea el nombre del servicio definido en `docker-compose.yml`.

Si se requiriera acceso a PostgreSQL desde una herramienta que se ejecuta en el host, como un cliente `psql` local, entonces sí sería necesario publicar el puerto correspondiente, por ejemplo mediante la directiva `ports` con la asignación `5432:5432`.

> [!TIP]
> En Compose, la publicación de puertos está orientada al acceso **desde fuera** de la red interna del proyecto. La comunicación entre servicios conectados a la misma red no depende de esa publicación, sino del mecanismo de resolución DNS interno provisto por Docker.

### El error de `localhost`

Dentro de un contenedor, `localhost` hace referencia al **propio contenedor**. No apunta ni a la máquina anfitriona ni a otros contenedores del mismo proyecto. Esta distinción es fundamental, ya que constituye uno de los errores más frecuentes al trabajar con aplicaciones multicontenedor.

```python
# INCORRECTO dentro de un contenedor
conn = psycopg2.connect("postgresql://app:secreto123@localhost:5432/visitasdb")

# CORRECTO dentro de la red de Compose
conn = psycopg2.connect("postgresql://app:secreto123@db:5432/visitasdb")
```

> [!IMPORTANT]
> Si su aplicación contenerizada intenta conectarse a `localhost` para alcanzar otro servicio, la conexión fallará. Utilice siempre el **nombre del servicio** como hostname.

## Persistencia con volúmenes en Compose

Los contenedores son, por naturaleza, elementos efímeros. Si un contenedor de base de datos se elimina y luego se recrea, la información almacenada únicamente en su sistema de archivos interno no se conserva. Los **volúmenes** permiten resolver este problema al ofrecer un mecanismo de persistencia desacoplado del ciclo de vida del contenedor, de modo que los datos permanezcan disponibles incluso después de su reemplazo o recreación.

En el archivo `docker-compose.yml`, la persistencia se configura en dos niveles:

1. **Declaración del volumen** en el nivel superior:

```yaml
volumes:
  pgdata:
```

2. **Montaje en el servicio**:

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data
```

Con esta configuración, los datos de PostgreSQL sobreviven a ciclos de `docker compose down` y `docker compose up`. Los datos solo se eliminan si se ejecuta `docker compose down -v` o si se borra el volumen manualmente con `docker volume rm`.

### Verificar persistencia

```bash
# Levantar, generar datos
$ docker compose up -d
$ curl http://localhost:8000    # Visitas: 1
$ curl http://localhost:8000    # Visitas: 2

# Detener sin eliminar volúmenes
$ docker compose down

# Volver a levantar
$ docker compose up -d
$ curl http://localhost:8000    # Visitas: 3 (los datos persisten)
```

## Configuración y variables de entorno

### Variables en el archivo Compose

Las variables de entorno pueden definirse directamente en el servicio:

```yaml
services:
  db:
    environment:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=secreto123
```

### Interpolación desde `.env`

Compose lee automáticamente un archivo `.env` ubicado en el mismo directorio que `docker-compose.yml`. Las variables definidas allí pueden interpolarse con la sintaxis `${VARIABLE}`:

```yaml
environment:
  - POSTGRES_USER=${POSTGRES_USER}
```

Esta separación permite mantener las credenciales fuera del archivo `docker-compose.yml`, facilitando su exclusión del control de versiones.

### Archivo de entorno por servicio

También es posible indicar un archivo de entorno específico para un servicio:

```yaml
services:
  web:
    env_file:
      - ./web.env
```

### Advertencia sobre secretos

> [!CAUTION]
> Las variables de entorno pueden ser consultadas desde el interior del contenedor y, en ciertos casos, quedar expuestas a través de registros, procesos en ejecución o mecanismos de inspección como `docker inspect`. Por esta razón, no deben asumirse como un mecanismo robusto de protección de información sensible. En entornos de producción, conviene utilizar soluciones especializadas para la gestión de secretos, como Docker Secrets en Swarm o los servicios de secretos ofrecidos por plataformas cloud. En entornos de desarrollo, el uso de `.env` puede considerarse una práctica razonable, siempre que dicho archivo no sea incorporado al repositorio ni distribuido de forma inadvertida.

## Errores comunes y troubleshooting

### Problema 1: Puerto del host ya en uso

**Síntoma**
Al ejecutar `docker compose up`, aparece un error similar a:

```plaintext
Error response from daemon: driver failed programming external connectivity:
Bind for 0.0.0.0:8080 failed: port is already allocated
```

**Causa**
Otro proceso en el host ya ocupa el puerto 8080.

**Solución**
Identifique el proceso con `lsof -i :8080` o `ss -tlnp | grep 8080` y deténgalo, o cambie el puerto en `compose.yml`:

```yaml
ports:
  - "9090:8080"   # Usar 9090 en el host
```

### Problema 2: La aplicación no logra conectarse a la base de datos

**Síntoma**  
La aplicación web presenta errores de conexión rechazada al intentar acceder a la base de datos.

**Causas**

1. **Dependencia declarada sin validación de disponibilidad**  
   El contenedor de la base de datos puede haberse iniciado, pero PostgreSQL aún no estar listo para aceptar conexiones. En este caso, `depends_on` por sí solo no resulta suficiente.

   **Acción recomendada**  
   Defina un `healthcheck` en el servicio `db` y, cuando la configuración lo requiera, utilice una condición como `service_healthy`. Además, resulta conveniente que la propia aplicación implemente reintentos controlados al establecer la conexión.

2. **Hostname incorrecto en la cadena de conexión**  
   La aplicación intenta conectarse usando `localhost` o una dirección IP fija.

   **Acción recomendada**  
   Verifique que la cadena de conexión utilice el nombre del servicio, por ejemplo `db`, como hostname dentro de la red de Compose.

3. **Inconsistencia en credenciales o parámetros de conexión**  
   Las variables de entorno definidas en la aplicación no coinciden con las configuradas en el servicio de base de datos.

   **Acción recomendada**  
   Revise cuidadosamente valores como usuario, contraseña, nombre de base de datos, hostname y puerto, y confirme que exista coherencia entre ambos servicios.

### Problema 3: Uso incorrecto de `localhost`

**Síntoma**  
La aplicación genera un error como `Connection refused` al intentar conectarse a `localhost:5432`.

**Causa**  
Dentro de un contenedor, `localhost` hace referencia al propio contenedor. No corresponde ni al host ni a otro servicio del mismo proyecto.

**Solución**  
Sustituya `localhost` por el nombre del servicio definido en `compose.yml`, por ejemplo `db`.

> [!TIP]
> En la comunicación entre servicios de Compose debe utilizarse el nombre del servicio y el puerto interno del contenedor.

### Problema 4: Pérdida de datos al recrear contenedores

**Síntoma**  
Después de ejecutar `docker compose down` y posteriormente `docker compose up`, los datos de la base de datos ya no están disponibles.

**Causas**

1. No se definió un volumen nombrado para la ruta de datos del servicio.
2. Se ejecutó `docker compose down -v`, lo que elimina los volúmenes asociados declarados en la configuración.

**Solución**  
Declare un volumen nombrado en el nivel superior del archivo `compose.yml` y móntelo en la ruta de persistencia del servicio correspondiente.

```yaml
volumes:
  pgdata:

services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
```
> [!WARNING]
> Si se utiliza docker compose down -v, el volumen persistente también será eliminado. En ese escenario, los datos no se conservarán.

### Problema 5: Error de sintaxis o indentación en YAML

**Síntoma**

Al ejecutar `docker compose up`, aparece un error de parseo:

```plaintext
yaml: line 12: mapping values are not allowed in this context
```

**Causa**

YAML es un formato estricto en aspectos como la indentación y la estructura. La indentación debe realizarse con espacios, no con tabulaciones, y la jerarquía entre mapas y listas debe mantenerse de forma consistente.

**Solución**

Revise cuidadosamente la indentación y la estructura general del archivo. Resulta recomendable utilizar un editor con soporte para YAML y validar la configuración con el siguiente comando:

```bash
$ docker compose config
```

Este comando parsea y muestra la configuración resultante. Si hay errores de sintaxis, los reporta.

### Problema 6: Servicio que depende de otro aún no preparado

**Síntoma**

La aplicación intenta conectarse repetidamente a la base de datos durante los primeros segundos y falla, pero eventualmente funciona si se reinicia manualmente.

**Causa**

`depends_on` sin condición solo garantiza que el contenedor se haya **iniciado**, no que el servicio interno esté listo.

**Soluciones**
- Agregar healthchecks y `condition: service_healthy`
- Implementar lógica de reintentos con *backoff* en la aplicación
- Usar la política `restart: unless-stopped` para que el contenedor se reinicie automáticamente si falla al inicio

## Resumen de comandos principales de Compose

| Comando | Descripción |
|---------|-------------|
| `docker compose up` | Construye, crea e inicia los servicios definidos en la configuración. |
| `docker compose up -d` | Inicia los servicios en modo desacoplado, es decir, en segundo plano. |
| `docker compose up --build` | Fuerza la construcción de las imágenes antes de iniciar los servicios. |
| `docker compose ps` | Lista los contenedores del proyecto y muestra su estado actual. |
| `docker compose logs` | Muestra los logs agregados de los servicios del proyecto. |
| `docker compose logs -f <servicio>` | Muestra y sigue en tiempo real los logs de un servicio específico. |
| `docker compose exec <servicio> <cmd>` | Ejecuta un comando dentro de un contenedor en ejecución asociado al servicio indicado. |
| `docker compose stop` | Detiene los servicios sin eliminar los contenedores. |
| `docker compose down` | Detiene y elimina los contenedores y las redes creadas para el proyecto. |
| `docker compose down -v` | Además de lo anterior, elimina los volúmenes nombrados definidos en Compose y los volúmenes anónimos asociados. |
| `docker compose config` | Valida, resuelve y muestra la configuración resultante en formato canónico. |

## Buenas prácticas iniciales

- **Un servicio, una responsabilidad**. Cada servicio debe encapsular una función claramente definida dentro de la aplicación. No resulta recomendable agrupar, por ejemplo, la base de datos y la aplicación web en un mismo contenedor, ya que ello dificulta la separación de responsabilidades, el mantenimiento y la evolución del sistema.
- **Nombres de servicio descriptivos**. Utilice nombres que reflejen la función del componente, como `db`, `web`, `cache` o `worker`. Evite nombres genéricos que no aporten significado técnico, como `service1` o `app1`.
- **Externalice la configuración**. Mantenga parámetros de configuración, credenciales y valores dependientes del entorno fuera del código fuente y fuera de los valores fijados directamente en el archivo `docker-compose.yml`. Para ello, utilice variables de entorno, archivos `.env` u otros mecanismos adecuados según el contexto.
- **No incorpore credenciales al repositorio**. El archivo `.env` y cualquier otro archivo que contenga información sensible deben excluirse del control de versiones, por ejemplo mediante `.gitignore`.
- **Utilice volúmenes para servicios con estado**. Todo servicio que requiera persistencia, como bases de datos o almacenamiento de archivos, debe apoyarse en volúmenes. Esto permite desacoplar los datos del ciclo de vida del contenedor y reduce el riesgo de pérdida de información.
- **No confunda Compose con una plataforma de orquestación de gran escala**. Docker Compose es especialmente útil en desarrollo, pruebas locales y escenarios controlados de despliegue. Sin embargo, en arquitecturas de mayor complejidad, con requisitos de escalabilidad, alta disponibilidad o múltiples nodos, suelen requerirse soluciones de orquestación más robustas, como Kubernetes o servicios gestionados de contenedores en la nube.
- **Documente la operación básica del proyecto**. Incluya en el `README` los comandos mínimos para construir, iniciar, detener, inspeccionar y depurar la aplicación. Esta práctica mejora la reproducibilidad y reduce la fricción de incorporación al proyecto.
- **Mantenga el archivo `docker-compose.yml` bajo control de versiones**. El archivo de Compose forma parte de la definición operativa de la aplicación y debe tratarse con el mismo rigor que el código fuente.

## Preguntas de autoevaluación

1. ¿Qué limitaciones presenta la gestión de una aplicación multicontenedor usando exclusivamente `docker run`?

2. ¿Cuál es la diferencia entre las directivas `build` e `image` en un servicio de Compose?

3. ¿Por qué `depends_on` sin condición adicional no garantiza que la base de datos esté lista para recibir conexiones? ¿Qué alternativas existen?

4. Un desarrollador reporta que su aplicación Flask contenerizada no puede conectarse a PostgreSQL, pero ambos contenedores están en estado `running`. La cadena de conexión usa `localhost:5432`. ¿Cuál es la causa probable y cómo se resuelve?

5. ¿Qué diferencia hay entre un puerto publicado (`ports`) y la comunicación entre servicios por la red interna de Compose?


## Referencias

- Docker Inc. *Docker Compose overview*. https://docs.docker.com/compose/
- Docker Inc. *Compose file reference*. https://docs.docker.com/reference/compose-file/
- Docker Inc. *Networking in Compose*. https://docs.docker.com/compose/how-tos/networking/
- Docker Inc. *Use volumes in Docker Compose*. https://docs.docker.com/compose/how-tos/use-volumes/
- Docker Inc. *Environment variables in Compose*. https://docs.docker.com/compose/how-tos/environment-variables/
- Nickoloff, J., & Kuenzli, S. *Docker in Action* (2nd ed.). Manning Publications.
- Stoneman, E. *Learn Docker in a Month of Lunches*. Manning Publications.
