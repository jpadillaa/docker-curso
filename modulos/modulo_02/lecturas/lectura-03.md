# Lectura 03 - Construcción de imágenes Docker con Dockerfile

## Prerequisitos

Antes de comenzar, asegúrese de tener:

- Docker instalado y en ejecución 
- Conocimientos básicos de Docker (contenedores, imágenes, volúmenes)
- Conocimientos básicos de línea de comandos de Linux
- Editor de texto o IDE de su preferencia
- Al menos 5 GB de espacio libre en disco
- Familiaridad con al menos un lenguaje de programación (Python, Node.js, Java, etc.)

Para verificar su instalación:

```bash
docker --version
docker info
```

## Introducción

Un **Dockerfile** es un archivo de texto plano que contiene una serie de instrucciones para construir una imagen Docker de forma automatizada y reproducible. Es el equivalente a una "receta" que especifica paso a paso cómo configurar el entorno, instalar dependencias y preparar una aplicación para ejecutarse en un contenedor.

### ¿Por qué usar Dockerfiles?

**Ventajas:**
- **Reproducibilidad**: La misma imagen se construye idénticamente en cualquier entorno
- **Versionamiento**: El Dockerfile se guarda en control de versiones (Git) junto con el código
- **Automatización**: Integración con CI/CD para construcción automática
- **Documentación**: El Dockerfile documenta el entorno y dependencias de la aplicación
- **Portabilidad**: La imagen resultante funciona en cualquier sistema con Docker

> [!NOTE]
> Un Dockerfile siempre debe llamarse exactamente `Dockerfile` (con mayúscula inicial y sin extensión) o puede usar un nombre personalizado con la opción `-f` en el comando `docker build`.

## Conceptos fundamentales
Una imagen Docker es una plantilla inmutable de solo lectura que contiene todo lo necesario para ejecutar una aplicación:
- Sistema operativo base
- Runtime (Python, Node.js, JVM, etc.)
- Librerías y dependencias
- Código de la aplicación
- Variables de entorno
- Comandos de inicio

### Arquitectura de capas

Las imágenes Docker están compuestas por **capas** (layers) apiladas:

```
┌──────────────────────────────────────────┐
│ Capa 4: CMD ["python", "app.py"]         │
├──────────────────────────────────────────┤
│ Capa 3: COPY app.py /app/                │
├──────────────────────────────────────────┤
│ Capa 2: RUN pip install flask            │
├──────────────────────────────────────────┤
│ Capa 1: FROM python:3.11                 │
└──────────────────────────────────────────┘
```

**Características importantes:**
- Cada instrucción en el Dockerfile crea una nueva capa
- Las capas son inmutables y se cachean
- Las capas se comparten entre imágenes (ahorro de espacio)
- Solo la última capa es de escritura cuando se crea un contenedor

### Contexto de construcción

El **build context** es el conjunto de archivos y directorios que Docker puede acceder durante la construcción:

```bash
docker build -t mi-app:1.0 .
                            ↑
                    contexto (directorio actual)
```

> [!WARNING]
> Docker envía **todo el contexto** al daemon antes de construir. Un contexto grande (con node_modules, .git, etc.) ralentiza significativamente la construcción. Use `.dockerignore` para excluir archivos innecesarios.

## Anatomía de un Dockerfile

Un Dockerfile está compuesto por instrucciones que se ejecutan secuencialmente. Cada instrucción crea una capa en la imagen final.

### Instrucciones básicas

#### FROM - Imagen base

Define la imagen base desde la cual construir. **Siempre es la primera instrucción** (excepto ARG).

```dockerfile
# Usar imagen oficial de Python
FROM python:3.11-slim

# Usar imagen de Node.js
FROM node:18-alpine

# Usar imagen de Ubuntu
FROM ubuntu:22.04

# Multi-stage build (múltiples FROM)
FROM node:18 AS builder
FROM nginx:alpine AS production
```

> [!TIP]
> Prefiera imágenes oficiales y use tags específicos. Las imágenes `-alpine` son más ligeras pero pueden requerir dependencias adicionales.

#### RUN - Ejecutar comandos

Ejecuta comandos durante la construcción de la imagen. Cada `RUN` crea una nueva capa.

