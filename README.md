# Secure Nexus Docker Registry with mTLS Authentication

## Overview

This project implements a secure private Docker Registry using **Sonatype Nexus Repository Manager**, **Nginx Reverse Proxy**, and **Mutual TLS (mTLS) authentication**.

The goal of this project is to create a production-like artifact repository environment where Docker clients must provide a valid client certificate before accessing the private registry.

This setup demonstrates how DevOps teams can secure internal artifact repositories used by applications, containers, and Kubernetes workloads.

---

# Architecture

```
                         Docker Client
                              |
                              |
                    Client Certificate
                              |
                              |
                         HTTPS :443
                              |
                              |
                    +----------------+
                    |     Nginx      |
                    | Reverse Proxy  |
                    +----------------+
                              |
                 +------------+------------+
                 |                         |
                 |                         |
             Nexus UI              Docker Registry
             :8081                     :5000

```

---

# Technologies Used

- Docker
- Docker Compose
- Sonatype Nexus Repository Manager
- Nginx
- OpenSSL
- TLS / mTLS Authentication
- Docker Registry API
- Linux

---

# Project Components

## Nexus Repository Manager

Nexus is used as a private artifact repository.

Configured repositories:

- Docker Hosted Repository
- Docker Proxy Repository
- Docker Group Repository
- Kubernetes image proxy repository
- Ubuntu package proxy repositories


Nexus responsibilities:

- Store private Docker images
- Proxy external repositories
- Manage internal artifacts
- Provide registry API


---

## Nginx Reverse Proxy

Nginx acts as the security gateway in front of Nexus.

Responsibilities:

- HTTPS termination
- TLS certificate handling
- Client certificate validation
- Reverse proxy routing
- Forward Docker Registry traffic


Traffic flow:

```
Docker Client

      |
      | HTTPS + Client Certificate

      v

Nginx :443

      |
      +----------------+
      |                |
      v                v

Nexus UI          Docker Registry
:8081                 :5000

```

---

# TLS / mTLS Authentication

This project uses Mutual TLS authentication.

Normal HTTPS only verifies the server identity.

With mTLS, both sides authenticate:

- Server verifies client
- Client verifies server


Authentication flow:

```
                 Certificate Authority
                         |
              +----------+----------+
              |                     |
              |                     |
        Nexus Server          Docker Client
        Certificate           Certificate

```

Connection process:

1. Docker client connects to Nexus using HTTPS.
2. Nginx sends the server certificate.
3. Docker client sends the client certificate.
4. Nginx validates the certificate against the internal CA.
5. Access is granted only to trusted clients.


---

# Certificate Structure

Certificates are generated using OpenSSL.


Directory structure:

```
certs/

├── ca.crt
├── ca.key

├── nexus.crt
├── nexus.key

├── client.crt
└── client.key

```

Certificate roles:

| File | Purpose |
|---|---|
| ca.crt | Internal Certificate Authority |
| ca.key | CA private key |
| nexus.crt | Nexus server certificate |
| nexus.key | Nexus server private key |
| client.crt | Docker client certificate |
| client.key | Docker client private key |


---

# Certificate Verification

Verify server certificate:

```bash
openssl verify \
-CAfile ca.crt \
nexus.crt
```


Verify client certificate:

```bash
openssl verify \
-CAfile ca.crt \
client.crt
```


Expected result:

```
nexus.crt: OK

client.crt: OK
```

---

# Nginx Configuration

Nginx requires client certificates:

```nginx
listen 443 ssl;

server_name nexus.company.local;


ssl_certificate /certs/nexus.crt;

ssl_certificate_key /certs/nexus.key;


ssl_client_certificate /certs/ca.crt;

ssl_verify_client on;

```

This enables Mutual TLS authentication.

---

# Docker Client Certificate Configuration

Docker requires certificates in:

```
/etc/docker/certs.d/nexus.company.local/
```


Required files:

```
ca.crt
client.cert
client.key
```


Restart Docker:

```bash
sudo systemctl restart docker
```

---

# Testing HTTPS Connection

## Without Client Certificate

Command:

```bash
curl https://nexus.company.local
```


Expected result:

```
400 No required SSL certificate was sent
```


The request is rejected because the client certificate is missing.


---

## With Client Certificate

Command:

```bash
curl \
--cert certs/client.crt \
--key certs/client.key \
--cacert certs/ca.crt \
https://nexus.company.local
```


Expected result:

Nexus response is returned successfully.


---

# Docker Registry Authentication

Login to private registry:

```bash
docker login nexus.company.local
```


Example:

```
Username: admin
Password:

Login Succeeded
```

---

# Docker Image Push Test

Tag image:

```bash
docker tag nginx-test \
nexus.company.local/nginx-test:latest
```


Push image:

```bash
docker push \
nexus.company.local/nginx-test:latest
```


After successful push, the image will be available inside Nexus Docker Hosted Repository.

---

# Troubleshooting

## 1. Missing Client Certificate

Error:

```
400 No required SSL certificate was sent
```


Cause:

The client certificate was not provided.


Solution:

Use:

```bash
--cert client.crt
--key client.key
```

---

## 2. Docker Certificate Naming Issue

Docker expects:

```
client.cert
```

not:

```
client.crt
```


Correct structure:

```
/etc/docker/certs.d/nexus.company.local/

├── ca.crt
├── client.cert
└── client.key

```

---

## 3. 413 Request Entity Too Large

Problem:

Docker image push fails during upload.


Cause:

Nginx request body size limitation.


Solution:

Increase Nginx configuration:

```nginx
client_max_body_size 2G;
```

---

# Current Implementation Status

Implemented:

- ✅ Nexus Repository Manager
- ✅ Docker Hosted Repository
- ✅ Docker Proxy Repository
- ✅ Docker Group Repository
- ✅ Nginx Reverse Proxy
- ✅ HTTPS Encryption
- ✅ Mutual TLS Authentication
- ✅ Docker Client Certificate Authentication
- ✅ Docker Image Push Test


---

# Future Improvements

Planned improvements:

- Connect Kubernetes cluster to Nexus Registry
- Configure Kubernetes nodes to pull private images
- Use Nexus as Kubernetes package mirror
- Automate certificate generation
- Add CI/CD pipeline integration
- Add monitoring and logging


---

# DevOps Skills Demonstrated

- Linux Administration
- Docker Container Management
- Docker Registry Implementation
- Nexus Repository Management
- Nginx Reverse Proxy Configuration
- TLS Certificate Management
- mTLS Authentication
- Network Troubleshooting
- Secure Artifact Management
- Infrastructure Automation


---

# Author

**Parsa Zarabi**

DevOps Learning Portfolio
