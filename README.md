# SignServer Docker on Ubuntu

This repository provides a simple way to run **Keyfactor SignServer Community** in Docker on Ubuntu, create a local management certificate authority with OpenSSL, generate a browser client certificate for administrator access, and open the SignServer web interface. The SignServer Community container is published by Keyfactor, and the documented quick-start flow uses a mounted Management CA certificate together with client-certificate authenticated access.
## Overview

SignServer is a PKI-based signing server used for code signing, document signing, timestamps, and related signing workflows. In the container-based quick start, the service is exposed on `/signserver/`, and administrator authentication is based on a client certificate trusted by the mounted Management CA certificate.

This repository is designed for a local lab, test VM, or Ubuntu server where the goal is to get a working SignServer instance quickly. The official container guidance also notes that the default embedded database mode is suitable for evaluation and ephemeral testing rather than hardened production use.

## Repository clone

Clone the repository and move into the project directory:

```bash
git clone https://github.com/Hishem123/signserver-docker.git
cd signserver-docker/
```

## Prerequisites

Use an Ubuntu server or VM with Docker Engine and the Docker Compose plugin installed. SignServer's current quick-start documentation for the community container assumes Docker is installed first, then the container is started with the required certificate mount and published ports.

The following tools should be available:

- `docker`
- `docker compose`
- `openssl`
- a modern web browser such as Firefox or Chrome

## Project structure

A practical repository layout looks like this:

```text
signserver-docker/
â”œâ”€â”€ docker-compose.yml
â”œâ”€â”€ certs/
â”‚   â”œâ”€â”€ ManagementCA.key.pem
â”‚   â”œâ”€â”€ ManagementCA.pem
â”‚   â”œâ”€â”€ client.key.pem
â”‚   â”œâ”€â”€ client.csr.pem
â”‚   â”œâ”€â”€ client.crt.pem
â”‚   â””â”€â”€ signserver-admin.p12
â””â”€â”€ README.md
```

The `ManagementCA.pem` file is mounted into the container as the management CA trust anchor, while the exported `.p12` file is imported into the browser for admin authentication.

## Step 1: Create the Management CA

Create a local certificate authority that SignServer will use to trust the administrator client certificate. OpenSSL-based CA creation follows the standard pattern of generating a private key first and then creating a self-signed X.509 certificate in PEM format.

```bash
mkdir -p certs

openssl genrsa -out certs/ManagementCA.key.pem 2048

openssl req -new -x509 -days 3650 \
  -key certs/ManagementCA.key.pem \
  -out certs/ManagementCA.pem \
  -subj "/C=TN/ST=Tunis/L=Tunis/O=MYCERT/OU=PKI/CN=MYCERT"
```

This produces two important files:

- `certs/ManagementCA.key.pem`: the private key of the CA
- `certs/ManagementCA.pem`: the public CA certificate mounted into SignServer

Keep the private key on the server and do not commit it to Git. Git ignore guidance commonly recommends ignoring certificate and key material such as `*.pem`, `*.crt`, `*.p12`, and `*.key` in repositories that contain deployment secrets.
## Step 2: Start SignServer with Docker

Run the SignServer container with Docker Compose:

```bash
sudo docker compose up -d
```

The official SignServer quick-start flow exposes the application on ports `8080` and `8443`, mounts the Management CA certificate into the container, and makes the application available under `/signserver/`.

To verify that the container is up:

```bash
sudo docker ps
```

A healthy container should show published ports for `8080` and `8443`. Docker's `ps` output documents published host-to-container port mappings, which is the easiest way to confirm the service is listening externally.

## Step 3: Verify HTTP access

Check that the web application responds on HTTP:

```bash
curl -I http://localhost:8080/signserver/
```

A successful response should return `HTTP/1.1 200 OK`, which confirms that the SignServer web application is deployed and reachable on the HTTP endpoint.

If needed, inspect the logs:

```bash
sudo docker logs --tail=200 signserver
```

When SignServer starts correctly, the logs typically show the application deployment, service startup, and the final startup completion of the `signserver.ear` application.

## Step 4: Create the administrator client certificate

SignServer admin access uses a **client certificate**, not just a username and password. The official quick-start guidance says the browser must present a client certificate that chains to the mounted Management CA certificate.

Generate the browser client key:

```bash
openssl genrsa -out certs/client.key.pem 2048
```

Create the certificate signing request:

```bash
openssl req -new \
  -key certs/client.key.pem \
  -out certs/client.csr.pem \
  -subj "/C=TN/ST=Tunis/L=Tunis/O=MYCERT/OU=PKI/CN=MYCERT"
```

Sign the client certificate with the CA created earlier:

```bash
openssl x509 -req \
  -in certs/client.csr.pem \
  -CA certs/ManagementCA.pem \
  -CAkey certs/ManagementCA.key.pem \
  -CAcreateserial \
  -out certs/client.crt.pem \
  -days 365
```

Export the certificate and private key as a PKCS#12 file for browser import:

```bash
openssl pkcs12 -export \
  -in certs/client.crt.pem \
  -inkey certs/client.key.pem \
  -out certs/signserver-admin.p12 \
  -name "MYCERT" \
  -passout pass:mycert
```

This PKCS#12 file contains the administrator certificate and private key in a browser-friendly format. OpenSSL documentation and certificate-handling references consistently use `openssl pkcs12 -export` for this purpose.

