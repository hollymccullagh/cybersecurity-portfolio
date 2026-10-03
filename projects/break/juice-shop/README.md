# Juice Shop
(OWASP Juice Shop Web Security Lab)

Detailed findings and vulnerability analysis can be viewed here:
[FINDINGS.md](FINDINGS.md)

## Overview

This project documents a web application security assessment performed against OWASP Juice Shop running in a controlled Docker lab environment.

The objective of this lab was to develop practical experience with:

- Web application reconnaissance
- HTTP request analysis
- Proxy interception
- Authentication testing
- Input validation weaknesses
- OWASP Top 10 vulnerabilities
- Vulnerability reporting
- Remediation recommendations

Testing was performed using OWASP ZAP and browser developer tools in an isolated local environment.



## Hardware Used

| Hardware | Purpose |
|---|---|
| Kali Linux Workstation | Security testing workstation |



## Software Used

| Software | Purpose |
|---|---|
| Docker | Container platform used to host OWASP Juice Shop |
| OWASP Juice Shop | Deliberately vulnerable web application used for security training |
| OWASP ZAP | Intercepting proxy and web application security testing |
| Firefox Developer Edition | Web browser used for testing and proxy configuration |


## Installation & Setup

### 1. Install Docker

Update package repositries:
```bash
sudo apt update
```

Install Docker:
```bash
sudo apt install docker.io -y
```

Enable and start the Docker service:
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verify Docker installation:
```bash
docker --version
```


### 2. Deploy OWASP Juice Shop

Pull and run the Juice Shop container:
```bash
docker run -d \
--name juice-shop \
-p 127.0.0.1:3000:3000 \
bkimminich/juice-shop
```
127.0.0.1 - Binds the application to localhost so it is only accessible from the local machine

3000:3000 - Maps local port 3000 to port 3000 inside the Docker container

Verify the container is running:
```bash
docker ps
```
Now try to access the application
http://127.0.0.1:3000


### 3. Install OWASP ZAP

Install ZAP
```bash
sudo apt install zaproxy -y
```
Launch ZAP
```bash
zaproxy
```

### 4. Configure Browser Proxy (Firefox)

When using a standard browser instead of the built-in ZAP browser, Firefox can be manually configured to route traffic through the OWASP ZAP proxy.

Proxy settings used:
- Proxy Type: Manual Proxy Configuration
- HTTP Proxy: 127.0.0.1
- Port: 8080

This allows browser traffic to pass through OWASP ZAP for interception and analysis.
Otherwise the built in ZAP browser that launches from within ZAP is preconfigured.

---

## Testing Methodologies

Testing followed a structured web application assessment methodology within a controlled local lab environment.

OWASP Juice Shop was deployed locally within a Docker container to provide an isolated and reproducible testing environment. 
Firefox Developer Edition was configured to route browser traffic through OWASP ZAP operating as an intercepting proxy.

Initial testing focused on reconnaissance and application mapping through manual browsing, request inspection, and OWASP ZAP spidering. 
This assisted in identifying accessible pages, application functionality, API endpoints, and user interaction workflows.

HTTP requests and responses were analysed to better understand authentication mechanisms, session handling, input processing, and client-server communication behaviour.

Manual testing techniques were then used to assess the application for common web application security weaknesses. 
This included request manipulation, authentication testing, parameter tampering, forced browsing attempts, and input validation testing.

Where vulnerabilities were identified, controlled exploitation was performed to validate behaviour and assess potential impact. 
Findings were documented alongside associated risks and remediation recommendations.

---

## Findings Report

Detailed findings and vulnerability analysis can be viewed here:
[FINDINGS.md](FINDINGS.md)

This project significantly improved my understanding of modern web application security concepts and common vulnerabilities through practical hands-on testing. 
Working directly with HTTP requests, proxy interception, application behaviour, and vulnerability validation provided a much deeper understanding than theory alone, particularly around how common web attacks occur and how insecure application design can introduce security risks.


