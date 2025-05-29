# Self-Hosting Chatwoot on Oracle Cloud VM

A comprehensive guide to install and configure Chatwoot on Oracle Cloud using a custom domain with SSL certificates and automatic startup.

## 📋 Prerequisites

- Oracle Cloud account (free tier works)
- Domain name with DNS management access
- SSH access to your Oracle Cloud VM
- Basic command line knowledge

## ⚠️ IMPORTANT: Port Conflicts Warning

**If you already have services running on your VM (like n8n, Supabase, etc.), READ THIS FIRST:**

Chatwoot defaults to **Port 3000**, which commonly conflicts with:
- Supabase Studio (Port 3000)
- Create React App dev servers (Port 3000)
- Other Node.js applications (Port 3000)

**Before starting the installation**, check which ports are already in use:

```bash
sudo netstat -tlnp | grep -E ':(3000|3001|5432|6379)'
```

**If Port 3000 is already occupied:**
- We'll configure Chatwoot to run on **Port 3001** instead
- This guide includes specific instructions for port conflicts
- The installation process will be slightly different (covered in Step 7)

**For fresh VM instances:** You can follow the standard installation without modifications.

---

## 🚀 Step 1: DNS Configuration

**CRITICAL:** Set up your subdomain BEFORE starting the installation for SSL certificate generation.

Create an A record with your domain provider:

| Record Type | Host | Value | TTL |
|-------------|------|--------|-----|
| A | `chat` | `YOUR_VM_IP` | 14400 |

**Example**: If your domain is `example.com` and VM IP is `123.456.78.90`:
- Host: `chat`
- Points to: `123.456.78.90`
- Result: `chat.example.com` → `123.456.78.90`

**Verify DNS propagation:**
```bash
nslookup chat.example.com
```

## 🖥️ Step 2: Connect to Your VM

```bash
ssh -i ~/path/to/your-ssh-key.key ubuntu@YOUR_VM_IP
```

## 🔍 Step 3: Pre-Installation Port Check

Check for existing services and port usage:

```bash
# Check critical ports
sudo netstat -tlnp | grep -E ':(3000|3001|5432|6379|80|443)'

# Check active nginx sites
sudo ls -la /etc/nginx/sites-enabled/ 2>/dev/null || echo "No nginx sites found"

# Check existing SSL certificates
sudo ls -la /etc/letsencrypt/live/ 2>/dev/null || echo "No SSL certificates found"
```

**Record the output** - you'll need this information for Step 7.

## 📦 Step 4: Download and Run Chatwoot Installer

```bash
cd ~
wget https://get.chatwoot.app/linux/install.sh
chmod +x install.sh
./install.sh --install
```

**During installation, answer these prompts:**

1. **Install Postgres and Redis?** → `yes` (unless you have external services)
2. **Domain setup?** → `y` (yes)
3. **Domain name?** → `chat.example.com` (your actual subdomain)
4. **Email for SSL?** → `your-email@example.com`
5. **Additional configurations** → Accept defaults

**Installation Progress:**
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

## 🔧 Step 5: Handle Installation Errors

### If SSL Setup Fails:

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

## 🌐 Step 6: Nginx Configuration

### If Nginx config wasn't created automatically:

```bash
sudo nano /etc/nginx/sites-available/chatwoot.conf
```

**Add this configuration:**
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

## ⚙️ Step 7: Handle Port Conflicts (If Applicable)

**⚠️ ONLY follow this section if Port 3000 is already in use by other services.**

### Check if Chatwoot is running correctly:

```bash
sudo systemctl status chatwoot.target
sudo netstat -tlnp | grep :3000
```

### If you see "Address already in use" errors:

```bash
journalctl -u chatwoot-web.1.service --lines=10 --no-pager
```

**Look for:**
```
Address already in use - bind(2) for "0.0.0.0" port 3000 (Errno::EADDRINUSE)
```

### Solution: Configure Chatwoot for Port 3001

#### 7.1 Stop Chatwoot Services
```bash
sudo systemctl stop chatwoot.target
```

#### 7.2 Update Environment Configuration
```bash
sudo -u chatwoot nano /home/chatwoot/chatwoot/.env
```

**Add/modify these lines:**
```env
PORT=3001
FRONTEND_URL=https://chat.example.com
```

**Ensure no line conflicts** - fix any lines like:
```
# BAD:
INSTALLATION_ENV=linux_scriptPORT=3001

# GOOD:
INSTALLATION_ENV=linux_script
PORT=3001
```

#### 7.3 Update SystemD Service File
```bash
sudo nano /etc/systemd/system/chatwoot-web.1.service
```

**Find and change:**
```ini
# Change this line:
Environment="PORT=3000"

# To this:
Environment="PORT=3001"
```

#### 7.4 Update Nginx Configuration
```bash
sudo nano /etc/nginx/sites-available/chatwoot.conf
```

**Change the proxy_pass line:**
```nginx
# Change this:
proxy_pass http://localhost:3000;

# To this:
proxy_pass http://localhost:3001;
```

#### 7.5 Reload and Restart Services
```bash
sudo systemctl daemon-reload
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl restart chatwoot.target
```

#### 7.6 Verify Port 3001 Usage
```bash
# Wait 30 seconds for startup
sleep 30

# Check if Chatwoot is running on port 3001
sudo netstat -tlnp | grep :3001
```

**Expected output:**
```
tcp        0      0 0.0.0.0:3001            0.0.0.0:*               LISTEN      12345/puma 6.4.3
```

