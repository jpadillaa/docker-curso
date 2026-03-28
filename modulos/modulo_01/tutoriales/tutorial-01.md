![Banner del proyecto](/modulos/assets/Banner.png)

# Tutorial 01 - Instalación de Docker

> **Objetivo:** Correcta instalación y verificación de Docker, asegurando que el entorno quede listo para ejecutar contenedores y desplegar aplicaciones completas con un solo comando.

Mantener actualizada la documentación de instalación de Docker en Linux es complejo, porque los pasos cambian con las distribuciones (y sus codenames), con el manejo de llaves/repositorios y con actualizaciones del sistema. Por esta razón, esta sección mantiene el procedimiento alineado con la **última versión de Ubuntu soportada por Docker** y, para otras distribuciones, remite a la documentación oficial correspondiente.

## Instalación de Docker Engine en Ubuntu (repositorio oficial)

Esta guía es un resumen alineado con la documentación oficial de Docker para Ubuntu.

### Requisitos

Una versión de **Ubuntu soportada por Docker Engine**. Al momento de esta edición:

| Versión de Ubuntu | Nombre código | Soporte |
|-------------------|---------------|---------|
| Ubuntu 25.10 | Questing | Estándar |
| Ubuntu 25.04 | Plucky | Estándar |
| Ubuntu 24.04 | Noble | **LTS** |
| Ubuntu 22.04 | Jammy | **LTS** |

> **Nota:** En distribuciones derivadas (por ejemplo, Linux Mint) puede funcionar, pero no está oficialmente soportado por Docker.

### 1. Desinstalar versiones antiguas y paquetes en conflicto (recomendado)

Antes de instalar Docker desde el repositorio oficial, elimine paquetes no oficiales o conflictivos. Ejecute:

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```
> [!NOTE]
> `apt` puede indicar que ninguno de estos paquetes está instalado.

> [!IMPORTANT]
> Desinstalar Docker no elimina automáticamente imágenes, contenedores, volúmenes y redes almacenados en `/var/lib/docker/`.

### 2. Configurar el repositorio oficial de Docker (APT)

#### 2.1. Preparar dependencias y llaves

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

#### 2.2. Registrar el repositorio 
```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```bash
sudo apt update
```

### 3. Instalar Docker Engine y complementos

Instale Docker Engine, CLI, containerd y los plugins oficiales (Buildx y Compose):

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

El servicio de Docker normalmente inicia automáticamente tras la instalación.

### 4. Verificación de la instalación

#### 4.1. Verificar el servicio

```bash
sudo systemctl status docker
```

Si el servicio no está activo, inícielo manualmente:

```bash
sudo systemctl start docker
```

#### 4.2. Ejecutar un contenedor de prueba (hello-world)
Este paso valida la instalación de Docker de extremo a extremo, confirmando que el servicio está activo, que el cliente puede comunicarse con el daemon y que el sistema puede descargar imágenes desde el registro y ejecutar contenedores correctamente. El contenedor hello-world es una imagen mínima diseñada para esta verificación: al ejecutarla, Docker mostrará un mensaje en consola indicando que la ejecución fue exitosa.

```bash
sudo docker run hello-world
```

#### 4.3. Verificación alternativa con Python (1 al 10)
Esta verificación complementa la prueba de hello-world con un escenario más representativo de uso real: Docker descarga (si no existe localmente) una imagen oficial de Python, inicia un contenedor interactivo y ejecuta un comando de Python dentro del entorno aislado. La opción --rm elimina el contenedor al terminar, evitando residuos, mientras que -it habilita un modo interactivo adecuado para ejecución en terminal. Si el comando imprime los números del 1 al 10, confirma que Docker puede obtener imágenes desde un registro, crear y ejecutar contenedores, y correr procesos de aplicación correctamente dentro de ellos.

```bash
sudo docker run --rm -it python:3.12 python -c "for i in range(1, 11): print(i)"
```

#### 5. Ejecutar Docker sin sudo (post-instalación)

Por defecto, Docker requiere privilegios de superusuario. Para permitir que su usuario ejecute Docker sin sudo, agréguelo al grupo docker.

> [!WARNING]
> **Advertencia de seguridad:** pertenecer al grupo `docker` otorga privilegios equivalentes a `root`. Aplíquelo solo si comprende las implicaciones de seguridad en su entorno.

#### 5.1. Crear el grupo docker (si no existe)

```bash
sudo groupadd docker
```

#### 5.2. Agregar el usuario actual al grupo

```bash
sudo usermod -aG docker $USER
```

#### 5.3. Aplicar los cambios

> [!TIP]
> **Método recomendado:** cierre sesión y vuelva a iniciarla.

Alternativa (solo para su terminal actual):

```bash
newgrp docker
```

