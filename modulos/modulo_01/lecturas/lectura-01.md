# Lectura 1 - Motivación: ¿por qué contenedores?

El desarrollo de software enfrenta desafíos recurrentes asociados con la **gestión de dependencias**, la **portabilidad entre entornos** y la **eficiencia en el uso de recursos computacionales**. La frase **“funciona en mi máquina”** refleja una situación frecuente: una aplicación opera correctamente en desarrollo, pero falla en pruebas o producción por diferencias sutiles en:

- versiones de bibliotecas
- variables de entorno
- configuración del sistema
- servicios auxiliares

---

## Máquinas virtuales: mitigación con overhead

Durante años, las **máquinas virtuales (VMs)** mitigaron parte del problema al encapsular un **sistema operativo completo** junto con la aplicación. Sin embargo, esta aproximación introduce costos relevantes:

- **Overhead significativo**: (duplicación de SO)
- **Mayor consumo**: de memoria y almacenamiento
- **Complejidad operacional**: que puede limitar velocidad y eficiencia en despliegues modernos

> [!WARNING]
> La virtualización completa suele ser efectiva, pero puede ser ineficiente cuando se requiere **escalar rápidamente** o desplegar con alta frecuencia.

---

## Contenedores: portabilidad con menor overhead

En este contexto, los **contenedores** emergen como una solución ligera y eficiente, dado que permiten **empaquetar** una aplicación con sus dependencias y configuración de ejecución, favoreciendo un despliegue consistente entre entornos.

### ¿Qué aporta un contenedor?
- Encapsula lo necesario para ejecutar una aplicación
- Reduce interferencias con otras aplicaciones y con el sistema anfitrión
- Disminuye fricciones operativas al estandarizar el ambiente

> [!TIP]
> Piense en contenedores como una forma de **definir explícitamente** la base de ejecución, en vez de depender del “estado” del entorno.

En un mismo host (servidor, máquina virtual o entorno local), múltiples contenedores:
- comparten el **mismo kernel Linux** del anfitrión
- operan en **espacios de usuario aislados**

### Ejemplo
Un servicio web que funciona localmente puede fallar en producción si allí existe:
- otra versión de Python/Java
- librerías diferentes
- variables de entorno faltantes

Al **contenerizar** el servicio, se define explícitamente la base de ejecución y sus dependencias, reduciendo la variabilidad entre ambientes.
> [!IMPORTANT]
> Esta arquitectura compartida de los contenedores tiene implicaciones directas en **eficiencia**, **velocidad** y **portabilidad**.

---
## Implicaciones de la arquitectura compartida de los contenedores

### Eficiencia de recursos
Al eliminar la necesidad de un **sistema operativo invitado** por cada instancia, los contenedores reducen drásticamente la huella de:

- memoria
- almacenamiento

> [!TIP]
> En el mismo hardware, es común ejecutar **muchos más contenedores** que VMs, gracias al menor overhead.

---

### Velocidad de arranque
Dado que no existe una secuencia de arranque del kernel, un contenedor puede iniciarse en **milisegundos**.

Desde la perspectiva del kernel, iniciar un contenedor es análogo a iniciar un **proceso convencional**, pero con:

- parámetros de aislamiento adicionales

> [!NOTE]
> Esta característica resulta especialmente valiosa en escenarios de **autoscaling**, despliegues frecuentes y entornos efímeros.

---

### Portabilidad
Un contenedor empaqueta no solo el **código** de la aplicación, sino también sus **dependencias exactas**, por ejemplo:

- bibliotecas
- binarios
- archivos de configuración

> [!WARNING]
> La portabilidad no elimina la necesidad de buenas prácticas (p. ej., variables de entorno bien definidas y manejo explícito de configuración), pero reduce significativamente la variabilidad entre ambientes.

Esto contribuye a que lo que se ejecuta en el portátil del desarrollador sea **idéntico** a lo que se ejecuta en producción, con alta consistencia entre entornos.

---
## ¿Cómo logran aislamiento los contenedores?

Los contenedores logran aislamiento principalmente mediante primitivas del **kernel Linux**:

- **Namespaces**: aíslan elementos como procesos, red y sistema de archivos desde la perspectiva del contenedor.
- **cgroups**: controlan y limitan recursos (CPU, memoria, I/O) asignados a procesos.

> [!IMPORTANT]
> A diferencia de una VM, los contenedores **comparten el kernel** del sistema anfitrión, evitando duplicación de SO y manteniendo un modelo de ejecución eficiente.

> [!CAUTION]
> En producción, la seguridad suele reforzarse con configuraciones adicionales (p. ej., **restricciones de capacidades del kernel** y **perfiles de seguridad**) según las políticas de la organización.

---

## Implicaciones arquitectónicas y prácticas modernas

La contenedorización favorece principios relevantes en ingeniería de software moderna:

- **Infraestructura inmutable**  
  Los artefactos de despliegue se **reemplazan** en lugar de modificarse manualmente, reduciendo deriva de configuración y aumentando reproducibilidad.
- **Paradigma cloud native**  
  Prioriza despliegues automatizables, elasticidad y administración consistente de recursos en entornos distribuidos.

> [!NOTE]
> Los contenedores habilitan (sin imponer) patrones como **microservicios**, pero también son útiles en **monolitos** cuando se busca estandarizar ambientes y despliegues.

---

## Nota sobre el ecosistema de contenedores

Docker hace parte de un ecosistema más amplio en evolución. Existen alternativas y componentes especializados, como:

- `containerd`
- `CRI-O`
- `Podman`

Estos pueden estar optimizados para distintos contextos (p. ej., integración con orquestadores o enfoques sin *daemon*). En este curso, el foco estará en **Docker** por su valor pedagógico y utilidad práctica como puerta de entrada; las alternativas se proponen como material complementario para profundización posterior.

---

## Preguntas de autoevaluación

1. ¿Qué diferencias explican el mayor consumo de recursos de una **VM** frente a un **contenedor**?
2. En un despliegue típico, ¿qué elementos debería definir explícitamente para reducir el problema de **“funciona en mi máquina”**?
