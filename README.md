# Reverse Proxy Setup

Nginx reverse proxy running in Docker Compose, sitting in front of three backend services and routing traffic by both URL path and subdomain. Includes load balancing across multiple instances, rate limiting, HTTPS with SSL termination, and a custom error page.

## Architecture

```
                      ┌─────────────────────┐
                      │    YOUR BROWSER      │
                      └─────────┬───────────┘
                                │
                      ┌─────────▼───────────┐
                      │   NGINX PROXY        │
                      │   :80  → :443        │
                      └──┬──────┬──────┬────┘
                         │      │      │
             ┌───────────▼─┐ ┌──▼───┐ ┌▼──────┐
             │   App 1     │ │App 2 │ │ App 3 │
             │  (x2 inst.) │ │ (x2) │ │  (x2) │
             └─────────────┘ └──────┘ └───────┘
```

## Features

- Routes traffic to the correct backend based on URL path (`/app1/`, `/app2/`, `/app3/`).
- Routes traffic to the correct backend based on subdomain (`app1.localhost`, `app2.localhost`, `app3.localhost`).
- Load balances across two instances of each backend service using Nginx `upstream`.
- Rate limits each IP to 1 request per second with a burst of 5 — returns a custom error page when exceeded.
- Terminates HTTPS at the proxy level using a self-signed SSL certificate.
- Automatically redirects all HTTP traffic to HTTPS.
- Shows a custom error page for 502, 503, and 504 errors.
- Exposes a live Nginx stats page at `/status`.
- All backend containers include health checks.

## Quick Start

**1. Clone the repo**
```bash
git clone https://github.com/ahmed-m-miqdad/reverse-proxy-setup.git
cd reverse-proxy-setup
```

**2. Generate a self-signed SSL certificate**
```bash
mkdir -p nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/nginx.key \
  -out nginx/ssl/nginx.crt \
  -subj "/CN=localhost"
```

**3. Add subdomains to your hosts file**
```bash
sudo nano /etc/hosts

# add these lines
127.0.0.1 app1.localhost
127.0.0.1 app2.localhost
127.0.0.1 app3.localhost
```

**4. Start everything**
```bash
docker compose up -d
```

**5. Visit**

| URL | What you see |
|---|---|
| `https://localhost/app1/` | App 1 via path routing |
| `https://localhost/app2/` | App 2 via path routing |
| `https://localhost/app3/` | App 3 via path routing |
| `https://app1.localhost` | App 1 via subdomain routing |
| `https://app2.localhost` | App 2 via subdomain routing |
| `https://app3.localhost` | App 3 via subdomain routing |
| `http://localhost/status` | Nginx live stats |

> **Note:** Since the SSL certificate is self-signed, your browser will show a security warning. Click **Advanced → Proceed** to continue. This is expected for local development.

## Project Structure

```
reverse-proxy-setup/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf        # full proxy config — routing, upstream, rate limit, SSL
│   ├── error.html        # custom error page for 502/503/504
│   └── ssl/              # not committed — generate locally
│       ├── nginx.crt
│       └── nginx.key
├── apps/
│   ├── app1/
│   │   └── index.html
│   ├── app2/
│   │   └── index.html
│   └── app3/
│       └── index.html
├── .gitignore
└── README.md
```

## Testing

```bash
# Test path-based routing
curl -sk https://localhost/app1/
curl -sk https://localhost/app2/
curl -sk https://localhost/app3/

# Test rate limiting — you should see 503 after a few rapid requests
for i in {1..20}; do curl -sk https://localhost/app1/ -o /dev/null -w "%{http_code}\n"; done

# Check Nginx stats
curl http://localhost/status

# Test error page — stop an app and visit its route
docker compose stop app1_1 app1_2
curl -sk https://localhost/app1/
```

## Useful Commands

```bash
# Start all containers
docker compose up -d

# Stop all containers
docker compose down

# View proxy logs
docker compose logs -f nginx-proxy

# Check container health
docker compose ps
```

## Requirements

- Docker
- Docker Compose
- `openssl` — for generating the self-signed certificate

## Author

Ahmed M Miqdad — [github.com/ahmed-m-miqdad](https://github.com/ahmed-m-miqdad)