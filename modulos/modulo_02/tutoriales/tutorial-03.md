# Tutorial 02 - Contenerización de GroceryListApp con Docker

> **Objetivo:** Contenerizar una aplicación creando un Dockerfile correcto y mantenible, definiendo imagen base, dependencias, copiado de código, configuración de ejecución y exposición de puertos para generar una imagen lista para despliegue.

## Prerequisitos

Antes de comenzar, asegúrese de cumplir con los siguientes requisitos:

| Requisito | Verificación |
|-----------|--------------|
| Docker instalado | `docker --version` |
| Docker daemon en ejecución | `docker info` |
| Conexión a internet | Necesaria para descargar imágenes |
| Puerto 8080 disponible | `lsof -i :8080` (debe estar vacío) |
| Navegador web | Para acceder a la aplicación |

**Verificar instalación de Docker:**

```bash
docker --version
```

Salida esperada:
```
Docker version 24.0.x, build xxxxxxx
```

> [!WARNING]
> Si el comando `docker` no es reconocido o muestra errores de permisos, consulte la guía de instalación de Docker para su sistema operativo.

## Escenario 

Este laboratorio es un paso a paso para contenerizar **GroceryListApp**, una aplicación web Flask para gestionar listas de compras con persistencia en SQLite.

## Paso 1: Clonar el repositorio

Descargue el código fuente de la aplicación desde GitHub.

**Ejecutar en el host:**

```bash
git clone https://github.com/jpadillaa/grocerylistapp.git
```

Ingrese al directorio del proyecto:

```bash
cd grocerylistapp
```

**Verificación:** Confirma que el repositorio se clonó correctamente listando su contenido:

```bash
ls -la
```

Debería ver una estructura similar a:

```text
.
├── app/
├── static/
├── templates/
├── tests/
├── requirements.txt
├── README.md
└── LICENSE
```

## Paso 2: Crear el Dockerfile

El `Dockerfile` define cómo construir la imagen del contenedor. Cree el archivo en la raíz del proyecto.

**Ejecutar en el host:**

```bash
touch Dockerfile
```

Abra el archivo con tu editor preferido y añade el siguiente contenido:

```dockerfile
FROM python:3.11-slim

# Configurar directorio de trabajo
WORKDIR /app

# Instalar dependencias
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar el código de la aplicación
COPY . .

# Variables de entorno por defecto
ENV PORT=8080
ENV DATA_DIR=/data
ENV DB_PATH=/data/shop.db
ENV FLASK_CONFIG=app.config.Config

# Asegurar que el directorio de datos existe
RUN mkdir -p /data

# Exponer puerto
EXPOSE 8080

# El comando de arranque usa gunicorn
# La inicialización de la DB ocurre dentro del create_app() en app/__init__.py
CMD ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT} \"app.main:app\""]
```

### Explicación línea por línea

| Instrucción | Propósito |
|-------------|-----------|
| `FROM python:3.11-slim` | Usa una imagen base Python 3.11 optimizada (slim reduce el tamaño ~100MB vs la versión completa). |
| `WORKDIR /app` | Establece `/app` como directorio de trabajo. Los comandos siguientes se ejecutan desde aquí. |
| `COPY requirements.txt .` | Copia **solo** el archivo de dependencias primero para aprovechar la caché de capas de Docker. |
| `RUN pip install --no-cache-dir ...` | Instala dependencias sin almacenar caché de pip (reduce tamaño de imagen). |
| `COPY . .` | Copia el resto del código fuente. Al estar después del `pip install`, cambios en el código no invalidan la capa de dependencias. |
| `ENV PORT=8080` | Define variables de entorno con valores por defecto. Pueden sobrescribirse en tiempo de ejecución. |
| `RUN mkdir -p /data` | Crea el directorio para la base de datos SQLite. La opción `-p` evita errores si ya existe. |
| `EXPOSE 8080` | Documenta el puerto que la aplicación usa. Es informativo, no abre el puerto automáticamente. |
| `CMD [...]` | Define el comando por defecto al iniciar el contenedor. Usa `sh -c` para expandir la variable `${PORT}`. |

> [!IMPORTANT]
> El orden de las instrucciones `COPY` está optimizado para aprovechar la **caché de capas** de Docker. Las dependencias (que cambian con poca frecuencia) se instalan antes de copiar el código fuente (que cambia frecuentemente).

## Paso 3: Crear el archivo .dockerignore

El archivo `.dockerignore` excluye archivos innecesarios del contexto de build, reduciendo el tamaño de la imagen y acelerando la construcción.

**Ejecutar en el host:**

```bash
touch .dockerignore
```

Añada el siguiente contenido:

```text
# Control de versiones
.git
.gitignore

# Entornos virtuales de Python
venv/
.venv/
__pycache__/
*.py[cod]
*.pyo

# Archivos de IDE/Editor
.vscode/
.idea/
*.swp
*.swo
.DS_Store

# Tests y documentación (no necesarios en producción)
tests/
*.md
!README.md

# Archivos de Docker (evita recursión)
Dockerfile
docker-compose*.yml
.dockerignore

# Datos locales (la DB debe persistir en volúmenes, no en la imagen)
data/
*.db

# Archivos de configuración local
.env
.env.*
```

