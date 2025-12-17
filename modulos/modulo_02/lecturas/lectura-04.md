# Lectura 04 - Buenas prácticas, optimización y troubleshooting

## Introducción

Construir imágenes Docker es relativamente sencillo, pero construir imágenes **eficientes, seguras y mantenibles** requiere conocer y aplicar buenas prácticas. Un Dockerfile mal optimizado puede resultar en:

- 📦 **Imágenes enormes** (GBs en lugar de MBs)
- 🐢 **Construcciones lentas** (minutos u horas innecesarios)
- 🔓 **Vulnerabilidades de seguridad** (ejecución como root, secretos expuestos)
- 💸 **Costos elevados** (almacenamiento, transferencia de red, tiempo de despliegue)
- 🔄 **Cache ineficiente** (reconstrucción completa ante cambios mínimos)

Esta guía presenta las **10 mejores prácticas** para escribir Dockerfiles profesionales, técnicas de optimización comprobadas, y soluciones a los problemas más comunes.

### Impacto de las buenas prácticas

| Métrica | Sin optimizar | Optimizado | Mejora |
|---------|---------------|------------|--------|
| Tamaño de imagen | 1.2 GB | 85 MB | **93%** |
| Tiempo de build | 8 min | 45 seg | **90%** |
| Tiempo de pull | 3 min | 15 seg | **92%** |
| Superficie de ataque | Alta | Mínima | ✅ |

## Buenas prácticas

### 1. Ordenar instrucciones por frecuencia de cambio

Docker utiliza un sistema de **cache por capas**. Cada instrucción crea una capa, y si una capa cambia, todas las capas subsecuentes se reconstruyen. Por lo tanto, coloque las instrucciones que cambian menos frecuentemente al inicio del Dockerfile.

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

**Orden recomendado de instrucciones:**

```
1. FROM        → Imagen base (rara vez cambia)
2. ARG/ENV     → Variables de construcción/entorno
3. RUN         → Instalación de dependencias del sistema
4. COPY        → Archivos de dependencias (package.json, requirements.txt)
5. RUN         → Instalación de dependencias de la aplicación
6. COPY        → Código fuente (cambia frecuentemente)
7. RUN         → Comandos de build (si aplica)
8. EXPOSE      → Puertos
9. USER        → Usuario no-root
10. CMD/ENTRY  → Comando de inicio
```

> [!TIP]
> **Regla de oro**: Las instrucciones que cambian más frecuentemente deben estar al final del Dockerfile.

### 2. Combinar comandos RUN

Cada instrucción `RUN` crea una nueva capa en la imagen. Múltiples capas innecesarias aumentan el tamaño de la imagen y reducen la eficiencia del cache.

```dockerfile
# ❌ MAL - Crea 3 capas innecesarias (y el cache de apt persiste)
RUN apt-get update
RUN apt-get install -y curl git vim
RUN apt-get clean

# ✅ BIEN - Una sola capa, más eficiente y limpia
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git \
        vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

**Beneficios de combinar comandos:**

| Aspecto | Comandos separados | Comandos combinados |
|---------|-------------------|---------------------|
| Número de capas | 3+ | 1 |
| Cache de apt | Persiste (~50MB) | Eliminado |
| Tamaño total | Mayor | Menor |
| Legibilidad | Alta | Media (usar `\`) |

> [!IMPORTANT]
> Cuando elimina archivos en una instrucción `RUN` separada, **los archivos siguen ocupando espacio** en la capa anterior. Siempre limpie en la misma instrucción `RUN` donde instala.

### 3. Usar imágenes específicas y ligeras

La elección de la imagen base tiene un impacto enorme en el tamaño final y la seguridad.

```dockerfile
# ❌ MAL - Sin versión específica (impredecible)
FROM python:latest

# ⚠️ ACEPTABLE - Versión específica pero imagen grande
FROM python:3.11

# ✅ BIEN - Versión específica + imagen slim
FROM python:3.11-slim