## ✅ Step 8: Verification and Testing

### Check Service Status
```bash
sudo systemctl status chatwoot.target
sudo systemctl status chatwoot-web.1.service
sudo systemctl status chatwoot-worker.1.service
```

### Check Database Services
```bash
sudo systemctl status postgresql
sudo systemctl status redis
```

### Test Web Access
Open your browser and navigate to: `https://chat.example.com`

You should see the Chatwoot welcome screen with account setup form.

## 🔧 Step 9: Initial Configuration

### Create Your Admin Account

Fill out the setup form with:
- **Name**: Your full name
- **Company Name**: Your company
- **Work Email**: Your email address
- **Password**: Strong password (6+ characters)

Click **"Finish Setup"** to complete the installation.

### Essential Environment Variables

Edit the environment file for additional configuration:

```bash
sudo -u chatwoot nano /home/chatwoot/chatwoot/.env
```

**Add these optional but recommended settings:**
```env
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

# Security (Auto-generated during install)
SECRET_KEY_BASE=your-secret-key
```

**After any .env changes:**
```bash
sudo systemctl restart chatwoot.target
```

## 🔄 Step 10: Auto-Start Configuration

Chatwoot should automatically start on boot. To verify:

```bash
sudo systemctl is-enabled chatwoot.target
```

**If not enabled:**
```bash
sudo systemctl enable chatwoot.target
```

## 📊 Port Overview

Your final setup should use these ports:

- **Port 3000**: Chatwoot (default) OR other existing service
- **Port 3001**: Chatwoot (if port 3000 was occupied)
- **Port 5432**: PostgreSQL (internal)
- **Port 6379**: Redis (internal)
- **Port 80**: HTTP (redirects to HTTPS)
- **Port 443**: HTTPS (SSL)

## 🛠️ Management Commands

### Chatwoot CLI (Available after installation)

```bash
# Install/Update Chatwoot CLI
sudo wget https://get.chatwoot.app/linux/install.sh -O /usr/local/bin/cwctl && chmod +x /usr/local/bin/cwctl

# Common commands
sudo cwctl --help          # Show help
sudo cwctl -r              # Restart Chatwoot
sudo cwctl -c              # Rails console
sudo cwctl -l web          # Web service logs
sudo cwctl -l worker       # Worker service logs
sudo cwctl --upgrade       # Upgrade Chatwoot
```

### Manual Service Management

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

### Database Management

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

## 🔄 Upgrading Chatwoot

### Using Chatwoot CLI (Recommended)
```bash
sudo cwctl --upgrade
```

### Manual Upgrade Process
```bash
# Login as chatwoot user
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

1. **Change default passwords** in the `.env` file
2. **Use strong, unique passwords** for all accounts
3. **Regularly update Chatwoot** for security patches
4. **Monitor logs** for suspicious activity
5. **Keep SSL certificates updated** (auto-renewal should work)
6. **Firewall configuration** (Oracle Cloud security groups handle this)

## 🛠️ Troubleshooting

### Common Issues

#### 502 Bad Gateway
**Cause:** Chatwoot service not running or wrong port configuration

**Solution:**
```bash
# Check service status
sudo systemctl status chatwoot-web.1.service

# Check port binding
sudo netstat -tlnp | grep -E ':(3000|3001)'

# Check logs
journalctl -u chatwoot-web.1.service --lines=20 --no-pager

# Restart services
sudo systemctl restart chatwoot.target
```

#### SSL Certificate Issues
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

#### Database Connection Errors
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

#### Port Conflict Errors
**Cause:** Multiple services trying to use the same port

**Solution:**
```bash
# Identify what's using the port
sudo lsof -i :3000

# Follow Step 7 to configure alternate port
# Or stop conflicting service if not needed
```

#### Asset Precompilation Failures
**Cause:** JavaScript/CSS compilation errors

**Solution:**
```bash
sudo -u chatwoot -i
cd chatwoot
RAILS_ENV=production rake assets:clean assets:clobber assets:precompile
exit
sudo systemctl restart chatwoot.target
```

### Log Locations

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
```

### Performance Monitoring

```bash
# Check system resources
htop
df -h
free -h

# Check port usage
sudo netstat -tlnp

# Check service status
sudo systemctl list-units --type=service --state=running | grep chatwoot
```

## 📚 API Usage

Once installed, your Chatwoot API will be available at:

- **Base URL**: `https://chat.example.com/`
- **API Endpoint**: `https://chat.example.com/api/v1/`
- **Webhooks**: `https://chat.example.com/webhooks/`

**Get your API credentials:**
1. Login to Chatwoot dashboard
2. Go to Settings → Account Settings → API Access
3. Generate new access token

## 🎉 Success!

You now have a fully functional, self-hosted Chatwoot instance that:

- ✅ **Runs 24/7** on Oracle Cloud free tier
- ✅ **Uses SSL encryption** with automatic renewal
- ✅ **Accessible via custom domain**
- ✅ **Handles port conflicts** with existing services
- ✅ **Auto-starts** after VM reboots
- ✅ **Includes full database and Redis setup**

---

**Next Steps:**
1. Configure your first inbox (Website, Email, etc.)
2. Set up team members and departments
3. Customize chat widget appearance
4. Configure automation rules
5. Integrate with your website or app

**Need help?** Check the [official Chatwoot documentation](https://www.chatwoot.com/docs) or [community forum](https://github.com/chatwoot/chatwoot/discussions).
