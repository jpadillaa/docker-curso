# Lectura 3 — Contenedores

Un contenedor responde a la misma necesidad de **encapsular una aplicación** y su entorno de ejecución, pero lo hace con una estructura más ligera que una máquina virtual (VM). En lugar de empaquetar un **sistema operativo completo**, un contenedor incluye la **aplicación**, sus **dependencias** y las **bibliotecas** necesarias para ejecutarse. En este modelo, la virtualización ocurre a nivel de **sistema operativo**: los contenedores **comparten el kernel** del sistema operativo anfitrión, y se diferencian entre sí por mecanismos de aislamiento y control de recursos.

<img src="../assets/diagrama7.png" alt="VMs vs Contenedores" width="640">

> **Idea clave:** un contenedor es, en esencia, un **conjunto de procesos aislados** que comparten el kernel del host.

## Contenedores de aplicación vs contenedores de sistema

Es importante distinguir dos enfoques:

- **Contenedores de aplicación (application containers):** buscan empaquetar y ejecutar **un servicio o proceso principal** de la forma más ligera posible. Este es el enfoque típico de **Docker** y el estándar predominante en el desarrollo de software moderno.

- **Contenedores de sistema (system containers):** buscan emular la experiencia de un “sistema” con múltiples servicios y un `init`, acercándose al uso de una VM, pero sin la sobrecarga de virtualizar hardware. Herramientas como **LXC** y gestores como **LXD** (históricamente basado en LXC) apuntan a este caso de uso; alternativas comunitarias como **Incus** continúan esa línea.

> **Regla práctica:** para aplicaciones modernas (microservicios, CI/CD, despliegues cloud), el estándar es el **contenedor de aplicación**.

## Por qué son más ligeros y rápidos

<a href="https://www.youtube.com/watch?v=a1LW8rDB874">
  <img src="https://img.youtube.com/vi/a1LW8rDB874/hqdefault.jpg" alt="Ver video en YouTube" width="480">
</a>

Los contenedores suelen ser **rápidos de crear e iniciar** porque no requieren arrancar un kernel ni un sistema operativo invitado. Su ciclo de vida se parece al de procesos nativos del sistema operativo, ya que **son procesos** con aislamiento adicional.

Sin embargo, conviene matizar dos ideas:

- Un contenedor puede ejecutarse bajo el patrón de “un proceso principal”, pero en la práctica puede tener **más de un proceso** (por ejemplo, inicialización, procesos auxiliares o herramientas de diagnóstico).
- Aunque no cargan un SO invitado, sí existe overhead asociado a:
  - el **runtime**,
  - el **filesystem por capas** (p. ej., overlay/union),
  - la **red virtual**,
  - y procesos auxiliares (logging, sidecars, etc.) en escenarios reales.

## Aislamiento y control de recursos en Linux

A diferencia de las VMs, que se apoyan en virtualización de hardware para emular una máquina completa, los contenedores utilizan capacidades nativas del kernel del sistema operativo anfitrión. En Linux, los mecanismos principales son:

- **Namespaces:** aíslan la “vista” que tiene un proceso sobre recursos del sistema, como:
  - árbol de procesos,
  - interfaces y stack de red,
  - IDs de usuario,
  - y sistemas de archivos.

- **Control Groups (cgroups):** limitan y monitorean el consumo de recursos, como:
  - CPU,
  - memoria,
  - y E/S de disco.

> **Trade-off:** los contenedores son más livianos, pero su aislamiento depende del **kernel compartido**. En producción, esto suele complementarse con controles como **seccomp**, **AppArmor** o **SELinux**, según la plataforma.

## Ecosistema de tecnologías para contenedores

En la oferta actual existen múltiples herramientas que cubren funciones similares (construcción, ejecución, distribución), pero con diferencias técnicas relevantes. Para compararlas, conviene considerar estos ejes:

- **Construcción de imágenes** (builder)
- **Runtime de ejecución** (runtime)
- **Modelo operativo** (con o sin daemon, privilegios)
- **Integración con Kubernetes**
- **Tipo de contenedor** (aplicación vs sistema)

### Docker: Moby, BuildKit y containerd

Docker se apoya en un ecosistema modular y maduro:

- **BuildKit:** construcción eficiente de imágenes (por ejemplo, mejoras en caché y paralelismo).
- **containerd:** runtime central ampliamente adoptado.
- **Moby:** base de componentes open source sobre los que se construye el stack.