# ✅ MEJOR - Versión específica + imagen Alpine (más ligera)
FROM python:3.11-alpine
```

**Comparación de tamaños de imagen base:**

| Imagen | Tamaño | Caso de uso |
|--------|--------|-------------|
| `python:3.11` | ~920 MB | Desarrollo, compatibilidad máxima |
| `python:3.11-slim` | ~125 MB | Producción general |
| `python:3.11-alpine` | ~49 MB | Producción optimizada |
| `node:18` | ~950 MB | Desarrollo |
| `node:18-slim` | ~240 MB | Producción general |
| `node:18-alpine` | ~170 MB | Producción optimizada |
| `ubuntu:22.04` | ~77 MB | Base general |
| `alpine:3.18` | ~7 MB | Base minimalista |
| `scratch` | 0 MB | Binarios estáticos |

> [!WARNING]
> **Alpine Linux** usa `musl libc` en lugar de `glibc`, lo que puede causar incompatibilidades con algunas librerías nativas compiladas (como ciertas extensiones de Python). Pruebe exhaustivamente antes de usar en producción.

### 4. No ejecutar como root

Por defecto, los contenedores Docker se ejecutan como `root`. Esto representa un **riesgo de seguridad significativo**: si un atacante compromete la aplicación, tiene acceso root al contenedor.

```dockerfile
# ❌ MAL - Ejecuta como root (usuario por defecto)
FROM ubuntu:22.04
COPY app.py /app/
CMD ["python", "/app/app.py"]

# ✅ BIEN - Crea y usa usuario no-root
FROM ubuntu:22.04

# Crear usuario sin privilegios
RUN useradd -m -u 1000 -s /bin/bash appuser

# Establecer directorio de trabajo
WORKDIR /app

# Copiar archivos con el usuario correcto
COPY --chown=appuser:appuser app.py .

# Cambiar a usuario no-root ANTES de CMD
USER appuser

CMD ["python", "app.py"]
```

**Crear usuario en diferentes imágenes base:**

```dockerfile
# Debian/Ubuntu
RUN useradd -m -u 1000 appuser

# Alpine
RUN adduser -D -u 1000 appuser

# Con grupo específico
RUN groupadd -g 1000 appgroup && \
    useradd -m -u 1000 -g appgroup appuser
```

> [!CAUTION]
> **Nunca ejecute contenedores como root en producción.** Esto viola el principio de mínimo privilegio y amplifica el impacto de cualquier vulnerabilidad.

### 5. Limpiar cache y archivos temporales

Los gestores de paquetes crean archivos de cache que ocupan espacio innecesario en la imagen final.

```dockerfile
# Python - Evitar cache de pip
RUN pip install --no-cache-dir -r requirements.txt

# Node.js - Limpiar cache de npm
RUN npm ci --only=production && \
    npm cache clean --force

