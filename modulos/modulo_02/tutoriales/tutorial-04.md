# Tutorial 03 - Contenerización de GroceryListApp: Dockerfile Optimizado

> **Objetivo:** Contenerizar una aplicación creando un Dockerfile siguiendo algunas prácticas recomendadas para entornos de producción. Se parte del Dockerfile generado en el tutorial anterior, agregando una configuración segura, eficiente y mantenible.

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

## Paso 1: Clonar el repositorio

Omita este paso si ya tiene el repositorio clonado en su equipo.

**Ejecutar en el host:**

```bash
git clone https://github.com/jpadillaa/grocerylistapp.git
cd grocerylistapp
```

## Paso 2: Crear el Dockerfile optimizado

Crea el archivo `Dockerfile` en la raíz del proyecto:

**Ejecutar en el host:**

```bash
touch Dockerfile
```

Añade el siguiente contenido optimizado:

```dockerfile
# ============================================================================
# ETAPA 1: Build - Instalación de dependencias
# ============================================================================
FROM python:3.11.9-slim-bookworm AS builder

# Evitar prompts interactivos y configurar Python
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /build

# Instalar dependencias en un virtualenv para multi-stage limpio
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copiar solo requirements para aprovechar caché de capas
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


# ============================================================================
# ETAPA 2: Runtime - Imagen final optimizada
# ============================================================================
FROM python:3.11.9-slim-bookworm AS runtime

# ------------------------------------------------------------------------------
# Metadatos de la imagen (OCI Image Spec)
# ------------------------------------------------------------------------------
LABEL org.opencontainers.image.title="GroceryListApp" \
      org.opencontainers.image.description="Aplicación web para gestionar listas de compras" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.vendor="jpadillaa" \
      org.opencontainers.image.source="https://github.com/jpadillaa/grocerylistapp" \
      org.opencontainers.image.licenses="GPL-3.0"

# ------------------------------------------------------------------------------
# Variables de entorno
# ------------------------------------------------------------------------------
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Configuración de la aplicación
    PORT=8080 \
    DATA_DIR=/data \
    DB_PATH=/data/shop.db \
    FLASK_CONFIG=app.config.Config \
    # Ruta del virtualenv
    PATH="/opt/venv/bin:$PATH"

# ------------------------------------------------------------------------------
# Instalación de dependencias del sistema y configuración de seguridad
# ------------------------------------------------------------------------------
RUN set -eux; \
    # Instalar tini para manejo correcto de señales
    apt-get update; \
    apt-get install -y --no-install-recommends \
        tini \
        curl \
    ; \
    # Limpiar caché de apt para reducir tamaño
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*; \
    # Crear usuario no-root para seguridad
    groupadd --gid 1000 appgroup; \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser; \
    # Crear directorio de datos con permisos correctos
    mkdir -p /data; \
    chown -R appuser:appgroup /data

# ------------------------------------------------------------------------------
# Copiar dependencias desde la etapa de build
# ------------------------------------------------------------------------------
COPY --from=builder /opt/venv /opt/venv

# ------------------------------------------------------------------------------
# Configurar directorio de trabajo y copiar código fuente
# ------------------------------------------------------------------------------
WORKDIR /app

# Copiar código con ownership del usuario no-root
COPY --chown=appuser:appgroup . .

# ------------------------------------------------------------------------------
# Configuración de seguridad final
# ------------------------------------------------------------------------------
# Cambiar a usuario no-root
USER appuser

# Declarar volumen para persistencia
VOLUME ["/data"]

# Exponer puerto (documentación)
EXPOSE 8080

# Health check para orquestadores
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl --fail http://localhost:${PORT}/health || exit 1

# ------------------------------------------------------------------------------
# Punto de entrada y comando
# ------------------------------------------------------------------------------
# Usar tini como init system para manejo correcto de señales y procesos zombie
ENTRYPOINT ["/usr/bin/tini", "--"]

# Comando de arranque con gunicorn
CMD ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT} --workers 2 --threads 4 --worker-class gthread --access-logfile - --error-logfile - \"app.main:app\""]
```

