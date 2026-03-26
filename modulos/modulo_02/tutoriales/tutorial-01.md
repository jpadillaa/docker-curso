![Banner del proyecto](/modulos/assets/Banner.png)

# Tutorial 01 - Gestión de contenedores y servicios con Docker

> **Objetivo:** Aplicar los comandos fundamentales de Docker para desplegar, inspeccionar y gestionar un servicio web y una base de datos con persistencia de datos, simulando un escenario de desarrollo real.

## Introducción

Hasta ahora ha ejecutado contenedores de demostración con un solo comando. En este tutorial irá más allá: desplegará un servidor web Apache sirviendo contenido personalizado, levantará una base de datos PostgreSQL con persistencia real y diagnosticará problemas comunes usando las herramientas que Docker ofrece.

Al finalizar, habrá experimentado de primera mano el ciclo de vida completo de un contenedor y comprenderá por qué conceptos como el mapeo de puertos, los volúmenes y los logs son indispensables en cualquier flujo de trabajo con Docker.

### Lo que construirá

| Fase | Servicio | Conceptos clave |
|------|----------|-----------------|
| 1 | Servidor web Apache | Tags, port mapping, contenido personalizado |
| 2 | Base de datos PostgreSQL | Volúmenes, variables de entorno, persistencia |
| 3 | Diagnóstico y limpieza | `inspect`, `logs`, `exec`, limpieza del sistema |

## Prerrequisitos

- Docker instalado y en ejecución 
- Terminal con acceso a Docker (sin necesidad de `sudo` si ya configuró el grupo `docker`)
- Puertos `8080` y `5432` disponibles en su máquina
- Editor de texto

Verifique su instalación antes de comenzar:

```bash
docker --version
docker ps
```

## Paso 1: Servidor web con contenido personalizado

### 1.1. Explorar tags de la imagen httpd

Antes de ejecutar cualquier contenedor, es importante entender qué versión está utilizando. Descargue la imagen del servidor web Apache:

```bash
docker pull httpd:2.4
```

Ahora compare con la variante Alpine:

```bash
docker pull httpd:2.4-alpine
```

Liste las imágenes descargadas y observe la diferencia de tamaño:

```bash
docker images httpd
```

**Salida esperada (aproximada):**

```
REPOSITORY   TAG          IMAGE ID       CREATED       SIZE
httpd        2.4          f84fe51ff5d3   10 days ago   250MB
httpd        2.4-alpine   c7b8040505e2   3 years ago   82.8MB
```

> [!NOTE]
> La imagen Alpine es significativamente más pequeña (~83 MB vs ~250 MB) porque usa Alpine Linux como base. Para este tutorial usaremos la versión estándar `2.4` por compatibilidad.

### 1.2. Iniciar el servidor web con port mapping

Ejecute el contenedor mapeando el puerto 8080 de su máquina al puerto 80 del contenedor:

```bash
docker run -d --name mi-apache -p 8080:80 httpd:2.4
```

**Desglose del comando:**

| Opción | Función |
|--------|---------|
| `-d` | Ejecuta en segundo plano (detached) |
| `--name mi-apache` | Asigna un nombre descriptivo |
| `-p 8080:80` | Puerto 8080 del host → Puerto 80 del contenedor |
| `httpd:2.4` | Imagen y tag específico |

Verifique que el contenedor está en ejecución:

```bash
docker ps
```

Abra su navegador y visite `http://localhost:8080`. Debería ver el mensaje **"It works!"**.

### 1.3. Servir contenido personalizado con un bind mount

Detener y eliminar el contenedor actual:

```bash
docker stop mi-apache
docker rm mi-apache
```

Cree un directorio local con una página HTML personalizada:

```bash
mkdir -p ~/docker-tutorial/sitio-web
```

Cree el archivo `index.html`:

```bash
cat > ~/docker-tutorial/sitio-web/index.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Mi Servidor Docker</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 600px; margin: 50px auto; text-align: center; }
        .container { background: #f0f8ff; border-radius: 10px; padding: 30px; }
        h1 { color: #2496ED; }
        .info { background: #e8f5e9; padding: 15px; border-radius: 5px; margin-top: 20px; text-align: left; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Servidor Apache en Docker</h1>
        <p>Esta página se sirve desde un contenedor Docker con contenido montado desde el host.</p>
        <div class="info">
            <strong>Detalles técnicos:</strong>
            <ul>
                <li>Imagen: httpd:2.4</li>
                <li>Puerto del contenedor: 80</li>
                <li>Puerto del host: 8080</li>
                <li>Volumen: bind mount</li>
            </ul>
        </div>
    </div>
</body>
</html>
EOF
```

Ahora ejecute el contenedor montando su directorio local como el directorio de contenido de Apache:

```bash
docker run -d \
  --name mi-apache \
  -p 8080:80 \
  -v ~/docker-tutorial/sitio-web:/usr/local/apache2/htdocs/:ro \
  httpd:2.4
```

> [!NOTE]
> La opción `:ro` al final del volumen monta el directorio en modo **solo lectura** (read-only). Esto es una buena práctica de seguridad: el contenedor puede leer los archivos pero no modificarlos.