# APT (Debian/Ubuntu) - Limpiar listas y cache
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        package1 \
        package2 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && rm -rf /var/tmp/*

# APK (Alpine) - --no-cache evita crear cache
RUN apk add --no-cache \
    package1 \
    package2
```

**Archivos comunes a eliminar:**

| Sistema | Archivos de cache | Tamaño típico |
|---------|-------------------|---------------|
| APT | `/var/lib/apt/lists/*` | 30-50 MB |
| pip | `~/.cache/pip` | 50-200 MB |
| npm | `~/.npm` | 100-500 MB |
| APK | `/var/cache/apk/*` | 10-20 MB |

### 6. Usar .dockerignore

El archivo `.dockerignore` excluye archivos del **contexto de construcción**, reduciendo el tiempo de build y evitando incluir archivos sensibles o innecesarios.

> [!TIP]
> Ver la sección [Dockerignore](#dockerignore) para ejemplos completos y patrones de exclusión.

### 7. Un proceso por contenedor

El principio de **"un proceso por contenedor"** facilita la escalabilidad, el monitoreo y la gestión del ciclo de vida.

```dockerfile
# ❌ MAL - Múltiples servicios en un contenedor
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx mysql-server
COPY start-services.sh /start.sh
CMD ["/start.sh"]  # Script que inicia nginx Y mysql

# ✅ BIEN - Un servicio por contenedor
# Dockerfile.nginx
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
CMD ["nginx", "-g", "daemon off;"]

# Dockerfile.mysql
FROM mysql:8.0
ENV MYSQL_ROOT_PASSWORD=secret
CMD ["mysqld"]
```

**Orquestar con Docker Compose:** Esta sección es solo ilustrativa; en lecturas siguientes desarrollaremos en detalle la orquestación con Docker Compose.

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.nginx
    ports:
      - "80:80"
    depends_on:
      - db

  db:
    build:
      context: .
      dockerfile: Dockerfile.mysql
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

> [!NOTE]
> **Excepciones válidas**: Procesos auxiliares como log shippers, sidecars de monitoreo, o init systems como `tini`/`dumb-init` que manejan señales correctamente.

### 8. Variables de entorno para configuración

Use variables de entorno para configurar la aplicación, permitiendo diferentes valores en desarrollo, staging y producción **sin modificar la imagen**.

```dockerfile
# Definir variables con valores por defecto
ENV APP_ENV=production \
    APP_PORT=8080 \
    DATABASE_URL=postgresql://localhost/mydb \
    LOG_LEVEL=info \
    MAX_CONNECTIONS=100 \
    WORKERS=4

# Usar en el comando
CMD ["gunicorn", "--bind", "0.0.0.0:${APP_PORT}", "--workers", "${WORKERS}", "app:app"]
```

**Sobrescribir al ejecutar:**

```bash
# Sobrescribir una variable
docker run -e DATABASE_URL=postgresql://prod-db/mydb mi-app

# Sobrescribir múltiples variables
docker run \
  -e DATABASE_URL=postgresql://prod-db/mydb \
  -e LOG_LEVEL=debug \
  -e WORKERS=8 \
  mi-app

# Usar archivo de variables
docker run --env-file .env.production mi-app
```

> [!WARNING]
> **Nunca incluya secretos (contraseñas, API keys) directamente en ENV del Dockerfile.** Los valores quedan expuestos en `docker history` y `docker inspect`. Use Docker Secrets, archivos `.env` (excluidos con `.dockerignore`), o gestores de secretos externos.

### 9. Documentar con LABEL

Los labels agregan metadatos a la imagen, facilitando la gestión, auditoría y automatización.

```dockerfile
# Labels estándar OCI (Open Container Initiative)
LABEL org.opencontainers.image.title="Mi Aplicación Web" \
      org.opencontainers.image.description="API REST para gestión de usuarios" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.authors="jesse@example.com" \
      org.opencontainers.image.url="https://example.com" \
      org.opencontainers.image.source="https://github.com/usuario/repo" \
      org.opencontainers.image.documentation="https://docs.example.com" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.created="2024-01-15T10:30:00Z" \
      org.opencontainers.image.revision="a]b1c2d3e4f5"

# Labels personalizados
LABEL com.example.team="backend" \
      com.example.maintainer="Jesse Padilla" \
      com.example.release-date="2024-01-15"
```

**Consultar labels:**

```bash
# Ver todos los labels
docker inspect --format='{{json .Config.Labels}}' mi-app:1.0 | jq

# Filtrar imágenes por label
docker images --filter "label=com.example.team=backend"
```

### 10. Healthchecks siempre

Los healthchecks permiten a Docker verificar si el contenedor está funcionando correctamente, habilitando reinicio automático y balanceo de carga inteligente.

```dockerfile
# Healthcheck con curl
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Healthcheck con wget (Alpine)
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1

# Healthcheck con script personalizado
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD /app/healthcheck.sh

# Healthcheck para base de datos PostgreSQL
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=5 \
  CMD pg_isready -U postgres || exit 1
```

**Parámetros del healthcheck:**

| Parámetro | Descripción | Valor recomendado |
|-----------|-------------|-------------------|
| `--interval` | Frecuencia de verificación | 30s |
| `--timeout` | Tiempo máximo de espera | 3-5s |
| `--start-period` | Tiempo de gracia al iniciar | 10-60s |
| `--retries` | Intentos antes de marcar unhealthy | 3 |

**Verificar estado de salud:**

```bash
# Ver estado de salud
docker ps --format "table {{.Names}}\t{{.Status}}"

# Inspeccionar healthcheck
docker inspect --format='{{json .State.Health}}' mi-contenedor | jq
```

## Optimización de imágenes

### Comparación de tamaños

La elección de la imagen base determina el tamaño mínimo de su imagen final:

```dockerfile
# Imagen completa con herramientas de desarrollo
FROM ubuntu:22.04        # ~77 MB

# Imagen slim (sin herramientas innecesarias)
FROM python:3.11-slim    # ~125 MB

# Imagen Alpine (basada en musl libc)
FROM python:3.11-alpine  # ~49 MB

# Imagen distroless (solo runtime)
FROM gcr.io/distroless/python3  # ~52 MB

# Imagen scratch (solo binario estático)
FROM scratch             # 0 MB (+ su binario)
```

### Técnicas de optimización

#### 1. Multi-stage builds

Los **multi-stage builds** permiten usar múltiples imágenes base, copiando solo los artefactos necesarios a la imagen final. Esto es especialmente útil para lenguajes compilados.

```dockerfile
# ==================== ETAPA DE CONSTRUCCIÓN ====================
FROM golang:1.21-alpine AS builder

WORKDIR /build

# Copiar archivos de dependencias
COPY go.mod go.sum ./
RUN go mod download

# Copiar código fuente y compilar
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# ==================== ETAPA DE PRODUCCIÓN ====================
FROM alpine:3.18

# Instalar certificados CA (necesarios para HTTPS)
RUN apk --no-cache add ca-certificates

WORKDIR /app

# Copiar SOLO el binario desde la etapa de construcción
COPY --from=builder /build/main .

# Usuario no-root
RUN adduser -D appuser
USER appuser

EXPOSE 8080
CMD ["./main"]
```

**Reducción de tamaño:**

| Etapa | Contenido | Tamaño |
|-------|-----------|--------|
| Builder | Go + herramientas + código | ~500 MB |
| Final | Alpine + binario | ~15 MB |
| **Reducción** | | **97%** |

> [!TIP]
> Para aplicaciones Node.js/Python, use multi-stage para instalar dependencias de desarrollo en la primera etapa y copiar solo `node_modules` o el virtualenv a la imagen final.

#### 2. Usar Alpine Linux

```dockerfile
# Antes: imagen grande
FROM node:18        # 950 MB

# Después: imagen Alpine
FROM node:18-alpine # 170 MB
```

> [!NOTE]
> Alpine usa `musl libc` en lugar de `glibc`, lo que puede causar incompatibilidades con algunas librerías compiladas. Siempre pruebe su aplicación en Alpine antes de usar en producción.

#### 3. Minimizar capas

```dockerfile
# ❌ 4 capas separadas
RUN command1
RUN command2
RUN command3
RUN command4

# ✅ 1 sola capa
RUN command1 && \
    command2 && \
    command3 && \
    command4
```

#### 4. Eliminar archivos en la misma capa

```dockerfile
# ❌ MAL - El cache de APT persiste en una capa anterior
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ BIEN - Todo se limpia en la misma capa
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

#### 5. Herramientas de análisis

```bash
# Analizar capas de una imagen
docker history mi-app:1.0

# Ver tamaño detallado por capa
docker history --no-trunc mi-app:1.0

# Herramienta dive para análisis visual detallado
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest mi-app:1.0
```

**Dive** muestra:
- Cada capa y su tamaño
- Archivos añadidos/modificados/eliminados por capa
- Eficiencia de la imagen (% de espacio desperdiciado)
- Potenciales optimizaciones

## Construcción y gestión

### Construir imágenes

```bash
# Construcción básica
docker build -t mi-app:1.0 .

# Especificar Dockerfile alternativo
docker build -t mi-app:1.0 -f Dockerfile.prod .

# Construir sin usar cache (reconstrucción completa)
docker build --no-cache -t mi-app:1.0 .

# Construir con argumentos de construcción
docker build \
  --build-arg VERSION=1.0.0 \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  -t mi-app:1.0 .

# Construir para múltiples plataformas (requiere buildx)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t mi-app:1.0 \
  --push .

# Construir solo hasta cierta etapa (multi-stage)
docker build --target builder -t mi-app:builder .

# Etiquetar con múltiples tags en una sola construcción
docker build -t mi-app:1.0 -t mi-app:latest -t mi-app:stable .

# Construcción con salida detallada
docker build --progress=plain -t mi-app:1.0 .
```

### Etiquetar imágenes

```bash
# Etiquetar imagen existente con nuevo tag
docker tag mi-app:1.0 mi-app:latest

# Etiquetar para registry privado
docker tag mi-app:1.0 registry.example.com/mi-app:1.0

# Etiquetar para Docker Hub
docker tag mi-app:1.0 usuario/mi-app:1.0

# Etiquetar para Amazon ECR
docker tag mi-app:1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/mi-app:1.0

# Etiquetar para Google Container Registry
docker tag mi-app:1.0 gcr.io/mi-proyecto/mi-app:1.0
```

### Publicar imágenes

```bash
# Login en Docker Hub
docker login

# Login en registry privado
docker login registry.example.com

# Login en Amazon ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Publicar imagen
docker push usuario/mi-app:1.0

# Publicar todas las tags de una imagen
docker push usuario/mi-app --all-tags
```

### Inspeccionar imágenes

```bash
# Ver información detallada (JSON)
docker inspect mi-app:1.0

# Ver historial de capas
docker history mi-app:1.0

# Ver historial con comandos completos
docker history --no-trunc mi-app:1.0

# Ver tamaño de imagen
docker images mi-app

# Listar con formato personalizado
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}"

# Ver labels
docker inspect --format='{{json .Config.Labels}}' mi-app:1.0 | jq

# Ver variables de entorno
docker inspect --format='{{json .Config.Env}}' mi-app:1.0 | jq

# Ver puertos expuestos
docker inspect --format='{{json .Config.ExposedPorts}}' mi-app:1.0 | jq
```

### Eliminar imágenes

```bash
# Eliminar imagen específica
docker rmi mi-app:1.0

# Eliminar imagen forzosamente (incluso si tiene contenedores)
docker rmi -f mi-app:1.0

# Eliminar imágenes sin tag (dangling)
docker image prune

# Eliminar todas las imágenes no utilizadas
docker image prune -a

# Eliminar imágenes más antiguas que 30 días (720 horas)
docker image prune -a --filter "until=720h"

# Eliminar imágenes con label específico
docker image prune -a --filter "label=environment=development"

# Ver espacio usado por Docker
docker system df

# Limpieza completa (imágenes, contenedores, volúmenes, networks)
docker system prune -a --volumes
```

> [!WARNING]
> `docker system prune -a --volumes` elimina **todo** lo no utilizado, incluyendo volúmenes con datos. Use con precaución.

## Dockerignore

El archivo `.dockerignore` funciona como `.gitignore`, excluyendo archivos del contexto de construcción. Esto:

- **Reduce tiempo de build** (menos archivos que enviar al daemon)
- **Reduce tamaño de imagen** (archivos excluidos no se copian con `COPY .`)
- **Mejora seguridad** (evita incluir archivos sensibles)

### Ejemplo completo

```dockerignore
# =====================================================
# CONTROL DE VERSIONES
# =====================================================
.git
.gitignore
.gitattributes
.svn
.hg

# =====================================================
# CI/CD
# =====================================================
.github
.gitlab-ci.yml
.travis.yml
Jenkinsfile
azure-pipelines.yml
.circleci

# =====================================================
# DEPENDENCIAS (se instalan en el contenedor)
# =====================================================
node_modules
vendor
venv
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.Python
pip-log.txt
pip-delete-this-directory.txt

# =====================================================
# ARCHIVOS DE ENTORNO Y SECRETOS
# =====================================================
.env
.env.local
.env.development
.env.test
.env.production
.env.*.local
*.env
secrets/
*.pem
*.key

# =====================================================
# LOGS
# =====================================================
*.log
logs/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# =====================================================
# SISTEMA OPERATIVO
# =====================================================
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
desktop.ini

# =====================================================
# IDE Y EDITORES
# =====================================================
.vscode
.idea
*.swp
*.swo
*.swn
*~
*.sublime-project
*.sublime-workspace
.project
.classpath
.settings/

# =====================================================
# BUILD Y DISTRIBUCIÓN
# =====================================================
dist
build
out
target
*.egg-info
*.egg
.eggs
*.whl

# =====================================================
# TESTING Y COVERAGE
# =====================================================
tests/
test/
__tests__/
*.test.js
*.test.ts
*.spec.js
*.spec.ts
coverage/
.coverage
htmlcov/
.pytest_cache/
.tox/
.nox/

# =====================================================
# DOCUMENTACIÓN
# =====================================================
docs/
doc/
*.md
!README.md
LICENSE
CHANGELOG
CONTRIBUTING

# =====================================================
# DOCKER (no necesarios dentro de la imagen)
# =====================================================
Dockerfile*
docker-compose*.yml
docker-compose*.yaml
.docker
.dockerignore

# =====================================================
# MISCELÁNEOS
# =====================================================
Makefile
*.bak
*.tmp
*.temp
.cache
```

> [!TIP]
> Un buen `.dockerignore` puede reducir el tiempo de construcción en un **50-90%** al excluir directorios grandes como `node_modules`, `.git`, etc.

### Patrones de exclusión

```dockerignore
# Excluir todo y luego incluir solo lo necesario
*
!src/
!package.json
!package-lock.json
!tsconfig.json

# Excluir por extensión
*.log
*.tmp
*.backup
*.bak

# Excluir directorios específicos
node_modules/
.git/
dist/

# Excluir con wildcard
temp*
*.local

# Comentarios
# Este es un comentario

# Negar exclusión (incluir archivo previamente excluido)
*.md
!README.md
!IMPORTANT.md

# Excluir archivos en cualquier subdirectorio
**/*.log
**/__pycache__
**/node_modules
```

## Troubleshooting

### Problema 1: Construcción lenta

**Síntomas:**
- Construcción tarda minutos u horas
- Mensaje "Sending build context to Docker daemon" muy lento

**Diagnóstico:**

```bash
# Verificar tamaño del contexto
du -sh .

