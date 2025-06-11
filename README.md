# Self-Hosting Chatwoot on Oracle Cloud

A comprehensive step-by-step guide to self-host Chatwoot on Oracle Cloud VMs with custom domains, SSL certificates, and automatic startup.

## 📋 Prerequisites

- Oracle Cloud account (free tier works)
- Domain name with DNS management access  
- SSH access to your Oracle Cloud VM
- Basic command line knowledge

## 🚨 ORACLE CLOUD FIREWALL CRITICAL INFO

**Oracle Cloud has TWO firewall layers that BOTH need configuration:**

1. **Cloud-Level:** Security Groups (Oracle Console)
2. **Server-Level:** iptables (Ubuntu instance)

**Why this matters:** Even with correct Security Groups, your server will block traffic due to Ubuntu's default iptables rules. This causes SSL certificate generation to fail.

## 🌐 Step 1: DNS Configuration

**CRITICAL:** Set up your subdomain BEFORE starting the installation for SSL certificate generation.

Create an A record with your domain provider:

| Record Type | Host | Value | TTL |
|-------------|------|-------|-----|
| A | sales | YOUR_VM_IP | 14400 |

**Example:** If your domain is `stardawnai.com` and VM IP is `130.61.220.9`:
- **Host:** `sales`
- **Points to:** `130.61.220.9`  
- **Result:** `sales.stardawnai.com` → `130.61.220.9`

**Verify DNS propagation:**
```bash
nslookup sales.stardawnai.com
dig sales.stardawnai.com
```

## 🔐 Step 2: Oracle Cloud Security Groups

In Oracle Console, configure these **Ingress Rules** for your VM's Security Group:

| Protocol | Source | Port Range | Description |
|----------|--------|------------|-------------|
| TCP | 0.0.0.0/0 | 22 | SSH |
| TCP | 0.0.0.0/0 | 80 | HTTP |
| TCP | 0.0.0.0/0 | 443 | HTTPS |
| TCP | 0.0.0.0/0 | 3000 | Chatwoot |
| TCP | 0.0.0.0/0 | 5432 | PostgreSQL |
| TCP | 0.0.0.0/0 | 6379 | Redis |

## 🔐 Step 3: Connect to Your VM

```bash
ssh -i ~/path/to/your-ssh-key.key ubuntu@YOUR_VM_IP
```

## ⚡ Step 4: Fix Oracle Cloud iptables (CRITICAL)

**Oracle Cloud blocks traffic despite Security Groups being correct. Fix this FIRST:**

```bash
# Check current iptables (you'll see REJECT rules blocking everything except SSH)
sudo iptables -L INPUT -n -v

# Add required ports BEFORE the REJECT rule
sudo iptables -I INPUT 4 -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 5 -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT 6 -p tcp --dport 3000 -j ACCEPT
sudo iptables -I INPUT 7 -p tcp --dport 5432 -j ACCEPT
sudo iptables -I INPUT 8 -p tcp --dport 6379 -j ACCEPT

# Make rules permanent
sudo apt install iptables-persistent -y
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# Verify ports are open
sudo iptables -L INPUT -n
```

**Why this is needed:** Oracle Cloud Ubuntu instances come with restrictive iptables that block all traffic except SSH, regardless of Security Group settings.

## 📦 Step 5: Download and Run Chatwoot Installer

```bash
cd ~
wget https://get.chatwoot.app/linux/install.sh
chmod +x install.sh
./install.sh --install
```

## ⚙️ Step 6: Installation Configuration

During installation, answer these prompts:

- **Install Postgres and Redis?** → `yes`
- **Domain setup?** → `y` (yes)
- **Domain name?** → `sales.stardawnai.com` (your actual subdomain)
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

## 🛠️ Step 7: Manual SSL Setup (If Needed)

**If SSL fails during installation, run manually:**

```bash
# Generate SSL certificate
sudo certbot certonly --standalone --agree-tos --email your-email@example.com -d sales.stardawnai.com

# Update existing nginx config with your domain
sudo sed -i 's/chatwoot.domain.com/sales.stardawnai.com/g' ~/nginx_chatwoot.conf

# Copy to nginx
sudo cp ~/nginx_chatwoot.conf /etc/nginx/sites-enabled/chatwoot

# Remove default nginx config
sudo rm -f /etc/nginx/sites-enabled/default

# Test and restart nginx
sudo nginx -t
sudo systemctl restart nginx
```

## ✅ Step 8: Verify Installation

