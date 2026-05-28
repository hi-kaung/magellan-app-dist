# magellan-app-dist

Pre-built production artifact of [magellan-app](https://github.com/) for self-hosting on AWS Ubuntu.

Built with `npm run build` (Next.js 16 App Router). Runs with `npm run start` on port **3001** under PM2.

---

## First-time setup (Ubuntu 22.04 / 24.04)

```bash
# 1. Install Node.js 20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. Install PM2 globally
sudo npm install -g pm2

# 3. Clone this dist repo
git clone git@github.com:hi-kaung/magellan-app-dist.git ~/magellan-app
cd ~/magellan-app

# 4. Install production dependencies only
npm ci --omit=dev

# 5. (Optional) Override .env.production values here if your server-side
#    MAGELLAN_API_BASE_URL differs from the public one. Note: NEXT_PUBLIC_*
#    values are baked in at build time and CANNOT be changed here.

# 6. Start under PM2
pm2 start ecosystem.config.js
pm2 save

# 7. Enable PM2 boot on reboot
pm2 startup systemd -u $USER --hp $HOME
# ^ run the command it prints
```

The app is now listening on `http://0.0.0.0:3001`. Front it with nginx/ALB for TLS and host routing.

---

## Updating to a new build

On your laptop (in the magellan-app source repo):

```bash
npm run deploy
```

On the Ubuntu server:

```bash
cd ~/magellan-app
git pull
npm ci --omit=dev
pm2 reload ecosystem.config.js
```

`pm2 reload` does a zero-downtime restart.

---

## Useful PM2 commands

```bash
pm2 status                    # process list
pm2 logs magellan-app         # tail logs
pm2 logs magellan-app --lines 200
pm2 restart magellan-app
pm2 stop magellan-app
pm2 monit                     # live CPU/memory dashboard
```

Logs land in `./logs/out.log` and `./logs/error.log` (configured in `ecosystem.config.js`).

---

## Environment variables

`.env.production` is loaded automatically by `next start`.

| Var | Side | Purpose |
|---|---|---|
| `MAGELLAN_API_BASE_URL` | server | Used by proxy + auth route handlers (read at runtime). |
| `NEXT_PUBLIC_MAGELLAN_API_BASE_URL` | browser | Used by `/signin` to POST magic-link. **Baked in at build time.** |
| `PORT` | server | Defaults to `3001` (set in `ecosystem.config.js`). |

To change a `NEXT_PUBLIC_*` value you must rebuild from the source repo and redeploy.

---

## nginx reverse-proxy example

```nginx
server {
  listen 443 ssl http2;
  server_name app.magellan.so;

  ssl_certificate     /etc/letsencrypt/live/app.magellan.so/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/app.magellan.so/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:3001;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "upgrade";
  }
}
```
