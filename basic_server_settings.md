Here are the essential packages for an Ubuntu web and database server:

## Web Server

**Core**
- `nginx` or `apache2` — HTTP server
- `certbot` + `python3-certbot-nginx` — SSL/TLS via Let's Encrypt

**Runtime / App Layer**
- `php-fpm` + `php-common` + `php-mysql` — PHP stack
- `nodejs` + `npm` — Node.js (or install via `nvm`)
- `python3` + `python3-pip` + `python3-venv` — Python stack

---

## Database Server

**Relational**
- `mysql-server` or `mariadb-server` — MySQL-compatible DB
- `postgresql` + `postgresql-contrib` — PostgreSQL

**NoSQL**
- `redis-server` — In-memory cache / session store
- `mongodb` — Document store (via MongoDB's own repo)

---

## Security & Firewall

- `ufw` — Uncomplicated Firewall (`ufw allow 80,443,22/tcp`)
- `fail2ban` — Brute-force protection
- `unattended-upgrades` — Automatic security patches
- `libpam-google-authenticator` — 2FA for SSH (optional)

---

## Monitoring & Logging

- `htop` — Interactive process viewer
- `iotop` — Disk I/O monitor
- `netstat` / `ss` (via `net-tools` / `iproute2`) — Network stats
- `logrotate` — Log file rotation (usually pre-installed)
- `rsyslog` — System logging daemon

---

## Utilities & Dev Tools

- `git` — Version control
- `curl` + `wget` — HTTP clients
- `vim` or `nano` — Text editors
- `tmux` or `screen` — Terminal multiplexer
- `zip` + `unzip` + `tar` — Archiving
- `build-essential` — GCC, make, and compilers
- `software-properties-common` — For adding PPAs

---

## One-liner to install the essentials

```bash
sudo apt update && sudo apt install -y \
  nginx certbot python3-certbot-nginx \
  mysql-server redis-server \
  php-fpm php-mysql php-common \
  git curl wget vim tmux \
  ufw fail2ban unattended-upgrades \
  build-essential software-properties-common \
  htop iotop net-tools
```

---

**Tips:**
- Run `sudo mysql_secure_installation` after installing MySQL
- Enable UFW: `sudo ufw enable` after setting your rules
- Enable auto-updates: `sudo dpkg-reconfigure --priority=low unattended-upgrades`

Here's a comprehensive setup guide for Ubuntu Server as a **web + database + storage server**.

---

## System Update First

```bash
sudo apt update && sudo apt upgrade -y
sudo apt dist-upgrade -y
sudo reboot
```

---

## 1. Core System Utilities

```bash
sudo apt install -y \
  build-essential \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release \
  curl wget \
  git \
  vim nano \
  tmux screen \
  zip unzip tar \
  htop iotop iftop \
  net-tools iproute2 \
  dnsutils \
  lsof \
  tree \
  jq
```

---

## 2. Security & Firewall

```bash
sudo apt install -y \
  ufw \
  fail2ban \
  unattended-upgrades \
  aide \
  rkhunter \
  lynis \
  libpam-pwquality
```

### UFW Firewall Setup

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential ports
sudo ufw allow 22/tcp       # SSH
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS
sudo ufw allow 7474/tcp     # Neo4j Browser
sudo ufw allow 7687/tcp     # Neo4j Bolt

sudo ufw enable
sudo ufw status verbose
```

### Fail2Ban

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Auto Security Updates

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 3. SSH Hardening

Edit `/etc/ssh/sshd_config`:

```ini
Port 2222                        # Change default port
PermitRootLogin no               # Disable root login
PasswordAuthentication no        # Use SSH keys only
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers yourusername          # Whitelist users
```

```bash
sudo systemctl restart sshd
sudo ufw allow 2222/tcp          # Allow new SSH port
sudo ufw delete allow 22/tcp
```

---

## 4. Web Server

### Nginx (Recommended)

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### SSL/TLS — Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

### PHP (if needed)

```bash
sudo apt install -y \
  php-fpm \
  php-mysql \
  php-pgsql \
  php-redis \
  php-curl \
  php-gd \
  php-mbstring \
  php-xml \
  php-zip \
  php-bcmath
```

### Node.js (via NVM — recommended)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts
node -v && npm -v
```

### Python

```bash
sudo apt install -y \
  python3 \
  python3-pip \
  python3-venv \
  python3-dev
```

---

## 5. Database Servers

### MySQL

```bash
sudo apt install -y mysql-server
sudo systemctl enable mysql
sudo mysql_secure_installation
```

### PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

### Redis (Cache / Session Store)

```bash
sudo apt install -y redis-server
sudo systemctl enable redis-server

# Edit /etc/redis/redis.conf
# Set: supervised systemd
# Set: maxmemory 256mb
# Set: maxmemory-policy allkeys-lru
```

### Neo4j (Graph Database)

```bash
curl -fsSL https://debian.neo4j.com/neotechnology.gpg.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/neo4j-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/neo4j-archive-keyring.gpg] \
  https://debian.neo4j.com stable latest" \
  | sudo tee /etc/apt/sources.list.d/neo4j.list

sudo apt update && sudo apt install -y neo4j
sudo systemctl enable neo4j
sudo neo4j-admin dbms set-initial-password YourStrongPassword
```

### MongoDB (Optional)

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-keyring.gpg

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-keyring.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt update && sudo apt install -y mongodb-org
sudo systemctl enable mongod
sudo systemctl start mongod
```

---

## 6. Storage Repository

### Samba (Windows File Sharing)

```bash
sudo apt install -y samba
sudo systemctl enable smbd nmbd

# Add share to /etc/samba/smb.conf
[storage]
  path = /srv/storage
  browseable = yes
  read only = no
  valid users = @samba_users
  create mask = 0660
  directory mask = 0770

sudo mkdir -p /srv/storage
sudo smbpasswd -a yourusername
sudo systemctl restart smbd
```

### NFS (Linux Network File System)

```bash
sudo apt install -y nfs-kernel-server

# Edit /etc/exports
/srv/nfs  192.168.1.0/24(rw,sync,no_subtree_check)

sudo mkdir -p /srv/nfs
sudo exportfs -a
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server
```

### MinIO (S3-Compatible Object Storage)

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

sudo mkdir -p /srv/minio/data
sudo useradd -r minio-user -s /sbin/nologin
sudo chown minio-user:minio-user /srv/minio/data

# Create systemd service
sudo nano /etc/systemd/system/minio.service
```

```ini
[Unit]
Description=MinIO Object Storage
After=network.target

[Service]
User=minio-user
Group=minio-user
Environment="MINIO_ROOT_USER=admin"
Environment="MINIO_ROOT_PASSWORD=YourStrongPassword"
ExecStart=/usr/local/bin/minio server /srv/minio/data --console-address ":9001"
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio
```

### Nextcloud (Optional — Self-hosted Cloud Storage)

```bash
sudo apt install -y nextcloud-desktop   # or deploy via snap
sudo snap install nextcloud
```

---

## 7. Monitoring & Observability

```bash
sudo apt install -y \
  prometheus-node-exporter \
  logrotate \
  rsyslog
```

### Netdata (Real-time monitoring dashboard)

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
# Dashboard at http://yourserver:19999
```

### Cockpit (Web-based server management UI)

```bash
sudo apt install -y cockpit
sudo systemctl enable --now cockpit.socket
sudo ufw allow 9090/tcp
# Dashboard at https://yourserver:9090
```

---

## 8. Backup Tools

```bash
sudo apt install -y \
  rsync \
  borgbackup \
  restic
```

### Rsync Example (Automated Backup)

```bash
# Backup /srv/storage to external/remote
rsync -avz --delete /srv/storage/ user@backup-server:/backups/storage/
```

### Cron Backup Schedule

```bash
crontab -e

# Daily database backup at 2 AM
0 2 * * * mysqldump -u root -pYourPassword --all-databases > /srv/backups/mysql_$(date +\%F).sql

# Daily Neo4j backup
0 3 * * * neo4j-admin database dump --to-path=/srv/backups/neo4j/

# Weekly full server sync
0 4 * * 0 rsync -avz /srv/ user@backup-server:/backups/
```

---

## 9. Port Reference

| Service | Port | Protocol |
|---|---|---|
| SSH | 2222 | TCP |
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| MySQL | 3306 | TCP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |
| Neo4j Browser | 7474 | TCP |
| Neo4j Bolt | 7687 | TCP |
| MongoDB | 27017 | TCP |
| MinIO API | 9000 | TCP |
| MinIO Console | 9001 | TCP |
| Samba | 445 | TCP |
| NFS | 2049 | TCP |
| Netdata | 19999 | TCP |
| Cockpit | 9090 | TCP |

---

## 10. Directory Structure (Recommended)

```
/srv/
├── www/          → Web files (Nginx root)
├── storage/      → Samba / NFS shares
├── nfs/          → NFS exports
├── minio/        → Object storage
├── backups/      → Local backups
│   ├── mysql/
│   ├── neo4j/
│   └── files/
└── logs/         → Centralized app logs
```

```bash
sudo mkdir -p /srv/{www,storage,nfs,minio,backups/{mysql,neo4j,files},logs}
sudo chown -R www-data:www-data /srv/www
```

---

**Pro Tips:**
- Always use **SSH keys**, never passwords in production
- Regularly run `sudo lynis audit system` for security checks
- Use **logrotate** to prevent disk fill-up from logs
- Monitor disk usage with `df -h` and `du -sh /srv/*` regularly
- Keep a **snapshot** of the VM before major changes
