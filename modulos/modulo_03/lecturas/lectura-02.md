![Banner del proyecto](/modulos/assets/Banner.png)

# Lectura 02 - Redes, descubrimiento de servicios y comunicación entre contenedores

## Prerrequisitos

La presente lectura supone que el estudiante:

- comprende los conceptos de imagen, contenedor y volumen en Docker
- ha utilizado `docker run`, incluidas opciones como `-d`, `-p`, `-v`, `-e` y `--name`
- ha definido y operado aplicaciones multicontenedor con Docker Compose
- posee familiaridad con la estructura de un archivo `compose.yml` o `docker-compose.yml`
- utiliza la terminal en un entorno Linux o equivalente
- cuenta con nociones básicas de redes TCP/IP, tales como direcciones IP, puertos y protocolos

## Introducción

La lectura anterior presentó Docker Compose como una herramienta para definir y operar aplicaciones multicontenedor de manera declarativa. En ese contexto, la comunicación entre servicios se introdujo como un mecanismo práctico: **un servicio establece conexión con otro mediante su nombre**. No obstante, detrás de esa aparente simplicidad subyace un modelo de red cuyo entendimiento resulta indispensable para diseñar, diagnosticar y operar soluciones contenerizadas con criterio técnico.

Surgen entonces varias preguntas fundamentales: ¿por qué un contenedor puede alcanzar a otro mediante un identificador como `db`? ¿Qué ocurre cuando un contenedor intenta conectarse a `localhost`? ¿Por qué algunos servicios deben publicar puertos y otros no? ¿Qué consecuencias se derivan de que dos servicios no compartan la misma red?

Esta lectura aborda estas preguntas. Su propósito es explicitar el **modelo de red de Docker**, examinar el mecanismo de **descubrimiento de servicios**, diferenciar con precisión la **comunicación interna** del **acceso desde el host** y ofrecer criterios de **diagnóstico** cuando se presentan fallos de conectividad.

## Motivación

Toda aplicación compuesta por más de un contenedor requiere, de manera explícita o implícita, algún mecanismo de comunicación entre sus componentes. Entre los patrones más frecuentes se encuentran los siguientes:

- una aplicación web que consulta una base de datos
- un frontend que consume una API expuesta por un backend
- un backend que almacena sesiones en un servicio de caché como Redis
- un servicio de procesamiento que obtiene trabajos desde una cola de mensajes

En cada uno de estos escenarios, un contenedor debe **localizar** a otro y **establecer una conexión TCP** con él. Para que ello sea posible, ambos contenedores deben compartir una red, disponer de un mecanismo de resolución de nombres a direcciones IP y exponer internamente los puertos requeridos para la comunicación.

### Por qué `docker run -p` no resuelve por sí solo la conectividad

La opción `-p` en `docker run` publica un puerto del contenedor hacia el host. Esto permite que un navegador o una herramienta ejecutada en la máquina local acceda al servicio. Sin embargo, **publicar un puerto no equivale a habilitar la comunicación entre contenedores**. Dos contenedores con puertos publicados no necesariamente pueden comunicarse entre sí a través de dichos puertos si no comparten una red común.

> [!IMPORTANT]
> La publicación de puertos (`-p`) resuelve el acceso **desde el host hacia el contenedor**. La comunicación **entre contenedores** se establece mediante redes compartidas y resolución de nombres.

Comprender esta distinción es esencial para modelar correctamente la conectividad de una aplicación multicontenedor y para evitar configuraciones erróneas que aparentan ser válidas, pero fallan en tiempo de ejecución.

## Modelo de red en Docker

### Redes virtuales

Docker incorpora un subsistema de red que permite crear **redes virtuales** dentro de la máquina anfitriona. Cada una de estas redes funciona como un segmento lógico aislado, en el que los contenedores conectados pueden intercambiar tráfico de manera controlada.

Desde la perspectiva del contenedor, esta red se comporta de forma similar a una red local convencional. El contenedor dispone de una interfaz de red, recibe direccionamiento dentro de la red correspondiente y puede establecer comunicaciones mediante protocolos como TCP o UDP con otros contenedores conectados al mismo segmento.
### Aislamiento mediante namespace de red

El mecanismo que hace posible este aislamiento es el **network namespace** del kernel Linux. Un *network namespace* define un espacio aislado con su propia pila de red, incluidas interfaces, tablas de enrutamiento, reglas de filtrado y puertos. En Docker, este mecanismo constituye una de las bases del modelo de conectividad de los contenedores.

