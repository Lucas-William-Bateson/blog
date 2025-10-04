# Mini Deploy Template - Setup Guide

A lightweight, production-ready deployment template using Docker and GitHub Actions. This guide will walk you through setting up your own deployment pipeline from scratch.

## 🚀 Quick Setup Checklist

Before you begin, ensure you have:

- [ ] Docker installed and running
- [ ] GitHub repository with Actions enabled
- [ ] Self-hosted runner configured (or modify for GitHub-hosted runners)
- [ ] Container registry access (Docker Hub, GitHub Packages, etc.)

## 📋 Required Configuration Changes

### 1. Container Registry Setup

**Update the registry URL in `.github/workflows/ci.yml`:**

```yaml
env:
  REGISTRY: localhost:5001 # 👈 CHANGE THIS
```

**Options:**

- Docker Hub: `docker.io/yourusername`
- GitHub Packages: `ghcr.io/yourusername`
- AWS ECR: `123456789012.dkr.ecr.region.amazonaws.com`
- Azure ACR: `yourregistry.azurecr.io`
- Google GCR: `gcr.io/your-project-id`

### 2. Application Configuration

**Copy and customize the environment file:**

```bash
cp .env.template .env.production
```

**Edit `.env.production` with your values:**

```bash
APP_NAME=your-app-name     # 👈 CHANGE: Your application name
APP_PORT=8000              # 👈 CHANGE: Your app's internal port
HOST_PORT=8000             # 👈 CHANGE: External port to expose
CPU_LIMIT=0.5              # 👈 ADJUST: CPU cores limit
MEMORY_LIMIT=512M          # 👈 ADJUST: Memory limit
```

### 3. Dockerfile Customization

**⚠️ REQUIRED**: Update `docker/Dockerfile` to include your application:

```dockerfile
# Update metadata
LABEL maintainer="your-email@example.com"        # 👈 CHANGE
LABEL project="your-project-name"                # 👈 CHANGE
LABEL description="Your app description"         # 👈 CHANGE

# Add your application files
COPY . /app                                      # 👈 ADD: Copy your code

# Install your dependencies
RUN apk add --no-cache python3 py3-pip         # 👈 ADD: Your runtime
RUN pip install -r requirements.txt            # 👈 ADD: Your deps

# Update the command to run your app
CMD ["python3", "app.py"]                      # 👈 CHANGE: Your start command
```

### 4. Health Check Configuration

**Update the health check in `docker/Dockerfile` and `docker/docker-compose.yml`:**

```dockerfile
# In Dockerfile - update to match your health endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:8000/health || exit 1
```

```yaml
# In docker-compose.yml - update the health check
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:${APP_PORT}/health"] # 👈 CHANGE
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

## 🔧 Step-by-Step Setup Process

### Step 1: Repository Setup

1. **Fork or clone this repository**
2. **Create necessary secrets in GitHub repository settings:**
   - Go to Settings → Secrets and variables → Actions
   - Add repository secrets if using private registries:
     ```
     DOCKER_USERNAME=your-username
     DOCKER_PASSWORD=your-password-or-token
     ```

### Step 2: Runner Configuration

**For Self-Hosted Runners (recommended for production):**

1. Configure a self-hosted runner in GitHub repository settings
2. Install Docker on the runner machine
3. Ensure the runner has access to your container registry

**For GitHub-Hosted Runners:**

Update `.github/workflows/ci.yml`:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest # 👈 CHANGE from [self-hosted]
```

### Step 3: Registry Authentication

**Add registry login to the CI workflow:**

```yaml
- name: Login to Container Registry
  run: |
    echo "${{ secrets.DOCKER_PASSWORD }}" | docker login $REGISTRY -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
```

### Step 4: Test Local Build

```bash
# Test build locally
docker build -f docker/Dockerfile -t test-app .

# Test run locally
docker run -d -p 8000:8000 --name test-container test-app

# Check if it's working
curl http://localhost:8000
# or
docker logs test-container

# Cleanup
docker stop test-container && docker rm test-container
```

### Step 5: Deploy

1. **Commit your changes:**

   ```bash
   git add .
   git commit -m "Configure deployment for my app"
   git push origin main
   ```

2. **Monitor the deployment:**
   - Go to GitHub → Actions tab
   - Watch the workflow execution
   - Check for any errors in the logs

## 🛠 Customization Options

### CI/CD Schedule Optimization

The current pipeline runs every 2 hours. Adjust in `.github/workflows/ci.yml`:

```yaml
schedule:
  - cron: "0 2 * * *" # 👈 CHANGE: Daily at 2 AM (recommended)
  # - cron: '0 */2 * * *'  # Current: Every 2 hours (might be excessive)
```

### Environment-Specific Configurations

Create additional environment files for different stages:

**Staging environment (`.env.staging`):**

```bash
APP_NAME=myapp-staging
APP_PORT=8000
HOST_PORT=8001          # Different port to avoid conflicts
CPU_LIMIT=0.25          # Lower resources for staging
MEMORY_LIMIT=256M
```

**Development environment (`.env.development`):**

```bash
APP_NAME=myapp-dev
APP_PORT=8000
HOST_PORT=8002
CPU_LIMIT=0.1
MEMORY_LIMIT=128M
```

### Multi-Environment Workflow

Create separate workflows for different environments:

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging
on:
  push:
    branches: [develop]
env:
  REGISTRY: your-registry.com
  ENVIRONMENT: staging
