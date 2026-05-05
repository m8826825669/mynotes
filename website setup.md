Good timing — you have a $6/month DO droplet (1 vCPU, 1GB RAM, 25GB disk) at IP `157.245.248.210`. Let me give you a realistic, step-by-step plan.

---

## ⚠️ Reality check on 1GB RAM first

| Demo Type | RAM needed | Feasible on your droplet? |
|---|---|---|
| Django/Next.js website | ~400MB | ✅ Yes |
| Web product demo (Medical Store, Accounting) | +150MB | ✅ Yes (tight) |
| JavaFX via VNC/noVNC (School ERP) | ~600MB per session | ❌ No — kills the server |
| Docker demo containers | ~300MB each | ❌ No |

**Verdict:** Run the website + web demos on this droplet. For desktop apps (School ERP), use **trial downloads** instead of live VNC. Upgrade to 2GB ($12/mo) when you get your first 5 customers.

---

## Phase 1 — Server Setup (Do this now, 30 mins)

SSH into your droplet:

```bash
ssh root@157.245.248.210
```

### 1.1 — Add swap (critical for 1GB RAM)

```bash
# Without swap, your server WILL crash when traffic spikes
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Tune swap aggressiveness (use swap only when memory is 90% full)
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p

# Verify
free -h
# Should now show 2GB swap
```

### 1.2 — Install dependencies

```bash
apt update && apt upgrade -y

# Python + Node
apt install -y python3 python3-pip python3-venv nodejs npm nginx certbot python3-certbot-nginx git curl

# Install Node 20 (apt gives old version)
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Verify
python3 --version   # should be 3.10+
node --version      # should be 20.x
nginx -v
```

---

## Phase 2 — Deploy the Website

### 2.1 — Upload your code

On your **Windows machine**, zip and upload:

```bat
REM On your Windows machine
cd D:\softwaresales
tar -czf softwaresales.tar.gz --exclude=node_modules --exclude=venv --exclude=.next --exclude=__pycache__ .

REM Upload to server
scp softwaresales.tar.gz root@157.245.248.210:/opt/
```

On the **server**:

```bash
mkdir -p /opt/softcraft
tar -xzf /opt/softwaresales.tar.gz -C /opt/softcraft
ls /opt/softcraft
# Should show: backend/ frontend/ nginx/ README.md etc.
```

### 2.2 — Backend setup

```bash
cd /opt/softcraft/backend

# Create venv and install
python3 -m venv venv
source venv/bin/activate
pip install -r requirements-dev.txt   # use dev requirements (no psycopg2/celery)
pip install gunicorn                   # need this for production

# Create production .env
cat > .env << 'EOF'
SECRET_KEY=CHANGE_THIS_TO_A_50_CHAR_RANDOM_STRING
DEBUG=False
ALLOWED_HOSTS=157.245.248.210,yourdomain.com,www.yourdomain.com
DATABASE_URL=sqlite:////opt/softcraft/backend/db.sqlite3
CORS_ALLOWED_ORIGINS=https://yourdomain.com,http://157.245.248.210
RAZORPAY_KEY_ID=rzp_test_XXXXXXXXXX
RAZORPAY_KEY_SECRET=your_secret_here
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=your@gmail.com
EMAIL_HOST_PASSWORD=your_app_password
DEFAULT_FROM_EMAIL=SoftCraft Solutions <noreply@yourdomain.com>
SITE_URL=https://yourdomain.com
SUPPORT_EMAIL=support@yourdomain.com
EOF

# Setup database
python manage.py migrate --run-syncdb
python manage.py seed_products
python manage.py collectstatic --noinput

# Create your admin account
python manage.py createsuperuser
```

Generate a real SECRET_KEY:
```bash
python -c "import secrets; print(secrets.token_urlsafe(50))"
# Copy this output into .env SECRET_KEY
```

### 2.3 — Frontend build

```bash
cd /opt/softcraft/frontend

# Create .env.local
cat > .env.local << 'EOF'
NEXT_PUBLIC_API_URL=https://yourdomain.com
NEXT_PUBLIC_RAZORPAY_KEY_ID=rzp_test_XXXXXXXXXX
NEXT_PUBLIC_SITE_URL=https://yourdomain.com
EOF

npm install --legacy-peer-deps
npm run build
# This takes 3-5 minutes — normal
```

### 2.4 — Systemd services (keeps them running after reboot)

**Backend service:**
```bash
cat > /etc/systemd/system/softcraft-backend.service << 'EOF'
[Unit]
Description=SoftCraft Django Backend
After=network.target

[Service]
User=www-data
WorkingDirectory=/opt/softcraft/backend
Environment="PATH=/opt/softcraft/backend/venv/bin"
ExecStart=/opt/softcraft/backend/venv/bin/gunicorn \
    config.wsgi:application \
    --bind 127.0.0.1:8000 \
    --workers 2 \
    --timeout 120 \
    --log-file /var/log/softcraft-backend.log
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**Frontend service:**
```bash
cat > /etc/systemd/system/softcraft-frontend.service << 'EOF'
[Unit]
Description=SoftCraft Next.js Frontend
After=network.target