En términos generales, cada contenedor opera dentro de su propio espacio de red, lo que implica que:

- dispone de sus propias interfaces de red dentro del contenedor, típicamente una interfaz como `eth0`
- puede recibir direccionamiento de red propio según la red a la que se conecte
- mantiene su propia noción de `localhost`
- administra su propio espacio de puertos, por lo que las aplicaciones dentro de distintos contenedores pueden escuchar en el mismo puerto sin colisionar entre sí a nivel de *namespace*

> [!NOTE]
> Esta descripción corresponde al comportamiento habitual en redes aisladas de Docker, como las de tipo *bridge*. Existen excepciones, por ejemplo cuando se utiliza el modo de red `host`, caso en el cual el contenedor comparte el *network namespace* de la máquina anfitriona y deja de tener aislamiento de red propio.

> [!NOTE]
> No es necesario dominar el detalle interno de los namespaces para trabajar con redes en Docker. Lo importante, en esta etapa, es comprender que cada contenedor posee un **entorno de red independiente** y que `localhost` dentro de un contenedor refiere exclusivamente a ese contenedor.

### Drivers de red
Docker utiliza **drivers de red** para implementar distintos modelos de conectividad entre contenedores, el host y redes externas. Cada driver define un comportamiento particular en términos de aislamiento, visibilidad, direccionamiento y alcance de la comunicación.

Los más relevantes para esta lectura son los siguientes:

| Driver | Descripción | Uso típico |
|--------|-------------|------------|
| `bridge` | Implementa una red virtual local dentro del host. Permite la comunicación entre contenedores conectados a la misma red y mantiene aislamiento frente a otros segmentos de red. | Desarrollo local, pruebas y aplicaciones multicontenedor desplegadas en un único host |
| `host` | El contenedor comparte directamente la pila de red de la máquina anfitriona, por lo que desaparece el aislamiento de red propio del contenedor. | Escenarios específicos donde se requiere máxima cercanía a la red del host y se acepta una reducción del aislamiento |
| `none` | Desactiva la conectividad de red del contenedor, salvo la interfaz de loopback. | Contenedores que deben permanecer completamente aislados desde la perspectiva de red |

Existen otros drivers que responden a necesidades más especializadas. Entre ellos se encuentran `overlay`, orientado a la comunicación entre múltiples hosts Docker dentro de una red distribuida, y `macvlan`, que permite que un contenedor aparezca en la red física como si fuera un dispositivo independiente.

No obstante, el análisis detallado de estos drivers excede el alcance de esta lectura. El foco se centrará en el driver **bridge**, ya que constituye el modelo de referencia para comprender la conectividad de aplicaciones Docker desplegadas en un único host.

## Redes bridge

### Qué es una red `bridge`

Una red **bridge** es una red virtual local implementada por Docker dentro de un mismo host. Su propósito es permitir la comunicación entre contenedores conectados a esa red, manteniendo al mismo tiempo un grado de aislamiento frente al exterior y frente a otros segmentos de red.

Desde una perspectiva conceptual, su funcionamiento puede compararse con el de un switch virtual. Cada contenedor conectado recibe direccionamiento dentro de un rango administrado por Docker y puede intercambiar tráfico con otros contenedores que pertenezcan a la misma red. Este modelo constituye la base más común para ejecutar aplicaciones multicontenedor en entornos locales o en un único servidor.

### La red `bridge` por defecto

Cuando Docker se instala, crea automáticamente una red llamada `bridge`. Si un contenedor se inicia con `docker run` y no se especifica una red explícita, Docker lo conecta a esta red por defecto.

