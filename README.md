# nginx-spnego

Docker image of **NGINX with SPNEGO (Kerberos) module** precompiled for GSSAPI-based authentication.  
This image is intended to be used in environments with [FreeIPA](https://www.freeipa.org/) or Windows Server for secure, enterprise-grade SSO authentication.


## 🐳 Usage

### 1. Create a  Service Principal for NGINX

#### 1.1 For FreeIPA

> Replace hostnames with your actual server FQDNs.

```bash
ipa service-add HTTP/test.example.com
```

#### 1.2 For Windows Server
> Replace hostnames with your actual server FQDNs.

```
setspn -A HTTP/test.example.com <user>
```
User must exist!

### 2. Generate a Keytab for the NGINX Host

#### 2.1 For FreeIPA server

```bash
ipa-getkeytab -s <freeipa-server> \
  -p HTTP/test.example.com@EXAMPLE.COM \
  -k nginx.keytab
```

#### 2.2 For Windows Server

```
ktpass -princ HTTP/test.example.com@<WINDOWS_DOMAIN> -mapuser <user>@<windows_domain> -pass <user_password> -out ./nginx.keytab -ptype KRB5_NT_PRINCIPAL -crypto RC4-HMAC-NT
```

### 3. Prepare Your NGINX Configuration

Place your SPNEGO-enabled NGINX site configuration file in `./default`. You can use default configuration as example

### 4. Run the Docker Container

```bash
docker run -it --rm \
  -v $(pwd)/default:/etc/nginx/sites-available/default \
  -v $(pwd)/nginx.keytab:/etc/nginx/nginx.keytab \
  --net=host \
  suprematic/nginx-spnego:latest
```

> Note: `--net=host` is used to allow NGINX to bind to system ports.

### 5. Test the Setup

From a Kerberos-authenticated client:

```bash
curl -vv --negotiate -u : http://test.example.com/
```
Depend on your system configuration, you should be authenticated before (for linux):
```bash
kinit <username>
```

If everything is configured correctly, the server should respond without prompting for credentials.

Some browsers may require additional configurations!
For example, MS Edge requires that site must be added into 'Local Intranet' list and available via HTTPS.


Here’s a `README.md` description for your SPNEGO-based authentication project to protect internal Kubernetes HTTP applications. It explains the architecture, purpose, and how to use it:

---

## Usage in Kubernetes

### 🔒 Use Case

This solution is ideal for internal web applications deployed in a Kubernetes cluster where centralized authentication via **FreeIPA**, **Active Directory**, or another Kerberos-compatible realm is required. It integrates seamlessly with an Ingress controller and supports secure header-based identity propagation.

---

### 🧩 Architecture Overview

* **NGINX + SPNEGO**: Runs as a standalone deployment behind an internal service.
* **Ingress Auth Integration**: The Ingress controller delegates authentication to the SPNEGO service using the `auth-url` and `auth-response-headers` annotations.
* **Kerberos Keytab**: Provided securely via Kubernetes Secret.
* **User Identity**: Passed to backend applications via the `X-Authenticated-User` HTTP header.

---

### 🚀 Components

#### 1. `ConfigMap`: NGINX Configuration

Defines the NGINX server block, enabling SPNEGO authentication via the `auth_gss` directive. The user identity is exposed with the `X-Authenticated-User` header.

#### 2. `Secret`: Kerberos Keytab

Contains the keytab file (`http-headers.keytab`) for the SPNEGO service principal (e.g., `HTTP/spnego-auth.default.svc.cluster.local@YOUR.REALM`), base64-encoded.

#### 3. `Deployment`: SPNEGO Auth Service

Runs the custom NGINX image (`suprematic/nginx-spnego`) with:

* Mounted keytab at `/etc/nginx/nginx.keytab`
* Mounted `nginx.conf` from the ConfigMap
* Logs to `stdout` and `stderr` for easy access

#### 4. `Service`: Internal Access

A ClusterIP service exposes the NGINX pod at port 80.

#### 5. `Ingress`: Authentication Gateway

Ingress configuration for the protected application. Uses `auth-url` to delegate requests to the SPNEGO service and propagates the `X-Authenticated-User` header to the backend.
