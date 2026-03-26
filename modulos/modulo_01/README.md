![Banner del proyecto](/modulos/assets/Banner.png)

# Módulo 1: Contenedores y Docker

> [!IMPORTANT]
> Al terminar este módulo, el lector podrá explicar por qué los contenedores se convirtieron en la unidad de despliegue estándar de la industria, construir y ejecutar imágenes Docker, y razonar sobre decisiones de diseño que afectan portabilidad, rendimiento y seguridad en entornos reales.

## Objetivos de aprendizaje

- [ ] Explicar las limitaciones de las VMs que motivaron el surgimiento de los contenedores
- [ ] Describir los mecanismos de aislamiento del kernel Linux: namespaces y cgroups
- [ ] Diferenciar imagen, contenedor y Dockerfile, y entender la relación entre ellos
- [ ] Instalar y verificar Docker en Ubuntu y en Windows con WSL 2
- [ ] Ejecutar contenedores con `docker run` y gestionar imágenes con los comandos esenciales

---

## Lecturas

| # | Título | Descripción |
|---|--------|-------------|
| 01 | [Motivación: ¿por qué contenedores?](lecturas/lectura-01.md) | Presenta el problema de portabilidad y consistencia entre entornos, el costo de las máquinas virtuales como solución previa, y cómo los contenedores abordan esas limitaciones mediante una arquitectura de kernel compartido. |
| 02 | [Virtualización](lecturas/lectura-02.md) | Explica la virtualización basada en hipervisores y máquinas virtuales: qué se virtualiza, tipos de hipervisor, ventajas y trade-offs, con énfasis en por qué este modelo abrió el camino a enfoques más livianos. |
| 03 | [Contenedores](lecturas/lectura-03.md) | Describe qué es un contenedor, cómo se diferencia de una VM, los mecanismos de aislamiento del kernel Linux (namespaces y cgroups), y un panorama del ecosistema actual de tecnologías para contenedores. |
| 04 | [Docker](lecturas/lectura-04.md) | Introduce Docker como plataforma: sus tres conceptos centrales (Dockerfile, imagen, contenedor), la arquitectura del Docker Engine, objetos fundamentales (imágenes, contenedores, volúmenes, redes), y consideraciones de seguridad y uso en producción. |

## Tutoriales

| # | Título | Descripción |
|---|--------|-------------|
| 01 | [Instalación de Docker](tutoriales/tutorial-01.md) | Guía paso a paso para instalar Docker Engine en Ubuntu (desde el repositorio oficial) y Docker Desktop en Windows con WSL 2, incluyendo verificación de la instalación y configuración post-instalación. |
| 02 | [Docker en acción](tutoriales/tutorial-02.md) | Demuestra el uso de Docker con tres ejemplos ejecutables desde un solo comando: un videojuego clásico, un entorno de desarrollo con VS Code y Python, y una plataforma de ciencia de datos con Jupyter. |