No obstante, conviene precisar que esta red por defecto no suele ser la opción más conveniente para aplicaciones compuestas por varios contenedores. En la práctica, resulta preferible definir redes `bridge` específicas para cada aplicación, ya que ello mejora el aislamiento y facilita una organización más clara de la conectividad.

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
f6e5d4c3b2a1   host      host      local
1a2b3c4d5e6f   none      null      local
```

> [!NOTE]
> En escenarios multicontenedor, las redes `bridge` definidas explícitamente suelen ofrecer una experiencia más adecuada que la red `bridge` por defecto, especialmente porque permiten una configuración más controlada y una resolución de nombres más predecible entre contenedores.

Sin embargo, la red bridge por defecto presenta limitaciones importantes:

- **No proporciona resolución DNS por nombre de contenedor**. Los contenedores conectados a esta red solo pueden comunicarse entre sí mediante direcciones IP, las cuales son efímeras y pueden cambiar cuando un contenedor se recrea.
- **Todos los contenedores sin una red explícita comparten este espacio**, lo que reduce el aislamiento lógico entre aplicaciones.

### Redes bridge definidas por el usuario

Cuando se crea una red con `docker network create`, el resultado es una red bridge **definida por el usuario**. Esta variante ofrece ventajas significativas respecto de la red por defecto:

| Característica | Red bridge por defecto | Red bridge definida por el usuario |
|----------------|------------------------|-------------------------------------|
| Resolución DNS por nombre | No disponible | Disponible |
| Aislamiento de otros contenedores | Todos comparten la red | Solo los contenedores conectados explícitamente |
| Creación automática por Compose | No | Sí |
| Recomendada para aplicaciones | No | Sí |

```bash
# Crear una red bridge definida por el usuario
$ docker network create mi-red

# Ejecutar contenedores en esa red
$ docker run -d --name servidor --network mi-red nginx
$ docker run -d --name cliente --network mi-red alpine sleep infinity

# El cliente puede alcanzar al servidor por nombre
$ docker exec cliente ping -c 2 servidor
```

> [!TIP]
> Cuando se trabaja con Docker Compose, no es necesario crear redes manualmente. Compose crea automáticamente una red bridge definida por el usuario para cada proyecto y habilita la resolución DNS entre servicios de manera predeterminada.

### Redes creadas por Compose

Al ejecutar `docker compose up` en un directorio llamado `mi-proyecto`, Compose crea, por defecto, una red cuyo nombre suele seguir el patrón `mi-proyecto_default`. Más precisamente, este nombre se deriva del **nombre efectivo del proyecto**, que normalmente coincide con el directorio de trabajo, aunque puede configurarse de forma explícita.

Todos los servicios definidos en `docker-compose.yml` se conectan automáticamente a esta red, lo que les permite comunicarse entre sí utilizando el nombre del servicio como identificador dentro de la red interna.

```bash
$ docker compose up -d
$ docker network ls
NETWORK ID     NAME                    DRIVER    SCOPE
...
7f8e9d0a1b2c   mi-proyecto_default     bridge    local
```

## Descubrimiento de servicios

### El problema de localizar a otro contenedor

Cuando un servicio necesita conectarse a otro, debe contar con algún mecanismo para identificar su destino. En entornos tradicionales, esta necesidad suele resolverse mediante direcciones IP fijas o nombres de host configurados manualmente. En un entorno contenerizado, sin embargo, esta estrategia resulta poco robusta, ya que las direcciones IP asignadas a los contenedores pueden cambiar cuando estos se eliminan, recrean o se vuelven a conectar a una red. Por esta razón, construir la conectividad de la aplicación sobre direcciones IP específicas conduce a configuraciones frágiles y difíciles de mantener.

### DNS interno de Docker

Docker resuelve este problema mediante un mecanismo de **resolución DNS interna** disponible en las redes definidas por el usuario. En este contexto, Docker mantiene información actualizada sobre los contenedores y aliases conectados a cada red, y responde las consultas de nombre con la dirección IP vigente del destino correspondiente.

En la práctica, esto significa que un contenedor puede intentar conectarse a otro utilizando un nombre lógico, como `db`, en lugar de depender de una dirección IP concreta. De este modo, si el contenedor de destino es recreado y recibe una nueva IP, la aplicación no necesita modificar su configuración, siempre que el nombre de servicio permanezca constante.

Cuando Docker Compose crea automáticamente la red de una aplicación, este mismo mecanismo permite que los servicios se descubran entre sí utilizando el **nombre del servicio** definido en `compose.yml`.

> [!IMPORTANT]
> La resolución DNS por nombre funciona en **redes definidas por el usuario**, incluidas las que Docker Compose crea automáticamente para un proyecto. Este comportamiento no está disponible de forma automática en la red `bridge` por defecto.

### Nombres de contenedor y nombres de servicio

En este contexto conviene distinguir entre dos tipos de nombres:

- **Nombre de contenedor**: se asigna con `--name` en `docker run` o se deriva automáticamente. En redes definidas por el usuario, puede funcionar como hostname.
- **Nombre de servicio**: se define como clave en la sección `services` de `docker-compose.yml`. En Docker Compose, este constituye el nombre principal para la resolución DNS.

```yaml
services:
  api:           # ← nombre de servicio (usado para DNS)
    container_name: mi-api   # ← nombre de contenedor (opcional)
    image: mi-api:latest