**Check service status:**
```bash
# Check Chatwoot services
sudo systemctl status chatwoot.target
sudo systemctl list-units | grep chatwoot

# Check port binding
sudo netstat -tlnp | grep :3000

# Check nginx status
sudo systemctl status nginx
```

**View logs if needed:**
```bash
journalctl -u chatwoot-web.1.service --lines=10 --no-pager
```

## 🌍 Step 9: Access Your Chatwoot Instance

Open your browser and navigate to: **https://sales.stardawnai.com**

You should see the Chatwoot welcome screen with account setup form.

**Fill out the setup form with:**
- **Name:** Your full name
- **Company Name:** Your company  
- **Work Email:** Your email address
- **Password:** Strong password (6+ characters)

Click **"Finish Setup"** to complete the installation.

## ⚙️ Step 10: Additional Configuration (Optional)

Edit the environment file for additional configuration:
```bash
sudo -u chatwoot nano /home/chatwoot/chatwoot/.env
```

Add these optional but recommended settings:
```bash
# Frontend URL (REQUIRED)
FRONTEND_URL=https://sales.stardawnai.com

# Mailer Configuration (Recommended)
MAILER_SENDER_EMAIL=noreply@stardawnai.com
SMTP_ADDRESS=smtp.gmail.com
SMTP_PORT=587
SMTP_DOMAIN=stardawnai.com
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

## 🔄 Step 11: Auto-Start Configuration

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

## 🚨 Oracle Cloud Firewall Troubleshooting

### Problem: SSL Certificate Generation Fails
**Error:** `Error getting validation data` or `Connection refused`

**Cause:** Oracle Cloud's double firewall blocking HTTP/HTTPS traffic

**Solution:**
```bash
# 1. Verify DNS resolves to correct IP
nslookup sales.stardawnai.com
curl ifconfig.me

# 2. Check Security Groups in Oracle Console (must allow ports 80, 443)

# 3. Check local iptables
sudo iptables -L INPUT -n -v

# 4. If you see REJECT rules, add ports before them:
sudo iptables -I INPUT 4 -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 5 -p tcp --dport 443 -j ACCEPT

# 5. Test with simple HTTP server
sudo python3 -m http.server 80

# 6. Try SSL generation again
sudo certbot certonly --standalone --agree-tos --email your-email@example.com -d sales.stardawnai.com
```

### Problem: 502 Bad Gateway
**Cause:** Chatwoot service not running or port 3000 blocked

**Solution:**
```bash
# Check if Chatwoot is running
sudo netstat -tlnp | grep :3000
sudo systemctl status chatwoot-web.1.service

# If port 3000 blocked by iptables:
sudo iptables -I INPUT 6 -p tcp --dport 3000 -j ACCEPT
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# Restart services
sudo systemctl restart chatwoot.target
sudo systemctl restart nginx
```

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
- **Oracle Cloud specific:** Always configure both Security Groups AND iptables

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

# Check iptables rules
sudo iptables -L INPUT -n -v
```

## 🔌 API Access

Once installed, your Chatwoot API will be available at:
- **Base URL:** `https://sales.stardawnai.com/`
- **API Endpoint:** `https://sales.stardawnai.com/api/v1/`
- **Webhooks:** `https://sales.stardawnai.com/webhooks/`

**Get your API credentials:**
- Login to Chatwoot dashboard
- Go to **Settings → Account Settings → API Access**
- Generate new access token

## 🎉 Success!

You now have a fully functional, self-hosted Chatwoot instance that:

- ✅ Runs 24/7 on Oracle Cloud free tier
- ✅ Uses SSL encryption with automatic renewal
- ✅ Accessible via custom domain (sales.stardawnai.com)
- ✅ Auto-starts after VM reboots
- ✅ Includes full database and Redis setup
- ✅ Properly configured for Oracle Cloud's double firewall

### Next Steps:
- Configure your first inbox (Website, Email, etc.)
- Set up team members and departments
- Customize chat widget appearance
- Configure automation rules
- Integrate with your website or app

## ⚠️ Oracle Cloud Specific Notes

**Why Oracle Cloud is different:**
- **Double Firewall:** Security Groups + iptables both need configuration
- **Restrictive Defaults:** Ubuntu instances block everything except SSH by default
- **iptables Persistence:** Rules don't survive reboots without `iptables-persistent`

**Other cloud providers (AWS, Google Cloud) typically:**
- Have only one firewall layer
- More permissive default configurations
- Better documentation for web applications

**This guide addresses Oracle Cloud's specific quirks that cause most installation failures.**

**Need help?** Check the [official Chatwoot documentation](https://www.chatwoot.com/docs) or [community forum](https://github.com/chatwoot/chatwoot/discussions).
