# Tutorial 01 - Instalación de Docker

Mantener actualizada la documentación de instalación de Docker en Linux es complejo, porque los pasos cambian con las distribuciones (y sus codenames), con el manejo de llaves/repositorios y con actualizaciones del sistema. Por esta razón, esta sección mantiene el procedimiento alineado con la **última versión de Ubuntu soportada por Docker** y, para otras distribuciones, remite a la documentación oficial correspondiente.

## Instalación de Docker Engine en Ubuntu (repositorio oficial)

Esta guía es un resumen alineado con la documentación oficial de Docker para Ubuntu.

### Requisitos

- Una versión de **Ubuntu soportada por Docker Engine**. Al momento de esta edición:
  - Ubuntu **Questing 25.10**
  - Ubuntu **Plucky 25.04**
  - Ubuntu **Noble 24.04 (LTS)**
  - Ubuntu **Jammy 22.04 (LTS)**

> **Nota:** En distribuciones derivadas (por ejemplo, Linux Mint) puede funcionar, pero no está oficialmente soportado por Docker.

### 1) Desinstalar versiones antiguas y paquetes en conflicto (recomendado)

Antes de instalar Docker desde el repositorio oficial, elimine paquetes no oficiales o conflictivos. Ejecute:

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```
> [!NOTE]
> `apt` puede indicar que ninguno de estos paquetes está instalado.

> [!IMPORTANT]
> Desinstalar Docker no elimina automáticamente imágenes, contenedores, volúmenes y redes almacenados en `/var/lib/docker/`.

### 2) Configurar el repositorio oficial de Docker (APT)

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

### 3) Instalar Docker Engine y complementos

Instale Docker Engine, CLI, containerd y los plugins oficiales (Buildx y Compose):

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

El servicio de Docker normalmente inicia automáticamente tras la instalación.

### 4) Verificación de la instalación

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

#### 5) Ejecutar Docker sin sudo (post-instalación)

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

## Referencias

- Docker Engine (Ubuntu): https://docs.docker.com/engine/install/ubuntu/
- Docker Engine (Debian): https://docs.docker.com/engine/install/debian/
- Docker Engine (Fedora): https://docs.docker.com/engine/install/fedora/
- Docker Engine (CentOS): https://docs.docker.com/engine/install/centos/
- Docker Desktop (Windows): https://docs.docker.com/desktop/install/windows-install/

> **Nota:** Pasos de instalación de WSL: https://learn.microsoft.com/es-mx/windows/wsl/install