Visite `http://localhost:8080` de nuevo. Ahora verá su página personalizada.

### 1.4. Editar contenido en tiempo real

Una de las ventajas del bind mount es que los cambios en el host se reflejan inmediatamente. Edite el archivo HTML:

```bash
echo '<p style="color: green; font-weight: bold;">¡Contenido actualizado sin reiniciar el contenedor!</p>' >> ~/docker-tutorial/sitio-web/index.html
```

Recargue la página en el navegador. El cambio aparece sin necesidad de reiniciar el contenedor.

### ¿Qué acaba de lograr?

- Desplegó un servidor web Apache con un tag específico
- Configuró port mapping para acceder al servicio desde el host
- Montó contenido personalizado con un bind mount
- Verificó que los cambios en el host se reflejan en tiempo real

> **Para detener:** `docker stop mi-apache` (no lo haga todavía, lo usaremos en la Fase 3).

---

## Paso 2: Base de datos PostgreSQL con persistencia

### 2.1. Crear un volumen nombrado

A diferencia del bind mount de la fase anterior, usaremos un **named volume** para la base de datos. Docker administra su ubicación y es la opción recomendada para datos de producción:

```bash
docker volume create pgdata-tutorial
```

Verifique que se creó correctamente:

```bash
docker volume ls
docker volume inspect pgdata-tutorial
```

### 2.2. Iniciar PostgreSQL con variables de entorno

Lance el contenedor de PostgreSQL usando un archivo `.env` para las credenciales (en lugar de exponerlas directamente en el comando):

```bash
cat > ~/docker-tutorial/.env << 'EOF'
POSTGRES_USER=estudiante
POSTGRES_PASSWORD=docker2025
POSTGRES_DB=universidad
EOF
```

```bash
docker run -d \
  --name mi-postgres \
  -p 5432:5432 \
  --env-file ~/docker-tutorial/.env \
  -v pgdata-tutorial:/var/lib/postgresql/data \
  postgres:17.5
```

> [!WARNING]
> En un proyecto real, el archivo `.env` **nunca** debe incluirse en el repositorio de código. Agréguelo siempre a su `.gitignore`.

Verifique que el contenedor está en ejecución y saludable:

```bash
docker ps
```

Espere unos segundos a que PostgreSQL termine su inicialización. Puede monitorear el progreso con:

```bash
docker logs mi-postgres
```

Busque la línea `database system is ready to accept connections` antes de continuar.

### 2.3. Crear datos de prueba

Conéctese al contenedor y ejecute comandos SQL:

```bash
docker exec -it mi-postgres psql -U estudiante -d universidad
```

Dentro de la sesión `psql`, cree una tabla y agregue datos:

```sql
CREATE TABLE cursos (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    creditos INTEGER NOT NULL,
    semestre VARCHAR(10)
);

INSERT INTO cursos (nombre, creditos, semestre) VALUES
    ('Desarrollo Cloud', 4, '2025-2'),
    ('Ingeniería de Software', 3, '2025-2'),
    ('Bases de Datos Avanzadas', 3, '2025-1'),
    ('Inteligencia Artificial', 4, '2025-2');

SELECT * FROM cursos;
```

**Salida esperada:**

```
 id |         nombre          | creditos | semestre
----+-------------------------+----------+----------
  1 | Desarrollo Cloud        |        4 | 2025-2
  2 | Ingeniería de Software  |        3 | 2025-2
  3 | Bases de Datos Avanzadas|        3 | 2025-1
  4 | Inteligencia Artificial |        4 | 2025-2
(4 rows)
```

Salga de `psql`:

```sql
\q
```

### 2.4. Demostrar persistencia de datos

Este es el momento clave. Destruya el contenedor y recréelo con el mismo volumen:

```bash
# Detener y eliminar el contenedor
docker stop mi-postgres
docker rm mi-postgres

# Verificar que el contenedor ya no existe
docker ps -a | grep mi-postgres

# Recrear el contenedor con el mismo volumen
docker run -d \
  --name mi-postgres \
  -p 5432:5432 \
  --env-file ~/docker-tutorial/.env \
  -v pgdata-tutorial:/var/lib/postgresql/data \
  postgres:17.5
```

Espere unos segundos y verifique que los datos persisten:

```bash
docker exec -it mi-postgres psql -U estudiante -d universidad -c "SELECT * FROM cursos;"
```

Los datos siguen ahí. El contenedor es efímero; el volumen es persistente.

### ¿Qué acaba de lograr?

- Creó y gestionó un volumen nombrado
- Usó un archivo `.env` para manejar credenciales de forma segura
- Desplegó PostgreSQL con persistencia de datos
- Demostró que los datos sobreviven a la destrucción del contenedor

## Paso 3: Diagnóstico y gestión

### 3.1. Inspeccionar contenedores con docker inspect

Obtenga la dirección IP interna del contenedor de PostgreSQL:

```bash
docker inspect mi-postgres | grep -i ipaddress
```

Para obtener datos específicos de forma estructurada, use el flag `--format`:

```bash
# Dirección IP
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mi-postgres

# Puertos mapeados
docker inspect --format='{{json .NetworkSettings.Ports}}' mi-postgres

# Variables de entorno
docker inspect --format='{{json .Config.Env}}' mi-postgres
```

> [!TIP]
> Instale `jq` para filtrar la salida JSON con mayor comodidad:
> ```bash
> docker inspect mi-postgres | jq '.[0].NetworkSettings.Networks'
> ```

### 3.2. Explorar logs

Revise los logs del servidor Apache:

```bash
# Últimas 20 líneas
docker logs --tail 20 mi-apache

# Logs en tiempo real
docker logs -f mi-apache
```

En otra terminal, genere tráfico visitando `http://localhost:8080` varias veces. Observará las peticiones HTTP reflejadas en los logs en tiempo real.

Presione `Ctrl + C` para salir del modo follow.

Para los logs de PostgreSQL con timestamps:

```bash
docker logs -t --tail 10 mi-postgres
```

### 3.3. Ejecutar comandos con exec

`docker exec` permite ejecutar comandos dentro de un contenedor en ejecución sin detenerlo.

**En el contenedor de Apache:**

```bash
# Ver la versión de Apache
docker exec mi-apache httpd -v

# Listar los archivos servidos
docker exec mi-apache ls -la /usr/local/apache2/htdocs/

# Ver el sistema operativo base
docker exec mi-apache cat /etc/os-release
```

**En el contenedor de PostgreSQL:**

```bash
# Verificar que la base de datos está lista
docker exec mi-postgres pg_isready -U estudiante

# Ejecutar una consulta directa
docker exec mi-postgres psql -U estudiante -d universidad -c "SELECT nombre, creditos FROM cursos WHERE creditos >= 4;"

# Abrir una sesión interactiva de bash
docker exec -it mi-postgres bash
```

> [!NOTE]
> Dentro de la sesión bash del contenedor, puede explorar el sistema de archivos, verificar procesos con `ps aux`, o revisar la configuración de PostgreSQL. Escriba `exit` para salir.

### 3.4. Ver el uso de recursos

Revise cuánto espacio consume Docker en su sistema:

```bash
docker system df
```

**Salida esperada (aproximada):**

```
TYPE            TOTAL   ACTIVE  SIZE      RECLAIMABLE
Images          3       2       332.8MB   82.8MB (24%)
Containers      2       2       63B       0B (0%)
Local Volumes   1       1       40MB      0B (0%)
Build Cache     0       0       0B        0B
```

## Paso 4: Limpieza

### 4.1. Detener y eliminar contenedores

```bash
docker stop mi-apache mi-postgres
docker rm mi-apache mi-postgres

# Verificar
docker ps -a
```

### 4.2. Eliminar volúmenes

```bash
docker volume rm pgdata-tutorial

# Verificar
docker volume ls
```

> [!CAUTION]
> Eliminar un volumen borra permanentemente los datos que contiene. En un entorno real, realice un respaldo antes de eliminar volúmenes con datos de producción.

### 4.3. Eliminar imágenes descargadas

```bash
docker rmi httpd:2.4 httpd:2.4-alpine postgres:17.5
```

### 4.4. Limpieza general

Si desea hacer una limpieza completa de recursos no utilizados:

```bash
# Ver qué se eliminará antes de confirmar
docker system df

# Eliminar todo lo no utilizado (sin volúmenes)
docker system prune

# Incluir volúmenes huérfanos (usar con precaución)
docker system prune --volumes
```

### 4.5. Eliminar archivos locales

```bash
rm -rf ~/docker-tutorial
```

## Solución de problemas

### Puerto 8080 ya está en uso

```bash
# Identificar el proceso que ocupa el puerto
sudo lsof -i :8080

# Usar un puerto alternativo
docker run -d --name mi-apache -p 9090:80 httpd:2.4
```

### Puerto 5432 ya está en uso

```bash
# Verificar si PostgreSQL local está ejecutándose
sudo lsof -i :5432

# Detener PostgreSQL local
sudo systemctl stop postgresql

# O usar un puerto alternativo
docker run -d --name mi-postgres -p 5433:5432 ...
```

### El contenedor se detiene inmediatamente

```bash
# Ver logs para diagnosticar
docker logs mi-postgres

# Verificar el estado de salida
docker ps -a --filter "name=mi-postgres"
```

### Permiso denegado al montar volúmenes

```bash
# Verificar permisos del directorio
ls -la ~/docker-tutorial/sitio-web/

# Alternativa: usar un named volume en lugar de bind mount
docker volume create apache-content
```

## Referencias

| Recurso | Enlace |
|---------|--------|
| Docker CLI reference | [docs.docker.com/engine/reference/commandline/cli](https://docs.docker.com/engine/reference/commandline/cli/) |
| Docker volumes | [docs.docker.com/storage/volumes](https://docs.docker.com/storage/volumes/) |
| Docker networking | [docs.docker.com/network](https://docs.docker.com/network/) |
| Imagen httpd | [hub.docker.com/\_/httpd](https://hub.docker.com/_/httpd) |
| Imagen PostgreSQL | [hub.docker.com/\_/postgres](https://hub.docker.com/_/postgres) |