```dockerfile
# Ejecutar un solo comando
RUN apt-get update

# Ejecutar múltiples comandos (shell form)
RUN apt-get update && apt-get install -y \
    curl \
    git \
    vim

# Ejecutar comando (exec form) - recomendado
RUN ["apt-get", "update"]

# Comando con múltiples líneas
RUN apt-get update && \
    apt-get install -y python3 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

> [!TIP]
> Combine comandos relacionados en un solo `RUN` para reducir el número de capas y el tamaño de la imagen.

#### COPY - Copiar archivos

Copia archivos o directorios del contexto de construcción a la imagen.

```dockerfile
# Copiar un archivo
COPY app.py /app/app.py

# Copiar múltiples archivos
COPY app.py requirements.txt /app/

# Copiar directorio completo
COPY ./src /app/src

# Copiar con permisos específicos
COPY --chown=appuser:appuser app.py /app/

# Copiar todos los archivos del contexto
COPY . /app/
```

#### ADD - Copiar archivos (con capacidades adicionales)

Similar a `COPY`, pero con funcionalidades adicionales:

```dockerfile
# Copiar y extraer archivo tar automáticamente
ADD archive.tar.gz /app/

# Descargar archivo desde URL
ADD https://example.com/file.zip /tmp/

# Uso normal (preferir COPY)
ADD app.py /app/
```

> [!WARNING]
> Prefiera `COPY` sobre `ADD` a menos que necesite específicamente la extracción automática de archivos tar o descarga de URLs. `COPY` es más explícito y predecible.

#### WORKDIR - Establecer directorio de trabajo

Define el directorio de trabajo para instrucciones subsecuentes.

```dockerfile
# Establecer directorio de trabajo
WORKDIR /app

# Se crea automáticamente si no existe
WORKDIR /app/src

# Todos los comandos siguientes se ejecutan en /app
RUN ls -la  # Lista contenido de /app
COPY . .    # Copia en /app
```

#### ENV - Variables de entorno

Define variables de entorno disponibles durante la construcción y en tiempo de ejecución.

```dockerfile
# Definir variable única
ENV NODE_ENV=production

# Definir múltiples variables
ENV APP_HOME=/app \
    APP_USER=appuser \
    PORT=8080

# Usar variables en el Dockerfile
ENV APP_DIR=/app
WORKDIR ${APP_DIR}
```

#### EXPOSE - Exponer puertos

Documenta qué puerto(s) escucha el contenedor (no publica el puerto).

```dockerfile
# Exponer un puerto
EXPOSE 8080

# Exponer múltiples puertos
EXPOSE 8080 8443

# Exponer puerto UDP
EXPOSE 53/udp
```

> [!NOTE]
> `EXPOSE` es solo documentación. Para publicar puertos, use `-p` en `docker run`: `docker run -p 8080:8080 mi-app`

#### CMD - Comando por defecto

Define el comando por defecto que se ejecuta cuando se inicia el contenedor.

```dockerfile
# Shell form (ejecuta en /bin/sh -c)
CMD python app.py

# Exec form (recomendado) - no invoca shell
CMD ["python", "app.py"]

# Proporcionar parámetros a ENTRYPOINT
CMD ["--port", "8080"]
```

> [!IMPORTANT]
> Solo puede haber **un CMD** en un Dockerfile. Si hay múltiples, solo el último surte efecto.

#### ENTRYPOINT - Punto de entrada

Define el ejecutable principal del contenedor. A diferencia de CMD, no se sobrescribe fácilmente.

```dockerfile
# Exec form (recomendado)
ENTRYPOINT ["python", "app.py"]

# Shell form
ENTRYPOINT python app.py

# Combinación con CMD (CMD proporciona parámetros por defecto)
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0"]
```

**Diferencia entre CMD y ENTRYPOINT:**

| Aspecto | CMD | ENTRYPOINT |
|---------|-----|------------|
| Propósito | Comando/parámetros por defecto | Ejecutable principal |
| Sobrescritura | Fácil (cualquier argumento en docker run) | Requiere --entrypoint |
| Uso típico | Parámetros por defecto | Comando principal inmutable |

### Instrucciones avanzadas

#### ARG - Argumentos de construcción

Define variables solo disponibles durante la construcción (no en runtime).

```dockerfile
# Definir argumento con valor por defecto
ARG PYTHON_VERSION=3.11
FROM python:${PYTHON_VERSION}

# Argumento sin valor por defecto
ARG BUILD_DATE
RUN echo "Build date: ${BUILD_DATE}"

