
# 🐹 animalsplit-on-pi

> Deploying a full-stack travel expense web app on a Raspberry Pi — fully self-hosted, HTTPS-secured, and DDNS-resilient.

## 🚀 Overview

This repo documents how I moved [animalsplit.com](https://animalsplit.com), a small side project for travel expense splitting, from AWS to a Raspberry Pi 5 sitting in my living room. The stack includes:

- 🐳 Dockerized frontend and backend
- 🔐 Nginx + Let's Encrypt SSL (via certbot and Route 53 DNS)
- 🔄 Dynamic IP handling with a DDNS cron job
- 📡 AWS Route 53 one-click domain setup script

It’s light, simple, and free to run (except your electricity).

---

## 🗂️ Project Structure

```
.
├── docker-compose.yml
├── route53-init.sh                  # Create or find Route 53 hosted zone
├── route53-ddns-cron.sh             # Cronjob to update dynamic IP to Route 53
├── animal-split-frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── ...
└── animal-split-core-pi/
    ├── Dockerfile
    └── docker-entrypoint.sh
```

---

## 🛠️ Setup Instructions

### 1. 🧠 Route 53 Hosted Zone Creation

Use `route53-init.sh` to create (or find) a public hosted zone and bind your domain to your IP:

```bash
./route53-init.sh
```

> Edit `DOMAIN` and `IP` as needed in the script.

---

### 2. 🔄 Dynamic DNS with Cron

Use the `route53-ddns-cron.sh` to regularly sync your home’s dynamic IP to Route 53:

```bash
crontab -e
# Add this line to run every minute:
* * * * * /path/to/route53-ddns-cron.sh >> /var/log/ddns.log 2>&1
```

---

### 3. 🐳 Docker Compose Up

Build and start everything:

```bash
docker compose up --build -d
```

---

## 🧩 Nginx Setup

`animal-split-frontend/nginx.conf`:

```
server {
    listen 80;
    server_name animalsplit.plus www.animalsplit.plus animalsplit.com www.animalsplit.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name animalsplit.plus www.animalsplit.plus animalsplit.com www.animalsplit.com;

    ssl_certificate     /etc/letsencrypt/live/animalsplit.plus/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/animalsplit.plus/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    root /usr/share/nginx/html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass         http://backend:3001;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header   Connection "";
    }

    location ~ /\.(?!well-known) {
        deny all;
    }
}
```

---

## 📦 Frontend Dockerfile

`animal-split-frontend/Dockerfile`:

```Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
ENV NODE_ENV=production
RUN yarn install
COPY . .
RUN yarn build                 

FROM nginx:1.25-alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY public/favicon.ico /usr/share/nginx/html/favicon.ico
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## ⚙️ Backend Dockerfile

`animal-split-core-pi/Dockerfile`:

```Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json yarn.lock* ./
RUN yarn install --frozen-lockfile
COPY . .
RUN npx prisma generate
RUN yarn build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist            ./dist
COPY --from=build /app/node_modules    ./node_modules
COPY --from=build /app/prisma          ./prisma          
COPY package*.json ./
COPY .env .env
ENV NODE_ENV=production
ENV PORT=3001
EXPOSE 3001
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
```

---

## 🧵 Docker Compose

`docker-compose.yml`: see full file in the repo.

---

## 💬 License

MIT — use freely, learn freely. Contributions welcome!

---

Want a blog post to go with it? [Here’s the story →](#)  