## Explicación de las optimizaciones

### 1. Multi-stage build

```dockerfile
FROM python:3.11.9-slim-bookworm AS builder
# ... instalación de dependencias ...

FROM python:3.11.9-slim-bookworm AS runtime
COPY --from=builder /opt/venv /opt/venv
```

| Aspecto | Beneficio |
|---------|-----------|
| **Separación de etapas** | La etapa `builder` contiene herramientas de compilación que no necesitamos en runtime. |
| **Imagen final más pequeña** | Solo se copia el virtualenv con las dependencias ya instaladas. |
| **Superficie de ataque reducida** | Menos herramientas = menos vulnerabilidades potenciales. |

### 2. Versión de imagen específica (pinning)

```dockerfile
# ❌ Evitar
FROM python:3.11-slim

# ✅ Preferir
FROM python:3.11.9-slim-bookworm
```

| Práctica | Razón |
|----------|-------|
| **Versión exacta** (`3.11.9`) | Garantiza reproducibilidad. `3.11` puede cambiar sin aviso. |
| **Variante explícita** (`bookworm`) | Identifica la versión de Debian base (Debian 12). |

### 3. Usuario no-root

```dockerfile
RUN groupadd --gid 1000 appgroup; \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

USER appuser
```

> [!CAUTION]
> Ejecutar contenedores como `root` es un riesgo de seguridad crítico. Si un atacante compromete la aplicación, tendría privilegios de root dentro del contenedor y potencialmente en el host.

**Beneficios:**

- Principio de mínimo privilegio
- Cumplimiento con estándares de seguridad (CIS Benchmark, PCI-DSS)
- Prevención de modificaciones accidentales al sistema

### 4. Init system con Tini

```dockerfile
RUN apt-get install -y --no-install-recommends tini

ENTRYPOINT ["/usr/bin/tini", "--"]
```

| Problema | Solución con Tini |
|----------|-------------------|
| **Señales no propagadas** | Tini reenvía correctamente `SIGTERM` y `SIGINT` a los procesos hijos. |
| **Procesos zombie** | Tini actúa como init (PID 1) y limpia procesos huérfanos. |
| **Apagado graceful** | Gunicorn puede terminar conexiones ordenadamente. |

> [!NOTE]
> Sin un init system adecuado, `docker stop` puede tardar 10 segundos (timeout) porque la señal `SIGTERM` no llega correctamente a Gunicorn.

### 5. Health check nativo

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl --fail http://localhost:${PORT}/health || exit 1
```

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `--interval` | 30s | Frecuencia de verificación. |
| `--timeout` | 10s | Tiempo máximo de espera por respuesta. |
| `--start-period` | 5s | Gracia inicial para que la app arranque. |
| `--retries` | 3 | Fallos consecutivos antes de marcar `unhealthy`. |

**Integración con orquestadores:**

- **Docker Swarm**: Reinicia contenedores `unhealthy` automáticamente.
- **Kubernetes**: Compatible con `livenessProbe` (aunque K8s usa su propia configuración).
- **Docker Compose**: Permite condicionar dependencias con `condition: service_healthy`.

### 6. Variables de entorno de Python optimizadas

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1
```

| Variable | Efecto |
|----------|--------|
| `PYTHONDONTWRITEBYTECODE=1` | No genera archivos `.pyc`. Reduce tamaño y evita problemas de permisos. |
| `PYTHONUNBUFFERED=1` | Output sin buffer. Los logs aparecen inmediatamente en `docker logs`. |
| `PIP_NO_CACHE_DIR=1` | No almacena caché de pip. Reduce tamaño de imagen. |
| `PIP_DISABLE_PIP_VERSION_CHECK=1` | Evita warnings sobre versiones de pip durante build. |

### 7. Metadatos OCI estándar

```dockerfile
LABEL org.opencontainers.image.title="GroceryListApp" \
      org.opencontainers.image.version="1.0.0" \
      ...
```