```

En la mayoría de los escenarios con Docker Compose, el **nombre del servicio** debe utilizarse como `hostname` en las cadenas de conexión. Esto se debe a que, dentro de la red del proyecto, los servicios son descubiertos automáticamente por ese nombre.

Aunque `container_name` puede influir en el nombre asignado al contenedor, su uso explícito como mecanismo principal de conectividad suele ser menos recomendable en configuraciones mantenibles. En la práctica, basar la comunicación interna en el nombre del servicio resulta más coherente con el modelo de Compose y favorece configuraciones más estables y portables.

### Nombres alternativos en red

Docker permite asignar **nombres alternativos** a un contenedor dentro de una red específica. Estos nombres funcionan como identificadores DNS adicionales dentro de esa red.

```yaml
services:
  db:
    image: postgres:16
    networks:
      backend:
        aliases:
          - postgres
          - database
```

Con esta configuración, cualquier servicio conectado a la red `backend` puede comunicarse con `db` utilizando cualquiera de estos nombres: `db`, `postgres` o `database`. Esta posibilidad resulta útil cuando distintos componentes de la aplicación requieren referirse al mismo servicio mediante nombres de host diferentes.

### Por qué los nombres sustituyen a las IP

Dado que Docker asigna direcciones IP de manera dinámica y mantiene actualizada esta información en su mecanismo de resolución DNS interna, los nombres de servicio constituyen una **referencia estable** independiente del ciclo de vida de cada contenedor. En consecuencia, si un contenedor de base de datos se elimina y luego se recrea, su dirección IP puede cambiar, pero el nombre `db` continuará resolviendo correctamente dentro de la red correspondiente.

```python
# FRÁGIL: depende de una IP que puede cambiar
conn = psycopg2.connect("postgresql://app:pass@172.18.0.3:5432/midb")

# ROBUSTO: usa el nombre del servicio
conn = psycopg2.connect("postgresql://app:pass@db:5432/midb")
```

## Puertos y conectividad

### Puerto interno vs. puerto publicado

Cada contenedor dispone de su propio espacio de puertos, determinado por el proceso que ejecuta en su interior. Por ejemplo, una aplicación web puede escuchar en el puerto `80` o `8000`, mientras que PostgreSQL suele hacerlo en el `5432`. Estos corresponden a **puertos internos** del contenedor y pueden ser utilizados por otros contenedores conectados a la misma red de Docker.

Un **puerto publicado**, en cambio, corresponde a un mapeo que Docker establece entre un puerto de la máquina anfitriona y un puerto interno del contenedor. Este mecanismo permite que procesos externos al entorno de red de Docker, como un navegador, `curl` o herramientas locales de administración, puedan acceder al servicio.

```yaml
services:
  web:
    ports:
      - "8000:8000"   # HOST:CONTENEDOR
```

### Dos planos de comunicación

```mermaid
flowchart TB
    subgraph Host["Máquina host"]
        Browser["Navegador / curl"]
    end
    
    subgraph Red["Red Docker: proyecto_default"]
        direction LR
        Web["web\n:8000"]
        API["api\n:5000"]
        DB["db\n:5432"]
    end
    
    Browser -->|"localhost:8000\n(puerto publicado)"| Web
    Web -->|"api:5000\n(red interna)"| API
    API -->|"db:5432\n(red interna)"| DB
    
    style Host fill:#f5f5f5,color:#000
    style Red fill:#e8f4fd,color:#000
    style Web fill:#ffc107,color:#000
    style API fill:#ffc107,color:#000
    style DB fill:#28a745,color:#fff
```

| Plano | Origen | Destino | Mecanismo | Requiere `ports` |
|-------|--------|---------|-----------|-------------------|
| Host → contenedor | Navegador, terminal, herramienta local | Contenedor con puerto publicado | Puerto mapeado (`localhost:8000`) | Sí |
| Contenedor → contenedor | Contenedor en la misma red | Otro contenedor en la misma red | Nombre de servicio + puerto interno (`api:5000`) | No |

### Publicación selectiva de puertos

No todos los servicios deben ser accesibles desde el host. En una arquitectura típica:

- el **frontend o web** publica un puerto porque constituye el punto de entrada para el usuario
- el **backend o API** puede no publicar puertos si solo es consumido por otros servicios dentro de la red Docker
- la **base de datos** normalmente no debería publicar su puerto si solo es accedida por el backend

```yaml
services:
  web:
    ports:
      - "8000:8000"    # accesible desde el host
  api:
    # sin ports: solo accesible dentro de la red Docker
    expose:
      - "5000"
  db:
    # sin ports: solo accesible dentro de la red Docker