# Ver qué archivos ocupan más espacio
du -sh * | sort -hr | head -20
```

**Soluciones:**

```bash
# 1. Crear/mejorar .dockerignore
cat >> .dockerignore << 'EOF'
node_modules
.git
dist
*.log
EOF

# 2. Usar contexto más específico
docker build -f docker/Dockerfile -t mi-app ./src

# 3. Habilitar BuildKit (más rápido)
DOCKER_BUILDKIT=1 docker build -t mi-app .

# 4. Usar cache mount para dependencias (BuildKit)
# syntax=docker/dockerfile:1.4
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### Problema 2: Cache no se aprovecha

**Síntoma:** Cada build reinstala todas las dependencias.

**Solución:**

```dockerfile
# ❌ MAL - Cambiar código invalida cache de dependencias
COPY . /app
RUN pip install -r requirements.txt

# ✅ BIEN - Dependencias se cachean independientemente
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt
COPY . /app
```

### Problema 3: Imagen muy grande

**Diagnóstico:**

```bash
# Ver tamaño total
docker images mi-app

# Ver tamaño por capa
docker history mi-app:1.0

# Análisis detallado con dive
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest mi-app:1.0
```

**Soluciones:**
1. Usar multi-stage builds
2. Cambiar a imagen base Alpine
3. Limpiar cache en la misma instrucción RUN
4. Eliminar archivos innecesarios
5. Usar `.dockerignore`