```

### Resource Scaling

Adjust resources based on your application needs:

**Light applications:**

```bash
CPU_LIMIT=0.1
MEMORY_LIMIT=128M
```

**Heavy applications:**

```bash
CPU_LIMIT=2.0
MEMORY_LIMIT=2G
```

## 🔍 Monitoring and Maintenance

### Health Check Endpoints

Implement health check endpoints in your application:

**Python Flask example:**

```python
@app.route('/health')
def health_check():
    return {'status': 'healthy', 'timestamp': datetime.now().isoformat()}
```

**Node.js Express example:**

```javascript
app.get("/health", (req, res) => {
  res.json({ status: "healthy", timestamp: new Date().toISOString() });
});
```

### Logging Configuration

The template includes structured logging. Monitor logs with:

```bash
# View real-time logs
docker compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml logs -f

# View specific service logs
docker logs your-app-name-prod -f
```

## 🐛 Troubleshooting Guide

### Common Setup Issues

**1. Registry Authentication Failed**

```bash
# Error: unauthorized: authentication required
# Solution: Check registry credentials
docker login your-registry.com
# Add secrets to GitHub repository settings
```

**2. Port Already in Use**

```bash
# Error: bind: address already in use
# Solution: Change HOST_PORT in .env.production
HOST_PORT=8001  # Use different port
```

**3. Health Check Failing**

```bash
# Error: health check failed
# Solutions:
# 1. Verify your app responds on the health endpoint
curl http://localhost:8000/health

# 2. Increase start_period in docker-compose.yml
start_period: 60s  # Give app more time to start

# 3. Check application logs
docker logs your-container-name
```

**4. Build Context Too Large**

```bash
# Error: build context is too large
# Solution: Update .dockerignore to exclude unnecessary files
echo "node_modules/" >> docker/.dockerignore
echo "*.log" >> docker/.dockerignore
```

**5. Resource Limits Exceeded**

```bash
# Error: container killed (OOMKilled)
# Solution: Increase memory limit
MEMORY_LIMIT=1G  # Increase from 512M
```

### CI/CD Pipeline Issues

**1. Workflow Permission Denied**

```yaml
# Add to .github/workflows/ci.yml if using GitHub Packages
permissions:
  contents: read
  packages: write
```

**2. Self-Hosted Runner Not Responding**

```bash
# Check runner status on the machine
sudo systemctl status actions.runner.your-repo.service
# Restart if needed
sudo systemctl restart actions.runner.your-repo.service
```

**3. Docker Compose Version Issues**

```bash
# Update docker-compose to latest version
pip install --upgrade docker-compose
# or
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

### Debugging Commands

**Check container status:**

```bash
docker compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml ps
```

**View container logs:**

```bash
docker compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml logs -f app
```

**Inspect container:**

```bash
docker inspect your-container-name
```

**Execute commands in container:**

```bash
docker exec -it your-container-name sh
```

**Check resource usage:**

```bash
docker stats your-container-name
```

## 📊 Performance Optimization

### Build Speed Optimization

**Enable BuildKit for faster builds:**

```bash
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1
```

**Use multi-stage builds efficiently:**

```dockerfile
# Use specific versions for cache consistency
FROM node:18.19-alpine AS builder
FROM nginx:1.25-alpine AS runtime
```

### Runtime Optimization

**Resource allocation tuning:**

```bash
# For CPU-intensive apps
CPU_LIMIT=1.0
MEMORY_LIMIT=512M

# For memory-intensive apps
CPU_LIMIT=0.5
MEMORY_LIMIT=2G

# For I/O-intensive apps
CPU_LIMIT=0.25
MEMORY_LIMIT=256M
```

**Enable swap accounting (Linux hosts):**

```bash
# Add to /etc/default/grub
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
sudo update-grub && sudo reboot
```

## 🔒 Security Best Practices

### Secrets Management

**Never commit secrets to git:**

```bash
# Use .env files (excluded in .gitignore)
echo "API_KEY=secret123" >> .env.production

# Use GitHub secrets for CI/CD
# Repository Settings → Secrets and variables → Actions
```

**Scan for secrets in commits:**

```bash
# Install git-secrets
git secrets --install
git secrets --register-aws
git secrets --scan
```

### Container Security

**Scan images for vulnerabilities:**

```bash
# Using Docker Scout
docker scout cves your-image:latest

# Using Trivy
trivy image your-image:latest
```

**Use minimal base images:**

```dockerfile
# Prefer distroless or minimal images
FROM gcr.io/distroless/static:nonroot
# or
FROM alpine:3.21.3
```

## 📈 Scaling Considerations

### Horizontal Scaling

**Multiple replicas with load balancer:**

```yaml
# docker-compose.yml
services:
  app:
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.role == worker
```

**Using Docker Swarm:**

```bash
# Initialize swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-compose.yml myapp
```

### Database Integration

**Add database service:**

```yaml
# docker-compose.yml
services:
  app:
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 📚 Additional Resources

### Documentation Links

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)

### Community Templates

- [Awesome Docker](https://github.com/veggiemonk/awesome-docker)
- [GitHub Actions Templates](https://github.com/actions/starter-workflows)

## 🆘 Getting Help

1. **Check the Issues**: Look through existing GitHub issues
2. **Create an Issue**: Use the issue template for bug reports
3. **Join Discussions**: Participate in GitHub Discussions
4. **Documentation**: Check official Docker and GitHub Actions docs

## 📄 License

[Add your license here - MIT, Apache 2.0, etc.]

---

## 🔧 Advanced Configuration

For advanced users who need additional customization options beyond the basic setup.