[Service]
User=www-data
WorkingDirectory=/opt/softcraft/frontend
Environment=NODE_ENV=production
Environment=PORT=3000
ExecStart=/usr/bin/node .next/standalone/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**Start both:**
```bash
# Fix permissions
chown -R www-data:www-data /opt/softcraft

systemctl daemon-reload
systemctl enable softcraft-backend softcraft-frontend
systemctl start softcraft-backend softcraft-frontend

# Check they're running
systemctl status softcraft-backend
systemctl status softcraft-frontend
```

### 2.5 — Nginx config

```bash
cat > /etc/nginx/sites-available/softcraft << 'EOF'
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com 157.245.248.210;

    client_max_body_size 50M;

    # Django static files
    location /static/ {
        alias /opt/softcraft/backend/staticfiles/;
        expires 1y;
    }

    # Media files
    location /media/ {
        alias /opt/softcraft/backend/media/;
        expires 1h;
    }

    # Django API
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }

    # Django admin
    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Next.js frontend (everything else)
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
}
EOF

ln -s /etc/nginx/sites-available/softcraft /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

Visit `http://157.245.248.210` — your site should be live.

---

## Phase 3 — Domain + SSL (Do this after getting a domain)

```bash
# Point your domain's A record to 157.245.248.210 first
# Then run:
certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Certbot auto-renews, but test it:
certbot renew --dry-run
```

---

## Phase 4 — Demo Setup (realistic for your droplet)

### Web products — hosted demo instance

```bash
# Deploy Medical Store as a demo on port 8001
# with a separate SQLite database that resets nightly

mkdir -p /opt/softcraft-demo/medical-store
cd /opt/softcraft-demo/medical-store

# Copy backend code and create demo .env
cat > .env << 'EOF'
SECRET_KEY=demo-secret-key-not-for-production
DEBUG=False
ALLOWED_HOSTS=157.245.248.210,yourdomain.com
DATABASE_URL=sqlite:////opt/softcraft-demo/medical-store/demo.sqlite3
DEMO_MODE=True
EOF

# Demo service on port 8001
cat > /etc/systemd/system/softcraft-demo-medical.service << 'EOF'
[Unit]
Description=SoftCraft Medical Store Demo
After=network.target

[Service]
User=www-data
WorkingDirectory=/opt/softcraft/backend
Environment="PATH=/opt/softcraft/backend/venv/bin"
Environment="DJANGO_SETTINGS_MODULE=config.settings"
Environment="DATABASE_URL=sqlite:////opt/softcraft-demo/medical-store/demo.sqlite3"
ExecStart=/opt/softcraft/backend/venv/bin/gunicorn \
    config.wsgi:application --bind 127.0.0.1:8001 --workers 1
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable softcraft-demo-medical
systemctl start softcraft-demo-medical
```

**Nightly reset cron:**
```bash
# Resets demo data every night at 2 AM
crontab -e
# Add:
0 2 * * * cd /opt/softcraft/backend && source venv/bin/activate && DATABASE_URL=sqlite:////opt/softcraft-demo/medical-store/demo.sqlite3 python manage.py flush --no-input && python manage.py seed_demo_data >> /var/log/demo-reset.log 2>&1
```

**Nginx route for demo:**
```nginx
# Add to your nginx config:
location /demo/medical-store/ {
    proxy_pass http://127.0.0.1:8001/;
    proxy_set_header Host $host;
    add_header X-Demo-Mode "true";
}
```

### Desktop products — trial download strategy

For School ERP (JavaFX), VNC is too heavy. Instead build a **time-limited JAR**:

```java
// Add to SchoolERP main class — checks trial expiry
public class App {
    private static final String TRIAL_FILE = System.getProperty("user.home") + "/.schoolerp_trial";
    private static final int TRIAL_DAYS = 15;

    public static void main(String[] args) {
        if (!isLicensed() && isTrialExpired()) {
            showTrialExpiredDialog();
            return;
        }
        // normal startup...
    }

    private static boolean isTrialExpired() {
        File f = new File(TRIAL_FILE);
        if (!f.exists()) {
            // First run — record install date
            try { 
                new FileWriter(f).append(String.valueOf(System.currentTimeMillis())).close(); 
            } catch (Exception ignored) {}
            return false;
        }
        try {
            long installed = Long.parseLong(new String(Files.readAllBytes(f.toPath())).trim());
            long daysUsed = (System.currentTimeMillis() - installed) / (1000 * 60 * 60 * 24);
            return daysUsed > TRIAL_DAYS;
        } catch (Exception e) { return true; }
    }
}
```

Build as `school-erp-trial.jar`, host at `/opt/softcraft/backend/media/trials/` and serve via the existing download endpoint.

---

## Phase 5 — Start Earning (The actual business part)

### 5.1 — Razorpay live keys