**Uso típico:** desarrollo local, CI/CD y despliegues en producción o entornos cloud.

### Podman: Buildah y runtimes OCI (p. ej., runc)

Podman y su ecosistema suelen destacarse por:

- **Arquitectura sin daemon** (modelo “daemonless”).
- Enfoque en **seguridad** y reducción de privilegios (por ejemplo, flujos rootless en escenarios compatibles).
- Integración con herramientas complementarias:
  - **Buildah** para construcción de imágenes,
  - runtimes compatibles con OCI como **runc** (y, en algunos entornos, alternativas como **crun**).

**Uso típico:** entornos que priorizan seguridad, administración alineada con prácticas Linux, y workflows flexibles.

### containerd como runtime en Kubernetes

**containerd** es un runtime ampliamente utilizado en entornos Kubernetes y puede operar de forma independiente del stack “Docker” como experiencia de usuario.

- Facilita integrar herramientas de construcción (por ejemplo, BuildKit) con la ejecución en Kubernetes.
- Favorece portabilidad al mantener una separación clara entre “construir” y “ejecutar”.

### LXC, LXD e Incus

Este grupo se asocia principalmente con **contenedores de sistema**:

- **LXC:** runtime ligero para contenedores de sistema.
- **LXD:** gestor que facilita administración (por ejemplo, perfiles, redes, almacenamiento, y funcionalidades de operación como clustering según configuración).
- **Incus:** fork comunitario asociado a la continuidad del enfoque de LXD, con énfasis en compatibilidad y migración en ciertos escenarios.

**Uso típico:** laboratorios, entornos donde se requiere administrar “sistemas” aislados con experiencia cercana a una VM, y ciertos casos de multi-servicio dentro de un mismo entorno.

## Contenedores en Windows

Microsoft introdujo soporte para contenedores en Windows Server 2016 y desde entonces ha evolucionado su integración con Docker y Kubernetes. Actualmente existen dos tipos principales:

### Windows Server Containers

- Comparten el **kernel de Windows**.
- Usan mecanismos de aislamiento y límites de recursos del sistema operativo (por ejemplo, job objects y otros controles).
- Son conceptualmente similares a los contenedores tradicionales en Linux, pero limitados a ejecutar binarios compatibles con Windows.

### Hyper-V Containers

- Ejecutan cada contenedor dentro de una **VM liviana** usando Hyper-V, con una instancia aislada del kernel.
- Mantienen una experiencia de uso similar a contenedores, pero con **mayor aislamiento**.
- Son útiles en escenarios multitenant o con requerimientos de seguridad más estrictos.

## Interoperabilidad: Open Container Initiative (OCI)

La interoperabilidad en el ecosistema de contenedores es posible gracias a la **Open Container Initiative (OCI)**, patrocinada por la Linux Foundation. OCI define especificaciones estándar para:

- el **formato de imágenes** (Image Specification),
- y el **runtime** (Runtime Specification).

Gracias a estas especificaciones, una imagen construida con una herramienta puede ser ejecutada por distintos runtimes compatibles (por ejemplo, mediante **containerd** u otros runtimes OCI). Un ejemplo de runtime de bajo nivel que implementa estas especificaciones es **runc**.

### Conexión con la motivación original

El costo de “empaquetar un sistema operativo completo por carga” en VMs motivó enfoques más livianos. Los contenedores reducen significativamente esa sobrecarga al virtualizar a nivel de sistema operativo, manteniendo portabilidad y repetibilidad del entorno con tiempos de arranque más cercanos a los de procesos nativos.

## Preguntas de autoevaluación

1. Considerando que el rendimiento y la rapidez de los contenedores provienen de ejecutarse como procesos aislados sobre un kernel compartido, ¿qué decisiones de diseño (builder, runtime, rootless, seccomp/AppArmor/SELinux, red, filesystem por capas) tomaría para equilibrar **portabilidad**, **seguridad** y **desempeño** en un despliegue real, y cuáles serían sus criterios para justificar esos trade-offs?

## Referencias

- Tanenbaum, A., & Bos, H. *Modern Operating Systems* (4th ed.). Pearson. (Capítulo: Virtualization).
- Tankersley, C. *Docker for Developers*. Leanpub Publishing.
- Raj, P., & Sing, V. *Learning Docker*. Packt Publishing.

