# Módulo 2: Uso de Docker — Comandos, Imágenes y Buenas Prácticas

> [!IMPORTANT]
> Al terminar este módulo, el lector podrá operar contenedores Docker con fluidez desde la línea de comandos, construir imágenes personalizadas mediante Dockerfiles, y aplicar buenas prácticas de optimización, seguridad y troubleshooting en sus flujos de trabajo.

## Objetivos de aprendizaje

- [ ] Gestionar el ciclo de vida completo de contenedores: descargar imágenes, instanciar, detener y eliminar contenedores
- [ ] Ejecutar contenedores en modo interactivo, attached y detached, y utilizar `docker exec` sobre contenedores activos
- [ ] Configurar tags, port mapping, volume mapping y variables de entorno con `docker run`
- [ ] Inspeccionar contenedores y consultar logs para diagnóstico
- [ ] Escribir Dockerfiles para construir imágenes de aplicaciones Python y Node.js
- [ ] Aplicar buenas prácticas de optimización, seguridad y mantenimiento de imágenes Docker

---

## Lecturas

| # | Título | Descripción |
|---|--------|-------------|
| 01 | [Comandos básicos de Docker](lecturas/lectura-01.md) | Explora la gestión de imágenes (`pull`, `images`, `rmi`) y el ciclo de vida de contenedores (`run`, `ps`, `stop`, `rm`, `exec`), incluyendo modos de ejecución attach, detach e interactivo, y comandos de limpieza del sistema. |
| 02 | [El comando `docker run`](lecturas/lectura-02.md) | Profundiza en opciones avanzadas de `docker run`: tags para control de versiones, port mapping, volume mapping con persistencia de datos (ejemplo con PostgreSQL), inspección de contenedores, logs, gestión de secrets y troubleshooting. |
| 03 | [Construcción de imágenes Docker con Dockerfile](lecturas/lectura-03.md) | Presenta los conceptos fundamentales de un Dockerfile, su anatomía e instrucciones principales, con ejemplos prácticos de construcción de imágenes para aplicaciones Python y Node.js. |
| 04 | [Buenas prácticas, optimización y troubleshooting](lecturas/lectura-04.md) | Aborda buenas prácticas para escribir Dockerfiles eficientes, técnicas de optimización de imágenes, estrategias de construcción y gestión, uso de `.dockerignore`, y resolución de problemas comunes. |

## Tutoriales

| # | Título | Descripción |
|---|--------|-------------|
| 01 | [Tutorial 01: Ejecución de contenedores Docker](tutoriales/tutorial-01.md) | En el tutorial se ejecutan contenedores Docker de forma correcta y reproducible, controlando puertos, volúmenes y variables de entorno para desplegar aplicaciones completas con docker run. |
| 02 | [Tutorial 02 - Contenerización de GroceryListApp con Docker](tutoriales/tutorial-02.md) | En el tutorial se conteneriza una aplicación creando un archivo Dockerfile. |
| 03 | [Tutorial 03 - Contenerización de GroceryListApp: Dockerfile Optimizado](tutoriales/tutorial-03.md) | En el tutorial se integran buenas prácticas al Dockerfile creado previamente . |
