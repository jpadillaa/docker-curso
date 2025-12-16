# Tutorial 02 - Docker en acción

> **Objetivo:** Demostrar el potencial de los contenedores Docker ejecutando aplicaciones completas con un solo comando.

## Introducción

¿Qué pasaría si pudiera ejecutar **cualquier aplicación** en su computadora sin instalarla? ¿Un videojuego clásico? ¿Un entorno de desarrollo completo? ¿Una plataforma de ciencia de datos?. Con Docker, todo esto es posible con **un solo comando**.

En este tutorial ejecutaremos 3 contenedores que demuestran el verdadero poder de esta tecnología:

| # | Ejemplo | Lo que demuestra |
|---|---------|------------------|
| 1 | **DOOM** | Docker puede ejecutar *cualquier cosa*, incluso videojuegos |
| 2 | **VS Code + Python** | Un IDE profesional completo en el navegador |
| 3 | **Jupyter Notebook** | Una plataforma de ciencia de datos lista para usar |

## Prerrequisitos

- Docker o Docker Desktop instalado y ejecutándose
- Conexión a internet (para descargar las imágenes)
- Un navegador web

## Ejemplo 1: DOOM — Sí, el Videojuego 

Si Docker puede ejecutar un videojuego clásico de los años 90 en tu navegador, entonces puede ejecutar **cualquier cosa**. 

### Ejecución

Abra su terminal y ejecute:

```bash
docker run --rm -p 8080:8080 b0nam/docker-doom
```

### Acceso

Abra su navegador y visite:

```
http://localhost:8080
```

### Controles del juego

| Tecla | Acción |
|-------|--------|
| `↑ ↓ ← →` | Movimiento |
| `Ctrl` | Disparar |
| `Espacio` | Abrir puertas |
| `Esc` | Menú/Pausa |

### ¿Qué acaba de lograr?

Sin instalar nada más que Docker, acaba de ejecutar:
- Un sistema operativo Linux
- Un servidor VNC
- Un emulador de DOS
- El videojuego DOOM completo

**Todo en un solo comando.**

> **Para detener:** Presione `Ctrl + C` en la terminal.

## Ejemplo 2: VS Code + Python en el Navegador

Imagine tener Visual Studio Code funcionando en cualquier dispositivo con un navegador, sin instalarlo. Además, con Python ya configurado y listo para programar.

### Ejecución

En su terminal, ejecute:

```bash
docker run --rm -p 8443:8443 \
  -e PUID=1000 \
  -e PGID=1000 \
  -e PASSWORD=docker123 \
  -e SUDO_PASSWORD=docker123 \
  -e DOCKER_MODS=linuxserver/mods:code-server-python3 \
  lscr.io/linuxserver/code-server:latest
```

> **Nota:** La primera ejecución tardará unos minutos mientras descarga e instala Python.

### Acceso

Abra su navegador y visite:

```
http://localhost:8443
```

**Contraseña:** `docker123`

### Probando Python

1. Abra la terminal integrada: `Terminal > New Terminal`
2. Verifique que Python está instalado:
   ```bash
   python3 --version
   ```
3. Cree un archivo `hola.py` con el siguiente contenido:
   ```python
   print("¡Hola desde Docker!")
   
   for i in range(1, 6):
       print(f"Iteración {i}")
   ```
4. Ejecute el script:
   ```bash
   python3 hola.py
   ```

### ¿Qué acaba de lograr?

- VS Code & Python 3 completamente configurado
- Terminal Linux integrada
- Todo accesible desde cualquier navegador

> **Para detener:** Presione `Ctrl + C` en la terminal.

## Ejemplo 3: Jupyter Notebook 

Jupyter es el estándar de la industria para ciencia de datos. Normalmente, configurar un entorno con todas las librerías necesarias toma horas. Con Docker, toma **segundos**.

### Ejecución
En su terminal, ejecute:

```bash
docker run --rm -p 8888:8888 quay.io/jupyter/scipy-notebook
```

### Acceso

1. Observe la terminal. Aparecerá una URL similar a:
   ```
   http://127.0.0.1:8888/lab?token=abc123...
   ```
2. Copie y pegue esa URL completa en su navegador.

### Probando el entorno

1. En JupyterLab, cree un nuevo Notebook: `File > New > Notebook`
2. Seleccione el kernel **Python 3**
3. En la primera celda, escriba y ejecute (`Shift + Enter`):

```python
# Importar librerías de ciencia de datos
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# Crear datos de ejemplo
datos = {
    'Mes': ['Ene', 'Feb', 'Mar', 'Abr', 'May'],
    'Ventas': [150, 200, 180, 220, 250]
}

df = pd.DataFrame(datos)
print(df)

# Crear gráfico
plt.figure(figsize=(8, 4))
plt.bar(df['Mes'], df['Ventas'], color='steelblue')
plt.title('Ventas por Mes')
plt.ylabel('Ventas ($)')
plt.show()
```

### ¿Qué acabas de lograr?

Sin instalar Python ni ninguna librería, tiene acceso a:

| Librería | Uso |
|----------|-----|
| **NumPy** | Computación numérica |
| **Pandas** | Análisis de datos |
| **Matplotlib** | Visualización |
| **SciPy** | Computación científica |
| **Seaborn** | Gráficos estadísticos |

> **Para detener:** Presiona `Ctrl + C` en la terminal (dos veces si es necesario).

### Resumen

```
┌─────────────────────────────────────────────────────────┐
│  "Si puede correr en Linux, puede correr en Docker"    │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
    ┌──────────────────────────────────────────────┐
    │  ✓ Videojuegos      ✓ IDEs                   │
    │  ✓ Bases de datos   ✓ Servidores web         │
    │  ✓ Machine Learning ✓ Aplicaciones completas │
    └──────────────────────────────────────────────┘
```

### El verdadero poder

Docker no es solo para DevOps o servidores. Es una herramienta que le permite:

1. **Probar cualquier tecnología** sin contaminar su sistema
2. **Compartir entornos** de desarrollo idénticos con su equipo de trabajo
3. **Desplegar aplicaciones** de manera consistente en cualquier lugar

## Comandos de limpieza

Para eliminar las imágenes descargadas:

```bash
# Ver imágenes descargadas
docker images

# Eliminar imágenes específicas
docker rmi b0nam/docker-doom
docker rmi lscr.io/linuxserver/code-server
docker rmi quay.io/jupyter/scipy-notebook

# O eliminar todas las imágenes no utilizadas
docker image prune -a
```

## Referencias

| Recurso | Enlace |
|---------|--------|
| Docker Hub | [hub.docker.com](https://hub.docker.com) |
| Imagen DOOM | [github.com/B0nam/DOCKER-DOOM](https://github.com/B0nam/DOCKER-DOOM) |
| LinuxServer Code-Server | [docs.linuxserver.io/images/docker-code-server](https://docs.linuxserver.io/images/docker-code-server/) |
| Jupyter Docker Stacks | [github.com/jupyter/docker-stacks](https://github.com/jupyter/docker-stacks) |