### Problema 4: Error "COPY failed: stat"

**Error:**
```
COPY failed: stat /var/lib/docker/tmp/docker-builder123/file.txt: no such file or directory
```

**Diagnóstico:**

```bash
# Verificar que el archivo existe
ls -la file.txt

# Verificar que NO está en .dockerignore
grep file.txt .dockerignore

# Ver el contexto efectivo
docker build --progress=plain -t test . 2>&1 | head -20
```

**Soluciones:**

```bash
# Verificar ruta relativa correcta
COPY ./src/file.txt /app/  # ✅
COPY /src/file.txt /app/   # ❌ (ruta absoluta)

# Construir con contexto diferente
docker build -f path/to/Dockerfile -t mi-app path/to/context
```

### Problema 5: Permisos denegados

**Error:**
```
permission denied while trying to connect to the Docker daemon socket
```

**Solución:**

```bash
# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Aplicar cambios (o cerrar sesión y volver a entrar)
newgrp docker

# Verificar
docker ps
```

### Problema 6: Error en Alpine vs Debian

**Error:**
```
/bin/sh: apt-get: not found
```

**Causa:** Comandos de Debian/Ubuntu no funcionan en Alpine.

**Solución:**

```dockerfile
# ❌ MAL - Sintaxis de APT en Alpine
RUN apt-get update && apt-get install -y curl

# ✅ BIEN - Sintaxis de APK
RUN apk add --no-cache curl
```

