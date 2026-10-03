# Internal HTTPS for Self-Hosted Applications
(NGINX Reverse Proxy with Internal PKI)

## Overview

This project documents the implementation of trusted HTTPS for internally hosted applications using NGINX as a reverse proxy and TLS termination point.

An internal Certificate Authority is used to issue an X.509 server certificate for the application hostname. NGINX presents this certificate to client devices, establishes the encrypted TLS connection, and forwards requests to the backend application over the internal Docker network.

The example configuration secures the following internal application:

```text
                           Client Device
                                 |
                                 | HTTPS (TCP/443)
                                 |
                                 v
                    +-----------------------+
                    |         NGINX         |
                    |     Reverse Proxy     |
                    |    TLS Termination    |
                    +-----------+-----------+
                                |
             HTTP over internal Docker proxy network
                                |
        +-----------------------+-----------------------+
        |                       |                       |
        v                       v                       v
  +------------+          +------------+          +------------+
  |    App 1   |          |    App 2   |          |    App 3   |
  |  app1:80   |          |  app2:80   |          |  app3:80   |
  +------------+          +------------+          +------------+
```

This design centralises certificate management at the reverse proxy while allowing the backend application to remain isolated on the internal Docker network.

This project demonstrates how internal HTTPS can provide:

* Encryption in transit
* Server identity validation
* Trusted browser connections
* Centralised TLS termination
* Simplified certificate management
* A scalable foundation for securing additional applications

---

## Prerequisites

This project assumes the following components have already been configured.

| Prerequisite | Purpose |
|--------------|---------|
| [Container Platform](../../build/docker/README.md) | Provides Docker Engine, Docker Compose, container networking, and persistent storage. |
| [NGINX Reverse Proxy](../../build/reverse-proxy/README.md) | Provides hostname-based routing, internal DNS configuration, and a central entry point for internal web applications. |
| Backend Application | Runs as the Docker service `app1` on the shared proxy network. |



Before starting, confirm that the application is accessible over HTTP:

```text
http://app1.home.arpa
```

The existing NGINX configuration should resemble:

```nginx
server {
    listen 80;
    server_name app1.home.arpa;

    location / {
        proxy_pass http://app1:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Objectives

The objectives of this project were to:

* Create an internal Certificate Authority
* Generate and protect the CA private key
* Issue a certificate for `app1.home.arpa`
* Include the application hostname in the Subject Alternative Name extension
* Configure NGINX for TLS termination
* Redirect HTTP requests to HTTPS
* Install the Root CA certificate on an authorised client
* Validate the certificate chain and encrypted connection
* Establish a repeatable process for securing additional applications

---

## Example Environment

| Component                     | Example          |
| ----------------------------- | ---------------- |
| Reverse Proxy Host            | `192.0.2.10`     |
| Internal Application Hostname | `app1.home.arpa` |
| Docker Backend Service        | `app1`           |
| Backend Port                  | `80`             |
| HTTPS Port                    | `443`            |
| Certificate Authority         | Internal Root CA |

> [!NOTE]
> RFC 5737 documentation addresses and generic application names are used throughout this project instead of real environment details.

---

## Request Flow

Once HTTPS has been configured, requests follow this sequence:

1. The client requests `https://app1.home.arpa`.
2. Internal DNS resolves the hostname to the reverse proxy host.
3. The client establishes a TLS connection with NGINX on TCP port 443.
4. NGINX presents the certificate issued for `app1.home.arpa`.
5. The client validates the certificate against the trusted internal Root CA.
6. NGINX decrypts the request.
7. NGINX forwards the request to `http://app1:80` over the Docker proxy network.
8. The backend application generates a response.
9. NGINX returns the response to the client over the encrypted TLS connection.

The external client connection is encrypted, while backend communication remains isolated within the Docker bridge network.

---

# Implementation Process

## 1. Prepare the Certificate Directory

Create a protected directory for Certificate Authority and server certificate files.

```bash
mkdir -p ~/pki/{ca,certs}
chmod 700 ~/pki
```

Navigate to the CA directory.

```bash
cd ~/pki/ca
```

The directory should be accessible only to the administrative account responsible for certificate management.

---

## 2. Create the Internal Certificate Authority

Generate an encrypted Root CA private key.

```bash
openssl genrsa \
  -aes256 \
  -out internal-root-ca.key \
  4096
```

Restrict access to the private key.

```bash
chmod 600 internal-root-ca.key
```

Create the self-signed Root CA certificate.

