# Complete Harbor Installation Guide - Step by Step

## Table of Contents
1. Prerequisites
2. Download Harbor
3. Configure harbor.yml
4. Generate SSL Certificates
5. Run Prepare
6. Run Installation
7. Verify Installation
8. Access Harbor
9. Post-Installation Setup
10. Troubleshooting

---

## Step 1: Prerequisites

### Check Docker Installation
```bash
docker --version
```
Expected output: `Docker version 20.10+`

### Check Docker Compose Installation
```bash
docker-compose --version
```
Expected output: `Docker Compose version 2.0+`

### If Docker/Docker Compose not installed
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Install Docker Compose (v2)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Verify permissions
```bash
sudo usermod -aG docker $USER
```

---

## Step 2: Download Harbor

### Download Harbor Offline Installer
```bash
# Create software directory
mkdir -p ~/software
cd ~/software

# Download Harbor (Choose latest version)
wget https://github.com/goharbor/harbor/releases/download/v2.9.0/harbor-offline-installer-v2.9.0.tgz

# Extract Harbor
tar xzvf harbor-offline-installer-v2.9.0.tgz

# Navigate to Harbor directory
cd harbor
```

### Verify Harbor contents
```bash
ls -la
```

You should see:
- `install.sh` - Installation script
- `prepare` - Configuration preparation script
- `harbor.yml.tmpl` - Configuration template
- `common.sh` - Common utilities
- `LICENSE` - License file

---

## Step 3: Configure harbor.yml

### Copy configuration template
```bash
cp harbor.yml.tmpl harbor.yml
```

### Edit harbor.yml
```bash
nano harbor.yml
```

### Key configurations to set:

**1. Hostname (Replace with your IP or domain)**
```yaml
hostname: 36.255.69.72
# or
hostname: your-domain.com
```

**2. HTTP Configuration**
```yaml
http:
  port: 8080
```

**3. HTTPS Configuration**
```yaml
https:
  port: 443
  certificate: /home/ashraful/software/data/cert/harbor.crt
  private_key: /home/ashraful/software/data/cert/harbor.key
```

**4. Admin Password**
```yaml
harbor_admin_password: YourStrongPassword123
```

**5. Database Password**
```yaml
database:
  password: db_root_password_123
```

**6. Data Volume**
```yaml
data_volume: /data
```

### Save and exit
Press `Ctrl + X`, then `Y`, then `Enter`

---

## Step 4: Generate SSL Certificates

### Create certificate directory
```bash
sudo mkdir -p /home/ashraful/software/data/cert
cd /home/ashraful/software/data/cert
```

### Option A: Generate Self-Signed Certificate (Development)
```bash
# Generate private key (2048-bit RSA)
sudo openssl genrsa -out harbor.key 2048

# Generate certificate (valid for 365 days)
sudo openssl req -new -x509 -key harbor.key -out harbor.crt -days 365 \
  -subj "/CN=36.255.69.72/O=MyCompany/C=BD"
```

### Option B: Use CA-Signed Certificate (Production)
```bash
# Copy your existing certificate and key
sudo cp /path/to/your/certificate.crt harbor.crt
sudo cp /path/to/your/private.key harbor.key
```

### Set proper permissions
```bash
sudo chmod 600 harbor.key
sudo chmod 644 harbor.crt
sudo chown -R $USER:$USER /home/ashraful/software/data
```

### Verify certificate created
```bash
ls -la /home/ashraful/software/data/cert/
```

Output should show:
```
-rw-r--r-- harbor.crt
-rw------- harbor.key
```

---

## Step 5: Run Prepare

### Navigate to Harbor directory
```bash
cd ~/software/harbor
```

### Run prepare script
```bash
sudo ./prepare
```

### Expected output
```
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
...
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
```

The script will:
- Download the prepare Docker image
- Generate configuration files
- Create secrets and certificates
- Generate docker-compose.yml

---

## Step 6: Run Installation

### Run install script
```bash
sudo ./install.sh
```

### What the installer does
1. Checks Docker installation
2. Checks Docker Compose installation
3. Prepares environment
4. Prepares Harbor configs
5. Pulls Harbor Docker images
6. Starts all Harbor containers

### Expected output
```
[Step 0]: checking if docker is installed ...
[Step 1]: checking docker-compose is installed ...
[Step 2]: preparing environment ...
[Step 3]: preparing harbor configs ...
[Step 4]: starting Harbor ...

[+] Running 10/10
 ✔ Network harbor_harbor        Created
 ✔ Container harbor-log         Started
 ✔ Container redis              Started
 ✔ Container registry           Started
 ✔ Container harbor-db          Started
 ✔ Container registryctl        Started
 ✔ Container harbor-portal      Started
 ✔ Container harbor-core        Started
 ✔ Container harbor-jobservice  Started
 ✔ Container nginx              Started

✔ ----Harbor has been installed and started successfully.----
```

### Installation time
First installation takes 3-5 minutes (downloads all images)

---

## Step 7: Verify Installation

### Check all containers are running
```bash
docker-compose ps
```

Output should show all 10 containers in "running" state:
```
NAME                COMMAND             STATUS
harbor-log          "/bin/sh -c ..."    Up 2 minutes
harbor-db           "docker-entrypoint..." Up 2 minutes
harbor-core         "/harbor/entry.sh"  Up 2 minutes
harbor-portal       "nginx -g daemon..." Up 2 minutes
harbor-registry     "/entrypoint.sh..." Up 2 minutes
redis               "redis-server ..."  Up 2 minutes
harbor-jobservice   "/harbor/entry.sh"  Up 2 minutes
registryctl         "/harbor/entry.sh"  Up 2 minutes
nginx               "nginx -g daemon..." Up 2 minutes
```

