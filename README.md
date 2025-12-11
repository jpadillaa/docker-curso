# Fundamentos de Docker

Este curso introduce **Docker** desde cero para personas con bases en Linux y Bash, pero sin experiencia previa en contenedores. Aprenderá a crear y ejecutar contenedores, construir imágenes con `Dockerfile`, gestionar redes y volúmenes, automatizar entornos con **Docker Compose** y aplicar buenas prácticas de construcción y operación. 

---

## Requisitos Previos

> [!IMPORTANT]
> Para aprovechar este curso al máximo, asegúrase de cumplir con los siguientes requisitos:

* **Linux CLI:** Conocimiento básico operativo de la interfaz de línea de comandos (gestión de procesos, permisos y sistemas de archivos).
* **Fundamentos de redes:** Comprensión del modelo OSI, protocolos TCP/IP, puertos y arquitectura cliente-servidor.
* **Git:** Nociones básicas de control de versiones.

---

## Resultados de aprendizaje

Al finalizar satisfactoriamente este módulo, el estudiante estará en capacidad de:

1.  **Virtualización**
    Diferenciar las arquitecturas de virtualización de hardware vs. virtualización a nivel de sistema operativo, analizando el impacto en el *rendimiento* y la asignación de recursos.

2.  **Instalación y configuración**
    Desplegar y configurar el motor de ejecución Docker en entornos Linux, aplicando estándares de instalación segura y gestión de privilegios.

3.  **Gestión del ciclo de vida**
    Operar el ciclo de vida de los contenedores e imágenes mediante la CLI, gestionando estados, señales de terminación y depuración de procesos.

4.  **Diseño de imágenes (Dockerfiles)**
    Diseñar imágenes de contenedores optimizadas y reproducibles, implementando técnicas de **multi-stage builds** y gestión eficiente de capas (`UnionFS`).

5.  **Persistencia y redes**
    Implementar estrategias de persistencia de datos y conectividad, discriminando el uso de `volumes` frente a `bind mounts` y configurando redes definidas por software.

6.  **Orquestación con Docker Compose**
    Orquestar aplicaciones multicontenedor complejas, definiendo **Infraestructura como Código (IaC)** de manera declarativa para entornos de desarrollo y pruebas.

7.  **Seguridad en contenedores**
    Vulnerabilidades comunes y mitigaciones como el *principio de menor privilegio* y el uso de usuarios **non-root**.

8.  **Proyecto**
    Sintetizar los conocimientos en el despliegue de una arquitectura de **microservicios completa**:
    * Frontend
    * Backend
    * Base de Datos
    * Caché
    * Proxy Inverso

> [!TIP]
> El proyecto asegura la operatividad y mantenibilidad del sistema, simulando un escenario del mundo real.