```

> [!TIP]
> La directiva `expose` documenta los puertos que un servicio utiliza internamente, pero **no publica** esos puertos hacia el host. Su principal utilidad es descriptiva y documental dentro del archivo Compose.

### Cuándo publicar y cuándo no

La regla general es la siguiente: **solo deben publicarse los puertos que realmente necesiten ser accesibles desde fuera de la red interna de Docker**. Publicar puertos de manera innecesaria expone servicios a la máquina anfitriona y, según la configuración de red del host, incluso a otros equipos de la red. Esto incrementa la superficie de ataque, dificulta el principio de mínimo acceso y puede generar una comprensión equivocada de la topología real de la aplicación.

Por esta razón, la publicación de puertos debe responder a una necesidad concreta de acceso externo y no utilizarse como mecanismo general de comunicación entre contenedores.

## El problema de `localhost`

Este aspecto merece atención particular porque constituye una de las causas más recurrentes de error en aplicaciones multicontenedor.

### Qué significa `localhost` dentro de un contenedor

Debido al aislamiento provisto por los *network namespaces*, cada contenedor dispone de su propia interfaz de *loopback*. Por ello, dentro de un contenedor, `localhost` o `127.0.0.1` hace referencia exclusivamente al propio contenedor. No corresponde ni a la máquina anfitriona ni a otros contenedores conectados a la misma red.

```mermaid
flowchart LR
    subgraph ContenedorA["Contenedor: web"]
        AppA["Aplicación Flask"]
        LoA["localhost = 127.0.0.1\n(solo este contenedor)"]
    end
    
    subgraph ContenedorB["Contenedor: db"]
        AppB["PostgreSQL"]
        LoB["localhost = 127.0.0.1\n(solo este contenedor)"]
    end
    
    AppA -. "localhost:5432 ✗\nno alcanza a db" .-> LoA
    AppA -->|"db:5432 ✓\nvía red Docker"| AppB
    
    style ContenedorA fill:#fff3cd,color:#000
    style ContenedorB fill:#d4edda,color:#000
```

### Casos correctos e incorrectos

| Escenario | Uso de `localhost` | ¿Correcto? | Explicación |
|-----------|-------------------|-------------|-------------|
| Aplicación dentro del contenedor conectándose a sí misma | `localhost:8000` | Sí | Hace referencia al propio contenedor |
| Aplicación en el contenedor `web` intentando conectarse al contenedor `db` | `localhost:5432` | No | `localhost` no apunta al servicio `db`, sino al propio contenedor `web` |
| `curl` ejecutado desde el host hacia un contenedor con puerto publicado | `localhost:8000` | Sí | El puerto del contenedor ha sido publicado hacia la máquina anfitriona |
| Aplicación en el contenedor `web` conectándose a `db` por nombre | `db:5432` | Sí | La resolución DNS interna de Docker permite localizar el servicio `db` dentro de la red |

> [!CAUTION]
> Si una aplicación contenerizada utiliza `localhost` en la cadena de conexión para intentar alcanzar a otro servicio, la conexión fallará, usualmente con errores como `Connection refused`. Este es uno de los errores de conectividad más frecuentes en aplicaciones multicontenedor. La corrección consiste en reemplazar `localhost` por el **nombre del servicio** correspondiente.
 
## Redes en Docker Compose

### Red implícita

Como se explicó en la lectura anterior, Docker Compose crea automáticamente una red de tipo `bridge` para cada proyecto. Si el directorio del proyecto se denomina `mi-proyecto`, la red suele adoptar el nombre `mi-proyecto_default`. Todos los servicios definidos en `compose.yml` se conectan a esta red y, a través de ella, pueden comunicarse utilizando el nombre del servicio.

En muchas aplicaciones, esta red creada de forma automática es suficiente y no requiere configuración adicional.

### Redes explícitas

Cuando se requiere **segmentar la comunicación** entre servicios, es posible definir redes explícitas en `docker-compose.yml`. Esto permite controlar qué servicios pueden comunicarse entre sí.

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    networks:
      - frontend

  api:
    build: ./api
    networks:
      - frontend
      - backend

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  pgdata:
```

En esta configuración:

