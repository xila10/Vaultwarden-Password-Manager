# Vaultwarden-Password-Manager
A project using VaultWarden, in combination with Nginx, Docker-Compose and Tailscale.

# Vaultwarden Deployment with Docker, Nginx, and Tailscale

## Overview

This project sets up a self-hosted password manager using Vaultwarden, deployed in Docker, fronted by Nginx as a reverse proxy, and secured with HTTPS over a private network using Tailscale.

The architecture ensures:

* Secure access via HTTPS
* No public exposure to the internet
* Private connectivity through a VPN (Tailscale)
* Clear separation of networking, TLS, and application logic.

---

## Architecture

→ Tailscale VPN
→ Nginx (TLS termination + reverse proxy)
→ Vaultwarden (Docker container)
→ Bitwarden (browser extension)

---

## 1. Docker Installation and Setup

Docker was installed and configured to allow non-root usage.

Vaultwarden was deployed using Docker Compose with a local volume for persistent storage and bound to localhost.

`sudo usermod -aG docker $USER`
`newgrp docker`

### docker-compose.yml

```yaml
version: '3'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      SIGNUPS_ALLOWED: "false"  # change to 'true' for user creation.
    volumes:
      - ./vw_data:/data
    ports:
      - "127.0.0.1:8080:80"
```

### Key Points

* Vaultwarden runs inside a container on port 80
* Exposed locally as `127.0.0.1:8080`
* Not directly accessible externally
* Data persists in `./vw_data`

---

## 2. Verifying Vaultwarden

Vaultwarden was verified locally using:

```
curl http://127.0.0.1:8080
```

This confirms that:

* The container is running
* The service is responding over HTTP

---

## 3. Nginx Reverse Proxy Configuration

Nginx was installed and configured to act as a reverse proxy.

### HTTP Configuration (Port 80)

```nginx
server {
    listen 80;
    server_name VM-name.my-tailscale-name.ts.net;
    return 301 https://$host$request_uri;
}
```

Purpose:

* Accept incoming HTTP traffic
* Redirect all requests to HTTPS

---

### HTTPS Configuration (Port 443)

```nginx
server {
    listen 443 ssl;
    server_name VM-name.my-tailscale-name.ts.net;

    ssl_certificate /home/user/vault_keys/VM-name.my-tailscale-name.ts.net.crt;
    ssl_certificate_key /home/user/vault_keys/VM-name.my-tailscale-name.ts.net.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Key Concepts

* `server_name` matches the Host header from the client request
* `location /` matches all paths
* `proxy_pass` forwards traffic to Vaultwarden
* Original header context preserved

---

## 4. Enabling Nginx Configuration

A sym-link was created to enable the site:

```
sudo ln -s /etc/nginx/sites-available/vaultwarden /etc/nginx/sites-enabled/
```
This projects all changes in `sites-available` to `sites-enabled` - avoiding potentialy missmatching duplicates.

Nginx configuration was tested and reloaded:

```
sudo nginx -t
sudo systemctl reload nginx
```
---

## 5. Port Binding and Traffic Flow

Vaultwarden is not exposed directly. Instead:

* Nginx listens on:

  * Port 80 (HTTP) (configured to redirect towards HTTPS)
  * Port 443 (HTTPS)
  
* Vaultwarden listens internally on:

  * `127.0.0.1:8080`

Traffic flow:

Browser → Nginx (80/443) → proxy_pass → 127.0.0.1:8080 → Vaultwarden

---

## 6. Tailscale Integration

Tailscale was installed on both:

* The VM (server)
* The host machine (client)

This creates a private network between devices. Follow instructions on https://tailscale.com/.

### Verification

```
tailscale status
```

Each device receives a Tailscale IP (100.x.x.x range).

---

## 7. DNS with MagicDNS

Tailscale MagicDNS provides hostname resolution.

Example hostname:

```
VM-name.my-tailscale-name.ts.net
```

This resolves internally to the VM’s Tailscale IP.

---

## 8. TLS Certificate Configuration

A TLS certificate was generated using Tailscale:

```
tailscale cert VM-name.my-tailscale-name.ts.net
```

The generated certificate and key were used in Nginx.

Important:

* Certificates must match the exact hostname
* Nginx must be configured with valid paths and syntax
* Enable signups to create users when application is live, then disable.
---

## 9. Minor HTTPS troubbleshooting:

- fixing configuration issues (including missing semicolons)

**Verification:**

- checking traffic on ports

```
ss -tulnp | grep 443
```

Access via browser:

```
https://VM-name.my-tailscale-name.ts.net
```

Result:

* No certificate warnings
* Secure connection established
* Vaultwarden login page accessible
* Creation of users accepted

---

## 10. Vaultwarden Account Setup

Vaultwarden does not include predefined users.

Process:

1. Temporarily enable signups (as before mentioned):

```yaml
SIGNUPS_ALLOWED: "true"
```

2. Recreate container:

```
docker compose up -d --force-recreate
```

3. Create account via web UI

4. Disable signups again:

```yaml
SIGNUPS_ALLOWED: "false"
```

---

## 11. Password Management Behavior

* Password generation rules are controlled by the client via Bitwarden (browser extension)
* Not enforced by the server
* Recommended:

  * 16+ character passwords
  * Strong master password
  * Number of itterations, KDF; Argon2 or PBKDF2-SHA256: 600.000+

---

## 12. Key Learnings

### Nginx Routing Logic

* `server_name` matches Host header
* `location` matches URL path
* `proxy_pass` forwards traffic internally

### Networking

* Tailscale provides secure connectivity
* No public exposure required
* Tailscale provides certifficates and direct peer-to-peer communication.
* No need for port forwarding, firewall configurations or a public domain.

### TLS

* TLS is terminated at Nginx, with internal communication over HTTP for simplicity and performance.
* Browser validates certificate independently

---

## Final Result

A fully functional, secure Vaultwarden deployment with:

* Docker-based application isolation
* Reverse proxy via Nginx
* HTTPS with valid certificates
* Private network access via Tailscale

---

## Potential future Improvements

* Add security headers (HSTS, CSP)
* Configure automated backups
* Enable 2FA in Vaultwarden
* Restrict access further via Tailscale ACLs

