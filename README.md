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