1. Go to [razorpay.com](https://razorpay.com) → complete KYC (takes 2-3 days)
2. Get live keys → update `/opt/softcraft/backend/.env`
3. Restart: `systemctl restart softcraft-backend`

### 5.2 — SEO quick wins (free traffic)

```bash
# Submit your sitemap to Google
# Visit: https://search.google.com/search-console
# Add property → verify → submit sitemap: https://yourdomain.com/sitemap.xml
```

Target these keywords (low competition, high intent):
- "school management software India free trial"
- "medical store billing software GST"  
- "tally alternative small business India"
- "pharmacy management software India"

### 5.3 — WhatsApp Business (your biggest sales channel)

Set up WhatsApp Business at +91 98765 43210:
- Auto-reply: "Thanks for reaching out! Visit softcraft.in to see our software or reply DEMO for a free trial."
- Catalog: Add all 3 products with price and buy link
- Broadcast list: Every demo requester gets a follow-up after 2 days

### 5.4 — JustDial / IndiaMART listing (free leads)

List on:
- **JustDial**: school software, medical software (free listing, gets real leads)
- **IndiaMART**: B2B software buyer intent is high
- **Sulekha**: Education/healthcare software

These 3 platforms alone can give you 5-10 demo requests/week for free.

### 5.5 — Monitor your server

```bash
# Install basic monitoring
apt install -y htop

# Watch memory live
watch -n 2 'free -h && echo "---" && ps aux --sort=-%mem | head -10'

# If memory goes above 85%, restart services:
systemctl restart softcraft-frontend

# Logs
journalctl -u softcraft-backend -f    # live backend logs
journalctl -u softcraft-frontend -f   # live frontend logs
tail -f /var/log/nginx/access.log     # who's visiting
```

---

## Your upgrade path as revenue grows

| Monthly Revenue | Action |
|---|---|
| ₹0 — ₹5,000 | Stay on $6 droplet, use trial downloads for desktop |
| ₹5,000 — ₹15,000 | Upgrade to $12/mo (2GB RAM) — add VNC demo for School ERP |
| ₹15,000+ | Add $6 demo droplet separately — never mix demo and production |
| ₹50,000+ | Move to managed DB (DO Managed PostgreSQL $15/mo), add CDN |

**First goal: Get 3 paying customers.** At ₹4,999 each that's ₹15,000 — covers your server for 40 months. Focus on demo requests and WhatsApp follow-ups above everything else.

Perfect. Let's configure everything for `vexenlabs.com` with files at `/var/www/vexenlabs`. Run these commands one by one on your droplet.

---

## Step 1 — Check what's there

```bash
ls /var/www/vexenlabs
# Should show: backend/ frontend/ or similar
```

```bash
# Check your folder structure
find /var/www/vexenlabs -maxdepth 2 -type d
```

Paste the output here if anything looks wrong. Assuming standard structure:

```
/var/www/vexenlabs/
├── backend/        ← Django
└── frontend/       ← Next.js
```

---

## Step 2 — Backend setup

```bash
cd /var/www/vexenlabs/backend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install --upgrade pip
pip install -r requirements-dev.txt
pip install gunicorn
```

```bash
# Create your .env file
cat > /var/www/vexenlabs/backend/.env << 'EOF'
SECRET_KEY=REPLACE_WITH_GENERATED_KEY
DEBUG=False
ALLOWED_HOSTS=vexenlabs.com,www.vexenlabs.com,157.245.248.210
DATABASE_URL=sqlite:////var/www/vexenlabs/backend/db.sqlite3
CORS_ALLOWED_ORIGINS=https://vexenlabs.com,https://www.vexenlabs.com
RAZORPAY_KEY_ID=rzp_test_XXXXXXXXXX
RAZORPAY_KEY_SECRET=your_secret_here
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
DEFAULT_FROM_EMAIL=SoftCraft Solutions <noreply@vexenlabs.com>
SITE_URL=https://vexenlabs.com
SUPPORT_EMAIL=support@vexenlabs.com
EOF
```

```bash
# Generate a real SECRET_KEY and paste it in
python3 -c "import secrets; print(secrets.token_urlsafe(50))"
```

Copy the output, then:

```bash
nano /var/www/vexenlabs/backend/.env
# Replace REPLACE_WITH_GENERATED_KEY with the output above
# Ctrl+X → Y → Enter to save
```

```bash
# Run migrations and seed data
python manage.py migrate --run-syncdb
python manage.py seed_products
python manage.py collectstatic --noinput

# Create admin account
python manage.py createsuperuser
# Enter: username, email (demo@vexenlabs.com), password
```

---

## Step 3 — Frontend build

```bash
cd /var/www/vexenlabs/frontend

# Create environment file
cat > .env.local << 'EOF'
NEXT_PUBLIC_API_URL=https://vexenlabs.com
NEXT_PUBLIC_RAZORPAY_KEY_ID=rzp_test_XXXXXXXXXX
NEXT_PUBLIC_SITE_URL=https://vexenlabs.com
NEXT_PUBLIC_SUPPORT_EMAIL=support@vexenlabs.com
EOF
```

```bash
# Install and build
npm install --legacy-peer-deps
npm run build
```

This takes 3-5 minutes. You should see `✓ Compiled successfully` at the end.

---

## Step 4 — Fix permissions

```bash
# Give www-data ownership of everything
chown -R www-data:www-data /var/www/vexenlabs
chmod -R 755 /var/www/vexenlabs
```

---

## Step 5 — Systemd services

### Backend service

```bash
cat > /etc/systemd/system/vexenlabs-backend.service << 'EOF'
[Unit]
Description=Vexenlabs Django Backend
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/vexenlabs/backend
EnvironmentFile=/var/www/vexenlabs/backend/.env
ExecStart=/var/www/vexenlabs/backend/venv/bin/gunicorn \
    config.wsgi:application \
    --bind 127.0.0.1:8000 \
    --workers 2 \
    --timeout 120 \
    --access-logfile /var/log/vexenlabs-backend.log \
    --error-logfile /var/log/vexenlabs-backend-error.log
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Frontend service

```bash
cat > /etc/systemd/system/vexenlabs-frontend.service << 'EOF'
[Unit]
Description=Vexenlabs Next.js Frontend
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/vexenlabs/frontend
Environment=NODE_ENV=production
Environment=PORT=3000
Environment=HOSTNAME=127.0.0.1
ExecStart=/usr/bin/node /var/www/vexenlabs/frontend/.next/standalone/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Enable and start both

```bash
systemctl daemon-reload
systemctl enable vexenlabs-backend vexenlabs-frontend
systemctl start vexenlabs-backend vexenlabs-frontend

# Check status
systemctl status vexenlabs-backend
systemctl status vexenlabs-frontend
```

Both should show `Active: active (running)`. If either fails:

```bash
# See the error
journalctl -u vexenlabs-backend -n 50 --no-pager
journalctl -u vexenlabs-frontend -n 50 --no-pager
```

---

## Step 6 — Nginx config

```bash
# Remove default site
rm -f /etc/nginx/sites-enabled/default

# Create vexenlabs config
cat > /etc/nginx/sites-available/vexenlabs << 'EOF'
server {
    listen 80;
    server_name vexenlabs.com www.vexenlabs.com 157.245.248.210;

    client_max_body_size 50M;

    # ── Django static files ────────────────────────────────────────
    location /static/ {
        alias /var/www/vexenlabs/backend/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        gzip_static on;
    }

    # ── Media files (installer uploads etc) ───────────────────────
    location /media/ {
        alias /var/www/vexenlabs/backend/media/;
        expires 1h;
    }

    # ── Django API ─────────────────────────────────────────────────
    location /api/ {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
        proxy_connect_timeout 10s;
    }

    # ── Django admin ───────────────────────────────────────────────
    location /admin/ {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }

    # ── Next.js frontend ───────────────────────────────────────────
    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade           $http_upgrade;
        proxy_set_header   Connection        'upgrade';
        proxy_cache_bypass $http_upgrade;
    }
}
EOF
```

```bash
# Enable the site
ln -s /etc/nginx/sites-available/vexenlabs /etc/nginx/sites-enabled/

# Test config — must say "syntax is ok"
nginx -t

# Reload nginx
systemctl reload nginx
```

Now visit `http://157.245.248.210` — your site should be live.

---

## Step 7 — Point domain to droplet

Go to wherever your domain DNS is managed (GoDaddy, Namecheap, Cloudflare, etc) and add:

| Type | Name | Value | TTL |
|---|---|---|---|
| A | `@` | `157.245.248.210` | 300 |
| A | `www` | `157.245.248.210` | 300 |

Wait 5-10 minutes, then test:

```bash
# On your server
ping vexenlabs.com
# Should resolve to 157.245.248.210
```

---

## Step 8 — SSL certificate (HTTPS)

```bash
# Install certbot if not already there
apt install -y certbot python3-certbot-nginx

# Get free SSL certificate
certbot --nginx -d vexenlabs.com -d www.vexenlabs.com

# It will ask:
# Email address → your email
# Agree to terms → Y
# Share email with EFF → N (your choice)
# Redirect HTTP to HTTPS → 2 (yes, always redirect)
```

After this, visiting `http://vexenlabs.com` auto-redirects to `https://vexenlabs.com`. Done.

```bash
# Test auto-renewal works
certbot renew --dry-run
# Should say "Congratulations, all simulated renewals succeeded"
```

---

## Step 9 — Verify everything works

```bash
# Quick health check script — run this anytime
echo "=== Services ===" && \
systemctl is-active vexenlabs-backend vexenlabs-frontend nginx && \
echo "=== Memory ===" && \
free -h && \
echo "=== Disk ===" && \
df -h / && \
echo "=== API Test ===" && \
curl -s http://127.0.0.1:8000/api/products/stats/ | python3 -m json.tool
```

You should see products stats JSON. Then test the full flow:

1. Visit `https://vexenlabs.com` → landing page loads
2. Visit `https://vexenlabs.com/products` → products list loads
3. Visit `https://vexenlabs.com/admin/` → Django admin login works
4. Register a new account → test purchase flow with Razorpay test keys

---

## Step 10 — Add swap memory (do this now)

```bash
# Critical for 1GB RAM droplet
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p

free -h
# Should show: Swap: 2.0G
```

---

## Useful commands going forward

```bash
# Restart after code changes
systemctl restart vexenlabs-backend
systemctl restart vexenlabs-frontend

# View live logs
journalctl -u vexenlabs-backend -f
journalctl -u vexenlabs-frontend -f

# Watch memory usage
watch -n 3 free -h

# Pull latest code and redeploy (once you set up git)
cd /var/www/vexenlabs && git pull
cd backend && source venv/bin/activate && python manage.py migrate
cd ../frontend && npm run build
systemctl restart vexenlabs-backend vexenlabs-frontend
```

---

Tell me which step you're on and paste any error messages — I'll fix them immediately.

I can't browse to `vexenlabs.com` directly — my web tools can only fetch URLs that appear in search results, and your site is too new to be indexed yet. 

**Share a screenshot** (screenshot the full page and upload it here) and I'll give you a detailed review. Or paste the output of this on your server:

```bash
# Check what's actually running
curl -I http://localhost:3000
curl -I http://localhost:8000/api/products/stats/
systemctl status vexenlabs-backend vexenlabs-frontend nginx
```

While you do that — here's the **checklist of things most likely missing** on a fresh deployment:

---

### 🔴 Critical (check these first)

```bash
# 1. Is the API returning real data?
curl https://vexenlabs.com/api/products/ 2>/dev/null | python3 -m json.tool | head -30

# 2. Are products seeded?
cd /var/www/vexenlabs/backend
source venv/bin/activate
python manage.py shell -c "from apps.products.models import Product; print(Product.objects.count(), 'products')"

# 3. Are static files loading?
curl -I https://vexenlabs.com/static/admin/css/base.css
# Should return 200, not 404
```

---

### 🟡 Things to fix right after deployment

**1. Replace placeholder content in Footer**

Open `frontend/src/components/Footer.tsx` and update:
```tsx
// Change these placeholders:
GST: 09AAAAA0000A1Z5        → your real GST number (or remove)
+91 98765 43210             → your real WhatsApp number  
support@softcraft.in        → support@vexenlabs.com
Noida, Uttar Pradesh        → your real city
SoftCraft Solutions         → Vexen Labs (or your brand name)
```

**2. Replace "SoftCraft" branding throughout**

```bash
# Find all instances of SoftCraft on your server
grep -r "SoftCraft" /var/www/vexenlabs/frontend/src --include="*.tsx" -l
```

**3. Update `.env` with real Razorpay keys**

```bash
nano /var/www/vexenlabs/backend/.env
# Change EMAIL_BACKEND to SMTP (right now emails go nowhere)
# Change RAZORPAY keys to your actual keys
```

**4. Test the full user journey**

Do this manually right now:
- [ ] Register a new account at `https://vexenlabs.com/auth/register`
- [ ] Go to `/products` — do products show?
- [ ] Click a product → does the detail page load?
- [ ] Try "Request Demo" → does the modal open?
- [ ] Go to `/validate` → test the license validator

---

Paste the `curl` outputs or a screenshot and I'll tell you exactly what's broken and how to fix it.

Good — the site is live and looking decent! Here's my full review:

---

## ✅ What's working well
- Dark theme rendering correctly
- Hero gradient text (Desktop Software Built for India) looks sharp
- Product cards loading with real data
- Trust badges showing (SSL, GST, Razorpay, 7-Day Refund, WhatsApp)
- Mobile-friendly card layout

---

## 🔴 Critical Issues to Fix Now

### 1. Wrong brand name everywhere — says "SoftCraft" not "Vexen Labs"

```bash
# On your server, do a brand replacement across all frontend files
cd /var/www/vexenlabs/frontend/src

# Find all SoftCraft references
grep -r "SoftCraft" . --include="*.tsx" --include="*.ts" -l
```

Files to update manually:
- `components/Navbar.tsx` — logo text "SoftCraft."
- `components/Footer.tsx` — company name, email, GST
- `app/layout.tsx` — metadata title/description
- `app/page.tsx` — hero badge "Trusted by 500+ businesses"
- `app/about/page.tsx` — company story

```bash
# Quick sed replace across all files
find /var/www/vexenlabs/frontend/src -name "*.tsx" -o -name "*.ts" | \
  xargs sed -i 's/SoftCraft Solutions/Vexen Labs/g'

find /var/www/vexenlabs/frontend/src -name "*.tsx" -o -name "*.ts" | \
  xargs sed -i 's/SoftCraft\./Vexen Labs./g'

find /var/www/vexenlabs/frontend/src -name "*.tsx" -o -name "*.ts" | \
  xargs sed -i 's/softcraft\.in/vexenlabs.com/g'

find /var/www/vexenlabs/frontend/src -name "*.tsx" -o -name "*.ts" | \
  xargs sed -i 's/SoftCraft/Vexen Labs/g'
```

Then rebuild:
```bash
cd /var/www/vexenlabs/frontend
npm run build
systemctl restart vexenlabs-frontend
```

---

### 2. Hero section has too much blank space at top

The area between navbar and headline is empty. Fix the hero padding in `app/page.tsx`:

```bash
nano /var/www/vexenlabs/frontend/src/app/page.tsx
```

Find the hero section and change:
```tsx
// FIND:
<section className="relative min-h-screen flex items-center overflow-hidden pt-20">

// CHANGE TO:
<section className="relative min-h-screen flex items-center overflow-hidden pt-28 pb-12">
```

---

### 3. Product cards show "View →" not "Buy Now" + "Try Demo"

The homepage product preview cards link correctly but the `/products` listing page cards should show both buttons. Check if the updated `products/page.tsx` deployed:

```bash
grep -n "TryDemoButton\|Buy Now" /var/www/vexenlabs/frontend/src/app/products/page.tsx
```

If `TryDemoButton` isn't there, the new version wasn't uploaded. Download and re-upload:
- `DemoModal.tsx` → `frontend/src/components/`
- `TryDemoButton.tsx` → `frontend/src/components/`
- `products-page.tsx` → `frontend/src/app/products/page.tsx`

---

### 4. Missing content below the fold

The homepage should have 8 sections below the hero. Scroll down and check which are missing. If sections like Stats, Features, Testimonials, FAQ are blank:

```bash
# Check if the full page.tsx is there
wc -l /var/www/vexenlabs/frontend/src/app/page.tsx
# Should be 250+ lines
```

---

## 🟡 Content to Personalise

### Update Footer with real details
```bash
nano /var/www/vexenlabs/frontend/src/components/Footer.tsx
```

Change these 6 things:
```tsx
// 1. Company name
'Vexen Labs'

// 2. Email
'support@vexenlabs.com'

// 3. Phone (your real WhatsApp number)
'+91 XXXXX XXXXX'

// 4. Location
'Your City, State, India'

// 5. GST (remove if you don't have one yet)
// Delete the GST line entirely for now

// 6. Domain links
'vexenlabs.com'
```

### Update hero stats (currently fake numbers)
```bash
nano /var/www/vexenlabs/frontend/src/app/page.tsx
```

Find the stats section and set realistic numbers for a new launch:
```tsx
// Change from fake "500+" to honest numbers
{ label: 'Happy Customers',       value: 12,  suffix: '+' },
{ label: 'Applications Delivered', value: 28,  suffix: '+' },
{ label: 'Cities Served',          value: 5,   suffix: '+' },
{ label: 'Uptime (Offline App)',    value: 100, suffix: '%' },
```

Fake "500+" will lose trust if a user asks around. Start honest, update as you grow.

### Update "Trusted by 500+ businesses" badge
```tsx
// Find in page.tsx hero section:
'Trusted by 500+ businesses across India'

// Change to:
'🚀 Now Live — Be Among Our First Customers'
// or
'Built for Indian Schools, Clinics & Businesses'
```

---

## 🟢 Next Steps Priority Order

| Priority | Task | Time |
|---|---|---|
| 1 | Fix branding (SoftCraft → Vexen Labs) | 10 min |
| 2 | Update Footer with real contact details | 5 min |
| 3 | Honest stats in hero | 5 min |
| 4 | Add your real WhatsApp number to contact page | 5 min |
| 5 | Connect Gmail SMTP so demo request emails actually send | 20 min |
| 6 | Upload Razorpay live keys after KYC | after KYC |
| 7 | Add Google Analytics | 15 min |
| 8 | Submit sitemap to Google Search Console | 10 min |

---

## Gmail SMTP setup (so demo requests actually reach you)

```bash
nano /var/www/vexenlabs/backend/.env
```

Change:
```env
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=yourgmail@gmail.com
EMAIL_HOST_PASSWORD=xxxx xxxx xxxx xxxx
DEFAULT_FROM_EMAIL=Vexen Labs <noreply@vexenlabs.com>
```

Get `EMAIL_HOST_PASSWORD` from: Google Account → Security → 2-Step Verification → App Passwords → Select app: Mail → Generate.

```bash
systemctl restart vexenlabs-backend
```

Test it:
```bash
cd /var/www/vexenlabs/backend
source venv/bin/activate
python manage.py shell -c "
from django.core.mail import send_mail
send_mail('Test', 'Email works!', 'noreply@vexenlabs.com', ['yourgmail@gmail.com'])
print('Email sent!')
"
```

---

Share a screenshot of the `/products` page and `/products/school-erp` detail page next and I'll review those too.

Which application are we converting? You haven't attached any files or specified the app. Based on our sessions, you have:

1. **School Management ERP** — Spring Boot + JavaFX (Java)
2. **Medical Store ERP** — FastAPI + Next.js (web-based)
3. **BharatBooks Accounting** — Django + Next.js (web-based)

The web-based ones (2 & 3) don't need this — they're already browser-based and license-protected via your website.

**For the School ERP (JavaFX app)**, I'll assume that's what you mean. Here's exactly what I'll build:

---

## What we'll create

```
SchoolERP-Setup.exe          ← Windows installer
├── Bundled JRE 21           ← No Java install required on customer PC  
├── school-erp.jar           ← Obfuscated with ProGuard
├── license-validator.dll    ← Native license check
└── uninstaller.exe          ← Clean uninstall
```

**License flow:**
```
First launch → enter license key
     ↓
Validate against vexenlabs.com/api/licenses/validate/
     ↓
✅ Valid → activate, store encrypted activation locally
     ↓
Daily background check (works offline for 30 days)
     ↓
❌ Expired/Revoked → grace period → lock
```

---

**Please share:**
1. Your School ERP project folder structure (run `tree /var/www` or share the zip)
2. Or confirm — should I build this from the School ERP code we already built in earlier sessions?

I'll then write:
- License validation service (Java)
- Machine fingerprinting (hardware ID)
- jpackage build script (JAR → EXE with bundled JRE)
- Inno Setup script (professional Windows installer)
- ProGuard config (code obfuscation)
- All wired to your `vexenlabs.com` backend

Let me build the complete system — license protection, machine fingerprinting, activation UI, obfuscation, and a professional Windows installer.Download and extract that zip. Here's the complete picture of what you got:

---

## 🏗️ Architecture — How it all fits together

```
Customer buys on vexenlabs.com
         ↓
Gets license key: ABCDE-FGHIJ-KLMNO-PQRST-UVWXY
         ↓
Downloads SchoolERP-Setup-1.0.0.exe (from dashboard)
         ↓
Installs on their PC (no Java needed — bundled inside)
         ↓
First launch → License dialog appears
         ↓
Types key → your server validates → activated ✅
         ↓
App opens — stores encrypted license.dat on their PC
         ↓
Every 30 days → silently re-validates online
         ↓
Offline? → 30 day grace period → then blocks
```

---

## 📋 Do this in order

**Step 1** — Extract zip into your School ERP project folder

**Step 2** — Copy the 4 `license/` classes + 2 `ui/license/` classes into your `src/`

**Step 3** — Add 5 lines to your existing `App.java`:
```java
boolean licenseValid = LicenseManager.checkLicense(primaryStage);
if (!licenseValid) { Platform.exit(); System.exit(0); return; }
```

**Step 4** — Install prerequisites: [Java 21 JDK](https://adoptium.net) + [Inno Setup 6](https://jrsoftware.org/isdl.php)

**Step 5** — Run `build-installer.bat` → produces `SchoolERP-Setup-1.0.0.exe`

**Step 6** — Upload that EXE to `vexenlabs.com/admin/upload`

**Step 7** — Buy a test license from your own site → test the full flow

---

## 🔐 What makes it hard to crack

| Protection | What it stops |
|---|---|
| **AES encrypted license.dat** | Manual editing of stored license |
| **Machine-bound decryption key** | Copying license.dat to another PC |
| **Hardware fingerprint** | VM cloning / sharing |
| **Online re-validation every 30 days** | Removing internet permanently |
| **ProGuard obfuscation** | Decompiling the JAR to remove checks |
| **Server-side revocation** | Sharing working keys publicly |

---------------------------------------------------------

You have two ways to manage content — the Django Admin (powerful, full control) and your custom Admin Panel (simpler, built into the website).

---

## Method 1 — Django Admin Panel
**URL:** `http://localhost:8000/admin/` (dev) or `https://vexenlabs.com/admin/` (live)

Login with your superuser credentials.

---

### Adding a New Product

**Step 1 — Create a Category (if needed)**

```
Django Admin → Products → Categories → Add Category
```
Fill in:
- **Name:** Healthcare
- **Slug:** healthcare *(auto-fills)*
- **Icon:** 🏥
- **Description:** Software for clinics and hospitals

Click **Save**.

---

**Step 2 — Add the Product**

```
Django Admin → Products → Products → Add Product
```

Fill every field:

| Field | Example | Notes |
|---|---|---|
| **Name** | Clinic Manager Pro | Shown on cards and detail page |
| **Slug** | clinic-manager | URL: `/products/clinic-manager` — auto-fills from name |
| **Emoji** | 🏥 | Big icon shown on product card |
| **Category** | Healthcare | Select from dropdown |
| **Tagline** | Patient records, prescriptions, billing in one app | One-line shown on product card |
| **Description** | Full paragraph(s) | Shown on product detail page Overview tab |
| **Version** | 1.0.0 | Shown on detail page |
| **Platform** | All Platforms | Windows / macOS / Linux / All |
| **File Size** | 48 MB | Shown on detail page |
| **Is Active** | ✅ checked | Unchecked = hidden from customers |
| **Is Featured** | ✅ checked | Shows "Most Popular" badge on card |
| **Sort Order** | 1 | Controls order on products page (1 = first) |
| **Demo Type** | request | online / trial / request / none |
| **Trial Days** | 15 | Only matters if Demo Type = trial |

Click **Save and continue editing**.

---

**Step 3 — Add Pricing Plans** *(inline on same page)*

Scroll down to the **Pricing Plans** section. Click **Add another Pricing Plan**.

| Field | Starter | Professional | Enterprise |
|---|---|---|---|
| Name | Starter | Professional | Enterprise |
| Price | 4999 | 7999 | 14999 |
| Original Price | 6999 | 11999 | *(leave blank)* |
| Billing Cycle | One Time | One Time | One Time |
| Max Devices | 1 | 3 | 10 |
| Max Users | 1 | 10 | 999 |
| Is Popular | ❌ | ✅ | ❌ |
| Sort Order | 0 | 1 | 2 |
| Features Included | `1 device,All modules,1 year updates,Email support,GST invoice` | `3 devices,All modules,2 years updates,WhatsApp support,Custom branding` | `10 devices,Lifetime updates,Dedicated support,On-site training` |

Add all three plans. Click **Save**.

---

**Step 4 — Add Testimonials** *(optional)*

```
Django Admin → Products → Testimonials → Add Testimonial
```

| Field | Value |
|---|---|
| Product | Clinic Manager Pro *(select)* |
| Author Name | Dr. Priya Sharma |
| Author Role | Cardiologist |
| Author Company | Apollo Clinic, Delhi |
| Content | Review text here... |
| Rating | 5 |
| Is Featured | ✅ checked *(shows on homepage)* |

---

**Step 5 — Upload the Installer File**

Two ways:

**Option A — Django Admin:**
```
Django Admin → Products → Products → Click your product → Installer File → Choose File
```
Upload your `.exe`, `.zip`, or `.jar` file. Click Save.

**Option B — Your custom upload page:**
```
https://vexenlabs.com/admin/upload
```
Select product → drag and drop the installer → click Upload. Progress bar shows upload status.

---

## Method 2 — Your Custom Admin Panel
**URL:** `http://localhost:3000/admin/` (dev) or `https://vexenlabs.com/admin/`

Login with your staff account. This gives you:

```
/admin           → Stats overview, recent orders, users
/admin/products  → Add / Edit / Delete products inline
/admin/upload    → Upload installer files
```

### Adding a product from the custom panel:

```
Go to /admin/products → Click "New Product" button (top right)
```

Fill the inline form that appears at the top:
- Name, Slug, Emoji, Tagline, Description
- Version, File Size, Platform
- Active ✅ / Featured checkboxes
- Sort Order

Click **Create Product** → it saves immediately.

**Limitation:** The custom panel currently edits only the product itself, not pricing plans. Use Django Admin to add/edit pricing plans.

---

## Quick Reference — What to edit where

| Task | Where |
|---|---|
| Add new product | Django Admin → Products → Add |
| Edit product name/tagline | Either admin panel |
| Add pricing plans | Django Admin only |
| Upload installer file | `/admin/upload` or Django Admin |
| Add/edit testimonials | Django Admin → Testimonials |
| View orders | `/admin` custom panel → Orders tab |
| View license activations | `/admin` custom panel → Licenses tab |
| Manage user accounts | Django Admin → Accounts → Users |
| Change site settings | Edit `.env` file on server + restart |

---

## Making the seed command your source of truth

For products you're building now, the fastest workflow is to update `seed_products.py` once and re-run it:

```bash
# On your server or local machine:
cd backend
python manage.py seed_products
```

Edit this file: `backend/apps/products/management/commands/seed_products.py`

Add a new product block:
```python
{
    'slug': 'clinic-manager',
    'name': 'Clinic Manager Pro',
    'category': health,
    'emoji': '🏥',
    'tagline': 'Patient records, prescriptions, billing in one app.',
    'description': 'Full description here...',
    'version': '1.0.0',
    'platform': 'all',
    'file_size': '52 MB',
    'is_featured': True,
    'sort_order': 2,
    'demo_type': 'request',
    'trial_days': 15,
    'plans': [
        {
            'name': 'Starter', 'price': '7999', 'original_price': '9999',
            'max_devices': 1, 'is_popular': False,
            'features_included': ['1 device', 'All modules', '1 year updates', 'Email support']
        },
        {
            'name': 'Professional', 'price': '12999', 'original_price': None,
            'max_devices': 3, 'is_popular': True,
            'features_included': ['3 devices', 'All modules', '2 years updates', 'WhatsApp support']
        },
    ],
    'testimonials': [],
},
```

Then run `python manage.py seed_products` — the new product appears on the site instantly.

---

## On the live server (vexenlabs.com)

After making any changes:

```bash
# SSH into your server
ssh root@157.245.248.210

# If you edited seed_products.py — re-run it:
cd /var/www/vexenlabs/backend
source venv/bin/activate
python manage.py seed_products

# If you edited frontend code — rebuild:
cd /var/www/vexenlabs/frontend
npm run build
systemctl restart vexenlabs-frontend
```