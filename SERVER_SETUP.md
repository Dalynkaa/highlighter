# Server Setup Commands

Complete guide to set up a fresh server for deploying the Highlighter project.

## 1. Initial Server Setup (as root)

```bash
# Update system packages
apt update && apt upgrade -y

# Install essential packages
apt install -y curl wget git nano htop unzip software-properties-common

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Verify Docker installation
docker --version
docker-compose --version
```

## 2. Create Deployment User

```bash
# Create new user for deployment
useradd -m -s /bin/bash highlighter
usermod -aG sudo highlighter
usermod -aG docker highlighter

# Set password for the user
passwd highlighter

# Switch to the new user
su - highlighter
```

## 3. SSH Key Setup (as highlighter user)

```bash
# Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Create authorized_keys file
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Add your public key (replace with your actual public key)
echo "ssh-rsa AAAAB3NzaC1yc2E... your-public-key-here" >> ~/.ssh/authorized_keys
```

## 4. Generate Deployment SSH Key

```bash
# Generate SSH key for GitHub Actions deployment
ssh-keygen -t rsa -b 4096 -f ~/.ssh/deploy_key -N ""

# Display public key (add this to server's authorized_keys)
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys

# Display private key (add this to GitHub secrets as DEPLOY_SSH_KEY)
cat ~/.ssh/deploy_key
```

## 5. Setup Traefik (if not already running)

```bash
# Create Traefik directory
sudo mkdir -p /opt/traefik
sudo chown highlighter:highlighter /opt/traefik
cd /opt/traefik

# Create Traefik network
docker network create traefik

# Create Traefik configuration
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Dashboard (remove in production)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json
    networks:
      - traefik

networks:
  traefik:
    external: true
EOF

# Create Traefik configuration file
cat > traefik.yml << 'EOF'
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true
  insecure: true  # Remove in production

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entrypoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
    network: traefik

certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@domain.com  # Change this
      storage: acme.json
      httpChallenge:
        entryPoint: web
EOF

# Create acme.json with proper permissions
touch acme.json
chmod 600 acme.json

# Start Traefik
docker-compose up -d
```

## 6. Setup Highlighter Project

```bash
# Create project directory
sudo mkdir -p /opt/highlighter
sudo chown highlighter:highlighter /opt/highlighter

cd /opt/highlighter

# Clone configuration (or create files manually)
# Option 1: Clone from your repo
git clone https://github.com/dalynkaa/highlighter.git .

# Option 2: Create files manually
# Create docker-compose.yml and .env files as provided in the deployment guide
```

## 7. Configure Environment

```bash
# Copy environment template
cp .env.example .env

# Edit environment file with actual values
nano .env
```

Edit the `.env` file with your actual values:

```env
GITHUB_USERNAME=dalynkaa
DATABASE_URL=postgresql://prod_user:your_secure_password@postgres:5432/highlighter_prod?schema=public&connection_limit=20&pool_timeout=20
POSTGRES_DB=highlighter_prod
POSTGRES_USER=prod_user
POSTGRES_PASSWORD=your_secure_password
JWT_SECRET=your_strong_jwt_secret_here
JWT_REFRESH_SECRET=your_strong_refresh_secret_here
CLOUDFLARE_R2_ENDPOINT=https://a2ba9c0946ce07a74584a0c50306a476.r2.cloudflarestorage.com
CLOUDFLARE_R2_ACCESS_KEY="21181afa37758cefdc427b6a6faf2243"
CLOUDFLARE_R2_SECRET_KEY="93783c6bf4f26d267e5d12c90ce99e5733baf2a36cecda4e87a644b6adae5f29"
CLOUDFLARE_R2_BUCKET_NAME="nextbit"
CLOUDFLARE_R2_PUBLIC_URL=https://cdn.nexbit.dev
SMTP_HOST=smtp.eu.mailgun.org
SMTP_PORT=25
SMTP_SECURE=false
SMTP_USER=web@nexbit.dev
SMTP_PASS=your_mailgun_password
SMTP_FROM=noreply@nexbit.dev
REDIS_PASSWORD=your_redis_password
```

## 8. Security Configuration

```bash
# Configure UFW firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable

# Disable root SSH access (optional)
sudo nano /etc/ssh/sshd_config
# Set: PermitRootLogin no
# Set: PasswordAuthentication no
sudo systemctl restart sshd
```

## 9. Initial Database Setup

```bash
# Start only the database first
docker-compose up -d postgres redis

# Wait for database to be ready
sleep 10

# Check if database is running
docker-compose ps
```

## 10. Test Deployment

```bash
# Pull latest images manually (first time)
docker-compose pull

# Start all services
docker-compose up -d

# Check logs
docker-compose logs -f

# Check if services are running
docker-compose ps
```

## 11. GitHub Repository Secrets

Add these secrets to both `highlighter-back` and `highlighter-web` repositories:

1. Go to GitHub → Repository → Settings → Secrets and variables → Actions
2. Add the following secrets:

```
DEPLOY_HOST=your-server-ip-or-domain
DEPLOY_USER=highlighter
DEPLOY_SSH_KEY=contents-of-~/.ssh/deploy_key-private-key
```

## 12. Domain Configuration

Make sure your domain `nexbit.dev` points to your server's IP address:

```bash
# Check DNS resolution
nslookup nexbit.dev
dig nexbit.dev

# Test if domain reaches your server
curl -H "Host: nexbit.dev" http://your-server-ip
```

## 13. Useful Maintenance Commands

```bash
# View all containers
docker ps -a

# View logs for specific service
docker-compose logs -f highlighter-back

# Restart specific service
docker-compose restart highlighter-back

# Update and restart all services
docker-compose pull && docker-compose up -d

# Clean up old images
docker system prune -f

# Backup database
docker-compose exec postgres pg_dump -U prod_user highlighter_prod > backup_$(date +%Y%m%d).sql

# Monitor resource usage
htop
docker stats
```

## 14. Automated Backups Setup

```bash
# Create backup script
cat > /opt/highlighter/backup.sh << 'EOF'
#!/bin/bash
cd /opt/highlighter
mkdir -p backups
docker-compose exec -T postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB > ./backups/backup_$(date +%Y%m%d_%H%M%S).sql
find ./backups -name "backup_*.sql" -mtime +7 -delete
EOF

chmod +x /opt/highlighter/backup.sh

# Add to crontab for daily backups at 2 AM
crontab -e
# Add: 0 2 * * * /opt/highlighter/backup.sh
```

## 15. Monitoring Setup (Optional)

```bash
# Install monitoring tools
sudo apt install -y prometheus-node-exporter

# Or use Docker-based monitoring
cat >> /opt/highlighter/docker-compose.yml << 'EOF'

  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.nexbit.dev`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"
    networks:
      - traefik

volumes:
  portainer_data:
EOF
```

This setup provides a complete production-ready server environment with proper security, monitoring, and maintenance procedures.