- `web` solo pertenece a la red `frontend`
- `api` pertenece simultáneamente a `frontend` y `backend`
- `db` solo pertenece a la red `backend`

Como consecuencia, `web` puede comunicarse con `api` porque ambos comparten `frontend`, y `api` puede comunicarse con `db` porque ambos comparten `backend`. Sin embargo, `web` **no puede** comunicarse directamente con `db`, dado que no existe una red común entre ambos.

```mermaid
flowchart TB
    subgraph frontend["Red: frontend"]
        Web["web :80"]
        API_f["api :5000"]
    end
    
    subgraph backend["Red: backend"]
        API_b["api :5000"]
        DB["db :5432"]
    end
    
    Host["Host"] -->|"localhost:80"| Web
    Web -->|"api:5000"| API_f
    API_b -->|"db:5432"| DB
    
    API_f -.- API_b
    
    style frontend fill:#e8f4fd,color:#000
    style backend fill:#f0e6ff,color:#000
    style Web fill:#ffc107,color:#000
    style API_f fill:#ffc107,color:#000
    style API_b fill:#ffc107,color:#000
    style DB fill:#28a745,color:#fff
```

> [!NOTE]
> `api` aparece dos veces en el diagrama únicamente con fines de representación visual. En la implementación real, se trata de un único contenedor con dos interfaces de red, una por cada red a la que está conectado.

### Aislamiento entre redes

Dos contenedores que no comparten al menos una red **no pueden comunicarse directamente** entre sí, incluso si ambos se ejecutan en el mismo host. Este aislamiento constituye una propiedad relevante del modelo de red de Docker y resulta útil para:

- impedir que el frontend acceda directamente a la base de datos
- separar el tráfico de administración del tráfico propio de la aplicación
- reducir la superficie de exposición de servicios internos

Esta capacidad de segmentación permite diseñar topologías de red más controladas, en las que cada servicio solo puede interactuar con los componentes estrictamente necesarios para su funcionamiento.

## Diseño básico de topologías

### Servicio público y servicio interno

Al diseñar la red de una aplicación multicontenedor, resulta útil distinguir entre dos tipos de servicios:

- **Servicios públicos**. Son aquellos que deben recibir tráfico desde fuera de la red interna de Docker, por ejemplo desde un navegador, una API externa o herramientas utilizadas directamente desde el host. Estos servicios suelen requerir la publicación de puertos.
- **Servicios internos**. Son aquellos cuyo consumo está restringido a otros contenedores de la misma aplicación. En consecuencia, no necesitan publicar puertos y no deberían ser accesibles directamente desde la máquina anfitriona.

Esta distinción permite definir con mayor claridad la topología de la aplicación y aplicar un criterio más estricto sobre qué componentes deben exponerse.

### Criterio para decidir qué puertos publicar

La decisión de publicar un puerto no debe tomarse por conveniencia, sino a partir de una pregunta central: **qué actor necesita acceder a ese servicio**.

| Consumidor | ¿Conviene publicar puerto? | Ejemplo |
|------------|-----------------------------|---------|
| Navegador del usuario | Sí | Frontend web, Adminer |
| Otro contenedor en la misma red | No | Base de datos, caché, API interna |
| Herramienta ejecutada en el host por el desarrollador | Depende del caso | PostgreSQL accesible desde `psql` local en un entorno de desarrollo |

> [!TIP]
> La publicación de puertos debe responder a una necesidad concreta de acceso externo. Si un servicio solo será consumido por otros contenedores, lo correcto es mantenerlo como servicio interno.

### Ejemplo de segmentación

```yaml
services:
  web:
    ports:
      - "80:80"       # público
    networks:
      - frontend

  api:
    # no publica puertos
    networks:
      - frontend
      - backend

  db:
    # no publica puertos
    networks:
      - backend

  redis:
    # no publica puertos
    networks:
      - backend
```

En esta topología, el único punto de entrada desde el host es `web` en el puerto 80. La base de datos y Redis permanecen aislados del acceso externo.

## Inspección y diagnóstico

Cuando la conectividad entre servicios falla, Docker ofrece herramientas que permiten diagnosticar el problema de manera sistemática.

### Comandos de inspección de redes

| Comando | Función |
|---------|---------|
| `docker network ls` | Lista todas las redes disponibles |
| `docker network inspect <red>` | Muestra detalles de una red: contenedores conectados, subnet, gateway |
| `docker compose ps` | Lista servicios del proyecto, sus puertos y estado |
| `docker compose logs <servicio>` | Muestra los logs de un servicio, útil para identificar errores de conexión |

