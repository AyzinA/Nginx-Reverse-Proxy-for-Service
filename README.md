# Nginx Reverse Proxy for Service Deployment

This setup provides a **secure reverse proxy** configuration using **Nginx** to handle HTTPS traffic, webhooks, and REST API requests for backend services such as **n8n**, **Taiga**, or similar applications.

---

## 🧩 Directory Structure

```
.
├── docker-compose.yaml
├── certs
│   ├── cert.crt
│   └── key.key
└── nginx
    └── conf.d
        └── nginx.conf
```

---

## 🐳 Docker Compose Overview

The `docker-compose.yaml` defines the Nginx service, which acts as a reverse proxy in front of your application containers.

You’ll need to:
- Replace `SERVICE_NAME` with the actual Docker service name of your app (e.g., `web`).
- Replace `SERVICE_PORT` with the internal port that your app listens on (e.g., `8080`).
- Update the `server_name` directives to match your actual domain (e.g., `web.example.com`).

Example snippet:

```yaml
services:
  nginx:
    image: nginx:1.28.0
    container_name: reverse-proxy
    restart: unless-stopped
    depends_on:
      - SERVICE_NAME
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
```

---

## ⚙️ Nginx Configuration

File: `nginx/conf.d/nginx.conf`

This configuration handles HTTP → HTTPS redirection, SSL termination, and traffic forwarding to the backend service.

### HTTP → HTTPS Redirect
```nginx
server {
  listen 80;
  server_name web.example.com;
  return 301 https://$host$request_uri;
}
```

### HTTPS Server Block
```nginx
server {
  listen 443 ssl http2;
  server_name web.example.com;

  ssl_certificate     /etc/nginx/certs/fullchain.crt;
  ssl_certificate_key /etc/nginx/certs/key.key;
  client_max_body_size 50m;
}
```

---

## 🔁 Proxying Configuration

### Webhooks (Long-Lived Connections)
The following block is optimized for webhook traffic, disabling buffering and extending timeouts:

```nginx
location ~ ^/(webhook|webhook-test)/ {
  proxy_pass http://SERVICE_NAME:SERVICE_PORT;
  proxy_http_version 1.1;
  proxy_set_header Host              $host;
  proxy_set_header X-Real-IP         $remote_addr;
  proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header Upgrade           $http_upgrade;
  proxy_set_header Connection        $connection_upgrade;
  proxy_buffering off;
  proxy_read_timeout 600s;
  proxy_send_timeout 600s;
}
```

### UI and REST API Requests
All other requests are routed through this standard proxy block:
```nginx
location / {
  proxy_pass http://SERVICE_NAME:SERVICE_PORT;
  proxy_http_version 1.1;
  proxy_set_header Host              $host;
  proxy_set_header X-Real-IP         $remote_addr;
  proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header Upgrade           $http_upgrade;
  proxy_set_header Connection        $connection_upgrade;
  proxy_read_timeout 600s;
  proxy_send_timeout 600s;
}
```

---

## 🔒 SSL Certificates

Mount your TLS certificates in the container under `/etc/nginx/certs`.

Expected files:
```
certs/
├── cert.crt
└── key.key
```

You can generate these using your intermediate CA or with Let's Encrypt.

---

## 🧠 Key Notes

- **`SERVICE_NAME`** → must match the Docker Compose service name of your backend app.
- **`SERVICE_PORT`** → must match the port the backend listens on internally.
- **Webhooks** require longer timeouts and buffering disabled.
- The configuration supports **WebSocket** upgrades (`Upgrade` and `Connection` headers).
- If you use a self-signed or intermediate CA, ensure clients trust the CA certificate chain.

---

## ✅ Verification

After starting the stack:

```bash
docker compose up -d
docker compose logs -f nginx
```

Then open your browser:
```
https://web.example.com
```

Health check endpoint:
```
https://web.example.com/healthz
```

If it returns “ok”, the proxy is correctly serving traffic.

---

## 🧾 License

This configuration is provided under the MIT License.
You may freely modify and redistribute it for your projects.