```bash
openssl req \
  -x509 \
  -new \
  -key internal-root-ca.key \
  -sha256 \
  -days 3650 \
  -out internal-root-ca.crt
```

Use a descriptive Common Name, such as:

```text
Internal Root Certificate Authority
```

The Root CA certificate establishes the internal trust anchor. Its private key is used to sign server certificates and must be protected from unauthorised access.

> [!IMPORTANT]
> This project uses a single internal CA to keep the implementation focused on securing internal services. Higher-security or production environments should consider an offline Root CA and a separate Intermediate CA for routine certificate issuance.

---

## 3. Validate the Root CA Certificate

Confirm that the certificate is marked as a Certificate Authority.

```bash
openssl x509 \
  -in internal-root-ca.crt \
  -noout \
  -text |
grep -A2 -i "Basic Constraints"
```

Expected output:

```text
X509v3 Basic Constraints: critical
    CA:TRUE
```

Review the certificate subject, issuer, and validity period.

```bash
openssl x509 \
  -in internal-root-ca.crt \
  -noout \
  -subject \
  -issuer \
  -dates
```

Because the certificate is self-signed, the subject and issuer should match.

---

## 4. Generate the Server Private Key

Navigate to the server certificate directory.

```bash
cd ~/pki/certs
```

Generate the server private key.

```bash
openssl genrsa \
  -out app1.home.arpa.key \
  2048
```

Restrict access to the private key.

```bash
chmod 600 app1.home.arpa.key
```
> [!CAUTION]
> The server private key should remain on the reverse proxy host and must not be distributed to client devices.

---

## 5. Create the Certificate Configuration

Create an OpenSSL configuration file for the server certificate.

```bash
nano app1.home.arpa.cnf
```

Add:

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = subject
req_extensions = request_extensions

[ subject ]
CN = app1.home.arpa

[ request_extensions ]
subjectAltName = @alternative_names

[ alternative_names ]
DNS.1 = app1.home.arpa
```

The Subject Alternative Name extension is required because modern clients validate the requested hostname against the SAN rather than relying only on the Common Name.

---

## 6. Create the Certificate Signing Request

Generate a Certificate Signing Request using the server private key and certificate configuration.

```bash
openssl req \
  -new \
  -key app1.home.arpa.key \
  -out app1.home.arpa.csr \
  -config app1.home.arpa.cnf
```

Inspect the CSR.

```bash
openssl req \
  -in app1.home.arpa.csr \
  -noout \
  -text
```

Confirm that the requested Subject Alternative Name includes:

```text
DNS:app1.home.arpa
```

---

## 7. Define the Server Certificate Extensions

Create a certificate extension file.

```bash
nano app1.home.arpa.ext
```

Add:

```ini
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName = DNS:app1.home.arpa
```

These extensions identify the certificate as an end-entity server certificate and restrict its use to TLS server authentication.

---

## 8. Sign the Server Certificate

Sign the CSR using the internal Root CA.

```bash
openssl x509 \
  -req \
  -in app1.home.arpa.csr \
  -CA ../ca/internal-root-ca.crt \
  -CAkey ../ca/internal-root-ca.key \
  -CAcreateserial \
  -out app1.home.arpa.crt \
  -days 825 \
  -sha256 \
  -extfile app1.home.arpa.ext
```

This produces the server certificate:

```text
app1.home.arpa.crt
```

The certificate contains the public key and identity information for the application. The corresponding private key remains in:

```text
app1.home.arpa.key
```

---

## 9. Validate the Server Certificate

Verify that the server certificate chains back to the internal Root CA.

```bash
openssl verify \
  -CAfile ../ca/internal-root-ca.crt \
  app1.home.arpa.crt
```

Expected output:

```text
app1.home.arpa.crt: OK
```

Confirm the Subject Alternative Name.

```bash
openssl x509 \
  -in app1.home.arpa.crt \
  -noout \
  -text |
grep -A2 -i "Subject Alternative Name"
```

Expected output:

```text
X509v3 Subject Alternative Name:
    DNS:app1.home.arpa
```

Review the certificate validity period.

```bash
openssl x509 \
  -in app1.home.arpa.crt \
  -noout \
  -dates
```

---

## 10. Prepare the NGINX Certificate Files

Create a certificate directory within the NGINX project.

```bash
mkdir -p /opt/docker/nginx/certs
```

Copy the server certificate and private key.

```bash
sudo cp \
  ~/pki/certs/app1.home.arpa.crt \
  /opt/docker/nginx/certs/