**Beneficios:**

- Documentación embebida en la imagen
- Trazabilidad en registries (Docker Hub, ECR, GCR)
- Integración con herramientas de escaneo de seguridad
- Información accesible via `docker inspect`

### 8. Gunicorn optimizado para contenedores

```dockerfile
CMD ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT} --workers 2 --threads 4 --worker-class gthread --access-logfile - --error-logfile - \"app.main:app\""]
```

| Opción | Valor | Justificación |
|--------|-------|---------------|
| `--workers 2` | 2 | Regla general: `2 * CPU cores + 1`. Para contenedores, 2-4 es típico. |
| `--threads 4` | 4 | Threads por worker para manejar I/O concurrente. |
| `--worker-class gthread` | gthread | Worker híbrido (procesos + threads). Eficiente para I/O bound. |
| `--access-logfile -` | stdout | Logs a stdout para `docker logs` y sistemas de logging. |
| `--error-logfile -` | stderr | Errores a stderr siguiendo convención de contenedores. |

### 9. Consolidación de comandos RUN

```dockerfile
# ❌ Evitar: Crea múltiples capas
RUN apt-get update
RUN apt-get install -y tini
RUN rm -rf /var/lib/apt/lists/*

# ✅ Preferir: Una sola capa optimizada
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends tini curl; \
    rm -rf /var/lib/apt/lists/*
```

| Práctica | Beneficio |
|----------|-----------|
| `set -eux` | Falla inmediatamente si hay error (`-e`), muestra comandos (`-x`), error en variables undefined (`-u`). |
| Consolidación | Menos capas = imagen más pequeña y builds más rápidos. |
| Limpieza en mismo RUN | Los archivos eliminados no ocupan espacio en capas anteriores. |

## Paso 3: Construir la imagen

**Ejecutar en el host:**

```bash
docker build \
  --tag grocerylistapp:1.0.0 \
  --tag grocerylistapp:latest \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  .
```

| Opción | Propósito |
|--------|-----------|
| `--tag grocerylistapp:1.0.0` | Versión específica para producción. |
| `--tag grocerylistapp:latest` | Tag de conveniencia para desarrollo. |
| `--build-arg BUILDKIT_INLINE_CACHE=1` | Habilita caché inline para builds futuros. |

**Verificación del tamaño:**

```bash
docker images grocerylistapp:1.0.0 --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

## Paso 4: Ejecutar el contenedor

**Ejecutar en el host:**

```bash
docker run -d \
  --name grocerylist \
  --restart unless-stopped \
  --memory 256m \
  --cpus 0.5 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --security-opt no-new-privileges:true \
  -p 8080:8080 \
  -v grocerylist-data:/data \
  grocerylistapp:1.0.0
