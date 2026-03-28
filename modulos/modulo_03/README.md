![Banner del proyecto](/modulos/assets/Banner.png)

# Módulo 3: Aplicaciones multicontenedor con Docker Compose

> [!IMPORTANT]
> Al terminar este módulo, podrá definir, levantar y operar aplicaciones compuestas por múltiples servicios usando Docker Compose, diseñar topologías de red segmentadas, gestionar persistencia y configuración de forma declarativa, y diagnosticar fallos de conectividad, arranque y configuración de manera sistemática.

## Objetivos de aprendizaje

- [ ] Definir una aplicación multicontenedor en un archivo `compose.yml` con servicios, redes, volúmenes y variables de entorno
- [ ] Levantar, detener, recrear e inspeccionar servicios con los comandos de `docker compose`
- [ ] Comprender el modelo de red de Docker: redes bridge, DNS interno y resolución por nombre de servicio
- [ ] Diferenciar entre comunicación interna entre contenedores y acceso desde el host mediante puertos publicados
- [ ] Diseñar topologías de red segmentadas para aislar servicios internos de los públicos
- [ ] Aplicar volúmenes nombrados para datos persistentes y bind mounts para iteración en desarrollo
- [ ] Configurar servicios mediante variables de entorno, archivos `.env` e interpolación en Compose
- [ ] Diagnosticar fallos de arranque, conectividad, configuración y persistencia con una estrategia ordenada

---

## Lecturas

| # | Título | Descripción |
|---|--------|-------------|
| 01 | [Docker Compose y aplicaciones multicontenedor](lecturas/lectura-01.md) | Presenta el problema de operar aplicaciones con múltiples contenedores usando solo `docker run`, introduce Docker Compose como solución declarativa, y cubre la anatomía de un archivo `compose.yml` con servicios, redes implícitas, volúmenes, `depends_on` y un ejemplo guiado con Flask, PostgreSQL y Adminer. |
| 02 | [Redes, descubrimiento de servicios y comunicación entre contenedores](lecturas/lectura-02.md) | Explica el modelo de red de Docker, las redes bridge, el DNS interno, la resolución por nombre de servicio, la diferencia entre puertos internos y publicados, el problema de `localhost` dentro de contenedores, y la segmentación con redes explícitas en Compose. Incluye un ejemplo con Nginx, Flask y PostgreSQL en dos redes. |
| 03 | [Persistencia y configuración en aplicaciones compuestas](lecturas/lectura-03.md) | Aborda la efimeridad de los contenedores, la diferencia entre volúmenes nombrados y bind mounts, estrategias de persistencia según el tipo de dato, configuración con variables de entorno y archivos `.env`, manejo de credenciales, y un ejemplo guiado que demuestra la supervivencia de datos entre ciclos de recreación. |
| PENDIENTE | [Operación y troubleshooting con Docker Compose](lecturas/lectura-04.md) | Cubre el ciclo de vida operativo de una aplicación Compose, la interpretación de estados y códigos de salida, el análisis de logs, la ejecución de comandos dentro de contenedores, las dependencias entre servicios con healthchecks, y una metodología sistemática de troubleshooting con escenarios de fallo reales. |

## Tutoriales

| # | Título | Descripción |
|---|--------|-------------|
| 01 | [Tutorial 01: Aplicación multicontenedor con Docker Compose](tutoriales/tutorial-01.md) | En el tutorial se construye una aplicación de marcadores (bookmarks) con Flask, PostgreSQL y Adminer, se define en un archivo `compose.yml`, se verifica la comunicación entre servicios por nombre, se inspecciona la topología de red y se valida la persistencia de datos con volúmenes nombrados. |
| 02 | [Tutorial 02: Persistencia, configuración y diagnóstico en aplicaciones compuestas](tutoriales/tutorial-02.md) | En el tutorial se configura una aplicación con Nginx, Flask y PostgreSQL usando dos redes segmentadas, bind mounts para desarrollo, volúmenes para persistencia y archivos `.env` para configuración, y se practican cuatro ejercicios de diagnóstico con fallos inducidos deliberadamente. |