**Beneficios de un `.dockerignore` bien configurado:**

- Reduce el tiempo de build al enviar menos archivos al daemon de Docker
- Disminuye el tamaño final de la imagen
- Evita incluir secretos o datos sensibles accidentalmente
- Previene conflictos entre archivos locales y del contenedor

## Paso 4: Construir la imagen

Con el `Dockerfile` y `.dockerignore` en su lugar, construya la imagen Docker.

**Ejecutar en el host:**

```bash
docker build -t grocerylistapp:1.0.0 .
```

| Parámetro | Descripción |
|-----------|-------------|
| `-t grocerylistapp:1.0.0` | Asigna nombre (`grocerylistapp`) y etiqueta de versión (`1.0.0`) a la imagen. |
| `.` | Indica que el contexto de build es el directorio actual. |

> [!TIP]
> Usa siempre etiquetas de versión específicas (ej: `1.0.0`) en lugar de `latest` para garantizar reproducibilidad en despliegues.

**Verificación:** Confirma que la imagen se creó correctamente:

```bash
docker images | grep grocerylistapp
```

Salida esperada:

```text
grocerylistapp   1.0.0   abc123def456   2 minutes ago   180MB
```

## Paso 5: Ejecutar el contenedor

Inicie un contenedor a partir de la imagen construida.

**Ejecutar en el host:**

```bash
docker run -d \
  --name grocerylist \
  -p 8080:8080 \
  -v grocerylist-data:/data \
  grocerylistapp:1.0.0
```

| Parámetro | Descripción |
|-----------|-------------|
| `-d` | Ejecuta el contenedor en segundo plano (detached). |
| `--name grocerylist` | Asigna un nombre al contenedor para fácil referencia. |
| `-p 8080:8080` | Mapea el puerto 8080 del host al puerto 8080 del contenedor. |
| `-v grocerylist-data:/data` | Monta un volumen nombrado para persistir la base de datos SQLite. |

> [!WARNING]
> Sin el volumen `-v grocerylist-data:/data`, **los datos se perderán** cuando el contenedor se detenga o elimine. SQLite almacena la base de datos en `/data/shop.db`.

**Verificación del estado:**

```bash
docker ps
```

Debería ver el contenedor en ejecución:

```text
CONTAINER ID   IMAGE                  STATUS          PORTS
a1b2c3d4e5f6   grocerylistapp:1.0.0   Up 30 seconds   0.0.0.0:8080->8080/tcp
```

## Verificación

Confirme que la aplicación funciona correctamente realizando las siguientes pruebas:

### 1. Verificar los logs del contenedor

```bash
docker logs grocerylist
```

Busque líneas que indiquen que Gunicorn inició correctamente:

```text
[INFO] Starting gunicorn 21.x.x
[INFO] Listening at: http://0.0.0.0:8080
```

### 2. Probar el endpoint de salud

```bash
curl http://localhost:8080/health
```

Respuesta esperada:

```json
{"status":"ok"}
```

### 3. Acceder a la interfaz web

Abra su navegador y visita: [http://localhost:8080](http://localhost:8080)

Debería ver la página principal de la lista de compras.

### 4. Verificar persistencia de datos

1. Cree algunos ítems en la aplicación
2. Detenga y reinicia el contenedor:

   ```bash
   docker stop grocerylist
   docker start grocerylist
   ```

3. Verifique que los ítems creados persisten después del reinicio

## Solución de problemas

### El contenedor no inicia

**Verificar logs detallados:**

```bash
docker logs grocerylist --tail 50
```

**Causas comunes:**

| Síntoma | Posible causa | Solución |
|---------|---------------|----------|
| `ModuleNotFoundError` | Dependencias no instaladas | Reconstruir imagen: `docker build --no-cache -t grocerylistapp:1.0.0 .` |
| `Permission denied` en `/data` | Problemas de permisos | Verificar que el volumen se creó correctamente |
| `Address already in use` | Puerto 8080 ocupado | Usar otro puerto: `-p 8081:8080` |

### La base de datos no persiste

Verifique que el volumen existe:

```bash
docker volume ls | grep grocerylist-data
```

Si no existe, créalo manualmente antes de ejecutar el contenedor:

```bash
docker volume create grocerylist-data
```

## Comandos de referencia rápida

| Acción | Comando |
|--------|---------|
| Detener contenedor | `docker stop grocerylist` |
| Iniciar contenedor | `docker start grocerylist` |
| Reiniciar contenedor | `docker restart grocerylist` |
| Ver logs en tiempo real | `docker logs -f grocerylist` |
| Acceder al shell del contenedor | `docker exec -it grocerylist /bin/bash` |
| Eliminar contenedor | `docker rm -f grocerylist` |
| Eliminar imagen | `docker rmi grocerylistapp:1.0.0` |
| Eliminar volumen de datos | `docker volume rm grocerylist-data` |
