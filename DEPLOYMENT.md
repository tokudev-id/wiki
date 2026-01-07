# Deployment Guide - wiki.tokudev.cloud

Panduan untuk deploy Wiki.js ke VPS dengan domain `wiki.tokudev.cloud` menggunakan Nginx sebagai reverse proxy.

## Prasyarat

- VPS dengan Ubuntu 20.04/22.04 atau Debian 11+
- Akses SSH ke VPS
- Docker dan Docker Compose terinstall
- Domain `wiki.tokudev.cloud` sudah diarahkan ke IP VPS

## Langkah 1: Siapkan DNS

Pastikan DNS record sudah dikonfigurasi:

```
Type: A
Name: wiki
Value: <IP_VPS_ANDA>
TTL: 3600
```

Cek dengan:
```bash
nslookup wiki.tokudev.cloud
# atau
dig wiki.tokudev.cloud
```

## Langkah 2: Upload Proyek ke VPS

### Opsi A: Menggunakan Git

```bash
# SSH ke VPS
ssh user@<IP_VPS>

# Buat direktori
mkdir -p /opt/wiki
cd /opt/wiki

# Clone repository (ganti dengan URL repo Anda)
git clone <URL_REPOSITORY> .
```

### Opsi B: Menggunakan SCP

```bash
# Dari komputer lokal
scp -r ./* user@<IP_VPS>:/opt/wiki/
```

## Langkah 3: Konfigurasi Environment

```bash
cd /opt/wiki

# Salin dan edit file environment
cp .env.example .env
nano .env
```

Edit isi `.env`:
```env
# Konfigurasi Wiki.js
DB_PASSWORD=<GANTI_DENGAN_PASSWORD_KUAT>
WIKI_PORT=3000
```

**Penting**: Ganti password database dengan password yang kuat!

## Langkah 4: Update Docker Compose untuk Production

Edit `docker-compose.yml` untuk production (binding ke localhost saja):

```bash
nano docker-compose.yml
```

Ubah bagian ports pada service `wiki`:
```yaml
    ports:
      - "127.0.0.1:3000:3000"  # Hanya bind ke localhost
```

## Langkah 5: Jalankan Container

```bash
# Pull images dan jalankan
docker compose up -d

# Cek status
docker compose ps

# Lihat logs
docker compose logs -f
```

## Langkah 6: Install dan Konfigurasi Nginx

### Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Buat Konfigurasi Virtual Host

```bash
sudo nano /etc/nginx/sites-available/wiki.tokudev.cloud
```

Isi dengan konfigurasi berikut:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name wiki.tokudev.cloud;

    # Redirect ke HTTPS (setelah SSL dikonfigurasi)
    # return 301 https://$server_name$request_uri;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }

    # Client upload size (untuk upload gambar/file)
    client_max_body_size 50M;
}
```

### Aktifkan Konfigurasi

```bash
# Buat symbolic link
sudo ln -s /etc/nginx/sites-available/wiki.tokudev.cloud /etc/nginx/sites-enabled/

# Hapus default config (opsional)
sudo rm /etc/nginx/sites-enabled/default

# Test konfigurasi nginx
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

## Langkah 7: Setup SSL dengan Let's Encrypt

### Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Generate SSL Certificate

```bash
sudo certbot --nginx -d wiki.tokudev.cloud
```

Ikuti instruksi di layar dan pilih opsi untuk redirect HTTP ke HTTPS.

### Auto-Renew (otomatis terinstall)

Certbot akan otomatis mengatur cron job untuk renew certificate. Cek dengan:

```bash
sudo certbot renew --dry-run
```

## Langkah 8: Konfigurasi Firewall (Opsional)

```bash
# Jika menggunakan UFW
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

## Langkah 9: Akses dan Setup Wiki.js

1. Buka browser: `https://wiki.tokudev.cloud`
2. Ikuti wizard setup Wiki.js:
   - Buat akun administrator
   - Konfigurasi pengaturan dasar
3. Import dokumentasi dari folder `docs/`

---

## Konfigurasi Nginx Lengkap dengan SSL

Setelah Certbot berhasil, konfigurasi lengkap akan terlihat seperti ini:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name wiki.tokudev.cloud;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name wiki.tokudev.cloud;

    # SSL Certificate (dikelola oleh Certbot)
    ssl_certificate /etc/letsencrypt/live/wiki.tokudev.cloud/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wiki.tokudev.cloud/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }

    # Client upload size
    client_max_body_size 50M;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;
}
```

---

## Troubleshooting

### Container tidak berjalan

```bash
# Cek logs
docker compose logs wiki
docker compose logs db

# Restart container
docker compose restart
```

### 502 Bad Gateway

```bash
# Cek apakah Wiki.js berjalan
curl http://127.0.0.1:3000

# Cek status container
docker compose ps
```

### SSL Certificate gagal

```bash
# Pastikan DNS sudah propagate
dig wiki.tokudev.cloud

# Pastikan port 80 dan 443 terbuka
sudo ufw status

# Retry certbot
sudo certbot --nginx -d wiki.tokudev.cloud --force-renewal
```

### Permission denied pada volume

```bash
# Fix permission SSH keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*
```

---

## Maintenance

### Backup Database

```bash
# Backup
docker exec wiki-db pg_dump -U wikijs wiki > /opt/wiki/backup-$(date +%Y%m%d).sql

# Restore
docker exec -i wiki-db psql -U wikijs wiki < backup.sql
```

### Update Wiki.js

```bash
cd /opt/wiki
docker compose pull wiki
docker compose up --force-recreate -d
```

### Renew SSL Certificate

```bash
sudo certbot renew
```

---

## Quick Deploy Script

Simpan script berikut sebagai `deploy.sh` untuk deployment cepat:

```bash
#!/bin/bash

# Quick Deploy Script for Wiki.js
set -e

echo "=== Wiki.js Deployment Script ==="

# Update system
echo "[1/6] Updating system..."
sudo apt update && sudo apt upgrade -y

# Install Docker (jika belum ada)
echo "[2/6] Installing Docker..."
if ! command -v docker &> /dev/null; then
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    rm get-docker.sh
fi

# Install Docker Compose plugin
echo "[3/6] Installing Docker Compose..."
sudo apt install docker-compose-plugin -y

# Install Nginx
echo "[4/6] Installing Nginx..."
sudo apt install nginx -y

# Setup environment
echo "[5/6] Setting up environment..."
if [ ! -f .env ]; then
    cp .env.example .env
    echo "Please edit .env file and rerun this script"
    exit 1
fi

# Start containers
echo "[6/6] Starting containers..."
docker compose up -d

echo "=== Deployment Complete ==="
echo "Next steps:"
echo "1. Configure Nginx: sudo nano /etc/nginx/sites-available/wiki.tokudev.cloud"
echo "2. Enable site: sudo ln -s /etc/nginx/sites-available/wiki.tokudev.cloud /etc/nginx/sites-enabled/"
echo "3. Install SSL: sudo certbot --nginx -d wiki.tokudev.cloud"
```

---

## Referensi

- [Wiki.js Documentation](https://docs.requarks.io/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Docker Documentation](https://docs.docker.com/)