# Múltiples argumentos
ARG APP_VERSION=1.0.0
ARG ENVIRONMENT=production
```

Usar al construir:

```bash
docker build --build-arg PYTHON_VERSION=3.12 --build-arg BUILD_DATE=$(date) -t mi-app .
```

#### USER - Cambiar usuario

Define el usuario (y opcionalmente grupo) para ejecutar comandos subsecuentes.

```dockerfile
# Crear usuario no-root
RUN adduser --disabled-password --gecos '' appuser

# Cambiar a usuario no-root
USER appuser

# Especificar usuario y grupo
USER appuser:appgroup

# Comandos siguientes se ejecutan como appuser
RUN whoami  # Retorna: appuser
```

> [!WARNING]
> **Nunca ejecute contenedores como root en producción**. Siempre cree y cambie a un usuario no privilegiado.

#### VOLUME - Punto de montaje

Crea un punto de montaje para volúmenes persistentes.

```dockerfile
# Declarar volumen único
VOLUME /data

# Declarar múltiples volúmenes
VOLUME ["/data", "/logs", "/config"]
```

#### LABEL - Metadatos

Agrega metadatos a la imagen en formato clave-valor.

```dockerfile
# Etiquetas individuales
LABEL version="1.0.0"
LABEL maintainer="jesse@example.com"

# Múltiples etiquetas
LABEL version="1.0.0" \
      maintainer="jesse@example.com" \
      description="Mi aplicación dockerizada" \
      org.opencontainers.image.source="https://github.com/usuario/repo"
```

#### HEALTHCHECK - Verificación de salud

Define cómo verificar si el contenedor está saludable.

```dockerfile
# Healthcheck con curl
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Healthcheck con comando personalizado
HEALTHCHECK CMD python health_check.py

# Deshabilitar healthcheck heredado
HEALTHCHECK NONE
```

#### ONBUILD - Triggers para imágenes base

Ejecuta instrucciones cuando la imagen se usa como base para otra.

```dockerfile
# En imagen base
ONBUILD COPY . /app
ONBUILD RUN npm install

# Cuando otra imagen usa FROM esta imagen, se ejecutan los ONBUILD
```

## Ejemplo práctico: Aplicación Python

Vamos a crear una aplicación Flask simple y dockerizarla.

### 1. Estructura del proyecto

```bash
flask-app/
├── app.py
├── requirements.txt
└── Dockerfile
```

### 2. Código de la aplicación

**app.py:**

```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from Docker!',
        'version': os.getenv('APP_VERSION', '1.0.0')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**requirements.txt:**

```
Flask==3.0.0
```

### 3. Dockerfile básico

```dockerfile
# Usar imagen base oficial de Python
FROM python:3.11-slim

# Establecer directorio de trabajo
WORKDIR /app

# Copiar archivo de dependencias
COPY requirements.txt .

# Instalar dependencias
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY app.py .

# Exponer puerto
EXPOSE 8080

# Variables de entorno
ENV APP_VERSION=1.0.0

# Comando por defecto
CMD ["python", "app.py"]
```

### 4. Construir la imagen

```bash
# Construir imagen
docker build -t flask-app:1.0 .

# Verificar imagen creada
docker images | grep flask-app
```

### 5. Ejecutar el contenedor

```bash
# Ejecutar contenedor
docker run -d -p 8080:8080 --name mi-flask-app flask-app:1.0

# Probar la aplicación
curl http://localhost:8080
curl http://localhost:8080/health

# Ver logs
docker logs mi-flask-app
```

### 6. Dockerfile optimizado

```dockerfile
# Usar imagen Alpine más ligera
FROM python:3.11-alpine

# Metadatos
LABEL maintainer="jesse@example.com" \
      version="1.0.0" \
      description="Flask application"

# Instalar dependencias del sistema si son necesarias
RUN apk add --no-cache gcc musl-dev linux-headers

# Crear usuario no-root
RUN adduser -D appuser

# Establecer directorio de trabajo
WORKDIR /app

# Copiar dependencias primero (mejor uso de cache)
COPY requirements.txt .

# Instalar dependencias Python
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de aplicación
COPY app.py .

# Cambiar permisos
RUN chown -R appuser:appuser /app

# Cambiar a usuario no-root
USER appuser

# Exponer puerto
EXPOSE 8080

# Variables de entorno
ENV APP_VERSION=1.0.0 \
    PYTHONUNBUFFERED=1

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"

# Comando por defecto
CMD ["python", "app.py"]
```

## Ejemplo práctico: Aplicación Node.js

### 1. Estructura del proyecto

