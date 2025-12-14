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

Al finalizar satisfactoriamente este curso, el estudiante estará en capacidad de:

1.  Diferenciar las arquitecturas de virtualización de hardware vs. virtualización a nivel de sistema operativo, analizando el impacto en el *rendimiento* y la asignación de recursos.

2.  Desplegar y configurar el motor de ejecución Docker en entornos Linux, aplicando estándares de instalación segura y gestión de privilegios.

3.  Operar el ciclo de vida de los contenedores e imágenes mediante la CLI, gestionando estados, señales de terminación y depuración de procesos.

4.  Diseñar imágenes de contenedores optimizadas y reproducibles, implementando técnicas de **multi-stage builds** y gestión eficiente de capas.

5. Implementar estrategias de persistencia de datos y conectividad, discriminando el uso de `volumes` frente a `bind mounts` y configurando redes definidas por software.

6.  Orquestar aplicaciones multicontenedor complejas, definiendo **Infraestructura como Código (IaC) con Docker Compose** de manera declarativa para entornos de desarrollo y pruebas.

7.  Indentificar vulnerabilidades comunes y mitigaciones como el *principio de menor privilegio* y el uso de usuarios **non-root**.

8.  **Proyecto**
    Sintetizar los conocimientos en el despliegue de una arquitectura de **microservicios completa**:
    * Frontend
    * Backend
    * Base de Datos
    * Caché
    * Proxy Inverso

> [!TIP]
> El proyecto asegura la operatividad y mantenibilidad del sistema, simulando un escenario del mundo real.

---

## Módulos

| Semana | Título del módulo | Foco | Actividades |
|:-----:|-------------------|----------------|------------------------|
| **1** | [Fundamentos de contenedores y Docker](./modulos/modulo_01/) | Conceptos, instalación, primeros comandos | Laboratorio guiado de comandos de Docker|
| **2** | Imágenes y Dockerfile | Construcción de imágenes, capas, tags | Laboratorio de Dockerfile |
| **3** | Volúmenes, redes y Docker Compose | Persistencia, redes, multi-servicio | Laboratorio mini-proyecto con Docker Compose |
| **4** | Buenas prácticas con Docker | Optimización, organización, observabilidad básica | Laboratorio de refactorización |
| **5** | Seguridad en contenedores | Principios de seguridad | Proyecto |

