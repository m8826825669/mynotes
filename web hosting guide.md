### Web Hosting guide ###

Here is a comprehensive, step-by-step guide to hosting websites on a VPS (Virtual Private Server). This guide assumes you are starting from zero technical knowledge and will cover choosing a provider, securing the server, setting up a LEMP/LLMP stack, and deploying a website.

---

# The Complete Step-by-Step Guide to Web Hosting on a VPS

## Table of Contents
1.  **Understanding VPS Hosting**
2.  **Choosing a VPS Provider & Plan**
3.  **Connecting to Your VPS (SSH)**
4.  **Initial Server Setup & Security**
5.  **Installing a Web Server (Nginx or Apache)**
6.  **Installing a Database (MySQL/MariaDB)**
7.  **Installing PHP (for WordPress, Laravel, etc.)**
8.  **Creating a Virtual Host / Server Block**
9.  **Uploading Your Website Files**
10. **Setting Up Domain & DNS**
11. **Installing SSL (HTTPS) for Free**
12. **Basic Maintenance & Monitoring**
13. **Troubleshooting Common Issues**

---

## Step 1: Understanding VPS Hosting

Unlike shared hosting (where you share resources with hundreds of users), a VPS is a virtual machine dedicated to you. You get:
- **Root access** (full control)
- **Dedicated resources** (RAM, CPU cores)
- **Higher performance & security**
- **Requires technical knowledge** (command line)

## Step 2: Choosing a VPS Provider & Plan

For beginners, choose **Unmanaged VPS** (you control everything) or **Semi-managed**.

**Recommended providers:**
- **DigitalOcean** ($4-6/mo) – best tutorials, beginner friendly
- **Linode** ($5/mo) – reliable, simple
- **Vultr** ($4-mo) – good global locations
- **AWS Lightsail** ($5/mo) – Amazon's simple VPS

**Minimum specs for a small website:**
- 1 vCPU
- 1-2 GB RAM
- 25 GB SSD
- 1 TB bandwidth
- **Linux OS: Ubuntu 22.04 LTS** (recommended for beginners)

## Step 3: Connecting to Your VPS (SSH)

**On Windows (using PowerShell or Windows Terminal):**
```bash
ssh root@your_server_ip
```
Example: `ssh root@203.0.113.10`

**On macOS/Linux (Terminal):**
```bash
ssh root@your_server_ip
```

**Password:** Enter the root password emailed to you by the VPS provider.

> **Tip:** If you get "Permission denied," you may need to use `ssh ubuntu@your_ip` (some providers disable root login by default).

## Step 4: Initial Server Setup & Security

Once connected, run these commands:

### 4.1 Update the system
```bash
apt update && apt upgrade -y
```

### 4.2 Create a new sudo user (instead of root)
```bash
add username
usermod -aG sudo username
su - username
```

### 4.3 Set up SSH key authentication (optional but recommended)
On your local computer, generate a key:
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
Copy the public key to the server:
```bash
ssh-copy-id username@your_server_ip
```

### 4.4 Disable root login & password auth (security)
Edit SSH config: `sudo nano /etc/ssh/sshd_config`
Change:
```
PermitRootLogin no
PasswordAuthentication no
```
Restart SSH: `sudo systemctl restart sshd`

### 4.5 Set up a basic firewall (UFW)
```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## Step 5: Installing a Web Server (Nginx recommended)

Nginx is faster and lighter than Apache for most modern sites.

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

**Test:** Open your browser and go to `http://your_server_ip`. You should see the Nginx welcome page.

## Step 6: Installing a Database (MariaDB)

MariaDB is a drop-in replacement for MySQL (more efficient).

```bash
sudo apt install mariadb-server mariadb-client -y
sudo mysql_secure_installation
```
Answer the prompts:
- Enter current password: (press Enter)
- Switch to unix_socket: N
- Change root password? Y (set a strong password)
- Remove anonymous users? Y
- Disallow root login remotely? Y
- Remove test database? Y
- Reload privileges? Y

## Step 7: Installing PHP

For WordPress, Laravel, Drupal, etc.

```bash
sudo apt install php8.1-fpm php8.1-mysql php8.1-common php8.1-cli php8.1-curl php8.1-gd php8.1-mbstring php8.1-xml php8.1-zip -y
```

To check it works: `php -v`

## Step 8: Creating a Virtual Host (Server Block)

This allows hosting multiple websites on one VPS.

### 8.1 Create website directory
```bash
sudo mkdir -p /var/www/yourdomain.com/html
sudo chown -R $USER:$USER /var/www/yourdomain.com/html
```

### 8.2 Create a sample index file
```bash
nano /var/www/yourdomain.com/html/index.html
```
Add: `<h1>Hello, world!</h1>`