```bash
node-app/
├── server.js
├── package.json
└── Dockerfile
```

### 2. Código de la aplicación

**package.json:**

```json
{
  "name": "node-docker-app",
  "version": "1.0.0",
  "description": "Node.js Docker example",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

**server.js:**

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Node.js Docker!',
    timestamp: new Date().toISOString(),
    nodeVersion: process.version
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

### 3. Dockerfile optimizado

```dockerfile
# Etapa 1: Construcción
FROM node:18-alpine AS builder

WORKDIR /app

# Copiar archivos de dependencias
COPY package*.json ./

# Instalar dependencias de producción
RUN npm ci --only=production

# Etapa 2: Producción
FROM node:18-alpine

# Instalar dumb-init para manejo apropiado de señales
RUN apk add --no-cache dumb-init

# Crear usuario no-root
RUN adduser -D nodeuser

# Establecer directorio de trabajo
WORKDIR /app

# Copiar node_modules desde etapa de construcción
COPY --from=builder --chown=nodeuser:nodeuser /app/node_modules ./node_modules

# Copiar código de aplicación
COPY --chown=nodeuser:nodeuser package*.json ./
COPY --chown=nodeuser:nodeuser server.js ./

# Cambiar a usuario no-root
USER nodeuser

# Exponer puerto
EXPOSE 3000

# Variables de entorno
ENV NODE_ENV=production \
    PORT=3000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Usar dumb-init para manejar señales correctamente
ENTRYPOINT ["dumb-init", "--"]

# Comando por defecto
CMD ["node", "server.js"]
```

## Buenas prácticas

### 1. Ordenar instrucciones por frecuencia de cambio

Coloque las instrucciones que cambian menos frecuentemente al inicio para aprovechar el cache.

```dockerfile
# ❌ MAL - Invalida cache en cada cambio de código
FROM python:3.11-slim
COPY . /app
RUN pip install -r requirements.txt

# ✅ BIEN - Cache se mantiene si solo cambia el código
FROM python:3.11-slim
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt
COPY . /app
```

### 2. Combinar comandos RUN

```dockerfile
# ❌ MAL - Crea 3 capas innecesarias
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ BIEN - Una sola capa, más eficiente
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Usar imágenes específicas y ligeras

```dockerfile
# ❌ MAL - Sin versión específica
FROM python:latest

# ⚠️ ACEPTABLE - Versión específica pero imagen grande
FROM python:3.11

# ✅ BIEN - Versión específica + imagen slim
FROM python:3.11-slim

# ✅ MEJOR - Versión específica + imagen Alpine (más ligera)
FROM python:3.11-alpine
```

### 4. No ejecutar como root

```dockerfile
# ❌ MAL - Ejecuta como root (usuario por defecto)
FROM ubuntu:22.04
COPY app.py /app/
CMD ["python", "/app/app.py"]

# ✅ BIEN - Crea y usa usuario no-root
FROM ubuntu:22.04
RUN useradd -m -u 1000 appuser
USER appuser
COPY --chown=appuser:appuser app.py /app/
CMD ["python", "/app/app.py"]
```

### 5. Limpiar cache y archivos temporales

```dockerfile
# Python
RUN pip install --no-cache-dir -r requirements.txt

# Node.js
RUN npm ci --only=production && npm cache clean --force

# APT (Debian/Ubuntu)
RUN apt-get update && \
    apt-get install -y package && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# APK (Alpine)
RUN apk add --no-cache package
```

### 6. Usar .dockerignore

Siempre cree un archivo `.dockerignore` para excluir archivos innecesarios del contexto.

### 7. Un proceso por contenedor

```dockerfile
# ❌ MAL - Múltiples servicios en un contenedor
CMD service nginx start && service mysql start

# ✅ BIEN - Un servicio por contenedor
# Use Docker Compose para orquestar múltiples contenedores
CMD ["nginx", "-g", "daemon off;"]
```

### 8. Variables de entorno para configuración

```dockerfile
# Usar ENV para valores configurables
ENV DATABASE_URL=postgresql://localhost/mydb \
    LOG_LEVEL=info \
    MAX_CONNECTIONS=100

# Sobrescribir al ejecutar
# docker run -e DATABASE_URL=postgresql://prod/db app
```

### 9. Documentar con LABEL

```dockerfile
LABEL maintainer="jesse@example.com" \
      version="1.0.0" \
      description="Aplicación web Flask" \
      org.opencontainers.image.source="https://github.com/user/repo" \
      org.opencontainers.image.licenses="MIT"
```

