# Self-Hosting Chatwoot on Oracle Cloud

A comprehensive step-by-step guide to self-host Chatwoot on Oracle Cloud VMs with custom domains, SSL certificates, and automatic startup.

## 📋 Prerequisites

- Oracle Cloud account (free tier works)
- Domain name with DNS management access  
- SSH access to your Oracle Cloud VM
- Basic command line knowledge

## 🌐 Step 1: DNS Configuration

**CRITICAL:** Set up your subdomain BEFORE starting the installation for SSL certificate generation.

Create an A record with your domain provider:

| Record Type | Host | Value | TTL |
|-------------|------|-------|-----|
| A | chat | YOUR_VM_IP | 14400 |

**Example:** If your domain is `example.com` and VM IP is `123.456.78.90`:
- **Host:** `chat`
- **Points to:** `123.456.78.90`  
- **Result:** `chat.example.com` → `123.456.78.90`

**Verify DNS propagation:**
```bash
nslookup chat.example.com
```

## 🔐 Step 2: Connect to Your VM

```bash
ssh -i ~/path/to/your-ssh-key.key ubuntu@YOUR_VM_IP
```

## 📦 Step 3: Download and Run Chatwoot Installer

```bash
cd ~
wget https://get.chatwoot.app/linux/install.sh
chmod +x install.sh
./install.sh --install
```

## ⚙️ Step 4: Installation Configuration

During installation, answer these prompts:

- **Install Postgres and Redis?** → `yes`
- **Domain setup?** → `y` (yes)
- **Domain name?** → `chat.example.com` (your actual subdomain)
- **Email for SSL?** → `your-email@example.com`
- **Additional configurations** → Accept defaults

### Installation Progress:
```
➥ 1/9 Installing dependencies. This takes a while.
➥ 2/9 Installing databases.
➥ 3/9 Installing webserver.
➥ 4/9 Setting up Ruby
➥ 5/9 Setting up the database.
➥ 6/9 Installing Chatwoot. This takes a long while.
➥ 7/9 Running database migrations.
➥ 8/9 Setting up systemd services.
➥ 9/9 Setting up SSL/TLS.
```

## 🛠️ Step 5: Manual SSL Setup (If Needed)

**Common error:**
```
Non-ASCII domain names not supported. To issue for an Internationalized Domain Name, use Punycode.
Some error has occured. Check '/var/log/chatwoot-setup.log' for details.
```

**Solution - Manual SSL Certificate:**
```bash
sudo certbot --nginx -d chat.example.com --email your-email@example.com --agree-tos --no-eff-email
```

**Alternative if above fails:**
```bash
sudo certbot certonly --standalone --agree-tos --email your-email@example.com -d chat.example.com
```

**Manual Nginx Configuration:**
```bash
sudo nano /etc/nginx/sites-available/chatwoot.conf
```

Add this configuration:
```nginx
server {
    server_name chat.example.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/chat.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chat.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    if ($host = chat.example.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    server_name chat.example.com;
    return 404;
}
```

**Enable the configuration:**
```bash
sudo ln -s /etc/nginx/sites-available/chatwoot.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## ✅ Step 6: Verify Installation

**Check service status:**
```bash
sudo systemctl status chatwoot.target
sudo netstat -tlnp | grep :3000
```

**View logs if needed:**
```bash
journalctl -u chatwoot-web.1.service --lines=10 --no-pager
```

## 🌍 Step 7: Access Your Chatwoot Instance

Open your browser and navigate to: **https://chat.example.com**

You should see the Chatwoot welcome screen with account setup form.

**Fill out the setup form with:**
- **Name:** Your full name
- **Company Name:** Your company  
- **Work Email:** Your email address
- **Password:** Strong password (6+ characters)

Click **"Finish Setup"** to complete the installation.

## ⚙️ Step 8: Additional Configuration (Optional)

Edit the environment file for additional configuration:
```bash
sudo -u chatwoot nano /home/chatwoot/chatwoot/.env
```

Add these optional but recommended settings:
```bash
# Frontend URL (REQUIRED)
FRONTEND_URL=https://chat.example.com

# Mailer Configuration (Recommended)
MAILER_SENDER_EMAIL=noreply@example.com
SMTP_ADDRESS=smtp.gmail.com
SMTP_PORT=587
SMTP_DOMAIN=example.com
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_AUTHENTICATION=plain
SMTP_ENABLE_STARTTLS_AUTO=true

# Storage Configuration
ACTIVE_STORAGE_SERVICE=local
```

**After any .env changes:**
```bash
sudo systemctl restart chatwoot.target
```

## 🔄 Step 9: Auto-Start Configuration

Chatwoot should automatically start on boot. To verify:
```bash
sudo systemctl is-enabled chatwoot.target
```

If not enabled:
```bash
sudo systemctl enable chatwoot.target
```

## 🔧 Port Overview

Your setup uses these ports:
- **Port 3000:** Chatwoot Application
- **Port 5432:** PostgreSQL (internal)
- **Port 6379:** Redis (internal)  
- **Port 80:** HTTP (redirects to HTTPS)
- **Port 443:** HTTPS (SSL)

## 🛠️ Management Commands

### Chatwoot CLI Tool
```bash
# Install/Update Chatwoot CLI
sudo wget https://get.chatwoot.app/linux/install.sh -O /usr/local/bin/cwctl && chmod +x /usr/local/bin/cwctl

