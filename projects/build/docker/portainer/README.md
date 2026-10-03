# Portainer
*(Portainer Deployment with Docker Compose)*


---

## Overview
Portainer is a web-based management interface for Docker environments. 
It provides a graphical interface for managing containers, images, networks, volumes, and Docker Compose stacks.
This project deploys Portainer Community Edition using Docker Compose and stores its configuration in a persistent host directory.

### Why Portainer?

Deploying Portainer helped me develop a practical understanding of containerised environments beyond the command line. Using a graphical interface made it easier to visualise how containers, images, networks, volumes, and Docker Compose stacks relate to one another, reinforcing core Docker concepts while providing a convenient platform for managing and troubleshooting self-hosted applications.

To learn more about Portainer, visit the official website or explore the source code on GitHub:

* [Portainer Official Website](https://www.portainer.io)
* [Portainer GitHub Repository](https://github.com/portainer)


---

## Prerequisites

Before starting, confirm that the following components are installed:

| Project | Description |
|---------|-------------|
| [Docker Engine](../README.md) | Container runtime used to build and manage isolated application environments. |
| [Docker Compose](../README.md) | Defines and deploys multi-container applications using declarative YAML configuration. |

Verify Docker and Docker Compose:

```bash
docker --version
docker compose version
```

Confirm that the Docker service is running:
```bash
sudo systemctl status docker
```

Directory Structure

Portainer files are stored under:
```text
/opt/docker/portainer/
├── compose.yaml
└── data/
```

The `compose.yaml` file defines the Portainer container.

The data directory stores Portainer configuration, users, environment details, and other persistent application data.


---

### 1. Create the Portainer Directory

Create the required directories:
```bash
sudo mkdir -p /opt/docker/portainer/data
```

Assign ownership of the Portainer directory to the current user:
```bash
sudo chown -R "$USER":"$USER" /opt/docker/portainer
```

Move into the Portainer directory:
```bash
cd /opt/docker/portainer
```


---

### 2. Create the Docker Compose File

Create the Compose file:
```bash
nano compose.yaml
```

Add the following configuration:
```bash
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped

    ports:
      - "9443:9443"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/docker/portainer/data:/data
```

Save and exit.


---

### 3. Compose Configuration


| Setting | Purpose |
|----------|----------|
| `image` | Specifies the Portainer Community Edition container image to deploy. |
| `container_name` | Assigns the container the name `portainer` for easier identification and management. |
| `restart` | Automatically restarts the container unless it has been manually stopped. |
| `9443:9443` | Publishes the Portainer HTTPS web interface on TCP port **9443** of the host. |
| `/var/run/docker.sock:/var/run/docker.sock` | Mounts the Docker socket into the container, allowing Portainer to communicate with and manage the local Docker Engine. |
| `/opt/docker/portainer/data:/data` | Stores Portainer's persistent configuration, users, endpoints, and settings on the host so they survive container recreation. |

> **Security Note:** The Docker socket (`/var/run/docker.sock`) provides Portainer with administrative access to the Docker host.
> Anyone with administrative access to Portainer can potentially create privileged containers, manage images, modify networks and volumes, and control other containers.
> For this reason, access to the Portainer web interface should be restricted to trusted administrators.


Validate the syntax before deployment:
```bash
docker compose config
```
If the configuration is valid, Docker Compose displays the resolved configuration without reporting an error.


---

### 4. Deploy Portainer

Start the Portainer container:
```bash
docker compose up -d
```

The `-d` option runs the container in detached mode.

Verify the deployment:
```bash
docker compose ps
```
Alternatively:
```bash
docker ps
```
The Portainer container should show a running status with TCP port 9443 published.


---

### 5. Access the Portainer Interface

Open the following address in a web browser:
`https://<SERVER-IP>:9443`

Portainer uses HTTPS and generates a self-signed certificate during the initial deployment.

A browser certificate warning is therefore expected.

The connection is encrypted, but the browser cannot verify the identity of the server because the certificate was not issued by a trusted Certificate Authority.

Proceed to the site only after confirming that the IP address belongs to the correct Docker host.

---

### 6. Initial Portainer Configuration

During the initial setup:

1. Create the Portainer administrator account.
2. Use a long and unique password.
3. Complete any enrollment or token prompt displayed by Portainer.
4. If prompted for an enrollment token, retrieve it from the Portainer logs:

   ```bash
   docker compose logs portainer
   ```

   Locate the enrollment token in the output, copy it, and paste it into the Portainer setup page.

5. Connect the local Docker environment.
6. Confirm that the Portainer dashboard displays the local Docker host.

The initial enrollment token does not normally need to be retained after the environment has been successfully registered.

Portainer may disable its initial setup page if the administrator account is not created within the permitted setup period.

The following message may appear:
```text
Your Portainer instance timed out for security purposes.
To re-enable your Portainer instance, you will need to restart Portainer.
```

Restart the container:
```bash
cd /opt/docker/portainer
docker compose restart
```
Refresh the Portainer page and complete the initial setup.

This timeout prevents another device on the network from claiming the first administrator account while the setup page is unattended.


---

## Docker Compose and Portainer Stacks

Portainer refers to Docker Compose deployments as stacks.
A stack may contain one or more related services, networks, and volumes.

For example:
```text
services:
  application:
    image: example/application

  database:
    image: example/database
```
Both services can be deployed and managed together as one stack.
Applications should be managed consistently.
If an application was deployed using a local compose.yaml file, changes should normally be made through that file rather than by manually editing the running container in Portainer.
This ensures that the documented configuration remains the source of truth.


---

## Persistent Data

Portainer configuration is stored in:
```bash
/opt/docker/portainer/data
```
This allows the container to be removed, updated, or recreated without losing:
- Administrator accounts
- Connected environments
- Portainer settings
- Endpoint configuration
- Stack metadata

The persistent data directory should be included in the host backup process.


---

## Useful Management Commands

Run the following commands from the Portainer directory:

```bash
cd /opt/docker/portainer
```

| Task                             | Command                  |
| -------------------------------- | ------------------------ |
| Check Portainer status           | `docker compose ps`      |
| Start Portainer                  | `docker compose start`   |
| Stop Portainer                   | `docker compose stop`    |
| Restart Portainer                | `docker compose restart` |
| View container logs              | `docker compose logs`    |
| Follow logs in real time         | `docker compose logs -f` |
| Remove the Portainer container   | `docker compose down`    |
| Recreate and start the container | `docker compose up -d`   |

> **Note:** Running `docker compose down` removes the Portainer container and its Compose network.
> It does not delete the persistent Portainer data stored in:
> ```text
> /opt/docker/portainer/data
> ```

Press `Ctrl+C` to stop following logs when using:

```bash
docker compose logs -f
```

This stops the live log display but does not stop the Portainer container.


---

## Update Portainer

Move into the Portainer directory:
```bash
cd /opt/docker/portainer
```

Use the following commands to update Portainer:

| Step                      | Command                | Purpose                                                                    |
| ------------------------- | ---------------------- | -------------------------------------------------------------------------- |
| 1. Pull the latest image  | `docker compose pull`  | Downloads the latest Portainer image defined in `compose.yaml`             |
| 2. Recreate the container | `docker compose up -d` | Recreates Portainer using the updated image and starts it in detached mode |
| 3. Verify the deployment  | `docker compose ps`    | Confirms that the updated Portainer container is running                   |
| 4. Review the logs        | `docker compose logs`  | Checks for startup errors after the update                                 |
| 5. Remove unused images   | `docker image prune`   | Removes unused Docker images that are no longer required                   |

The complete update sequence is:
```bash
cd /opt/docker/portainer
docker compose pull
docker compose up -d
docker compose ps
docker compose logs
```

After confirming that Portainer is working correctly, unused images can be removed:
```bash
docker image prune
```

> **Caution:** Review the confirmation prompt before approving image removal.
> Docker will only remove unused images, but the listed items should still be checked before continuing.


## Security Considerations

The following controls were applied:

- Portainer is accessed using HTTPS on TCP port 9443
- The web interface is restricted to an authorised workstation using UFW
- The administrator account uses a unique password
- Portainer data is stored persistently outside the container
- Only required ports are published
- The Docker socket is not exposed over the network
- Container configuration is documented in Docker Compose

The Docker socket grants significant control over the Docker host. 
Anyone who gains administrative access to Portainer may be able to create privileged containers, access mounted data, and control other containers.
Portainer should therefore be treated as a privileged administrative interface.


## Summary

Portainer Community Edition was successfully deployed using Docker Compose.

The deployment provides a central web interface for managing the local Docker environment while retaining a reproducible Compose configuration and persistent application data.

The environment is now ready for additional Docker Compose applications to be deployed and managed as separate services or stacks.


---