**Tabla de equivalencias:**

| Debian/Ubuntu (APT) | Alpine (APK) |
|---------------------|--------------|
| `apt-get update` | `apk update` |
| `apt-get install -y pkg` | `apk add pkg` |
| `apt-get install --no-install-recommends` | `apk add --no-cache` |
| `apt-get clean && rm -rf /var/lib/apt/lists/*` | (innecesario con `--no-cache`) |

### Problema 7: Contenedor se detiene inmediatamente

**Causa:** El proceso principal (PID 1) termina.

**Diagnóstico:**

```bash
# Ver logs
docker logs nombre-contenedor

# Ver comando configurado
docker inspect nombre-contenedor | jq '.[0].Config.Cmd'

# Ejecutar interactivamente
docker run -it mi-app /bin/sh
```

**Soluciones:**

```dockerfile
# ❌ MAL - El proceso termina
CMD echo "Hello World"

# ✅ BIEN - Proceso persistente
CMD ["python", "-u", "app.py"]
CMD ["node", "server.js"]
CMD ["nginx", "-g", "daemon off;"]

# Para debugging
CMD ["tail", "-f", "/dev/null"]
CMD ["sleep", "infinity"]
```

### Problema 8: BuildKit no habilitado

**Síntomas:**
- `RUN --mount` no funciona
- Construcción sin paralelización

**Solución:**

```bash
# Habilitar temporalmente
DOCKER_BUILDKIT=1 docker build -t mi-app .

# Habilitar permanentemente (agregar a ~/.bashrc)
export DOCKER_BUILDKIT=1

# O configurar en Docker daemon
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "features": {
    "buildkit": true
  }
}
EOF
sudo systemctl restart docker
```

## Referencias

- [Dockerfile reference oficial](https://docs.docker.com/engine/reference/builder/)
- [Best practices para Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [Docker image optimization](https://docs.docker.com/develop/dev-best-practices/)
- [Security best practices](https://docs.docker.com/engine/security/)
- [.dockerignore file](https://docs.docker.com/engine/reference/builder/#dockerignore-file)
- [Dive - Herramienta de análisis de imágenes](https://github.com/wagoodman/dive)
- [Hadolint - Linter para Dockerfiles](https://github.com/hadolint/hadolint)

> [!TIP]
> **Siguiente paso:** Implemente estas prácticas gradualmente en sus Dockerfiles existentes. Comience con `.dockerignore` y el ordenamiento de instrucciones, que ofrecen mejoras inmediatas con mínimo esfuerzo.