## Diagnóstico de redes en Docker

### Verificar resolución DNS desde un contenedor

```bash
# Opción 1: usar getent (si disponible en la imagen)
docker compose exec <servicio-origen> getent hosts <servicio-destino>

# Opción 2: usar Python
docker compose exec <servicio-origen> python -c "import socket; print(socket.gethostbyname('<servicio-destino>'))"

# Opción 3: usar nslookup (requiere bind-tools o dnsutils)
docker compose exec <servicio-origen> nslookup <servicio-destino>
```

Si la resolución falla, las causas más probables son las siguientes:

- Los servicios no comparten una red.
- El nombre del servicio está mal escrito.
- El contenedor de destino no se encuentra en ejecución.

### Verificar si un puerto está abierto desde un contenedor

```bash
docker compose exec <servicio-origen> python -c "
import socket
s = socket.socket()
s.settimeout(3)
try:
    s.connect(('<servicio-destino>', <puerto>))
    print('Conexión exitosa')
except Exception as e:
    print(f'Error: {e}')
finally:
    s.close()
"
```

### Inspeccionar la IP de un contenedor

```bash
docker inspect <nombre-contenedor> \
    --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

> [!NOTE]
> El nombre del contenedor en Docker Compose sigue la convención `<proyecto>-<servicio>-<réplica>`. Para identificar el nombre exacto, ejecutar `docker compose ps`.

> [!WARNING]
> Las direcciones IP de los contenedores son efímeras. No deben utilizarse en configuraciones ni en cadenas de conexión. La práctica recomendada consiste en utilizar siempre nombres de servicio.

## Errores comunes y troubleshooting

### 1. Uso incorrecto de `localhost`

**Síntoma**  
Aparece un error como `Connection refused` al intentar conectarse a otro servicio mediante `localhost`.

**Causa**  
Dentro de un contenedor, `localhost` hace referencia exclusivamente al propio contenedor. No corresponde a otro servicio de la aplicación.

**Solución**  
Sustituya `localhost` por el nombre del servicio en la cadena de conexión.

### 2. Publicación innecesaria de puertos

**Síntoma**  
La base de datos queda accesible desde el host sin requerirlo o se presentan colisiones de puertos.

**Causa**  
Se ha publicado un puerto, por ejemplo `5432:5432`, en un servicio que no necesita ser consumido desde fuera de la red interna de Docker.

**Solución**  
Elimine la directiva `ports` en aquellos servicios que solo deban ser consumidos por otros contenedores. La comunicación interna entre servicios no requiere publicación de puertos.

### 3. Asumir que publicar un puerto habilita comunicación interna

**Síntoma**  
Un contenedor no logra conectarse a otro, aunque el servicio de destino tenga puertos publicados.

**Causa**  
La publicación de puertos habilita acceso desde el host, no garantiza conectividad entre contenedores. Si los servicios no comparten una red, no podrán comunicarse entre sí.

**Solución**  
Verifique que ambos servicios estén conectados a una misma red de Docker.

### 4. Servicios sin red compartida

**Síntoma**  
Un servicio no logra resolver el nombre de otro servicio.

**Causa**  
En una configuración con redes explícitas, los servicios que deben comunicarse no comparten al menos una red.

**Solución**  
Revise la sección `networks` de cada servicio y confirme que los componentes que requieren comunicarse estén conectados a una red común.

### 5. Nombre de servicio incorrecto

**Síntoma**  
Se presentan errores de resolución DNS, como `Name does not resolve`, o fallos de conexión.

**Causa**  
La cadena de conexión utiliza un nombre que no coincide con la clave definida para el servicio en `compose.yml`. Entre los errores más frecuentes se encuentran usar `postgres` cuando el servicio se llama `db`, utilizar `container_name` en lugar del nombre del servicio o cometer errores tipográficos.

**Solución**  
Verifique que el nombre utilizado en la cadena de conexión coincida exactamente con la clave del servicio en la sección `services`.

### 6. Colisión de puertos en el host

**Síntoma**

Error al iniciar un servicio porque el puerto ya está asignado.

```plaintext
Bind for 0.0.0.0:80 failed: port is already allocated
```

**Causa**
Otro proceso del host, o de otro proyecto Compose, ya utiliza ese puerto.

**Solución**
Identificar el proceso con `lsof -i :80` o `ss -tlnp | grep :80` y detenerlo, o modificar el puerto del host en `compose.yml`:

```yaml
ports:
  - "8080:80"   # usar 8080 en el host en lugar de 80