```

```bash
sudo cp \
  ~/pki/certs/app1.home.arpa.key \
  /opt/docker/nginx/certs/
```

Restrict access to the private key.

```bash
sudo chmod 600 \
  /opt/docker/nginx/certs/app1.home.arpa.key
```

---

## 11. Mount the Certificate Directory

Add the certificate directory to the NGINX Docker Compose configuration.

```yaml
services:
  nginx:
    image: nginx:stable
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - proxy
```

The read-only mount allows NGINX to use the certificate files without modifying them.

Confirm that TCP port 443 is published by the NGINX container.

---

## 12. Configure the HTTPS Virtual Host

Update the NGINX configuration for `app1.home.arpa`.

```nginx
server {
    listen 443 ssl;
    server_name app1.home.arpa;

    ssl_certificate /etc/nginx/certs/app1.home.arpa.crt;
    ssl_certificate_key /etc/nginx/certs/app1.home.arpa.key;

    location / {
        proxy_pass http://app1:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

NGINX now terminates the TLS connection and forwards the decrypted request to the backend container.

The backend reference remains:

```text
http://app1:80
```

This uses Docker's internal DNS rather than the client-facing hostname.

---

## 13. Redirect HTTP to HTTPS

Replace the original HTTP virtual host with a redirect.

```nginx
server {
    listen 80;
    server_name app1.home.arpa;

    return 301 https://$host$request_uri;
}
```

Requests made to:

```text
http://app1.home.arpa
```

will be redirected to:

```text
https://app1.home.arpa
```

This ensures normal client access uses the encrypted HTTPS connection.

---

## 14. Recreate the NGINX Container

Validate the Docker Compose configuration.

```bash
docker compose config
```

Recreate the NGINX container with the updated port and volume configuration.

```bash
docker compose up -d
```

Confirm that the container is running.

```bash
docker compose ps
```

---

## 15. Validate the NGINX Configuration

Test the NGINX configuration from inside the container.

```bash
docker exec nginx nginx -t
```

Expected output should indicate that the configuration syntax is valid.

Reload NGINX if required.

```bash
docker exec nginx nginx -s reload
```

Review the logs for certificate or configuration errors.

```bash
docker logs nginx --tail=50
```

---

## 16. Install the Root CA Certificate on a Client

Transfer only the public Root CA certificate to the client device.

```text
internal-root-ca.crt
```

Do not transfer:

```text
internal-root-ca.key
```

The Root CA certificate must be installed within the client's trusted Root certificate store.

On Windows, open the certificate manager:

```powershell
certmgr.msc
```

Import the certificate into:

```text
Trusted Root Certification Authorities
```

In an enterprise environment, trusted Root certificates would normally be distributed through centralised management systems such as:

* Group Policy
* Mobile device management
* Endpoint management
* Configuration management

Manual installation is suitable for a limited laboratory environment.

---

## 17. Test the HTTPS Connection

Open a browser and navigate to:

```text
https://app1.home.arpa
```

The connection should load without a certificate warning when:

* Internal DNS resolves the hostname correctly.
* The certificate includes `app1.home.arpa` in its SAN.
* The certificate is within its validity period.
* The Root CA certificate is trusted by the client.
* NGINX is presenting the correct server certificate.
* TCP port 443 is reachable.

The browser should display a secure HTTPS connection.

---

## 18. Inspect the Presented Certificate

Use OpenSSL to inspect the certificate presented by NGINX.

```bash
openssl s_client \
  -connect app1.home.arpa:443 \
  -servername app1.home.arpa \
  -showcerts
```

This can be used to verify:

* The presented certificate
* The certificate subject
* The issuing CA
* The negotiated TLS protocol
* The negotiated cipher suite
* Certificate-chain validation

To display only the certificate details:

```bash
openssl s_client \
  -connect app1.home.arpa:443 \
  -servername app1.home.arpa \
  </dev/null 2>/dev/null |
openssl x509 \
  -noout \
  -subject \
  -issuer \
  -dates
```

---

## 19. Confirm HTTP Redirection

Test the HTTP endpoint.

```bash
curl -I http://app1.home.arpa
```

Expected response:

```text
HTTP/1.1 301 Moved Permanently
Location: https://app1.home.arpa/
```

Test HTTPS after the Root CA has been trusted.

```bash
curl -I https://app1.home.arpa
```

A successful response confirms that HTTPS routing is functioning.

---

## Adding Additional Applications

The same process can be repeated for additional internal applications.

Examples:

```text
app2.home.arpa
app3.home.arpa
```

A separate certificate can be issued for each application, or one certificate can contain multiple Subject Alternative Names.

Example SAN configuration:

```ini
[ alternative_names ]
DNS.1 = app1.home.arpa
DNS.2 = app2.home.arpa
DNS.3 = app3.home.arpa
```

Each NGINX virtual host can then reference the appropriate certificate and backend service.

```nginx
server {
    listen 443 ssl;
    server_name app2.home.arpa;

    ssl_certificate /etc/nginx/certs/internal-services.crt;
    ssl_certificate_key /etc/nginx/certs/internal-services.key;

    location / {
        proxy_pass http://app2:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Centralising TLS termination at the reverse proxy allows additional services to be secured without configuring HTTPS separately inside every application container.

---

## Security Considerations

The implementation was designed with the following security considerations:

* The CA private key is encrypted and permission-restricted.
* The server private key is accessible only to authorised administrators and NGINX.
* Certificate directories are mounted read-only inside the container.
* Backend applications remain isolated on the Docker proxy network.
* Only NGINX publishes web ports to the host.
* HTTP requests are redirected to HTTPS.
* Certificates contain explicit Subject Alternative Names.
* Trust is distributed only to authorised client devices.
* Private keys are never transferred to clients.
* Real hostnames, addresses, and environment-specific details are excluded from public documentation.

> [!NOTE]
> HTTPS protects traffic between the client and the reverse proxy. In this design, communication between NGINX and the backend container remains unencrypted but is restricted to the internal Docker bridge network. End-to-end TLS may be introduced where the threat model requires encryption between the reverse proxy and backend services.

---

## Certificate Renewal

Certificates should be monitored and renewed before expiration.

Check the certificate expiry date:

```bash
openssl x509 \
  -in app1.home.arpa.crt \
  -noout \
  -enddate
```

After renewing a certificate:

1. Replace the existing certificate file.
2. Confirm the private key matches the certificate.
3. Test the NGINX configuration.
4. Reload NGINX.
5. Verify the certificate presented to clients.

```bash
docker exec nginx nginx -t
docker exec nginx nginx -s reload
```

A documented renewal process reduces the risk of service disruption caused by expired certificates.

---

## Troubleshooting

### Certificate Warning in the Browser

Confirm that:

* The Root CA certificate is installed in the correct trust store.
* The browser uses the operating-system trust store.
* The requested hostname matches the certificate SAN.
* The certificate has not expired.
* NGINX presents the expected certificate.

---

### Hostname Does Not Resolve

Test internal DNS resolution.

```bash
nslookup app1.home.arpa
```

or:

```bash
dig app1.home.arpa
```

The hostname should resolve to the reverse proxy host.

---

### NGINX Configuration Test Fails

Run:

```bash
docker exec nginx nginx -t
```

Check:

* Certificate paths
* Private-key paths
* File permissions
* Virtual-host syntax
* Duplicate `server_name` values
* Whether TCP port 443 is published

---

### NGINX Cannot Reach the Application

Inspect the proxy network.

```bash
docker network inspect proxy
```

Confirm that both NGINX and the `app1` container are connected.

Test backend resolution from the NGINX container.

```bash
docker exec nginx getent hosts app1
```

Test backend connectivity.

```bash
docker exec nginx curl -I http://app1:80
```

---

### Certificate and Private Key Do Not Match

Compare their public-key hashes.

```bash
openssl x509 \
  -in app1.home.arpa.crt \
  -noout \
  -pubkey |
openssl sha256
```

```bash
openssl pkey \
  -in app1.home.arpa.key \
  -pubout |
openssl sha256
```

The resulting hashes should match.

---

## Summary

This project extended an existing NGINX reverse proxy deployment by introducing trusted internal HTTPS for self-hosted applications.

An internal Certificate Authority was created and used to issue a server certificate for `app1.home.arpa`. The certificate and private key were integrated with the containerised NGINX reverse proxy, allowing NGINX to terminate TLS connections and forward requests to the backend application over the internal Docker network.

The implementation also introduced:

* Subject Alternative Name validation
* Root CA trust distribution
* HTTP-to-HTTPS redirection
* Centralised TLS termination
* Certificate validation
* Certificate renewal considerations
* A repeatable approach for securing additional applications

This project strengthened my understanding of how internal DNS, reverse proxies, Docker networking, X.509 certificates, trust stores, and TLS work together to provide secure application delivery.


