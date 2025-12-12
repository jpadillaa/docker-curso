# Semana 01 – Fundamentos de contenedores y Docker

Este módulo introduce los **conceptos esenciales de contenedores** y el **uso básico de Docker**. El enfoque es práctico: instalar (a nivel conceptual) y ejecutar contenedores, comprender su relación con imágenes y registros, y dominar comandos mínimos para operar en la terminal.

---

## Objetivos de aprendizaje
- **Objetivo 1:** Comprender qué es un contenedor y qué problemas resuelve Docker en entornos de desarrollo y despliegue.
- **Objetivo 2:** Diferenciar contenedores y máquinas virtuales en términos de aislamiento, uso de recursos y portabilidad.
- **Objetivo 3:** Identificar los componentes principales de Docker (daemon, CLI, imágenes, contenedores, registros) y su rol en el flujo de trabajo.
- **Objetivo 4:** Ejecutar, inspeccionar y administrar contenedores con comandos esenciales de Docker.
- **Objetivo 5:** Ejecutar contenedores en modo interactivo y en segundo plano, incluyendo mapeo de puertos.

---

## Resultados esperados
Al finalizar este módulo, el estudiante será capaz de:

- **Outcome 1:** Explicar, con ejemplos concretos, por qué los contenedores mejoran portabilidad y reproducibilidad frente a instalaciones manuales.
- **Outcome 2:** Comparar contenedores vs. máquinas virtuales, identificando al menos **tres diferencias** y **dos casos de uso** típicos.
- **Outcome 3:** Describir la arquitectura básica de Docker (daemon, CLI, imágenes, contenedores, registro) y el flujo **pull → run → stop → remove**.
- **Outcome 4:** Ejecutar y administrar contenedores usando comandos esenciales (`docker run`, `docker ps`, `docker stop`, `docker rm`, `docker images`, `docker pull`, `docker rmi`).
- **Outcome 5:** Ejecutar un contenedor en modo interactivo (`-it`) y en modo detached (`-d`) con mapeo de puertos (`-p`), verificando su funcionamiento desde el navegador o terminal.

> Nota: Los resultados de aprendizaje están redactados en términos observables y medibles.

---

## Resumen 

### 1. Tema central
**Fundamentos de contenedores y operación básica con Docker.**  
Se estudia la motivación de Docker y se realizan primeras ejecuciones de contenedores para entender el ciclo de vida completo.

**Conexión con semanas posteriores:**  
Esta base habilita el módulo 2 (construcción de imágenes y `Dockerfile`) y prepara el terreno para redes, volúmenes y orquestación ligera con Compose (módulo 3).

### 2. Conceptos clave
- **Motivación de contenedores:** portabilidad, consistencia entre entornos, despliegues repetibles.
- **Contenedores vs. máquinas virtuales:** aislamiento, consumo de recursos, empaquetamiento y tiempos de arranque.
- **Arquitectura de Docker:** `Docker daemon`, `Docker CLI`, **imágenes**, **contenedores**, **registros** (Docker Hub).
- **Ciclo de vida del contenedor:** crear/ejecutar, listar, detener, eliminar.
- **Registros e imágenes:** descarga (`pull`), listado local, eliminación.

### 3. Actividades principales
1. **Lecturas:** Problemas que resuelve Docker y comparación contenedor vs VM.  
2. **Demostración guiada:** Arquitectura de Docker y comandos iniciales (`docker version`, `docker info`).  
3. **Laboratorio guiado:** Ejecutar un servicio web simple en contenedor (Nginx), mapear puertos, verificar acceso, y limpiar recursos.

---

## Requisitos
- **Conocimientos recomendados:**
  - Uso básico de terminal en Linux: navegación, permisos, ejecución de comandos.
  - Conceptos mínimos de red: puertos, localhost, cliente/servidor.
- **Conexión con contenidos previos del curso:**
  - Si el curso parte de Linux básico, esta semana aprovecha esa base para introducir herramientas de empaquetamiento y ejecución.
- **Cómo prepara para las siguientes semanas:**
  - Dominar el ciclo `pull/run/stop/rm` y la noción de imágenes es requisito directo para **construir imágenes** con `Dockerfile` (módulo 2) y operar ambientes multi-servicio (módulo 3).

---

## Criterios generales de evaluación
- **Criterio 1:** El estudiante explica correctamente (sin ambigüedades) qué es un contenedor y por qué se usa Docker en escenarios reales.
- **Criterio 2:** Ejecuta contenedores con comandos sencillos y demuestra control del ciclo de vida (listar, detener, eliminar).
- **Criterio 3:** Realiza una ejecución funcional de un contenedor en modo detached con mapeo de puertos, validando que el servicio responde.
- **Criterio 4:** Gestiona imágenes a nivel básico (descargar, listar y eliminar cuando aplique) y reconoce el rol de Docker Hub/registro.

---

## Notas para el profesor
- Mantener la instalación como **conceptual**: referenciar guías oficiales y advertir sobre diferencias por distribución (Ubuntu/Debian vs Fedora/CentOS).
- Enfatizar desde el inicio el hábito de **limpieza**: detener/eliminar contenedores y, si aplica, remover imágenes para evitar consumo de disco.
- Recomendación: usar Nginx por ser un ejemplo visible (navegador) y reforzar el concepto de **mapeo de puertos**.
- Variación si el entorno tiene restricciones.

---

## Evaluaciones de la semana

### Quiz (conceptos básicos)
- Contenedores y motivación de Docker.
- Contenedores vs VMs.
- Arquitectura de Docker.
- Comandos esenciales y su propósito.

### Laboratorio 
**Actividad:** Ejecutar una aplicación simple (p. ej., servidor web Nginx) mapeando puertos; listar contenedores; detenerlos y eliminarlos.  
**Evidencia mínima:** Capturas o salida de terminal mostrando:
- Ejecución del contenedor con `-d -p`.
- `docker ps` / `docker ps -a`.
- Detención (`docker stop`) y eliminación (`docker rm`).
- Verificación del servicio (respuesta en navegador o `curl`).
