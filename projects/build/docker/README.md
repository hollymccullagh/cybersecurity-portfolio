# Docker
*(Installing Docker Engine and Compose plugin on Linux)*


---

## Understanding Docker

Docker is a platform used to package and run applications inside isolated environments known as containers.
Unlike traditional software installations, applications running in Docker do not install their dependencies directly onto the host operating system. 
Instead, each container includes its own application files, libraries and runtime environment while sharing the host operating system's kernel.

This approach provides:

- Application isolation
- Consistent deployments
- Simplified upgrades
- Easier backup and recovery
- Reduced dependency conflicts

Containers are designed to be disposable. 
If a container is deleted, it can be recreated from its image. 
Persistent application data should therefore be stored outside the container on the host operating system.


---

### 1. Update the Operating System

Refresh the package catalogue and install the latest available updates.

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

`apt update` refreshes the list of available software packages.
`apt full-upgrade` installs available updates, including packages requiring dependency changes.
`reboot` ensures the system is running the latest kernel and libraries before installing Docker.


---

### 2. Install Repository Prerequisites

Install the packages required to securely communicate with Docker's software repository.

```bash
sudo apt install -y ca-certificates curl
```
| Package | Purpose |
|----------|---------|
| `ca-certificates` | Provides trusted Certificate Authority (CA) certificates used to verify the authenticity of HTTPS connections. |
| `curl` | Downloads files and data from web servers using supported protocols such as HTTP and HTTPS. |


---

### 3. Configure Docker's Package Repository

Create a directory to store trusted repository signing keys.
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's public signing key.
```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
-o /etc/apt/keyrings/docker.asc
```

Allow the package manager to read the signing key.
```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Why is this required?
Docker digitally signs every package it publishes.
The signing key allows the package manager to verify:

- the package was published by Docker
- the package has not been modified
- the download has not been tampered with


---

### 4. Add Docker's Repository

Create a repository definition for Docker's official stable repository.

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Refresh the package catalogue.
```bash
sudo apt update
```

Rather than installing Docker from the operating system repository, this configuration instructs the package manager to retrieve Docker packages directly 
from Docker's official repository.

Benefits include:

- Official Docker releases
- Faster security updates
- Current Docker Compose plugin
- Access to the latest stable features


---

### 5. Install Docker

Install Docker Engine and supporting components.
```bash
sudo apt install -y \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

#### Installed Components

| Component | Purpose |
|-----------|---------|
| `docker-ce` | Installs Docker Engine, the core service responsible for building, running and managing containers. |
| `docker-ce-cli` | Installs the Docker command-line interface (CLI) used to interact with Docker Engine. |
| `containerd.io` | Installs **containerd**, the container runtime responsible for creating, starting and managing containers. Docker uses containerd behind the scenes. |
| `docker-buildx-plugin` | Installs Docker Buildx, an extended image builder that supports advanced build features such as multi-platform images and improved build performance. |
| `docker-compose-plugin` | Installs the Docker Compose plugin, allowing applications consisting of multiple containers to be defined and managed using a `compose.yml` file. |


---

### 6. Verify Docker

Confirm Docker is running.
```bash
sudo systemctl status docker
```

Expected status:
Active: active (running)

Configure Docker to automatically start during system boot.
```bash
sudo systemctl enable docker
sudo systemctl enable containerd
```


---

### 7. Test Docker

Run Docker's test container.
```bash
sudo docker run --rm hello-world
```
Docker automatically performs the following actions:

1. Searches for the image locally.
2. Downloads the image if required.
3. Creates a container.
4. Starts the container.
5. Executes the application.
6. Removes the temporary container after completion.

Successful output:
`Hello from Docker!`


---

### 8. Configure User Permissions

By default, Docker commands require administrative privileges.
Add the current user to the Docker group.
```bash
sudo usermod -aG docker $USER
```
Reconnect to the system.

Confirm group membership.
```bash
groups
```

Test Docker without administrative privileges.
```bash
docker ps
docker run --rm hello-world
```


---

### Understanding Docker Storage

Containers are intended to be temporary. Deleting a container also deletes its internal filesystem unless persistent storage has been configured.
Docker allows directories on the host operating system to be mounted inside containers. This enables application data to survive container recreation.

Example:

```text
Host
└── /opt/docker/application/data
            │
            │ Mounted into
            ▼
Container
└── /data
```

When the application writes data to /data inside the container, it is actually writing to the directory on the host.
This separation allows containers to be updated or recreated without losing configuration, databases or uploaded files.


---

## Summary

Docker was installed using the vendor's official package repository and configured to start automatically during system boot. 
User permissions were configured to allow Docker administration without requiring `sudo`, which is fine for a home lab set up.

Understanding the relationship between images, containers, and persistent host storage is fundamental to managing Docker environments. 
Containers should be treated as disposable runtime instances, while important application data should always be stored on the host through bind mounts or Docker volumes.


---

## Related Projects

* [Self-Hosting Portainer with Docker Compose](../docker/portainer/README.md)
  Deploys Portainer as a Docker container to provide a web-based interface for managing Docker environments, containers, networks, volumes, and Compose stacks.

* [Self-Hosting Juice Shop with Docker](../../break/juice-shop/README.md)
   Deploys OWASP Juice Shop using the `docker run` command, demonstrating manual container configuration without Docker Compose.





