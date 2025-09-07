# Highlighter Project Deployment Guide

This guide covers the complete CI/CD setup for deploying the Highlighter project (both backend and frontend) with GitHub Actions, Docker, and Traefik.

## Architecture Overview

- **Frontend**: Next.js app served at `nexbit.dev`
- **Backend**: NestJS API served at `nexbit.dev/api`
- **Reverse Proxy**: Traefik (existing instance)
- **Database**: PostgreSQL
- **Cache**: Redis
- **Container Registry**: GitHub Container Registry (GHCR)

## Prerequisites

### 1. Server Requirements

- Linux server with Docker and Docker Compose installed
- Existing Traefik instance running
- Domain `nexbit.dev` pointing to your server

### 2. GitHub Repository Setup

Both `highlighter-back` and `highlighter-web` repositories need to be set up with the following secrets:

#### Required GitHub Secrets

Navigate to your repository → Settings → Secrets and variables → Actions, and add:

```
DEPLOY_HOST=your-server-ip-or-hostname
DEPLOY_USER=your-server-username
DEPLOY_SSH_KEY=your-private-ssh-key
```

**Generate SSH Key for Deployment:**
```bash
# On your local machine
ssh-keygen -t rsa -b 4096 -f ~/.ssh/deploy_key -N ""

# Add public key to server's authorized_keys
ssh-copy-id -i ~/.ssh/deploy_key.pub user@your-server

# Copy private key content to GitHub secret DEPLOY_SSH_KEY
cat ~/.ssh/deploy_key
```

## Server Setup

### 1. Directory Structure

Create the deployment directory on your server:

```bash
# SSH into your server
ssh user@your-server

# Create deployment directory
sudo mkdir -p /opt/highlighter
sudo chown $USER:$USER /opt/highlighter
cd /opt/highlighter

# Clone the main configuration
git clone https://github.com/your-username/highlighter.git .
```

### 2. Environment Configuration

Copy and configure the environment file:

```bash
cp .env.example .env
```

Edit `.env` with your actual values:

```bash
nano .env
```

Required configuration:
- `GITHUB_USERNAME`: Your GitHub username
- `DATABASE_URL`: PostgreSQL connection string
- `JWT_SECRET` and `JWT_REFRESH_SECRET`: Secure random strings
- Email configuration for notifications
- AWS S3 credentials for file storage
- Redis password

### 3. Traefik Network Setup

Ensure your existing Traefik instance has the required network:

```bash
# Check if traefik network exists
docker network ls | grep traefik

# If not, create it (usually should exist with your Traefik setup)
docker network create traefik
```

### 4. Initial Database Setup

Start only the database to run initial migrations:

```bash
# Start PostgreSQL
docker compose up -d postgres

# Wait for database to be ready
sleep 10

# You'll need to run migrations after first deployment
```

## Deployment Process

### 1. GitHub Actions Workflow

The CI/CD pipeline triggers on:
- Push to `master` branch with version tags (`v*`)
- Pull requests to `master` branch (tests only)

#### Workflow Steps:

1. **Test Stage**: 
   - Sets up test environment
   - Runs linting
   - Executes unit and e2e tests
   - For backend: includes PostgreSQL service for database tests

2. **Build & Deploy Stage** (only on version tags):
   - Builds Docker image
   - Pushes to GitHub Container Registry
   - Deploys to server via SSH
   - Runs database migrations (backend only)

### 2. Manual Deployment

To deploy manually without GitHub Actions:

```bash
# On your server
cd /opt/highlighter

# Pull latest images
docker compose pull

# Start services
docker compose up -d

# For backend: run migrations
docker compose exec highlighter-back bunx prisma migrate deploy

# Check logs
docker compose logs -f
```

## Version Tagging and Releases

Deployment only occurs when pushing version tags to the `master` branch:

```bash
# Create and push a version tag
git tag v1.0.0
git push origin v1.0.0

# This will trigger the CI/CD pipeline
```