#### 5.4. Confirmar membresía y probar sin sudo
```bash
groups
docker run hello-world
```
---

## Instalación de Docker en Microsoft Windows (Docker Desktop + WSL 2)

En Windows, la vía recomendada para ejecutar contenedores Linux es **Docker Desktop** usando el backend de **WSL 2**. Este enfoque integra el daemon de Docker con una distribución Linux en WSL, evitando configuraciones manuales complejas.

> [!IMPORTANT]
> Para evitar conflictos, desinstale cualquier Docker Engine/CLI instalado directamente dentro de sus distribuciones WSL antes de instalar Docker Desktop.  
> Referencia: [documentación oficial de Docker sobre WSL 2](https://docs.docker.com/desktop/wsl/).

## Requisitos

| Componente | Especificación |
|------------|----------------|
| **Sistema Operativo** | Windows 11 o Windows 10 22H2 (Enterprise/Pro/Education), 64-bit |
| **WSL** | WSL 2 instalado y actualizado |
| **Virtualización** | Habilitada a nivel de BIOS/UEFI |
| **Memoria RAM** | Mínimo 4 GB (sugerido) |

> [!NOTE]
> Los requisitos exactos (edición/build de Windows y versión mínima de WSL) pueden cambiar. Verifique siempre la [documentación oficial de Docker Desktop](https://docs.docker.com/desktop/install/windows-install/).

## Pasos de instalación

### 1. Instalar o actualizar WSL 2

Abra **PowerShell como administrador** y ejecute:
```powershell
wsl --install
```

Reinicie el equipo cuando se lo solicite. Luego, actualice WSL:
```powershell
wsl --update
```

Opcionalmente, establezca WSL 2 como versión predeterminada:
```powershell
wsl --set-default-version 2
```

Para verificar el estado y las distribuciones instaladas:
```powershell
wsl --status
wsl -l -v
```

### 2. Instalar Docker Desktop

1. Descargue e instale **Docker Desktop for Windows** desde la [página oficial de Docker](https://docs.docker.com/desktop/install/windows-install/).
2. Durante la instalación, asegúrese de seleccionar el uso de **WSL 2** cuando el instalador lo solicite (o actívelo posteriormente en la configuración).

> [!TIP]
> Después de instalar, abra Docker Desktop y valide en **Settings > General** que esté activa la opción `Use WSL 2 based engine`. Luego, en **Settings > Resources > WSL Integration**, habilite la integración con la distribución WSL que utilizará (por ejemplo, Ubuntu).

### 3. Verificación de la instalación

Desde **PowerShell**, verifique las versiones instaladas:
```powershell
docker version
docker compose version
```

Ejecute un contenedor de prueba con Python:
```powershell
docker run --rm -it python:3.12 python -c "for i in range(1, 11): print(i)"
```

## Solución de problemas frecuentes

### Mensaje: "WSL 2 installation is incomplete"

Si Docker Desktop no inicia o muestra este mensaje:

1. **Actualice WSL y reinicie:**
```powershell
   wsl --update
```

2. **Verifique que su distribución esté en WSL 2:**
```powershell
   wsl -l -v
```

3. Si su versión de Windows no soporta `wsl --install` (o está en un build antiguo), siga la [instalación manual de Microsoft](https://learn.microsoft.com/es-es/windows/wsl/install-manual).

> [!NOTE]
> La guía manual de Microsoft incluye comandos `dism.exe` para habilitar *Windows Subsystem for Linux* y *VirtualMachinePlatform*, así como la instalación del paquete de actualización del kernel de WSL 2.

## Referencias

### Docker Engine (Linux)

| Distribución | Documentación |
|--------------|---------------|
| Ubuntu | [docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu/) |
| Debian | [docs.docker.com/engine/install/debian](https://docs.docker.com/engine/install/debian/) |
| Fedora | [docs.docker.com/engine/install/fedora](https://docs.docker.com/engine/install/fedora/) |
| CentOS | [docs.docker.com/engine/install/centos](https://docs.docker.com/engine/install/centos/) |

### Docker Desktop (Windows)

| Recurso | Documentación |
|---------|---------------|
| Instalación y requisitos | [Docker Docs - Windows install](https://docs.docker.com/desktop/install/windows-install/) |
| Backend WSL 2 | [Docker Docs - WSL feature](https://docs.docker.com/desktop/wsl/) |

### WSL (Windows Subsystem for Linux)

| Recurso | Documentación |
|---------|---------------|
| Instalación de WSL | [Microsoft Learn - WSL install](https://learn.microsoft.com/es-es/windows/wsl/install) |
| Instalación manual (builds antiguos) | [Microsoft Learn - WSL manual install](https://learn.microsoft.com/es-es/windows/wsl/install-manual) |