```

### 7. Servicio que inicia, pero aún no está disponible

**Síntoma**  
La aplicación presenta fallos intermitentes al conectarse a la base de datos durante el arranque y solo logra funcionar correctamente después de un reinicio manual.

**Causa**  
El contenedor de base de datos ha sido iniciado, pero el motor aún se encuentra en proceso de inicialización. En este contexto, `depends_on`, sin condiciones adicionales, únicamente establece un orden de arranque entre contenedores, pero no garantiza que el servicio interno se encuentre listo para aceptar conexiones.

**Solución**  
Se recomienda utilizar `depends_on` junto con `condition: service_healthy` y definir un `healthcheck` en el servicio de base de datos. Como complemento, o en su defecto, también conviene implementar lógica de reintentos en la aplicación para manejar de forma más robusta la disponibilidad inicial de dependencias externas.

### 8. Dependencia de direcciones IP efímeras

**Síntoma**  
La aplicación funciona inicialmente, pero pierde conectividad después de recrear los contenedores.

**Causa**  
La cadena de conexión depende de una dirección IP fija, por ejemplo `172.18.0.3`, que deja de ser válida cuando el contenedor es eliminado y posteriormente recreado.

**Solución**  
La dirección IP debe sustituirse por el nombre del servicio. De este modo, la aplicación se apoya en el mecanismo de resolución DNS interna de Docker, que actualiza automáticamente la asociación entre el nombre lógico del servicio y su dirección IP vigente.

## Buenas prácticas

- **Evite el acoplamiento a direcciones IP**. Las direcciones IP de los contenedores no deben asumirse como estables. Por ello, la configuración de conectividad y las cadenas de conexión deben construirse a partir de nombres de servicio.
- **Publique únicamente los puertos estrictamente necesarios**. Si un servicio solo será consumido por otros contenedores de la misma aplicación, no requiere la directiva `ports`. Exponer puertos sin necesidad incrementa la superficie de acceso y puede introducir complejidad innecesaria.
- **Aísle servicios internos mediante redes explícitas cuando la arquitectura lo justifique**. En aplicaciones con varios componentes, la segmentación por redes permite controlar con mayor precisión qué servicios pueden comunicarse entre sí y cuáles deben permanecer aislados.
- **No confunda la conectividad interna con el acceso desde el host**. Que un servicio sea accesible por nombre dentro de la red de Docker no implica que pueda ser consumido directamente desde la máquina anfitriona. Para ello, se requiere la publicación explícita de puertos.
- **Documente la topología de red de la aplicación**. Incluir descripciones claras de servicios, redes y puertos publicados, e incluso diagramas sencillos cuando resulte pertinente, facilita la comprensión del sistema y mejora la incorporación de nuevos integrantes al proyecto.

## Preguntas de autoevaluación

1. ¿Qué diferencias existen entre la red bridge por defecto de Docker y una red bridge definida por el usuario? ¿Por qué Docker Compose utiliza esta última?

2. En un archivo `docker-compose.yml`, el servicio `api` no declara la directiva `ports`, pero `web` logra conectarse a `api:5000` sin inconvenientes. ¿Cómo se explica este comportamiento?

3. ¿Qué mecanismo utiliza Docker para resolver el nombre de un contenedor a una dirección IP? ¿En qué tipos de red se encuentra disponible?

4. ¿Cuál es la diferencia conceptual y operativa entre las directivas `ports` y `expose` en un archivo Compose?

5. ¿Por qué constituye una mala práctica utilizar direcciones IP fijas en las cadenas de conexión entre contenedores?

## Referencias

- Docker Inc. *Networking overview*. https://docs.docker.com/engine/network/
- Docker Inc. *Bridge network driver*. https://docs.docker.com/engine/network/drivers/bridge/
- Docker Inc. *Networking in Compose*. https://docs.docker.com/compose/how-tos/networking/
- Docker Inc. *docker network command reference*. https://docs.docker.com/reference/cli/docker/network/
- Nickoloff, J., & Kuenzli, S. *Docker in Action* (2nd ed.). Manning Publications. (Capítulo 5: Single-host networking).
- Stoneman, E. *Learn Docker in a Month of Lunches*. Manning Publications.
- McKendrick, R. *Mastering Docker* (4th ed.). Packt Publishing.
