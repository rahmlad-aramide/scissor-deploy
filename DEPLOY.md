# Deploying Scissor

Production topology: one VPS running Docker Compose — Postgres, Redis, the API,
a BullMQ worker, and Caddy (TLS termination + reverse proxy + the SPA's static
files) — serving three subdomains that must all point at the VPS's IP:

| Subdomain | Purpose |
|---|---|
| `scissor.rahmlad.com` | the React app |
| `api.scissor.rahmlad.com` | the Express API |
| `s.rahmlad.com` | short links (`s.rahmlad.com/abc123`) |

## 1. DNS

Create three `A` (or `AAAA`) records pointing at the VPS's IP:

```
scissor.rahmlad.com      A   <vps-ip>
api.scissor.rahmlad.com  A   <vps-ip>
s.rahmlad.com            A   <vps-ip>
```

Caddy will automatically request and renew Let's Encrypt certificates for all
three the first time it starts, as long as ports 80/443 are reachable.

## 2. Accounts you need to set up yourself

- **Resend**: create an account, verify the `rahmlad.com` domain (DNS TXT/DKIM
  records Resend gives you), and generate an API key.
- **Google OAuth** (optional — the "Continue with Google" button hides itself
  if unset): create an OAuth 2.0 Client ID in Google Cloud Console, add
  `https://api.scissor.rahmlad.com/auth/google/callback` as an authorized
  redirect URI.
- **MaxMind** (optional — only needed for the country breakdown in Analytics):
  create a free GeoLite2 account, download `GeoLite2-Country.mmdb`, and place
  it somewhere the API container can read (mount it in and set `GEOIP_DB_PATH`).

## 3. Server setup

Scissor is three separate git repos — `scissor-api`, `scissor-client`, and this one
(`scissor-deploy`, holding only `docker-compose.prod.yml`, `Caddyfile`, and this
file). Clone all three as siblings under one parent directory; the compose file's
`../scissor-api` and `../scissor-client/dist` paths depend on exactly this layout:

```bash
mkdir -p /opt/scissor && cd /opt/scissor
git clone <scissor-api repo url> scissor-api
git clone <scissor-client repo url> scissor-client
git clone <scissor-deploy repo url> scissor-deploy
cp scissor-api/.env.example scissor-api/.env
```

Edit `scissor-api/.env` for production:

```
DATABASE_URL=postgresql://scissor:<POSTGRES_PASSWORD>@postgres:5432/scissor?schema=public
REDIS_URL=redis://redis:6379
NODE_ENV=production
PORT=4000
SESSION_COOKIE_DOMAIN=.scissor.rahmlad.com
CLIENT_ORIGIN=https://scissor.rahmlad.com
CLIENT_URL=https://scissor.rahmlad.com
SHORT_BASE_URL=https://s.rahmlad.com
ENCRYPTION_KEY=<openssl rand -hex 32>
RESEND_API_KEY=<your Resend key>
MAIL_FROM="Scissor <no-reply@rahmlad.com>"
GOOGLE_CLIENT_ID=<optional>
GOOGLE_CLIENT_SECRET=<optional>
GOOGLE_CALLBACK_URL=https://api.scissor.rahmlad.com/auth/google/callback
MAXMIND_LICENSE_KEY=<optional>
```

Set the Postgres password used by Compose:

```bash
echo "POSTGRES_PASSWORD=$(openssl rand -hex 24)" > scissor-deploy/.env
```

(Docker Compose reads `.env` from the same directory as the compose file —
`scissor-deploy/.env` — for `${POSTGRES_PASSWORD}`. Make sure the same
password appears in `scissor-api/.env`'s `DATABASE_URL`.)

## 4. Build the frontend and bring the stack up

All `docker compose` commands below are run from inside `scissor-deploy/` —
its relative paths (`../scissor-api`, `../scissor-client/dist`) assume that.

```bash
cd /opt/scissor/scissor-client
cp .env.example .env.production
# edit .env.production:
#   VITE_API_URL=https://api.scissor.rahmlad.com
#   VITE_SHORT_BASE=https://s.rahmlad.com
npm ci
npm run build   # outputs scissor-client/dist, which Caddy serves directly

cd /opt/scissor/scissor-deploy
docker compose -f docker-compose.prod.yml up -d --build
```

The `api` service runs `prisma migrate deploy` on boot, so the schema is
applied automatically on first start and on every redeploy.

## 5. Redeploying

Pull whichever repo(s) changed, then rebuild from `scissor-deploy/`:

```bash
cd /opt/scissor/scissor-api && git pull && cd -
cd /opt/scissor/scissor-client && git pull && npm run build && cd -
cd /opt/scissor/scissor-deploy && git pull
docker compose -f docker-compose.prod.yml up -d --build
```

## 6. What to double-check after going live

- Visit `https://scissor.rahmlad.com`, sign up, confirm the verification
  email actually arrives (Resend dashboard shows the send).
- Create a link, confirm `https://s.rahmlad.com/<code>` redirects.
- Check `docker compose -f docker-compose.prod.yml logs worker` shows click
  events being recorded.