### Check logs
```bash
# View all logs
docker-compose logs

# View specific service logs
docker-compose logs harbor-core

# Follow logs in real-time
docker-compose logs -f
```

### Check port availability
```bash
sudo netstat -tulpn | grep -E ':(80|443|8080|4433)'
```

You should see:
- Port 80 (HTTP) - forwarding to 8080
- Port 443 (HTTPS) - assigned to nginx
- Port 8080 (HTTP backend)

---

## Step 8: Access Harbor

### Open your browser

**For HTTP:**
```
http://36.255.69.72:8080
```

**For HTTPS:**
```
https://36.255.69.72:443
```

### Login credentials
- **Username:** `admin`
- **Password:** `YourStrongPassword123` (from harbor.yml)

### Handle SSL Certificate Warning
If using self-signed certificate:
1. Click "Advanced" or "Details"
2. Click "Proceed to 36.255.69.72" or similar
3. Login to Harbor

### Successful login
After login, you should see:
- Dashboard with system information
- Navigation menu on left
- Projects section
- Administration section

---

## Step 9: Post-Installation Setup

### 1. Change Admin Password (Recommended)

After first login:
1. Click on your avatar (top right)
2. Select "User Profile"
3. Click "Change Password"
4. Enter current password and new password
5. Click "Save"

### 2. Create a New Project

1. Go to "Projects" in left menu
2. Click "New Project"
3. Enter project name: `myproject`
4. Select "Public" or "Private"
5. Click "Create Project"

### 3. Test Docker Push/Pull

**Configure Docker to trust Harbor (self-signed cert):**
```bash
# Create daemon.json
sudo nano /etc/docker/daemon.json
```

Add this:
```json
{
  "insecure-registries": ["36.255.69.72:443"]
}
```

Restart Docker:
```bash
sudo systemctl restart docker
```

**Login to Harbor from Docker:**
```bash
docker login -u admin -p YourPassword 36.255.69.72:443
```

**Tag and push an image:**
```bash
# Tag an image
docker tag nginx:latest 36.255.69.72:443/myproject/nginx:latest

# Push to Harbor
docker push 36.255.69.72:443/myproject/nginx:latest

# Pull from Harbor
docker pull 36.255.69.72:443/myproject/nginx:latest
```

### 4. Enable Trivy Vulnerability Scanner

In Harbor UI:
1. Go to "Administration" → "Configuration"
2. Select "Vulnerability"
3. Ensure Trivy is enabled
4. Set scanning policy

---

## Step 10: Troubleshooting

### Issue 1: Port 443 already in use

**Error message:**
```
Error response from daemon: failed to bind host port for 0.0.0.0:443
```

**Solution:**
```bash
# Find process using port 443
sudo lsof -i :443

# Kill the process
sudo kill -9 PID

# Or change Harbor port in harbor.yml
https:
  port: 4433  # Use different port
```

### Issue 2: Docker Compose not found

**Solution:**
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Issue 3: Cannot connect to Harbor

**Check containers status:**
```bash
docker-compose ps
```

**Restart containers:**
```bash
docker-compose restart
```

**Check logs:**
```bash
docker-compose logs harbor-core
docker-compose logs nginx
```

### Issue 4: SSL Certificate Error

**Regenerate certificate:**
```bash
cd ~/software/data/cert
sudo rm harbor.crt harbor.key
sudo openssl genrsa -out harbor.key 2048
sudo openssl req -new -x509 -key harbor.key -out harbor.crt -days 365 \
  -subj "/CN=36.255.69.72/O=MyCompany/C=BD"
```

**Rerun installation:**
```bash
cd ~/software/harbor
sudo ./prepare
sudo ./install.sh
```

### Issue 5: Database connection error

**Check database logs:**
```bash
docker-compose logs harbor-db
```

**Restart database:**
```bash
docker-compose restart harbor-db
```

### Issue 6: Cannot push/pull images

**Login again:**
```bash
docker logout 36.255.69.72:443
docker login -u admin -p YourPassword 36.255.69.72:443
```

**Check firewall:**
```bash
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp
```

---

## Useful Commands

### Start Harbor
```bash
cd ~/software/harbor
docker-compose up -d
```

### Stop Harbor
```bash
cd ~/software/harbor
docker-compose down
```

### Restart Harbor
```bash
cd ~/software/harbor
docker-compose restart
```

### View logs
```bash
docker-compose logs -f
```

### Check disk usage
```bash
du -sh /data/
```

### Backup Harbor database
```bash
docker-compose exec harbor-db pg_dump -U postgres harbor > harbor_backup.sql
```

### Restore Harbor database
```bash
docker-compose exec harbor-db psql -U postgres harbor < harbor_backup.sql
```

---

## Summary Checklist

- [ ] Docker and Docker Compose installed
- [ ] Harbor downloaded and extracted
- [ ] harbor.yml configured with correct hostname
- [ ] SSL certificates generated/obtained
- [ ] Prepare script executed successfully
- [ ] Installation script completed
- [ ] All containers running
- [ ] Accessed Harbor in browser
- [ ] Admin password changed
- [ ] Project created
- [ ] Docker image pushed/pulled successfully
- [ ] Trivy scanner enabled

---

## Next Steps

1. Create projects for your applications
2. Configure Harbor authentication (LDAP, OAuth)
3. Set up replication policies
4. Enable vulnerability scanning
5. Configure webhooks for CI/CD integration
6. Set up backups and disaster recovery

---

## Useful Resources

- Official Harbor Documentation: https://goharbor.io/docs/
- Docker Registry Documentation: https://docs.docker.com/registry/
- SSL Certificate Guide: https://letsencrypt.org/
- Harbor GitHub: https://github.com/goharbor/harbor