### 10. Healthchecks siempre

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

## Optimización de imágenes

### Comparación de tamaños

```bash
# Imagen completa
FROM ubuntu:22.04        # ~77 MB

# Imagen slim
FROM python:3.11-slim    # ~125 MB

# Imagen Alpine
FROM python:3.11-alpine  # ~49 MB

# Imagen scratch (solo binario estático)
FROM scratch             # ~5-10 MB
```

### Técnicas de optimización

#### 1. Multi-stage builds

Ya visto anteriormente - reduce tamaño al 90-95%.

#### 2. Usar Alpine Linux

```dockerfile
# Cambiar de imagen grande a Alpine
FROM node:18        # 950 MB
FROM node:18-alpine # 170 MB
```

> [!NOTE]
> Alpine usa `musl libc` en lugar de `glibc`, lo que puede causar incompatibilidades con algunas librerías compiladas.

#### 3. Minimizar capas

```dockerfile
# ❌ 4 capas
RUN command1
RUN command2
RUN command3
RUN command4

# ✅ 1 capa
RUN command1 && \
    command2 && \
    command3 && \
    command4
```

#### 4. Eliminar archivos en la misma capa

```dockerfile
# ❌ MAL - El cache de APT persiste en una capa
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ BIEN - Se limpia en la misma capa
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

#### 5. Herramientas de análisis

```bash
# Analizar capas de una imagen
docker history mi-app:1.0

# Herramienta dive para análisis detallado
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest mi-app:1.0
```

## Construcción y gestión

### Construir imágenes

```bash
# Construcción básica
docker build -t mi-app:1.0 .

# Especificar Dockerfile alternativo
docker build -t mi-app:1.0 -f Dockerfile.prod .

# Construir sin usar cache
docker build --no-cache -t mi-app:1.0 .

# Construir con build args
docker build --build-arg VERSION=1.0 --build-arg ENV=prod -t mi-app:1.0 .

# Construir para múltiples plataformas
docker buildx build --platform linux/amd64,linux/arm64 -t mi-app:1.0 .

# Construir solo hasta cierta etapa (multi-stage)
docker build --target builder -t mi-app:builder .

# Etiquetar múltiples tags en la construcción
docker build -t mi-app:1.0 -t mi-app:latest .
```

### Etiquetar imágenes

```bash
# Etiquetar imagen existente
docker tag mi-app:1.0 mi-app:latest

# Etiquetar para registry privado
docker tag mi-app:1.0 registry.example.com/mi-app:1.0

# Etiquetar para Docker Hub
docker tag mi-app:1.0 usuario/mi-app:1.0
```

### Inspeccionar imágenes

```bash
# Ver información detallada
docker inspect mi-app:1.0

# Ver historial de capas
docker history mi-app:1.0

# Ver tamaño de imagen
docker images mi-app

# Filtrar con formato
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Eliminar imágenes

```bash
# Eliminar imagen específica
docker rmi mi-app:1.0

# Eliminar imagen forzosamente
docker rmi -f mi-app:1.0

# Eliminar imágenes sin tag
docker image prune

# Eliminar todas las imágenes no utilizadas
docker image prune -a

# Eliminar imágenes más antiguas que X
docker image prune -a --filter "until=720h"
```

## Dockerignore

El archivo `.dockerignore` funciona como `.gitignore`, excluyendo archivos del contexto de construcción.

### Ejemplo completo

```dockerignore
# Archivos de control de versiones
.git
.gitignore
.gitattributes

# Archivos de CI/CD
.github
.gitlab-ci.yml
Jenkinsfile

# Dependencias (se instalan en el contenedor)
node_modules
venv
__pycache__
*.pyc
*.pyo
*.pyd
.Python

# Archivos de entorno
.env
.env.local
.env.*.local
*.env

# Archivos de logs
*.log
logs
npm-debug.log*
yarn-debug.log*

# Archivos de sistema operativo
.DS_Store
Thumbs.db
desktop.ini

# Archivos de IDE
.vscode
.idea
*.swp
*.swo
*~

# Archivos de construcción
dist
build
*.egg-info

# Archivos de documentación
README.md
docs
*.md

# Archivos de testing
tests
test
*.test.js
coverage

# Docker files (no necesarios dentro de la imagen)
Dockerfile*
docker-compose*.yml
.dockerignore
```