## Traefik Configuration

The docker-compose.yml includes Traefik labels for:

### Frontend (highlighter-web)
- **Host**: `nexbit.dev`
- **Port**: 3000
- **SSL**: Automatic via Let's Encrypt
- **Priority**: 1 (lower priority than API routes)

### Backend (highlighter-back)
- **Host**: `nexbit.dev`
- **Path**: `/api/*`
- **Port**: 4000
- **SSL**: Automatic via Let's Encrypt
- **Middleware**: Strip `/api` prefix before forwarding to backend

## Monitoring and Maintenance

### 1. Health Checks

Both containers include health checks:

```bash
# Check container health
docker compose ps

# View detailed health status
docker inspect highlighter-back | grep -A 10 '"Health"'
```

### 2. Logs

Monitor application logs:

```bash
# View all logs
docker compose logs -f

# View specific service logs
docker compose logs -f highlighter-back
docker compose logs -f highlighter-web
```

### 3. Database Backups

Set up automated PostgreSQL backups:

```bash
# Manual backup
docker compose exec postgres pg_dump -U highlighter_user highlighter_db > backup_$(date +%Y%m%d_%H%M%S).sql

# Automated backup script (add to crontab)
cat > /opt/highlighter/backup.sh << 'EOF'
#!/bin/bash
cd /opt/highlighter
docker compose exec -T postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB > ./backups/backup_$(date +%Y%m%d_%H%M%S).sql
# Keep only last 30 days of backups
find ./backups -name "backup_*.sql" -mtime +30 -delete
EOF

chmod +x /opt/highlighter/backup.sh

# Add to crontab for daily backups at 2 AM
crontab -e
# Add: 0 2 * * * /opt/highlighter/backup.sh
```

### 4. SSL Certificate Management

Traefik automatically handles SSL certificates via Let's Encrypt. Ensure your domain DNS is properly configured.

### 5. Updates and Rollbacks

#### Update to Latest Version:
```bash
cd /opt/highlighter
docker compose pull
docker compose up -d
```

#### Rollback to Previous Version:
```bash
# Use specific image tag
docker compose stop
# Edit docker-compose.yml to use previous tag
# Or pull specific version
docker pull ghcr.io/username/highlighter-back:v1.0.0
docker compose up -d
```

## Troubleshooting

### Common Issues:

1. **Container won't start**:
   ```bash
   docker compose logs service-name
   ```

2. **Database connection issues**:
   - Check DATABASE_URL in .env
   - Ensure PostgreSQL is running: `docker compose ps postgres`

3. **Traefik routing issues**:
   - Verify Traefik network exists
   - Check Traefik dashboard for route configuration
   - Ensure DNS points to correct server

4. **SSL certificate issues**:
   - Check Traefik logs for Let's Encrypt challenges
   - Verify domain DNS configuration

### Useful Commands:

```bash
# Restart specific service
docker compose restart highlighter-back

# View resource usage
docker stats

# Clean up unused images
docker system prune -f

# Access container shell
docker compose exec highlighter-back sh

# Check database connectivity
docker compose exec postgres psql -U highlighter_user -d highlighter_db -c "SELECT version();"
```

## Security Considerations

1. **Environment Variables**: Never commit actual `.env` files to Git
2. **SSH Keys**: Use dedicated deployment keys with minimal permissions
3. **Database**: Use strong passwords and consider IP restrictions
4. **Updates**: Regularly update base images and dependencies
5. **Backups**: Encrypt sensitive backup files
6. **Firewall**: Ensure only necessary ports are open (80, 443, 22)

## GitHub Actions Configuration Summary

The workflows are configured to:
- Run tests on every PR and push to master
- Deploy only on version tags to master branch
- Use multi-stage Docker builds for optimization
- Implement proper caching for dependencies
- Include security scanning (can be extended)
- Clean up old images after deployment

This setup provides a robust, automated deployment pipeline with proper testing, security, and monitoring capabilities.