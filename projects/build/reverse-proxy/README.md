# NGINX Reverse Proxy with Docker

## Overview

This project documents the architecture of an NGINX reverse proxy running as a Docker Compose container.

The reverse proxy provides a single entry point for web traffic destined for multiple self-hosted applications, while allowing each application to remain isolated on an internal Docker network. 

Rather than exposing multiple TCP ports to the local network, clients communicate with NGINX over TCP port 80 and NGINX forwards requests to the appropriate backend container based on the requested hostname.

An internal DNS service resolves each application hostname to the reverse proxy host.

This project focuses on implementing a secure and scalable foundation that can later be extended with HTTPS, certificates, authentication, and additional services.

---


## Design Principles

The implementation was designed around the following principles:

- Centralise inbound web traffic through a single entry point.
- Minimise exposed services by publishing only the reverse proxy.
- Use container networking for backend communication.
- Separate infrastructure components to improve maintainability.
- Design for future scalability and HTTPS implementation.


---

## Environment

| Component | Value |
|-----------|-------|
| Container Runtime | Docker Engine |
| Orchestration | Docker Compose |
| Reverse Proxy | NGINX |
| DNS | Internal DNS |
| Network | Docker Bridge Network |
| Documentation IPs | RFC 5737 Example Addresses |


---

## Solution Architecture

The reverse proxy acts as the single entry point for all web applications hosted on the proxy host.

Client devices send HTTP requests to NGINX using application hostnames rather than connecting directly to individual containers. NGINX inspects the requested hostname and forwards the request to the appropriate backend container over an isolated Docker bridge network.

This architecture allows backend applications to communicate privately while exposing only the reverse proxy to the local network.

```text
                     Client Devices
      +--------------------------------------+
      |  Desktop  Laptop  Mobile  Tablet     |
      +-------------------+------------------+
                          |
                          |
                     Proxy Host
                          |
                  +-----------------+
                  |      NGINX      |
                  | Reverse Proxy   |
                  +--------+--------+
                           |
            Docker Proxy Network (bridge)
                           |
        +------------------+------------------+
        |                  |                  |
        |                  |                  |
+---------------+  +---------------+  +---------------+
| Application 1 |  | Application 2 |  | Application 3 |
+---------------+  +---------------+  +---------------+
```


---

## Request Flow

The following sequence describes how requests are processed.

1. A client enters an application hostname into a web browser.
2. Internal DNS resolves the hostname to the reverse proxy host.
3. The client establishes an HTTP connection to NGINX on TCP port 80.
4. NGINX evaluates the HTTP `Host` header.
5. The request is forwarded to the appropriate backend container across the Docker bridge network.
6. The backend application generates a response.
7. NGINX returns the response to the client.

At no stage does the client communicate directly with the backend application container.


---

## Hostname Resolution

Application access is performed using hostnames rather than IP addresses.

Each application is assigned a unique hostname within the internal DNS service, which resolves requests to the reverse proxy host.

Example:

| Hostname | Backend Service |
|----------|-----------------|
| `app1.home.arpa` | Application 1 |
| `app2.home.arpa` | Application 2 |
| `app3.home.arpa` | Application 3 |


---

## Docker Networking

All application containers are connected to a shared external Docker bridge network.

This network allows NGINX to communicate directly with backend containers using Docker's built-in DNS service.

Example backend references:

- `http://app1:80`
- `http://app2:80`
- `http://app3:80`

Because communication occurs over the Docker bridge network, backend containers do not require knowledge of the host IP address.


---

## Deployment Configuration

The reverse proxy was deployed as a standalone Docker Compose project to provide a central entry point for multiple self-hosted web applications.

Key configuration decisions included:

- Deploying NGINX as an independent container.
- Creating a dedicated Docker bridge network for reverse proxy communication.
- Connecting backend applications to the shared proxy network.
- Routing requests using NGINX virtual hosts.
- Resolving application hostnames using Internal DNS.
- Publishing only the reverse proxy service while backend applications communicate internally.

This design provides a scalable foundation that allows additional applications to be integrated with minimal configuration changes while maintaining a consistent architecture.


---

## Implementation Process

The reverse proxy was implemented using the following staged approach.

### 1. Deploy NGINX

Create a dedicated Docker Compose project for NGINX to provide a single entry point for inbound HTTP traffic.

```bash
mkdir -p /opt/docker/nginx
cd /opt/docker/nginx
```

Deploy the reverse proxy using Docker Compose.

```bash
docker compose up -d
```

Verify the container is running.

```bash
docker compose ps
```

---

### 2. Create the Proxy Network

Create a dedicated Docker bridge network to allow secure communication between the reverse proxy and backend application containers.

```bash
docker network create proxy
```

Confirm the network was created successfully.

```bash
docker network ls
```

---

### 3. Connect Backend Applications

Attach each application container to the shared `proxy` network.

Within each Docker Compose file:

```yaml
networks:
  proxy:
    external: true
```

This allows NGINX to communicate with backend applications using Docker's internal DNS rather than host IP addresses.

> [!NOTE]
> Feel free to leave the existing host port mappings in place while testing the reverse proxy.
> Once request routing has been validated and applications are accessible through their configured hostnames, remove the published ports so backend services are accessible only through the reverse proxy.


---

### 4. Configure Virtual Hosts

Create an NGINX server block for each application.

Each virtual host listens for a specific hostname and proxies requests to the corresponding backend container.

Example:

```nginx
server {
    listen 80;
    server_name app1.home.arpa;

    location / {
        proxy_pass http://app1:80;
    }
}
```

Validate the configuration before reloading NGINX.

```bash
docker exec nginx nginx -t
```

Reload the configuration.

```bash
docker exec nginx nginx -s reload
```

---

### 5. Configure Internal DNS

Create DNS records for each application hostname so client devices resolve requests to the reverse proxy rather than individual application containers.

Example hostnames include:

* `app1.home.arpa`
* `app2.home.arpa`
* `app3.home.arpa`

---

### 6. Validate the Deployment

Confirm the deployment is functioning as expected.

Useful validation commands include:

```bash
docker compose ps
docker network inspect proxy
docker logs nginx
```

Verify:

* DNS resolves the application hostname.
* NGINX routes requests to the correct backend container.
* Applications are accessible using their configured hostname.
* Reverse proxy logs show successful client requests.

---

### 7. Reduce the Attack Surface

Once request routing has been validated, **remove unnecessary published ports from backend application containers where appropriate.**

This centralises inbound HTTP traffic through NGINX while allowing backend services to remain accessible only across the internal Docker bridge network.

This implementation separates infrastructure from applications while providing a scalable foundation that can easily accommodate additional services and future HTTPS support.



---

## Security Considerations

The deployment was designed with security and maintainability in mind.

Key security considerations included:

- Reducing the number of externally exposed services.
- Restricting inbound web traffic to a single entry point.
- Isolating backend applications using Docker networking.
- Using internal DNS rather than exposing applications through IP addresses and custom ports.
- Designing the solution to support future HTTPS implementation.


---

## Summary

This project demonstrates the deployment of an NGINX reverse proxy using Docker Compose to provide a scalable entry point for multiple self-hosted web applications.

By combining Docker networking, virtual host routing and internal DNS, the solution improves usability, simplifies future expansion and reduces the external attack surface compared to exposing individual application ports.

The architecture provides a flexible foundation that can be extended to support additional applications, HTTPS, authentication and other reverse proxy capabilities as requirements evolve.


---

## Next Steps

Future enhancements include:

- HTTPS using TLS certificates
- HTTP to HTTPS redirection
- Security headers
- Authentication and access control
- Load balancing
- Reverse proxy monitoring
