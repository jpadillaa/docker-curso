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

## Referencias

- [Dockerfile reference oficial](https://docs.docker.com/engine/reference/builder/)
- [Best practices para Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [Docker image optimization](https://docs.docker.com/develop/dev-best-practices/)