```

### Explicación de opciones de seguridad y recursos

| Opción | Propósito |
|--------|-----------|
| `--restart unless-stopped` | Reinicio automático excepto si se detiene manualmente. |
| `--memory 256m` | Límite de memoria. Previene que un contenedor agote recursos del host. |
| `--cpus 0.5` | Límite de CPU (50% de un core). |
| `--read-only` | Sistema de archivos de solo lectura. Previene modificaciones maliciosas. |
| `--tmpfs /tmp:...` | Directorio temporal en memoria para archivos transitorios. |
| `--security-opt no-new-privileges:true` | Previene escalación de privilegios dentro del contenedor. |

> [!IMPORTANT]
> La opción `--read-only` requiere que todos los archivos escribibles estén en volúmenes (`/data`) o tmpfs (`/tmp`). SQLite escribe en `/data`, que está montado como volumen.

## Verificación

### 1. Estado del health check

```bash
docker inspect grocerylist --format='{{.State.Health.Status}}'
```

Espera unos 30 segundos después del inicio y deberías ver:

```text
healthy
```

Ver historial de health checks:

```bash
docker inspect grocerylist --format='{{json .State.Health}}' | jq
```

### 2. Verificar usuario no-root

```bash
docker exec grocerylist whoami
```

Salida esperada:

```text
appuser
```

### 3. Verificar que el filesystem es read-only

```bash
docker exec grocerylist touch /app/test.txt
```

Salida esperada:

```text
touch: cannot touch '/app/test.txt': Read-only file system
```

### 4. Probar el endpoint de salud

```bash
curl -s http://localhost:8080/health | jq
```

Salida esperada:

```json
{
  "status": "ok"
}
```

### 5. Verificar metadatos de la imagen

```bash
docker inspect grocerylistapp:1.0.0 --format='{{json .Config.Labels}}' | jq
```

## Comparativa: Antes vs Después

### Dockerfile original vs optimizado

| Aspecto | Original | Optimizado |
|---------|----------|------------|
| **Multi-stage** | ❌ No | ✅ Sí (builder + runtime) |
| **Usuario** | root | appuser (UID 1000) |
| **Versión de imagen** | `python:3.11-slim` | `python:3.11.9-slim-bookworm` |
| **Init system** | ❌ Ninguno | ✅ Tini |
| **Health check** | ❌ No | ✅ Nativo en Dockerfile |
| **Metadatos OCI** | ❌ No | ✅ Labels estándar |
| **Gunicorn workers** | Por defecto | Configuración explícita |
| **Logs** | Por defecto | stdout/stderr |

### Mejoras de seguridad en runtime

| Control | Original | Optimizado |
|---------|----------|------------|
| Usuario no-root | ❌ | ✅ |
| Filesystem read-only | ❌ | ✅ |
| Límites de recursos | ❌ | ✅ |
| no-new-privileges | ❌ | ✅ |
| Reinicio automático | ❌ | ✅ |

### Tamaño estimado de imagen

| Versión | Tamaño aproximado |
|---------|-------------------|
| Original (single-stage) | ~180-200 MB |
| Optimizado (multi-stage) | ~150-170 MB |

> [!TIP]
> El ahorro de tamaño en multi-stage es más significativo cuando hay dependencias de compilación (gcc, build-essential). En este caso, el beneficio principal es la separación de responsabilidades.

## Solución de problemas

### Health check fallando

**Verificar estado detallado:**

```bash
docker inspect grocerylist --format='{{json .State.Health}}' | jq '.Log[-1]'
```

**Causas comunes:**

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `connection refused` | App no ha iniciado | Aumentar `--start-period` |
| `curl not found` | curl no instalado | Verificar que está en el Dockerfile |
| `404` en `/health` | Endpoint no existe | Verificar ruta en la aplicación |

### Errores de permisos en /data

```bash
# Verificar permisos del volumen
docker run --rm -v grocerylist-data:/data alpine ls -la /data

# Si es necesario, corregir ownership
docker run --rm -v grocerylist-data:/data alpine chown -R 1000:1000 /data
```

### Container killed por OOM (Out of Memory)

```bash
# Verificar si fue OOM
docker inspect grocerylist --format='{{.State.OOMKilled}}'

# Aumentar límite de memoria
docker update --memory 512m grocerylist
```

### Gunicorn no responde a SIGTERM

Verificar que Tini está funcionando como PID 1:

```bash
docker exec grocerylist ps aux
```

Salida esperada:

```text
PID   USER     COMMAND
1     appuser  /usr/bin/tini -- sh -c gunicorn ...
7     appuser  gunicorn: master [app.main:app]
```

## Comandos de referencia rápida

| Acción | Comando |
|--------|---------|
| Ver estado del health check | `docker inspect grocerylist --format='{{.State.Health.Status}}'` |
| Ver uso de recursos | `docker stats grocerylist --no-stream` |
| Ver logs en tiempo real | `docker logs -f grocerylist` |
| Acceder al shell | `docker exec -it grocerylist /bin/bash` |
| Escanear vulnerabilidades | `docker scout cves grocerylistapp:1.0.0` |
| Ver capas de la imagen | `docker history grocerylistapp:1.0.0` |