## Step 5: Import the certificate into the browser

Import `certs/signserver-admin.p12` into Firefox or Chrome. The SignServer quick-start documentation describes importing the client certificate into the browser certificate store and then using that certificate when accessing the admin interface.

In Firefox, the usual path is:

- **Settings**
- **Privacy & Security**
- **Certificates**
- **View Certificates**
- **Your Certificates**
- **Import**

Select `signserver-admin.p12` and use the export password:

```text
mycert
```

## Step 6: Access the web interface

After importing the certificate, open SignServer in the browser:

```text
https://localhost:8443/signserver
```

The SignServer documentation states that the service is accessed under `/signserver/`, and the admin-related flow relies on browser client-certificate authentication.

If HTTPS is not yet available in the local environment, the HTTP endpoint may still answer successfully at:

```text
http://localhost:8080/signserver/
```

In lab setups, HTTP may come up first while TLS configuration is still being initialized or adjusted, but the container quick start is designed around HTTPS and client-certificate use for administration.

## Admin access path

The main application is available at `/signserver/`, while the administrative web interface is commonly available at:

```text
https://localhost:8443/signserver/adminweb/
```

The SignServer Administration Web manual documents the AdminWeb endpoint under `/signserver/adminweb/`.

## Common workflow after login

Once access works, the next common tasks are creating a crypto worker, activating it with its authentication code, and then creating signer workers such as `PDFSigner`. SignServer's worker documentation explains that signer workers rely on a crypto token worker and are managed through the AdminWeb worker pages or the administration CLI.

For example, a `PDFSigner` worker must reference a configured crypto worker through the `CRYPTOTOKEN` property; otherwise SignServer reports that no cryptotoken is configured.

## Troubleshooting

### Container is running but HTTPS fails

If `curl -k -I https://localhost:8443/signserver/` returns a TLS handshake error, inspect the logs and confirm that the Management CA certificate is mounted correctly inside the container. The quick-start container flow depends on the CA certificate mount, and TLS handshake failures are commonly tied to trust or listener configuration issues.

Useful commands:

```bash
sudo docker logs --tail=200 signserver
ls -l certs/ManagementCA.pem
openssl x509 -in certs/ManagementCA.pem -noout -subject -issuer -dates
sudo docker exec -it signserver sh
ls -l /mnt/external/secrets/tls/cas/
```

### HTTP works but admin login does not

If the browser opens the page but shows a client-certificate authentication warning, the client certificate is either missing from the browser, not signed by the mounted CA, or not selected during the TLS prompt. Keyfactor's quick-start instructions explicitly require browser import of the client certificate before admin login succeeds.

### Worker activation fails with cryptotoken errors

If a signer such as `PDFSigner` shows an error like `No cryptotoken configured`, create and activate a crypto worker first, then configure the signer to reference it by name with the `CRYPTOTOKEN` property. SignServer's crypto token and worker documentation describe this as the normal design pattern.

## Security notes

This setup is suitable for testing, learning, and lab validation. The SignServer container logs warn that the default H2 in-memory database is intended only for ephemeral testing, and the official documentation recommends an external database for more durable deployments.

Private keys, exported browser certificates, and CA material should not be committed to GitHub. GitHub's ignore-file guidance recommends tracking the `.gitignore` file itself while excluding local secret files from version control.

## Quick command summary

```bash
# Clone repository
git clone https://github.com/Hishem123/signserver-docker.git
cd signserver-docker/

# Create CA
mkdir -p certs
openssl genrsa -out certs/ManagementCA.key.pem 2048
openssl req -new -x509 -days 3650 \
  -key certs/ManagementCA.key.pem \
  -out certs/ManagementCA.pem \
  -subj "/C=TN/ST=Tunis/L=Tunis/O=MYCERT/OU=PKI/CN=MYCERT"

# Start SignServer
sudo docker compose up -d

# Check HTTP
curl -I http://localhost:8080/signserver/

# Create client certificate
openssl genrsa -out certs/client.key.pem 2048
openssl req -new \
  -key certs/client.key.pem \
  -out certs/client.csr.pem \
  -subj "/C=TN/ST=Tunis/L=Tunis/O=MYCERT/OU=PKI/CN=MYCERT"
openssl x509 -req \
  -in certs/client.csr.pem \
  -CA certs/ManagementCA.pem \
  -CAkey certs/ManagementCA.key.pem \
  -CAcreateserial \
  -out certs/client.crt.pem \
  -days 365
openssl pkcs12 -export \
  -in certs/client.crt.pem \
  -inkey certs/client.key.pem \
  -out certs/signserver-admin.p12 \
  -name "MYCERT" \
  -passout pass:mycert
```

## Suggested .gitignore

A practical `.gitignore` for this repository should exclude certificate and key material while keeping the folder structure and documentation in Git:

```gitignore
# Certificates and keys
*.pem
*.crt
*.key
*.p12
*.pfx

# Logs
*.log

# Local env
.env

# Keep docs
!README.md
!certs/
!certs/README.md
```

Ignoring secret and certificate files aligns with common Git ignore practices for OpenSSL and deployment repositories.
# Export P12 for browser
openssl pkcs12 -export -in certs/client.crt.pem -inkey certs/client.key.pem \
  -out certs/signserver-admin.p12 -name "MYCERT" -passout pass:mycert

  ## add certificat to browser
  ## ACCESS TO web
  https://localhost:8443/signserver