# Common commands
sudo cwctl --help           # Show help
sudo cwctl -r               # Restart Chatwoot
sudo cwctl -c               # Rails console
sudo cwctl -l web           # Web service logs
sudo cwctl -l worker        # Worker service logs
sudo cwctl --upgrade        # Upgrade Chatwoot
```

### Manual Service Control
```bash
# Stop Chatwoot
sudo systemctl stop chatwoot.target

# Start Chatwoot  
sudo systemctl start chatwoot.target

# Restart Chatwoot
sudo systemctl restart chatwoot.target

# View logs
journalctl -u chatwoot-web.1.service -f --lines=50
journalctl -u chatwoot-worker.1.service -f --lines=50
```

### Database Operations
```bash
# Access Rails console
sudo -u chatwoot -i
cd chatwoot
RAILS_ENV=production bundle exec rails c

# Database migrations (after updates)
sudo -u chatwoot -i
cd chatwoot
RAILS_ENV=production bundle exec rake db:migrate
exit
sudo systemctl restart chatwoot.target
```

## 🔄 Manual Updates

```bash
# Quick update using CLI
sudo cwctl --upgrade

# Manual update process
sudo -i -u chatwoot
cd chatwoot

# Pull latest version
git checkout master && git pull

# Update Ruby version
rvm install "ruby-3.3.3"
rvm use 3.3.3 --default

# Update dependencies
bundle
pnpm i

# Precompile assets
rake assets:precompile RAILS_ENV=production

# Migrate database
RAILS_ENV=production bundle exec rake db:migrate

# Switch back to root
exit

# Update systemd services
sudo cp /home/chatwoot/chatwoot/deployment/chatwoot-web.1.service /etc/systemd/system/
sudo cp /home/chatwoot/chatwoot/deployment/chatwoot-worker.1.service /etc/systemd/system/
sudo cp /home/chatwoot/chatwoot/deployment/chatwoot.target /etc/systemd/system/

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart chatwoot.target
```

## 🔐 Security Considerations

- Change default passwords in the `.env` file
- Use strong, unique passwords for all accounts
- Regularly update Chatwoot for security patches
- Monitor logs for suspicious activity
- Keep SSL certificates updated (auto-renewal should work)

## 🛠️ Troubleshooting

### 502 Bad Gateway
**Cause:** Chatwoot service not running

**Solution:**
```bash
# Check service status
sudo systemctl status chatwoot-web.1.service

# Check port binding
sudo netstat -tlnp | grep :3000

# Check logs
journalctl -u chatwoot-web.1.service --lines=20 --no-pager

# Restart services
sudo systemctl restart chatwoot.target
```

### SSL Certificate Issues
**Cause:** Domain not propagated or Certbot errors

**Solution:**
```bash
# Test domain resolution
nslookup chat.example.com

# Manual certificate generation
sudo certbot --nginx -d chat.example.com --email your-email@example.com

# Check certificate status
sudo certbot certificates
```

### Database Connection Issues
**Cause:** PostgreSQL not running or configuration issues

**Solution:**
```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Start PostgreSQL if needed
sudo systemctl start postgresql

# Check database exists
sudo -u postgres psql -l | grep chatwoot
```

### Asset Compilation Issues
**Cause:** JavaScript/CSS compilation errors

**Solution:**
```bash
sudo -u chatwoot -i
cd chatwoot
RAILS_ENV=production rake assets:clean assets:clobber assets:precompile
exit
sudo systemctl restart chatwoot.target
```

## 📊 Monitoring & Logs

```bash
# Chatwoot application logs
journalctl -u chatwoot-web.1.service
journalctl -u chatwoot-worker.1.service

# Nginx logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# SSL certificate logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log

# Installation logs
sudo cat /var/log/chatwoot-setup.log

# Check system resources
htop
df -h
free -h

# Check port usage
sudo netstat -tlnp

# Check service status
sudo systemctl list-units --type=service --state=running | grep chatwoot
```

## 🔌 API Access

Once installed, your Chatwoot API will be available at:
- **Base URL:** `https://chat.example.com/`
- **API Endpoint:** `https://chat.example.com/api/v1/`
- **Webhooks:** `https://chat.example.com/webhooks/`

**Get your API credentials:**
- Login to Chatwoot dashboard
- Go to **Settings → Account Settings → API Access**
- Generate new access token

## 🎉 Success!

You now have a fully functional, self-hosted Chatwoot instance that:

- ✅ Runs 24/7 on Oracle Cloud free tier
- ✅ Uses SSL encryption with automatic renewal
- ✅ Accessible via custom domain
- ✅ Auto-starts after VM reboots
- ✅ Includes full database and Redis setup

### Next Steps:
- Configure your first inbox (Website, Email, etc.)
- Set up team members and departments
- Customize chat widget appearance
- Configure automation rules
- Integrate with your website or app

**Need help?** Check the [official Chatwoot documentation](https://www.chatwoot.com/docs) or [community forum](https://github.com/chatwoot/chatwoot/discussions).