> [!TIP]
> Un buen `.dockerignore` puede reducir el tiempo de construcción en un 50-90% al excluir archivos grandes innecesarios como `node_modules`, `.git`, etc.

### Patrones de exclusión

```dockerignore
# Excluir todo y luego incluir específicos
*
!src
!package.json
!package-lock.json

# Excluir archivos por extensión
*.log
*.tmp
*.backup

# Excluir directorios
node_modules
.git
dist

# Comentarios
# Este es un comentario

# Negar exclusión (incluir archivo)
!important-file.txt
```

## Troubleshooting

### Problema 1: Construcción lenta

**Síntomas:**
- Construcción tarda minutos u horas
- Mensaje "Sending build context to Docker daemon"

**Causas y soluciones:**

```bash
# Verificar tamaño del contexto
du -sh .

# Revisar qué archivos se están enviando
docker build --progress=plain -t mi-app .

# Crear/mejorar .dockerignore
echo "node_modules" >> .dockerignore
echo ".git" >> .dockerignore

# Usar cache mount (BuildKit)
# RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

### Problema 2: Cache no se aprovecha

**Solución: Ordenar instrucciones correctamente**

```dockerfile
# ❌ MAL - Cambiar código invalida cache de dependencias
COPY . /app
RUN pip install -r requirements.txt

# ✅ BIEN - Dependencias se cachean
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app
```

### Problema 3: Imagen muy grande

**Diagnóstico:**

```bash
# Ver tamaño por capa
docker history mi-app:1.0

# Analizar con dive
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest mi-app:1.0
```

**Soluciones:**
1. Usar multi-stage builds
2. Cambiar a imagen Alpine
3. Limpiar cache en la misma capa RUN
4. Eliminar archivos innecesarios

### Problema 4: Error "COPY failed: stat"

**Error:**
```
COPY failed: stat /var/lib/docker/tmp/docker-builder123/file.txt: no such file or directory
```

**Causa:** El archivo no existe en el contexto de construcción.

**Solución:**

```bash
# Verificar que el archivo existe
ls -la file.txt

# Verificar que no está en .dockerignore
cat .dockerignore | grep file.txt

# Construir con contexto diferente
docker build -f path/to/Dockerfile -t mi-app path/to/context
```

### Problema 5: Permisos denegados en COPY

**Error:**
```
permission denied while trying to connect to the Docker daemon socket
```

**Solución:**

```bash
# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Cerrar sesión y volver a entrar
# O recargar grupos
newgrp docker

# Verificar
docker ps
```

### Problema 6: Error al instalar paquetes en Alpine

**Error:**
```
ERROR: unable to select packages
```

**Causa:** Diferencias entre APT (Debian/Ubuntu) y APK (Alpine).

**Solución:**

```dockerfile
# ❌ MAL - Sintaxis de APT en Alpine
RUN apt-get install python3

# ✅ BIEN - Sintaxis de APK
RUN apk add --no-cache python3

# Tabla de equivalencias comunes
# apt-get update          → apk update
# apt-get install package → apk add package
# apt-get clean           → (innecesario con --no-cache)
```

### Problema 7: Contenedor se detiene inmediatamente

**Causa:** El proceso principal termina.

**Diagnóstico:**

```bash
# Ver logs del contenedor
docker logs container-name

# Ver el comando ejecutado
docker inspect container-name | grep -A 5 Cmd

# Ejecutar interactivamente para depurar
docker run -it mi-app /bin/sh
```

**Solución:**

```dockerfile
# ❌ MAL - Proceso termina
CMD echo "Hello"

# ✅ BIEN - Proceso persistente
CMD ["python", "app.py"]

# Para debugging
CMD ["tail", "-f", "/dev/null"]
```

### Problema 8: BuildKit no está habilitado

**Síntomas:**
- No funciona `RUN --mount`
- No hay salida detallada de construcción

**Solución:**

```bash
# Habilitar BuildKit temporalmente
DOCKER_BUILDKIT=1 docker build -t mi-app .

# Habilitar BuildKit permanentemente
# Agregar a ~/.bashrc o ~/.zshrc
export DOCKER_BUILDKIT=1

# O configurar en daemon.json
sudo nano /etc/docker/daemon.json
# Agregar: { "features": { "buildkit": true } }
sudo systemctl restart docker
```

## Referencias

- [Dockerfile reference oficial](https://docs.docker.com/engine/reference/builder/)
- [Best practices para Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [Docker image optimization](https://docs.docker.com/develop/dev-best-practices/)
- [.dockerignore file](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