### 8.3 Create Nginx server block
```bash
sudo nano /etc/nginx/sites-available/yourdomain.com
```
Paste this:
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    root /var/www/yourdomain.com/html;
    index index.html index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }
}
```

### 8.4 Enable the site
```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
sudo nginx -t  # Test config
sudo systemctl reload nginx
```

## Step 9: Uploading Your Website Files

### Option A: Git (recommended)
```bash
cd /var/www/yourdomain.com/html
git clone https://github.com/yourusername/your-repo.git .
```

### Option B: SFTP (using FileZilla or WinSCP)
- Protocol: SFTP
- Host: `your_server_ip`
- User: `username`
- Key: your SSH private key (if set)

Upload files to `/var/www/yourdomain.com/html/`

### Option C: SCP (command line)
```bash
scp -r /local/project/folder/* username@your_server_ip:/var/www/yourdomain.com/html/
```

## Step 10: Setting Up Domain & DNS

1. Buy a domain from **Namecheap**, **Cloudflare**, **GoDaddy**, etc.
2. Find your VPS public IP address (e.g., `203.0.113.10`)
3. In your domain registrar's DNS settings, add these **A records**:
   - **Type:** A, **Name:** @, **Value:** your_vps_ip
   - **Type:** A, **Name:** www, **Value:** your_vps_ip
4. Wait 5-30 minutes for propagation.

**Test:** `ping yourdomain.com` should return your VPS IP.

## Step 11: Installing SSL (HTTPS) for Free — Using Certbot (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow the prompts:
- Enter email (for renewal notices)
- Agree to terms
- Choose whether to redirect HTTP to HTTPS (choose 2 for automatic redirect)

Your site is now accessible via `https://yourdomain.com`.

**Auto-renewal:** Certbot renews automatically. Test with `sudo certbot renew --dry-run`

## Step 12: Basic Maintenance & Monitoring

### 12.1 Updates (run weekly)
```bash
sudo apt update && sudo apt upgrade -y
```

### 12.2 Monitor resources
```bash
htop                   # CPU/memory usage
df -h                  # Disk space
sudo tail -f /var/log/nginx/access.log   # Live traffic log
```

### 12.3 Backups (critical!)
For a simple site backup:
```bash
tar -czvf backup.tar.gz /var/www/yourdomain.com
```
For automatic daily backups, learn `rsync` or use `Duplicacy`/`Borg`.

### 12.4 Set up fail2ban (prevents brute force attacks)
```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

## Step 13: Troubleshooting Common Issues

| Problem | Likely Fix |
|--------|-------------|
| **"Unable to connect"** | Check firewall: `sudo ufw status`. Ensure ports 80/443 open |
| **Nginx 403 Forbidden** | Directory permissions: `sudo chmod 755 /var/www/yourdomain.com` |
| **PHP not working** | Check PHP-FPM is running: `sudo systemctl status php8.1-fpm` |
| **SSL not working** | Run `sudo certbot renew --force-renewal` |
| **Database connection error** | Restart MariaDB: `sudo systemctl restart mariadb` |
| **Site loads but not my files** | Nginx `root` path wrong. Check `/etc/nginx/sites-enabled/` |

## Final Checklist for a Live Website

- [ ] VPS is updated and secured (non-root user, firewall, fail2ban)
- [ ] Web server (Nginx) is running
- [ ] PHP and MySQL/MariaDB are installed and working
- [ ] Virtual host points to correct directory
- [ ] Domain DNS points to VPS IP
- [ ] SSL certificate installed and auto-renewal tested
- [ ] Website files uploaded
- [ ] Backups configured
- [ ] Monitoring (e.g., UptimeRobot) is set up

## Optional Next Steps

- **Email hosting** – Use a third-party service like Zoho or MXroute (do not run your own mail server on a VPS unless you are advanced)
- **Control panel** – Install a free panel like HestiaCP or aaPanel to manage everything via GUI
- **Caching** – Add Redis or Nginx fastcgi caching for speed
- **WAF** – Install ModSecurity or use Cloudflare's free plan

---

**You are now running your own VPS-hosted website.** Maintain it by applying security updates weekly and monitoring access logs. For beginners, consider a control panel like **HestiaCP** if the command line feels overwhelming.


Hosting Python web applications like Django, Flask, and FastAPI on a VPS is a natural next step after mastering basic web hosting. While static sites can be served directly, Python applications require an **application server** (like Gunicorn or uWSGI) to run your code and a **web server** (like Nginx) to handle incoming client requests.

Here is a step-by-step guide to get your Python application live.

---

### Quick Navigation
- [Phase 1: Prepare Your Application for Production](#phase-1-prepare-your-application-for-production)
- [Phase 2: Set Up the Server Environment (VPS)](#phase-2-set-up-the-server-environment-vps)
- [Phase 3: Deploy with a Production Stack (Gunicorn & Nginx)](#phase-3-deploy-with-a-production-stack-gunicorn--nginx)
- [Phase 4: Configure the Web Server (Nginx)](#phase-4-configure-the-web-server-nginx)
- [Phase 5: Set Up Your Domain and SSL (HTTPS)](#phase-5-set-up-your-domain-and-ssl-https)
- [Phase 6: Automate with Systemd (Keep it Alive)](#phase-6-automate-with-systemd-keep-it-alive)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)

---

## Phase 1: Prepare Your Application for Production

Before touching the server, get your application ready.

1.  **Requirements File**: Ensure you have a `requirements.txt` file in your project root.
    ```bash
    pip freeze > requirements.txt
    ```

2.  **WSGI/ASGI Entry Point**:
    - **Django**: This is already handled by `wsgi.py` created in your project folder.
    - **Flask**: Ensure your main file (e.g., `app.py`) creates the `app` object.
    - **FastAPI**: Requires an ASGI server. You will need an `app` object (usually defined in `main.py`).

3.  **Environment Variables**: Never hardcode secrets. Use environment variables (e.g., `os.getenv('SECRET_KEY')`).

4.  **Local Testing**: Run your app locally with `python manage.py runserver` (Django) or `flask run` to ensure it works.

## Phase 2: Set Up the Server Environment (VPS)

SSH into your VPS and follow the initial setup steps from the previous guide (updating system, creating a user, firewall). Then, install the necessary packages.

```bash
# Update the server
sudo apt update && sudo apt upgrade -y

# Install Python, pip, and virtual environment tools
sudo apt install -y python3-pip python3-venv nginx

# For database support (PostgreSQL example)
sudo apt install -y postgresql postgresql-contrib libpq-dev

# For building Python packages
sudo apt install -y build-essential
```

## Phase 3: Deploy with a Production Stack (Gunicorn & Nginx)

Unlike PHP, Python cannot be served directly by Nginx. You need an application server like **Gunicorn** (for WSGI apps like Django/Flask) or **Uvicorn** (for ASGI apps like FastAPI) to run your code. Nginx acts as a reverse proxy, passing web requests to Gunicorn.

### Step 3.1: Upload Your Code
Navigate to your web directory (e.g., `/var/www/`) and clone your repository or upload files via SFTP.

```bash
sudo mkdir -p /var/www/myapp
sudo chown -R $USER:$USER /var/www/myapp
cd /var/www/myapp
```

### Step 3.2: Set Up a Virtual Environment
It is best practice to keep your project's dependencies isolated.

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Install production server
pip install gunicorn  # For Django/Flask
# OR pip install uvicorn  # For FastAPI
```

### Step 3.3: Test Gunicorn/Uvicorn Manually
- **For Django/Flask (WSGI):**
    ```bash
    # Django
    gunicorn --bind 0.0.0.0:8000 myproject.wsgi
    # Flask (assuming app is in app.py and named 'app')
    gunicorn --bind 0.0.0.0:8000 app:app
    ```
- **For FastAPI (ASGI):**
    ```bash
    uvicorn main:app --host 0.0.0.0 --port 8000
    ```
Visit `http://your_server_ip:8000`. If you see your site, stop the process (`CTRL + C`) and proceed.

## Phase 4: Configure the Web Server (Nginx)

Nginx will listen on port 80 (HTTP) and pass requests to Gunicorn (running on port 8000).

### Step 4.1: Create an Nginx Server Block
```bash
sudo nano /etc/nginx/sites-available/myapp
```

### Step 4.2: Paste the Configuration
This config works for **Django, Flask, and FastAPI** with Gunicorn/Uvicorn:

```nginx
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;

    location / {
        # Forward requests to Gunicorn/Uvicorn
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Serve static files directly (Django/Flask)
    location /static/ {
        alias /var/www/myapp/static/;
    }

    # Serve media files directly
    location /media/ {
        alias /var/www/myapp/media/;
    }
}
```

**Static Files Note**: For Django, you must run `python manage.py collectstatic` to gather static files into the directory defined in the Nginx config.

### Step 4.3: Enable the Site and Test
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Phase 5: Set Up Your Domain and SSL (HTTPS)

Use the same Certbot method described in the general guide, but ensure Nginx is configured to proxy to your app.

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
```

## Phase 6: Automate with Systemd (Keep it Alive)

If you close your SSH session, Gunicorn stops. You need a **systemd service** to run it in the background and restart it automatically if it crashes.

### Step 6.1: Create a Systemd Service File
```bash
sudo nano /etc/systemd/system/myapp.service
```

### Step 6.2: Paste the Configuration
**For Django/Flask (Gunicorn):**
```ini
[Unit]
Description=Gunicorn instance to serve myapp
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
Environment="PATH=/var/www/myapp/venv/bin"
Environment="SECRET_KEY=your-secret-key-here"
ExecStart=/var/www/myapp/venv/bin/gunicorn --workers 3 --bind unix:myapp.sock -m 007 myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

**For FastAPI (Uvicorn):**
Change the `ExecStart` line to:
```bash
ExecStart=/var/www/myapp/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
```

### Step 6.3: Start and Enable the Service
```bash
sudo systemctl start myapp
sudo systemctl enable myapp
sudo systemctl status myapp
```

## Troubleshooting Common Issues

| Problem | Likely Cause & Fix |
| :--- | :--- |
| **502 Bad Gateway** | Gunicorn isn't running. Check with `sudo systemctl status myapp`. Also ensure Nginx is pointing to the correct socket/port. |
| **Static Files 404** | Nginx cannot find the static files. Run `collectstatic` (Django) or ensure the `alias` path in Nginx matches your static folder location . |
| **ModuleNotFoundError** | The wrong Python environment is active, or `requirements.txt` wasn't fully installed. Activate the venv and run `pip install -r requirements.txt` again . |
| **Permission Denied (sock)** | The user running Gunicorn (often `www-data`) doesn't have write access to the directory where the `.sock` file is created. Adjust `Group` and `WorkingDirectory` permissions. |
| **FastAPI doesn't load** | Azure App Service (and standard WSGI servers) require a custom startup command for FastAPI because it uses ASGI. Ensure you are using `uvicorn` or `gunicorn -k uvicorn.workers.UvicornWorker` . |

---

### Summary for Framework-Specific Deployments

*   **Django**:
    *   Entry Point: `<project_name>.wsgi:application`
    *   Static Files: Must run `python manage.py collectstatic`.
    *   Allowed Hosts: Add your domain to `ALLOWED_HOSTS` in `settings.py`.
*   **Flask**:
    *   Entry Point: `app:app` (where `app` is the filename and variable name).
    *   Simpler static file handling via Nginx proxy.
*   **FastAPI**:
    *   Entry Point: `main:app`.
    *   Runtime: Requires ASGI server (`Uvicorn` or `Hypercorn`) rather than pure WSGI.

By following this guide, you move from a simple static file server to a dynamic application server capable of handling complex Python web frameworks.

-----------------------

Hosting Node.js applications (including backend APIs, React, Angular, and Next.js) requires a different approach than Python or static PHP. The key differences: Node.js applications are **long-running processes** that need process management, and frontend frameworks often require **build steps** before deployment.

Here is a comprehensive guide for hosting Node.js apps on a VPS.

---

## Quick Navigation
- [Understanding the Stack](#understanding-the-stack)
- [Phase 1: Prepare Your Node.js Application](#phase-1-prepare-your-nodejs-application)
- [Phase 2: Set Up the VPS Environment](#phase-2-set-up-the-vps-environment)
- [Phase 3: Deploy a Backend API (Express, NestJS, Koa)](#phase-3-deploy-a-backend-api-express-nestjs-koa)
- [Phase 4: Deploy a React/Angular SPA (Static Build)](#phase-4-deploy-a-reactangular-spa-static-build)
- [Phase 5: Deploy Next.js with PM2 & Nginx](#phase-5-deploy-nextjs-with-pm2--nginx)
- [Phase 6: Set Up Domain, SSL, and Environment Variables](#phase-6-set-up-domain-ssl-and-environment-variables)
- [Phase 7: Process Management with PM2 (Critical)](#phase-7-process-management-with-pm2-critical)
- [Deployment Automation with GitHub Webhooks](#deployment-automation-with-github-webhooks)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)

---

## Understanding the Stack

Unlike Python (which often uses Gunicorn), Node.js applications:
- **Run as a single process** (or clustered)
- **Handle requests directly** (no separate WSGI server needed)
- **Need a process manager** (PM2) to keep them alive
- **Frontend frameworks** require build steps (`npm run build`)

**Production stack for Node.js:**
```
[User] → [Nginx (Port 80/443)] → [Node.js App (Port 3000)] → [Database]
                                   (Managed by PM2)
```

Nginx acts as a **reverse proxy** (forwarding requests) and also serves static files (images, CSS, JS).

---

## Phase 1: Prepare Your Node.js Application

### Step 1.1: Application Structure

Ensure your app has these files in its root:

**Backend API (Express, Fastify, NestJS):**
```javascript
// server.js or app.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => res.send('API Running'));
app.listen(PORT, () => console.log(`Server on port ${PORT}`));
```

**package.json must have:**
```json
{
  "name": "my-app",
  "scripts": {
    "start": "node server.js",
    "build": "echo 'No build needed for backend'"
  },
  "dependencies": { ... }
}
```

### Step 1.2: Environment Configuration

Use a `.env` file for local development (never commit to Git):
```
PORT=3000
DATABASE_URL=postgresql://user:pass@localhost/db
JWT_SECRET=your-secret-key
```

Install dotenv: `npm install dotenv`

Load in your app: `require('dotenv').config()`

### Step 1.3: .gitignore Setup
```
node_modules/
.env
dist/
build/
*.log
```

---

## Phase 2: Set Up the VPS Environment

SSH into your VPS and run:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 20.x (LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2 globally (process manager)
sudo npm install -g pm2

# Install Nginx
sudo apt install -y nginx

# Verify installations
node --version  # v20.x
npm --version
pm2 --version
```

### Step 2.1: Basic Firewall Setup
```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## Phase 3: Deploy a Backend API (Express, NestJS, Koa)

This applies to pure backend services (REST APIs, GraphQL, WebSockets).

### Step 3.1: Upload Your Code
```bash
sudo mkdir -p /var/www/myapi
sudo chown -R $USER:$USER /var/www/myapi
cd /var/www/myapi

# Clone your repository
git clone https://github.com/yourusername/myapi.git .
# OR upload via SCP/SFTP
```

### Step 3.2: Install Dependencies
```bash
npm install --production
# Or for development: npm install
```

### Step 3.3: Test Your App
```bash
node server.js
# Open another terminal and run: curl http://localhost:3000
```
If it works, stop with `CTRL+C`.

### Step 3.4: Configure Nginx as Reverse Proxy

Create Nginx config:
```bash
sudo nano /etc/nginx/sites-available/myapi
```

Paste this configuration:
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/myapi /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 3.5: Start with PM2 (Critical for keeping it alive)

```bash
pm2 start server.js --name myapi
pm2 save  # Save current process list
pm2 startup  # Generate startup script to auto-start on boot
```

**PM2 useful commands:**
```bash
pm2 list              # View all running apps
pm2 logs myapi        # View logs
pm2 restart myapi     # Restart app
pm2 stop myapi        # Stop app
pm2 delete myapi      # Remove from PM2
pm2 monit             # Monitor resources
```

---

## Phase 4: Deploy a React/Angular SPA (Static Build)

React, Angular, and Vue apps compile to **static files** (HTML, JS, CSS). They don't need Node.js at runtime—Nginx serves them directly.

### Step 4.1: Build Your App Locally
```bash
# On your local machine
cd my-react-app
npm run build

# This creates a 'build' folder (React) or 'dist' folder (Angular/Vue)
```

### Step 4.2: Upload Build Files to VPS
```bash
# From your local machine
scp -r build/* username@your_server_ip:/var/www/myapp/
```

Or on the VPS:
```bash
sudo mkdir -p /var/www/myapp
sudo chown -R $USER:$USER /var/www/myapp
# Upload via SFTP or Git
```

### Step 4.3: Configure Nginx for SPA Routing

SPAs use client-side routing (React Router, Angular Router). Nginx must redirect all routes to `index.html`.

```bash
sudo nano /etc/nginx/sites-available/myapp
```

```nginx
server {
    listen 80;
    server_name myapp.com www.myapp.com;
    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Enable and restart:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

**Note:** No PM2 needed for SPAs since they're static files.

---

## Phase 5: Deploy Next.js with PM2 & Nginx

Next.js is special—it's **hybrid**: static + server-side rendering (SSR). You need Node.js runtime AND build step.

### Step 5.1: Prepare Next.js App for Production

Update your `package.json`:
```json
{
  "scripts": {
    "build": "next build",
    "start": "next start -p 3000",
    "dev": "next dev"
  }
}
```

### Step 5.2: Upload and Build on VPS
```bash
cd /var/www/mynextapp
git clone https://github.com/yourusername/mynextapp.git .
npm install --production
npm run build  # Creates .next folder
```

### Step 5.3: Test Production Build
```bash
npm start
# Visit http://localhost:3000
```

### Step 5.4: Same Nginx Reverse Proxy as Backend API
Use the same Nginx config from Phase 3 (proxy_pass to port 3000).

### Step 5.5: Start with PM2
```bash
pm2 start npm --name "nextjs-app" -- start
pm2 save
pm2 startup
```

---

## Phase 6: Set Up Domain, SSL, and Environment Variables

### Step 6.1: DNS Configuration
Add A records pointing your domain/subdomain to your VPS IP:
- `api.yourdomain.com` → VPS IP (for backend API)
- `app.yourdomain.com` → VPS IP (for frontend)
- `yourdomain.com` → VPS IP (for main site)

### Step 6.2: Install SSL with Certbot
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

For multiple subdomains:
```bash
sudo certbot --nginx -d api.yourdomain.com -d app.yourdomain.com
```

### Step 6.3: Environment Variables on VPS

**Option A: .env file (simple)**
```bash
nano /var/www/myapi/.env
```
```
PORT=3000
DATABASE_URL=postgresql://localhost/mydb
JWT_SECRET=super-secret-key-123
```

**Option B: PM2 ecosystem file (recommended for sensitive keys)**
```bash
pm2 ecosystem  # Generates ecosystem.config.js
```

Edit `ecosystem.config.js`:
```javascript
module.exports = {
  apps: [{
    name: 'myapi',
    script: 'server.js',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
      JWT_SECRET: 'your-secret-key'
    }
  }]
}
```

Start with: `pm2 start ecosystem.config.js`

---

## Phase 7: Process Management with PM2 (Critical)

PM2 is essential for Node.js in production.

### Step 7.1: Zero-Downtime Deployments
```bash
# After code update
git pull
npm install --production
pm2 reload myapi  # Reload without downtime
```

### Step 7.2: Clustering (Use multiple CPU cores)
```bash
pm2 start server.js -i max --name myapi
# -i max uses all available CPU cores
```

### Step 7.3: Log Management
```bash
pm2 logs myapi --lines 100
pm2 flush  # Clear all logs
pm2 install pm2-logrotate  # Auto-rotate logs
```

### Step 7.4: Monitoring
```bash
pm2 monit  # Terminal dashboard
# Or web-based: pm2 plus (paid)
```

---

## Deployment Automation with GitHub Webhooks

Automatically deploy when you push to GitHub.

### Step 1: Create a Deployment Script
```bash
nano /var/www/myapi/deploy.sh
```
```bash
#!/bin/bash
cd /var/www/myapi
git pull origin main
npm install --production
pm2 reload myapi
echo "Deployed successfully at $(date)"
```

Make executable: `chmod +x deploy.sh`

### Step 2: Create a Webhook Endpoint (using Express)
```javascript
// webhook.js
const express = require('express');
const { exec } = require('child_process');
const app = express();

app.use(express.json());

app.post('/webhook', (req, res) => {
  if (req.headers['x-github-event'] === 'push') {
    exec('/var/www/myapi/deploy.sh', (error, stdout) => {
      console.log(stdout);
    });
    res.status(200).send('Deploying...');
  } else {
    res.status(200).send('OK');
  }
});

app.listen(9000);
```

Run with PM2: `pm2 start webhook.js --name webhook`

### Step 3: Configure GitHub
1. Go to your GitHub repo → Settings → Webhooks
2. Add webhook: `http://yourdomain.com:9000/webhook`
3. Content type: `application/json`
4. Save

---

## Troubleshooting Common Issues

| Problem | Likely Cause & Fix |
|---------|-------------------|
| **Cannot GET /route** | SPA not configured for client-side routing. Fix: Add `try_files $uri $uri /index.html` to Nginx config. |
| **502 Bad Gateway** | Node.js app not running. Check `pm2 list` and restart with `pm2 restart myapp`. Also verify port matches Nginx proxy_pass. |
| **Module not found** | `node_modules` missing. Run `npm install` on the server. |
| **Port already in use** | Another process using port 3000. Kill it: `sudo lsof -i :3000` then `kill -9 PID`. |
| **Next.js build fails** | Insufficient RAM on VPS (need at least 1GB for Next.js build). Add swap: `sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
| **WebSockets not working** | Nginx needs upgrade headers. Ensure your Nginx config includes: `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";` |
| **Environment variables undefined** | PM2 doesn't read `.env` by default. Either use `pm2 ecosystem` or install: `npm install dotenv` and require it in your app. |
| **CORS errors (API calls)** | Your API needs CORS enabled. In Express: `npm install cors` then `app.use(cors())`. Or configure Nginx to add CORS headers. |

---

## Summary: Framework-Specific Deployment Commands

| Framework | Deployment Method | PM2 Command | Nginx Config |
|-----------|-----------------|-------------|--------------|
| **Express/Fastify** | Node.js process | `pm2 start server.js` | Reverse proxy |
| **NestJS** | Node.js process | `pm2 start dist/main.js` | Reverse proxy |
| **React SPA** | Static files only | Not needed | Static file server + fallback |
| **Angular SPA** | Static files only | Not needed | Static file server + fallback |
| **Next.js (SSR)** | Node.js process | `pm2 start npm -- start` | Reverse proxy |
| **Nuxt.js (SSR)** | Node.js process | `pm2 start npm -- start` | Reverse proxy |
| **WebSocket server** | Node.js process | `pm2 start server.js` | Reverse proxy with upgrade headers |

---

## Performance Optimization Checklist

- [ ] **Enable compression** in Nginx (`gzip on; gzip_types text/css application/javascript;`)
- [ ] **Set up caching** for static assets (see Phase 4 Nginx config)
- [ ] **Use PM2 clustering** (`pm2 start server.js -i max`)
- [ ] **Configure Nginx buffer sizes** for large uploads:
      ```nginx
      client_max_body_size 10M;
      proxy_buffer_size 128k;
      proxy_buffers 4 256k;
      ```
- [ ] **Add a CDN** (Cloudflare free tier) for global delivery
- [ ] **Monitor with pm2 monit** or install PM2 Plus for advanced metrics

You now have a production-ready Node.js hosting setup! The key differences from PHP/Python hosting are the build process (for frontend frameworks) and PM2 for process management.

-------------------------

Hosting Java applications on a VPS is different from PHP, Python, or Node.js because Java introduces the **Java Virtual Machine (JVM)** as an intermediary layer. This means you aren't just running code directly; you are running a platform that executes your compiled code, which requires a different approach to process management and server configuration.

This guide separates Java hosting into three modern categories: **Standalone JARs (Spring Boot)**, **Traditional Web Apps (WAR files)**, and **Microservices**. We will focus on production-grade methods using systemd and Docker, as JavaFX (a desktop technology) is not designed for server hosting. For the best balance of performance and maintainability in 2025-2026, **Docker** is the recommended approach for Java on a VPS.

---

### Phase 1: The Fundamentals (Your First Java App on a VPS)

Before deploying complex frameworks, it is crucial to understand how to run a raw Java process on a server.

#### Step 1.1: Install the Java Runtime Environment (JRE) or JDK
SSH into your VPS and install Java. OpenJDK is the standard open-source implementation.
```bash
# Update package list
sudo apt update

# Install OpenJDK 17 (LTS version - most common for 2026)
sudo apt install openjdk-17-jdk -y

# Verify installation
java -version
```
> *Note:* OpenJDK 11, 17, and 21 are the most common Long-Term Support (LTS) versions used in production. Check your application's requirements.

#### Step 1.2: Upload and Run a Simple JAR
Unlike scripts that need interpreters running constantly, a Java app is a binary archive (JAR). To run it manually:
```bash
# Navigate to your app folder
cd /var/www/myapp

# Run the application (this blocks the terminal)
java -jar my-application.jar
```
To stop it, press `CTRL+C`.

#### Step 1.3: Running Java in the Background (The "Ampersand" Method)
To detach the process from your terminal session, you can use `&` and `disown`, but this is **not** production-ready because it won't restart if it crashes.
```bash
java -jar my-application.jar &
disown
```

---

### Phase 2: Production Deployment Strategies (The Right Way)

Since the basic `java -jar` command dies when you close the terminal or if the app crashes, we need a robust process manager. For Java, the standard approaches are `systemd` (for simplicity) and **Docker** (for modern isolation).

#### Strategy A: The "systemd" Approach (Simplest for Spring Boot)
This method registers your JAR as a Linux service so it starts on boot and restarts if it fails.

**Step 2.1: Create a Service User (Security)**
Running Java applications as `root` is dangerous. Create a dedicated user:
```bash
sudo useradd -r -s /bin/false javauser
```

**Step 2.2: Create a systemd Service File**
Create a configuration file for your application:
```bash
sudo nano /etc/systemd/system/my-java-app.service
```

Paste the following configuration (adjust paths and memory as needed):
```ini
[Unit]
Description=My Spring Boot Application
After=network.target

[Service]
Type=simple
User=javauser
Group=javauser
# Working directory where your JAR is located
WorkingDirectory=/var/www/myapp
# The actual command to start Java
ExecStart=/usr/bin/java -Xmx512m -Xms256m -jar /var/www/myapp/my-application.jar
# Restart policy
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Step 2.3: Start and Enable the Service**
```bash
# Reload systemd to recognize the new file
sudo systemctl daemon-reload

# Start the app
sudo systemctl start my-java-app

# Enable auto-start on boot
sudo systemctl enable my-java-app

# Check status and logs
sudo systemctl status my-java-app
sudo journalctl -u my-java-app -f
```

#### Strategy B: The Docker Approach (Best for Microservices & Isolation)
Containerization is the industry standard for Java in 2025-2026 because it guarantees the environment (Java version, OS settings) is identical everywhere.

**Step 2.1: Install Docker on your VPS**
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Log out and back in for the group change to take effect
```

**Step 2.2: Create a Dockerfile**
In the root of your Java project (where your `pom.xml` or `build.gradle` is), create a file named `Dockerfile`.

*Example for a Spring Boot application (build with Maven):*
```dockerfile
# Multi-stage build: Reduces final image size significantly
# Stage 1: Build the application
FROM maven:3.8.6-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run the application
FROM openjdk:17-jdk-slim
WORKDIR /app
# Copy the built JAR from the previous stage
COPY --from=build /app/target/*.jar app.jar
# Create a non-root user to run the app (Security best practice)
RUN useradd -m javauser && chown javauser:javauser app.jar
USER javauser
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Step 2.3: Build and Run the Container**
```bash
# Build the image (run this in the directory with the Dockerfile)
docker build -t my-java-app .

# Run the container
docker run -d -p 8080:8080 --name my-app --restart always my-java-app
```
> *Why Docker?* It handles restart policies natively (`--restart always`) and isolates your Java app from other services on the VPS.

---

### Phase 3: Framework-Specific Deep Dive

How you package your application dictates how you deploy it.

#### 3.1 Spring Boot (Modern Web Apps)
Most modern Spring Boot applications package dependencies into a "Fat JAR" with an embedded server (Tomcat/Jetty).
- **Artifact:** `.jar`
- **Deployment:** `java -jar app.jar`
- **Reverse Proxy:** Nginx simply passes traffic to the internal port (e.g., 8080). You do **not** need to install Tomcat separately; Spring Boot has it built-in.

#### 3.2 Traditional WAR Files (Jakarta EE / Legacy Apps)
Some enterprise apps deploy to an external application server like Tomcat or Wildfly.
- **Artifact:** `.war`
- **Deployment:** You install Tomcat, then drop the WAR file into the `webapps` folder. Tomcat expands it automatically.
- **Install Tomcat:**
    ```bash
    sudo apt install tomcat9 -y
    # Deploy your WAR
    sudo cp my-app.war /var/lib/tomcat9/webapps/
    sudo systemctl restart tomcat9
    ```

#### 3.3 Java Microservices (Spring Cloud)
Microservices are usually deployed as **Docker containers** orchestrated by **Docker Compose** or **Kubernetes**. Since you are on a single VPS, Docker Compose is the practical choice.

Create a `docker-compose.yml`:
```yaml
version: '3.8'
services:
  service-a:
    build: ./service-a
    ports:
      - "8081:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod

  service-b:
    build: ./service-b
    ports:
      - "8082:8080"
    depends_on:
      - service-a
```
Run all services: `docker-compose up -d`

#### 3.4 JavaFX on a Server (A Nuance)
*A clarification:* You cannot "host" a JavaFX desktop application on a VPS for users to see the GUI remotely. However, you *can* run a JavaFX app headlessly on the server if you need its backend logic, or use a library like **JPro** to render the JavaFX UI to a web browser. For standard web hosting, stick to Spring Boot.

---

### Phase 4: Setting up the Reverse Proxy (Nginx) & SSL

Unless your Java app runs on port 80 (requires root privileges), you need Nginx to forward requests from port 80/443 to your Java port (e.g., 8080).

**Step 4.1: Install Nginx**
```bash
sudo apt install nginx -y
```

**Step 4.2: Configure the Proxy**
Create an Nginx config file for your domain:
```bash
sudo nano /etc/nginx/sites-available/my-java-app
```

Paste the configuration (adjust `proxy_pass` to your app's port):
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Step 4.3: Enable Site and Install SSL**
```bash
sudo ln -s /etc/nginx/sites-available/my-java-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# Install SSL (Certbot)
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your-domain.com
```

---

### Phase 5: Performance Tuning the JVM

Java applications require explicit memory management. If your app crashes with an `OutOfMemoryError`, you have allocated too little RAM.

**Memory Flags for `systemd` or Command Line:**
| Flag | Meaning | Typical Value (1GB VPS) |
| :--- | :--- | :--- |
| `-Xms` | Initial heap size | `256m` |
| `-Xmx` | Maximum heap size (Crucial) | `512m` (Leave ~512mb for OS) |
| `-XX:+UseG1GC` | Garbage Collector algorithm | Default (Good for most apps) |

**Example optimized start command:**
```bash
java -Xms256m -Xmx512m -XX:+UseG1GC -jar my-app.jar
```

If you are using Docker, you can use the `JAVA_OPTS` environment variable without rebuilding the image:
```bash
docker run -e JAVA_OPTS="-Xmx512m" -p 8080:8080 my-java-app
```

---

### Troubleshooting Common Java Hosting Issues

| Problem | Likely Cause & Solution |
| :--- | :--- |
| **Process dies randomly (Exit code 137)** | The OS "OOM Killer" terminated Java because it used too much RAM. Reduce `-Xmx` value or upgrade your VPS. |
| **Port already in use (Bind Exception)** | Another process is on that port. Find it: `sudo lsof -i :8080` and kill it, or change your app's port. |
| **Spring Boot loads but page is blank / 404** | Spring Boot serves on context root `/`. If you deployed a WAR to Tomcat, it will be at `http://IP:8080/your-war-name/` |
| **Docker container exits immediately** | Check the logs: `docker logs my-app -f`. Usually a missing environment variable or invalid `JAVA_OPTS` syntax. |
| **Connection Refused (Nginx to Java)** | Java app is bound to localhost (127.0.0.1) but should be bound to 0.0.0.0 to accept external proxy connections. In Spring Boot, set `server.address=0.0.0.0`. |
| **Memory usage is 2x the Xmx value** | The JVM uses `Xmx` for heap, plus non-heap memory (Metaspace, threads, native code). A 512mb VPS often needs `-Xmx256m`. |

### Summary Checklist for Production

- [ ] **Java Version:** Verified (`java -version` matches app requirements).
- [ ] **Process Management:** App runs via `systemd` or Docker (with `--restart always`).
- [ ] **Reverse Proxy:** Nginx configured to forward requests to Java's internal port.
- [ ] **SSL:** Certbot installed and HTTPS is enforced.
- [ ] **Memory:** `-Xmx` flag is set to 50-70% of VPS RAM (e.g., 512mb on a 1GB VPS).
- [ ] **Firewall:** Ports 80 and 443 are open; 8080 is closed to the public (Nginx handles it internally).

By following this guide, your Java application will leverage the JVM's robust performance while running securely on your VPS. For most users, **Strategy B (Docker)** combined with **Phase 4 (Nginx)** is the most maintainable solution for 2025-2026.

--------------------------------

Deploying a **multi-tenant SaaS** application on a VPS is a strategic decision that balances architectural complexity, performance, and cost. Unlike standard web apps, a SaaS must handle **data isolation**, **scalability**, and **efficient resource utilization** simultaneously.

Based on the search results and industry best practices, here is a comprehensive guide covering the **isolation models**, the **recommended tech stacks for 2026**, and a **practical VPS configuration guide** to help you balance performance and budget.

---

### Part 1: Understanding SaaS Multi-Tenancy Models

Before choosing a stack or a server, you must decide **how** you will separate customer data. There are three primary models, each with different cost and complexity implications.

#### 1. The Pooled Model (Shared Database)
- **How it works:** All tenants share the same database tables. Tenants are distinguished by a `tenant_id` column in every record.
- **Pros:** Maximum use of server resources; cheapest to run.
- **Cons:** Highest risk of data leakage if code has bugs; "Noisy neighbor" effect (one tenant's high traffic can slow down everyone).
- **Best for:** MVPs, internal tools, or low-risk B2C apps.

#### 2. The Silo Model (Dedicated Resources)
- **How it works:** Every tenant gets their own isolated database (or even a separate infrastructure stack).
- **Pros:** Strong security; no "noisy neighbor" issues.
- **Cons:** Extremely expensive and hard to maintain (updating 100 databases is hard).
- **Best for:** Enterprise clients subject to HIPAA or financial regulations.

#### 3. The Hybrid Model (Most Common for Growth)
- **How it works:** Free/Small tenants share pooled resources. High-tier tenants get siloed (dedicated) resources.
- **Pros:** Best of both worlds. Optimizes cost while offering premium isolation to paying customers.
- **Best for:** Serious SaaS businesses aiming for profitability.

> **Recommendation:** Start with **Pooled (Database per tenant)** using a `tenant_id` filter. If you succeed, migrate to **Hybrid**.

---

### Part 2: The Best Technology Stack for SaaS in 2026

Based on an analysis of over 800 monetized SaaS sites and expert recommendations for 2026, the "best" stack prioritizes **type safety** and **speed of iteration**.

#### The "Full-Stack Type-Safe" Stack (Recommended)
This stack is currently the industry standard for new SaaS products because it catches errors before runtime and allows you to share code between the server and the browser.

**Frontend (UI):**
- **Framework:** **Next.js 15+** (Over 60% of monetized SaaS use this).
- **Language:** TypeScript (Non-negotiable for safety).
- **Styling:** Tailwind CSS + shadcn/ui (Pre-built, accessible components).

**Backend & API:**
- **Runtime:** Node.js (with Next.js API routes) or **Python (FastAPI)** if you are doing heavy AI work.
- **API Layer:** **tRPC** or **GraphQL** (These provide end-to-end type safety, meaning your frontend "knows" exactly what data the backend returns).
- **Auth:** Clerk or Supabase Auth (Don't build your own login screen).

**Database (Crucial for Multi-Tenancy):**
- **Primary DB:** **PostgreSQL** (Using Supabase or Neon). It supports **Row Level Security (RLS)** , which is a native database feature to isolate tenants automatically.
- **Caching:** Redis (to prevent database bottlenecks).

**Infrastructure (VPS Focus):**
- **Containerization:** **Docker** + Docker Compose.
- **Process Manager:** PM2 (for Node) or Systemd.
- **Proxy:** Nginx.

#### Alternative Stacks (Context Matters)
- **The Enterprise Stack:** **Angular + ASP.NET Core / Spring Boot + PostgreSQL + Azure**. Best for large teams building B2B products.
- **The AI-First Stack:** **React + FastAPI (Python) + PostgreSQL + pgvector**. If your SaaS is built around LLMs or embeddings, Python is mandatory.
- **The Solo Founder Stack:** **Next.js + Prisma + Postgres + Stripe**. This is the "boring" stack that just works.

---

### Part 3: How to implement Multi-Tenancy (The Code Strategy)

You cannot just deploy a standard app. You need a strategy to isolate data.

**Strategy A: Database Row-Level Security (Best for Pooled)**
Add a `tenant_id` to every table.
- *Example:* Table `invoices` has columns `id`, `tenant_id`, `amount`.
- *Logic:* Every `SELECT` query *must* include `WHERE tenant_id = current_tenant_id`.
- *Pro Tip:* Use **PostgreSQL RLS** to enforce this at the database level so you can't forget the filter in your code.

**Strategy B: Database per Tenant (Best for Hybrid/Silo)**
- Your main app connects to a "Tenant Directory" DB.
- When a user logs in, you look up their company name and dynamically connect to `database_company_id`.
- *Pro:* Easy to move high-value customers to a dedicated server later.

**Strategy C: Namespace per Tenant (Kubernetes/Docker)**
- Used by advanced users (like Wefox).
- Each tenant gets a separate Kubernetes Namespace. This isolates CPU and memory usage, preventing the "noisy neighbor" problem.

---

### Part 4: VPS Configuration for Performance & Economics

Balancing performance and cost is the hardest part of self-hosting SaaS. You need a configuration that handles traffic spikes (like a viral tweet) without paying for idle servers.

#### Recommended Baseline Configuration (For MVPs up to 1,000 active users)

| Component | Minimum Spec (Startup/Economy) | Recommended (Growth Stage) | Why? |
| :--- | :--- | :--- | :--- |
| **vCPU** | 2 cores | 4 cores | SaaS apps handle many concurrent requests. 2 cores allows Nginx + Node to run simultaneously. |
| **RAM** | 4 GB | 8 GB | **This is critical.** Node.js/Python apps + PostgreSQL + Redis need room. 2GB will crash under load. |
| **Storage** | 80 GB NVMe SSD | 150 GB NVMe SSD | **NVMe is mandatory** for fast database lookups. SATA SSDs are too slow for multi-tenant DBs. |
| **Bandwidth** | 4 TB | 10 TB | SaaS APIs are chatty. You will serve more data than a brochure site. |
| **Network** | 300 Mbps | 500 Mbps - 1 Gbps | Prevents latency spikes during backups or high traffic. |

#### The Architecture Layout on a Single VPS (Docker Compose example)

Instead of installing everything directly on Linux, use Docker to wrap your services. This makes migration easier.

**Recommended Resource Split (for an 8GB VPS):**
1.  **Nginx (Proxy):** ~100MB (Handles SSL and routing)
2.  **Node.js (Next.js Backend):** ~2.5GB (Your app logic)
3.  **PostgreSQL (Database):** ~2GB (Handles queries and RLS)
4.  **Redis (Cache):** ~1GB (Manages sessions and queues)
5.  **Reserved for OS:** ~2.4GB (Leaves breathing room for spikes)

#### Cost Analysis: VPS vs. Managed Cloud
A major lesson from successful founders: start with a VPS, even for 100k users.
- **The $20 VPS Reality:** One team served **102,000 active users** on a $20/month VPS by optimizing database queries.
- **The Cloud Threshold:** Managed services (AWS RDS, Google Cloud Run) become cheaper *only* when you factor in the cost of your own time. For bootstrapped founders, a **$40-$80/month VPS** is the "sweet spot" for SaaS.

---

### Part 5: Step-by-Step Deployment Checklist

Follow this order to deploy your multi-tenant SaaS on a VPS:

**Step 1: Server Provisioning**
- Rent a VPS (DigitalOcean, Hetzner, or Vultr) with the specs above.
- Choose Ubuntu 22.04 or 24.04 LTS.

**Step 2: Install Docker & Environment**
- SSH into server.
- Install Docker Engine and Docker Compose.
- Set up UFW firewall (Allow 22, 80, 443 only).

**Step 3: Database Setup (Multi-Tenant Ready)**
- Run a PostgreSQL container (using Docker volumes for persistence).
- Enable Row Level Security (RLS) on your tables.
- Run a Redis container for caching and rate limiting.

**Step 4: Application Configuration**
- Upload your code (via Git).
- Create a `.env` file with `DATABASE_URL` and `REDIS_URL`.
- *Crucial:* Ensure your app passes `tenant_id` in the `X-Tenant-ID` header or JWT token.

**Step 5: Reverse Proxy & SSL**
- Run an Nginx container (or install bare metal).
- Configure Nginx to point to your app's port (e.g., 3000).
- Use Certbot (or Caddy) to issue free SSL certificates.

**Step 6: Process Management**
- **Do not use `node index.js` directly.** Run the app with `pm2` (if Node) or Docker's `restart: always` policy.
- Set up a Systemd service to restart Docker/Pm2 if the server reboots.

**Step 7: Monitoring (The "Noisy Neighbor" Defense)**
- Set up **Prometheus + Grafana** or use a simple tool like **Netdata**.
- Create alerts for CPU spikes, but specifically track **Database Connection Pool** exhaustion. This is the first sign a tenant is misbehaving.

### Summary Checklist

| Layer | Check |
| :--- | :--- |
| **Architecture** | [ ] Decided on Pooled vs. Hybrid model. |
| | [ ] Database tables include `tenant_id` or Row Level Security. |
| **Stack** | [ ] Chosen Next.js + TypeScript + Postgres (Recommended). |
| | [ ] Integrated Stripe for billing & Clerk for Auth. |
| **Hardware** | [ ] VPS has NVMe storage (not standard SSD). |
| | [ ] Configuration uses at least 4GB RAM and 2 vCPUs. |
| **Security** | [ ] All tenants routed through a single Nginx proxy. |
| | [ ] Database backups are automated and encrypted. |

This setup gives you enterprise-level isolation patterns with indie-hacker economics. As you scale and hit the limits of a single VPS (roughly 150k active users or heavy database write loads), you can then migrate to a Kubernetes cluster (like EKS) without rewriting your application logic.

