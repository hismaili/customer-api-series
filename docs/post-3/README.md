# Containerizing and Deploying Spring Boot to Oracle Cloud: A Production Guide

*Part 3 of 9 in the series: "Production-Ready Spring Boot on OCI with Vault"*

![Docker and Cloud Deployment](https://images.unsplash.com/photo-1605745341112-85968b19335b?w=1200&h=400&fit=crop)

## What You'll Build

In this hands-on tutorial, you'll take the Customer Management API we built in Post 1 and deploy it to Oracle Cloud Infrastructure. By the end, your API will be running in a Docker container on OCI, accessible from anywhere in the world.

**What You'll Accomplish**:
- ✅ Containerize your Spring Boot application with Docker
- ✅ Push your container to a registry
- ✅ Deploy and run on OCI compute instance
- ✅ Configure networking and firewall rules
- ✅ Test your live API from the internet
- ✅ Set up automatic container restart on failures

**Time to complete**: 45 minutes  
**Difficulty**: Intermediate  
**Cost**: $0 (using OCI Free Tier)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Why Containers?](#why-containers)
3. [Installing Docker/Podman](#installing-docker-podman)
4. [Creating the Dockerfile](#creating-the-dockerfile)
5. [Building the Container Image](#building-the-container-image)
6. [Testing Locally](#testing-locally)
7. [Preparing Your OCI Instance](#preparing-your-oci-instance)
8. [Deploying to OCI](#deploying-to-oci)
9. [Configuring the Firewall](#configuring-the-firewall)
10. [Testing Your Live API](#testing-your-live-api)
11. [Container Management](#container-management)
12. [Troubleshooting](#troubleshooting)
13. [What's Next](#whats-next)

---

## Prerequisites

### From Previous Posts

**Required**:
- ✅ [Post 1: Customer Management API](https://link-to-post-1) - The Spring Boot application we're deploying
- ✅ [Post 2: OCI Infrastructure Setup](https://link-to-post-2) - OCI account, VCN, and compute instance

**What You Should Have**:
- Spring Boot Customer API source code
- OCI compute instance running Oracle Linux 8
- SSH access to your instance
- Public IP address of your instance

### Tools You'll Need

- **Docker Desktop** or **Podman** installed locally
- **Git** (to manage your code)
- **Maven** (should be installed from Post 1)
- **Text editor** or IDE

**Check Your Setup**:
```bash
# Verify Java
java -version  # Should show Java 17+

# Verify Maven
mvn -version   # Should show Maven 3.8+

# Verify Docker
docker --version  # Or: podman --version
```

---

## Why Containers?

Before we dive in, let's understand why containerization matters.

### The "Works on My Machine" Problem

Traditional deployment:
```
Developer's Laptop (Works!) 
    → Production Server (Fails!)
    
Why? Different:
- OS versions
- Installed libraries
- Java versions
- Configuration
- Environment variables
```

### The Container Solution

```
Developer's Laptop
    ↓
Container Image (Everything bundled!)
    ↓
Production Server (Works exactly the same!)
```

**What's in a Container?**
- Your application code
- Java runtime
- All dependencies
- Configuration files
- Operating system libraries

**Benefits**:
- 🔒 **Consistency**: Runs the same everywhere
- 🚀 **Fast startup**: Seconds, not minutes
- 📦 **Isolation**: No conflicts with other apps
- 🔄 **Easy rollback**: Just run previous version
- 📈 **Scalability**: Spin up multiple instances easily

---

## Installing Docker/Podman

You have two options: **Docker** (industry standard) or **Podman** (Docker-compatible, rootless).

### Option 1: Docker Desktop (Recommended for Beginners)

**macOS**:
1. Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
2. Install the `.dmg` file
3. Launch Docker Desktop
4. Verify: `docker --version`

**Windows**:
1. Enable WSL2 (Windows Subsystem for Linux)
2. Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
3. Install and restart
4. Verify: `docker --version`

**Linux (Ubuntu/Debian)**:
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (avoid sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
```

### Option 2: Podman (Alternative)

**macOS** (via Homebrew):
```bash
brew install podman
podman machine init
podman machine start
podman --version
```

**Linux (RHEL/Oracle Linux)**:
```bash
sudo yum install -y podman
podman --version
```

**Note**: Podman commands are identical to Docker. Just replace `docker` with `podman`.

For this tutorial, I'll use `docker`, but everything works with `podman` too.

---

## Creating the Dockerfile

A **Dockerfile** is a recipe for building your container image.

### Multi-Stage Build Strategy

We'll use a **multi-stage build** for efficiency:
- **Stage 1**: Build the application with Maven
- **Stage 2**: Run the application with a minimal JRE

This keeps the final image small and secure.

### Create the Dockerfile

In your project root (`customer-api/`), create a file named `Dockerfile`:

```dockerfile
# Stage 1: Build stage
FROM maven:3.9-eclipse-temurin-17-alpine AS build

# Set working directory
WORKDIR /app

# Copy pom.xml and download dependencies (cached layer)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build the application (skip tests for faster builds)
RUN mvn clean package -DskipTests

# Stage 2: Runtime stage
FROM eclipse-temurin:17-jre-alpine

# Create a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Set working directory
WORKDIR /app

# Copy the JAR from build stage
COPY --from=build /app/target/*.jar app.jar

# Change ownership to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose the application port
EXPOSE 8080

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Understanding the Dockerfile

**Multi-stage benefits**:
- Build stage: ~800 MB (Maven + dependencies)
- Runtime stage: ~200 MB (JRE only)
- **Savings**: 75% smaller image!

**Security features**:
- Uses official Eclipse Temurin images (trusted)
- Runs as non-root user (`appuser`)
- Minimal base image (Alpine Linux)

**Best practices**:
- Layers cached for faster rebuilds
- Dependencies downloaded separately
- Health check for monitoring

### Add .dockerignore

Create `.dockerignore` in project root to exclude unnecessary files:

```
# Build output
target/
*.class

# IDE files
.idea/
.vscode/
*.iml

# OS files
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Logs
*.log

# Maven wrapper (optional - comment out if you want to include it)
.mvn/
mvnw
mvnw.cmd
```

This makes builds faster and images smaller.

---

## Building the Container Image

### Build the Image Locally

From your project root:

```bash
# Build the image
docker build -t customer-api:1.0.0 .

# This will take 2-5 minutes on first build
# Subsequent builds are much faster (cached layers)
```

**Explanation**:
- `-t customer-api:1.0.0`: Tag (name) the image
- `.`: Use current directory (where Dockerfile is)

**Expected output**:
```
[+] Building 125.3s (15/15) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [build 1/6] FROM maven:3.9-eclipse-temurin-17-alpine
 => [build 2/6] WORKDIR /app
 => [build 3/6] COPY pom.xml .
 => [build 4/6] RUN mvn dependency:go-offline -B
 => [build 5/6] COPY src ./src
 => [build 6/6] RUN mvn clean package -DskipTests
 => [stage-1 1/5] FROM eclipse-temurin:17-jre-alpine
 => [stage-1 2/5] RUN addgroup -S appgroup && adduser -S appuser -G appgroup
 => [stage-1 3/5] WORKDIR /app
 => [stage-1 4/5] COPY --from=build /app/target/*.jar app.jar
 => [stage-1 5/5] RUN chown -R appuser:appgroup /app
 => exporting to image
 => => exporting layers
 => => writing image sha256:abc123...
 => => naming to docker.io/library/customer-api:1.0.0
```

### Verify the Image

```bash
# List images
docker images

# Should show:
# REPOSITORY      TAG       IMAGE ID       CREATED         SIZE
# customer-api    1.0.0     abc123def456   2 minutes ago   215MB
```

---

## Testing Locally

Before deploying to OCI, let's test the container locally.

### Run the Container

```bash
# Run in detached mode (-d) with port mapping
docker run -d \
  --name customer-api-test \
  -p 8080:8080 \
  customer-api:1.0.0

# Check if it's running
docker ps
```

**Expected output**:
```
CONTAINER ID   IMAGE               COMMAND           STATUS         PORTS
abc123456789   customer-api:1.0.0  "java -jar app.jar"  Up 10 seconds  0.0.0.0:8080->8080/tcp
```

### Test the API

Wait ~30 seconds for the application to start, then test:

```bash
# Health check
curl http://localhost:8080/actuator/health

# Should return:
# {"status":"UP"}

# Create a customer
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "companyName": "Test Corp",
    "contactName": "Test User",
    "phone": "+1-555-0100"
  }'

# Get all customers
curl http://localhost:8080/api/v1/customers
```

**Success!** Your containerized API works locally.

### View Logs

```bash
# Follow logs in real-time
docker logs -f customer-api-test

# Press Ctrl+C to stop following
```

### Stop and Remove the Test Container

```bash
# Stop the container
docker stop customer-api-test

# Remove the container
docker rm customer-api-test
```

---

## Preparing Your OCI Instance

Now let's prepare the OCI instance to run our container.

### SSH into Your Instance

```bash
# Use your instance's public IP from Post 2
ssh -i ~/.ssh/oci_key opc@144.24.XXX.XXX
```

### Install Container Runtime

Oracle Linux 8 comes with Podman pre-installed, but let's verify and update:

```bash
# Check if podman is installed
podman --version

# Update system packages
sudo yum update -y

# Install additional tools
sudo yum install -y git wget curl jq

# Verify podman
podman --version
# Should show: podman version 4.x.x or higher
```

**Why Podman on OCI?**
- Pre-installed on Oracle Linux
- Rootless by default (more secure)
- Docker-compatible commands
- No daemon required

### Configure Firewall (on the Instance)

Oracle Linux has a local firewall (firewalld) in addition to OCI security lists:

```bash
# Check firewall status
sudo firewall-cmd --state

# Add rule for port 8080
sudo firewall-cmd --permanent --add-port=8080/tcp

# Reload firewall
sudo firewall-cmd --reload

# Verify the rule
sudo firewall-cmd --list-ports
# Should show: 8080/tcp
```

---

## Deploying to OCI

There are two ways to get your image to OCI:

### Option 1: Build Directly on OCI (Recommended)

**Transfer your code to OCI**:

```bash
# On your local machine, from project directory
# Create a tarball of your project
tar -czf customer-api.tar.gz \
  --exclude=target \
  --exclude=.git \
  --exclude=.idea \
  .

# Copy to OCI instance (replace with your IP)
scp -i ~/.ssh/oci_key customer-api.tar.gz opc@144.24.XXX.XXX:~/

# SSH into the instance
ssh -i ~/.ssh/oci_key opc@144.24.XXX.XXX

# Extract the project
tar -xzf customer-api.tar.gz -C ~/customer-api
cd ~/customer-api

# Build the image on OCI
podman build -t customer-api:1.0.0 .
```

**Advantages**:
- No registry needed
- Faster for small teams
- Simpler setup

### Option 2: Use a Container Registry (Production Approach)

**Push to Docker Hub**:

```bash
# On your local machine

# Login to Docker Hub
docker login

# Tag image with your Docker Hub username
docker tag customer-api:1.0.0 yourusername/customer-api:1.0.0

# Push to Docker Hub
docker push yourusername/customer-api:1.0.0

# On OCI instance, pull the image
podman pull docker.io/yourusername/customer-api:1.0.0
```

**Or use OCI Container Registry** (Oracle's registry):

```bash
# Create a repository in OCI Console:
# Developer Services → Container Registry → Create Repository
# Name: customer-api
# Access: Public (for learning)

# Tag and push (on local machine)
docker tag customer-api:1.0.0 \
  <region>.ocir.io/<tenancy-namespace>/customer-api:1.0.0

docker push <region>.ocir.io/<tenancy-namespace>/customer-api:1.0.0

# Pull on OCI instance
podman pull <region>.ocir.io/<tenancy-namespace>/customer-api:1.0.0
```

For this tutorial, we'll use **Option 1** (build directly on OCI).

---

## Running the Container on OCI

### Start the Container

SSH into your OCI instance and run:

```bash
# Run the container
podman run -d \
  --name customer-api \
  --restart unless-stopped \
  -p 8080:8080 \
  customer-api:1.0.0

# Verify it's running
podman ps
```

**Flags explained**:
- `-d`: Detached mode (background)
- `--name customer-api`: Container name
- `--restart unless-stopped`: Auto-restart on failures
- `-p 8080:8080`: Map host port 8080 to container port 8080

### Check Container Health

```bash
# View logs
podman logs customer-api

# Follow logs in real-time
podman logs -f customer-api

# Check health status
podman inspect customer-api | grep -A 5 "Health"

# Test locally from the instance
curl http://localhost:8080/actuator/health
```

**Expected log output**:
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.1)

2026-01-12 10:30:15.123  INFO 1 --- [main] c.e.c.CustomerApiApplication : Starting CustomerApiApplication
2026-01-12 10:30:18.456  INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer : Tomcat started on port(s): 8080 (http)
2026-01-12 10:30:18.789  INFO 1 --- [main] c.e.c.CustomerApiApplication : Started CustomerApiApplication in 4.567 seconds
```

---

## Configuring the Firewall

Your container is running, but it's not accessible from the internet yet. We need to update the OCI security list.

### Update Security List

1. **Go to OCI Console** → **Networking** → **Virtual Cloud Networks**
2. Click your VCN (`customer-api-vcn`)
3. Click **Security Lists** (left menu)
4. Click **Default Security List for customer-api-vcn**
5. Click **Add Ingress Rules**
6. Configure:
   - **Source Type**: CIDR
   - **Source CIDR**: `0.0.0.0/0` (allow from anywhere)
   - **IP Protocol**: TCP
   - **Destination Port Range**: `8080`
   - **Description**: `Spring Boot Customer API`
7. Click **Add Ingress Rules**

**Security Note**: In production, restrict `0.0.0.0/0` to specific IP ranges (your office, your CDN, etc.).

### Verify the Rule

After adding the rule, verify it appears in the Ingress Rules list:

```
Source CIDR: 0.0.0.0/0
IP Protocol: TCP
Destination Port Range: 8080
Description: Spring Boot Customer API
```

---

## Testing Your Live API

Now the moment of truth - access your API from anywhere!

### Get Your Public IP

From OCI Console:
1. **Compute** → **Instances**
2. Click your instance
3. Copy the **Public IP Address**: `144.24.XXX.XXX`

### Test from Your Local Machine

```bash
# Replace with your actual public IP
export API_URL=http://144.24.XXX.XXX:8080

# Health check
curl $API_URL/actuator/health

# Create a customer
curl -X POST $API_URL/api/v1/customers \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@acmecorp.com",
    "companyName": "Acme Corporation",
    "contactName": "John Doe",
    "phone": "+1-555-0123"
  }'

# Get all customers
curl $API_URL/api/v1/customers

# Get customer by ID
curl $API_URL/api/v1/customers/1

# Create an invoice
curl -X POST $API_URL/api/v1/invoices \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1,
    "invoiceNumber": "INV-2026-001",
    "amount": 1500.00,
    "currency": "USD",
    "dueAt": "2026-02-15"
  }'

# Get customer invoices
curl $API_URL/api/v1/invoices/customer/1
```

### Test from Browser

Open your browser and visit:

```
http://144.24.XXX.XXX:8080/api/v1/customers
```

You should see a JSON response with your customers!

**🎉 Congratulations!** Your API is live on the internet!

---

## Container Management

### Essential Commands

```bash
# View running containers
podman ps

# View all containers (including stopped)
podman ps -a

# Stop the container
podman stop customer-api

# Start the container
podman start customer-api

# Restart the container
podman restart customer-api

# View logs
podman logs customer-api

# Follow logs in real-time
podman logs -f customer-api

# View resource usage
podman stats customer-api

# Execute commands in running container
podman exec -it customer-api /bin/sh

# Remove the container (must be stopped first)
podman stop customer-api
podman rm customer-api

# Remove the image
podman rmi customer-api:1.0.0
```

### Auto-Start on Boot

Make the container start automatically when the instance reboots:

```bash
# Generate systemd service file
podman generate systemd --new --name customer-api \
  --files

# Move to systemd directory
sudo mv container-customer-api.service \
  /etc/systemd/system/

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable container-customer-api.service
sudo systemctl start container-customer-api.service

# Check status
sudo systemctl status container-customer-api.service
```

Now your container will automatically start after instance reboots!

### Update Deployment

When you have a new version:

```bash
# Stop and remove old container
podman stop customer-api
podman rm customer-api

# Build new version
podman build -t customer-api:1.0.1 .

# Run new version
podman run -d \
  --name customer-api \
  --restart unless-stopped \
  -p 8080:8080 \
  customer-api:1.0.1
```

---

## Troubleshooting

### Container Won't Start

**Check logs**:
```bash
podman logs customer-api
```

**Common issues**:
- Port 8080 already in use
- Insufficient memory
- Java version mismatch

**Solution**:
```bash
# Check what's using port 8080
sudo lsof -i :8080

# Check available memory
free -h

# If memory is low, stop other services
```

### Can't Access API from Internet

**Checklist**:
1. ✅ Container is running: `podman ps`
2. ✅ Container is healthy: `podman logs customer-api`
3. ✅ Local test works: `curl http://localhost:8080/actuator/health`
4. ✅ Firewalld allows 8080: `sudo firewall-cmd --list-ports`
5. ✅ Security list has ingress rule for port 8080
6. ✅ Using correct public IP

**Test from instance**:
```bash
# This should work from the instance itself
curl http://localhost:8080/actuator/health

# If local works but external doesn't, it's a firewall issue
```

### Application Returns 500 Errors

**Check application logs**:
```bash
podman logs customer-api | grep ERROR
```

**Common causes**:
- Database connection issues (we're using H2, so shouldn't happen)
- Invalid input data
- Missing environment variables

### Container Keeps Restarting

```bash
# Check restart count
podman ps -a

# View last 50 log lines
podman logs --tail 50 customer-api

# Common causes:
# - Application crashes immediately
# - Port conflict
# - Resource limits exceeded
```

### Build Fails on OCI Instance

**Out of memory during build**:
```bash
# Check available memory
free -h

# If low, create swap space
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make swap permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Performance Optimization

### Monitor Resource Usage

```bash
# Real-time container stats
podman stats customer-api

# Expected output:
# CONTAINER      CPU %  MEM USAGE / LIMIT  MEM %  NET I/O     BLOCK I/O
# customer-api   0.5%   250MB / 1GB       25%    1.2kB/890B  0B/0B
```

### Java Heap Size Tuning

By default, Java uses 25% of available memory. For better performance:

```bash
# Stop current container
podman stop customer-api
podman rm customer-api

# Run with custom JVM options
podman run -d \
  --name customer-api \
  --restart unless-stopped \
  -p 8080:8080 \
  -e JAVA_OPTS="-Xms256m -Xmx512m" \
  customer-api:1.0.0
```

**Explanation**:
- `-Xms256m`: Initial heap size (256 MB)
- `-Xmx512m`: Maximum heap size (512 MB)

### Update Dockerfile for Production

Modify your `Dockerfile` to accept JVM options:

```dockerfile
# In the ENTRYPOINT line, change:
ENTRYPOINT ["java", "-jar", "app.jar"]

# To:
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

Rebuild the image after this change.

---

## What's Next

Incredible work! You've successfully deployed a containerized Spring Boot application to Oracle Cloud Infrastructure. Let's recap what you've achieved:

✅ Created a production-ready Dockerfile with multi-stage builds  
✅ Built and tested a Docker image locally  
✅ Deployed the container to OCI compute instance  
✅ Configured networking and firewall rules  
✅ Made your API accessible from the internet  
✅ Set up auto-restart and systemd integration  

### Current Architecture

```
Internet
    ↓
OCI Security List (Firewall: Allow 8080)
    ↓
OCI Compute Instance (144.24.XXX.XXX)
    ├── Firewalld (Allow 8080)
    └── Podman Container
        └── Spring Boot API (Port 8080)
            └── H2 Database (In-memory)
```

### Coming Up in This Series

**Post 4: Secrets Management with HashiCorp Vault - Local Setup**  
Right now, our application uses hardcoded configuration and an in-memory database. In the next post, we'll introduce HashiCorp Vault for secure secrets management. You'll learn:
- Vault fundamentals and architecture
- Running Vault locally
- Storing and retrieving secrets
- Integrating Spring Boot with Vault
- Dynamic database credentials

**Post 5: Zero-Trust Authentication - OCI Instance Principals with Vault**  
We'll deploy Vault to OCI and configure it to use OCI Instance Principals for authentication - completely eliminating the need for passwords or API keys!

---

## Production Considerations

Before going to production, consider these improvements:

### 1. Use a Persistent Database

Replace H2 with PostgreSQL or MySQL:

```bash
# Run PostgreSQL container
podman run -d \
  --name customer-api-db \
  -e POSTGRES_DB=customerdb \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=secure_password \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15-alpine

# Update application.properties
# We'll cover this in detail in Post 4
```

### 2. Add HTTPS/TLS

Use a reverse proxy like Caddy or Nginx:

```bash
# Run Caddy as reverse proxy
podman run -d \
  --name caddy \
  -p 80:80 \
  -p 443:443 \
  -v caddy-data:/data \
  -v caddy-config:/config \
  caddy:latest \
  caddy reverse-proxy --from yourdomain.com --to localhost:8080
```

### 3. Implement Monitoring

```bash
# Add Prometheus metrics to Spring Boot
# Add to pom.xml:
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

# Expose metrics endpoint
# In application.properties:
management.endpoints.web.exposure.include=health,info,prometheus
```

### 4. Set Up Log Aggregation

```bash
# Configure JSON logging
# In application.properties:
logging.pattern.console=%d{ISO8601} %-5level [%thread] %logger{36} - %msg%n

# Ship logs to external service (CloudWatch, Elasticsearch, etc.)
```

### 5. Implement CI/CD

In Post 9, we'll set up GitHub Actions to automatically:
- Build the Docker image
- Push to registry
- Deploy to OCI
- Run tests
- Rollback on failures

---

## Complete Deployment Checklist

Use this checklist for future deployments:

**Local Development**:
- [ ] Application builds successfully (`mvn clean package`)
- [ ] All tests pass (`mvn test`)
- [ ] Docker image builds without errors
- [ ] Container runs locally
- [ ] API endpoints respond correctly

**OCI Preparation**:
- [ ] OCI instance is running
- [ ] SSH access works
- [ ] Podman is installed and updated
- [ ] Firewalld configured for port 8080
- [ ] Security list has ingress rule for port 8080

**Deployment**:
- [ ] Code transferred to OCI or image pulled from registry
- [ ] Image built successfully on OCI
- [ ] Container starts without errors
- [ ] Health check passes
- [ ] Logs show no errors

**Testing**:
- [ ] Local test from instance works (`curl localhost:8080`)
- [ ] External test from internet works (`curl PUBLIC_IP:8080`)
- [ ] All API endpoints function correctly
- [ ] Performance is acceptable

**Production Readiness**:
- [ ] Auto-restart configured
- [ ] Systemd service enabled
- [ ] Monitoring in place
- [ ] Backups configured (for persistent data)
- [ ] Documentation updated

---

## Quick Reference Commands

```bash
# Build image
podman build -t customer-api:1.0.0 .

# Run container
podman run -d --name customer-api --restart unless-stopped -p 8080:8080 customer-api:1.0.0

# View logs
podman logs -f customer-api

# Restart container
podman restart customer-api

# Stop and remove
podman stop customer-api && podman rm customer-api

# Check resource usage
podman stats customer-api

# Execute shell in container
podman exec -it customer-api /bin/sh

# Update deployment
podman stop customer-api
podman rm customer-api
podman build -t customer-api:1.0.1 .
podman run -d --name customer-api --restart unless-stopped -p 8080:8080 customer-api:1.0.1
```

---

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Podman Documentation](https://docs.podman.io/)
- [Spring Boot Docker Guide](https://spring.io/guides/gs/spring-boot-docker/)
- [OCI Container Instances](https://docs.oracle.com/iaas/Content/container-instances/home.htm)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

---

## Connect and Share

🎉 **Your API is live!** Share your achievement:

- Tweet your deployment: `@YourHandle #SpringBoot #OCI #Docker #CloudNative`
- Share on LinkedIn with your public IP
- Star the GitHub repo (link below)
- Leave a comment with your questions or success stories!

---

## Source Code Repository

All code for this series is available on GitHub:

📦 **[github.com/your-username/customer-api-series](https://github.com/your-username/customer-api-series)**

**Branch for this post**: `post-3-containerize-deploy`

**What's included**:
- Complete Spring Boot application
- Production-ready Dockerfile
- Deployment scripts
- OCI setup instructions
- Troubleshooting guide

```bash
# Clone the repository
git clone https://github.com/your-username/customer-api-series.git
cd customer-api-series

# Checkout this post's branch
git checkout post-3-containerize-deploy

# Follow the README for setup
```

---

## Homework Challenge

Want to level up? Try these exercises:

### Beginner Level
1. **Add a custom banner**: Create `src/main/resources/banner.txt` with ASCII art
2. **Customize the port**: Change from 8080 to 9000 (update Dockerfile and security lists)
3. **Add environment variables**: Pass `SPRING_PROFILES_ACTIVE=prod` to the container

### Intermediate Level
1. **Multi-container setup**: Run PostgreSQL in a separate container
2. **Volume persistence**: Mount a volume for logs: `-v /var/log/customer-api:/app/logs`
3. **Resource limits**: Set memory limits: `--memory=512m --cpus=1.0`

### Advanced Level
1. **Blue-Green Deployment**: Run two versions simultaneously on different ports
2. **Container networking**: Create a custom Podman network for multiple services
3. **Health probes**: Implement custom health checks with Spring Boot Actuator

Share your solutions in the comments!

---

## Common Questions

**Q: Why use Podman instead of Docker on OCI?**  
A: Podman is pre-installed on Oracle Linux, runs rootless by default (better security), and doesn't require a daemon. However, Docker works too if you prefer it.

**Q: How much does this cost on OCI?**  
A: $0! The VM.Standard.E2.1.Micro instance is part of the Always Free tier. You can run this indefinitely without charges.

**Q: Can I use this for production?**  
A: This setup is production-ready for small to medium applications. For high-traffic production, add:
- Load balancer
- Multiple instances
- External database (with backups)
- HTTPS/TLS
- Monitoring and alerting

**Q: What if I want to use my own domain name?**  
A: You'll need to:
1. Register a domain (Namecheap, Google Domains, etc.)
2. Point an A record to your OCI public IP
3. Set up a reverse proxy (Caddy/Nginx) with TLS
4. Update security lists for ports 80/443

We'll cover this in a future bonus post!

**Q: How do I scale to handle more traffic?**  
A: Several approaches:
- Vertical scaling: Use a bigger compute shape
- Horizontal scaling: Run multiple instances behind a load balancer
- Container orchestration: Use Kubernetes (covered in Post 8)

**Q: Can I deploy multiple applications on one instance?**  
A: Yes! Run containers on different ports:
```bash
podman run -d -p 8080:8080 customer-api:1.0.0
podman run -d -p 8081:8080 another-api:1.0.0
podman run -d -p 8082:8080 third-api:1.0.0
```
Just add security list rules for each port.

---

## Troubleshooting Deep Dive

### Issue: "Port is already allocated"

**Error message**:
```
Error: cannot listen on the TCP port: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**Solution**:
```bash
# Find what's using port 8080
sudo lsof -i :8080
# Or
sudo netstat -tulpn | grep 8080

# Kill the process
sudo kill -9 <PID>

# Or use a different port
podman run -d -p 9090:8080 customer-api:1.0.0
```

### Issue: Container exits immediately

**Check exit code**:
```bash
podman ps -a

# Look at STATUS column:
# "Exited (0)" = clean shutdown
# "Exited (1)" = error
# "Exited (137)" = killed (OOM - out of memory)
```

**Check logs**:
```bash
podman logs customer-api

# Common causes:
# - Java out of memory
# - Missing environment variables
# - Port conflict
# - Application startup error
```

**Solution for OOM**:
```bash
# Add memory limit
podman run -d \
  --memory=1g \
  --memory-swap=1g \
  -e JAVA_OPTS="-Xmx768m" \
  -p 8080:8080 \
  customer-api:1.0.0
```

### Issue: Slow build times

**Problem**: Maven downloads dependencies every time

**Solution**: Use a build cache

```bash
# Create a volume for Maven cache
podman volume create maven-cache

# Build with cache
podman build \
  -v maven-cache:/root/.m2 \
  -t customer-api:1.0.0 .
```

**Or optimize your Dockerfile** (already done in our example):
- Copy `pom.xml` first
- Run `mvn dependency:go-offline`
- This caches dependencies in a separate layer

### Issue: Can't access from browser but curl works

**Cause**: Browser might be caching or using HTTPS

**Solution**:
1. Use incognito/private mode
2. Force HTTP: `http://144.24.XXX.XXX:8080` (not `https://`)
3. Clear browser cache
4. Try a different browser

### Issue: Application works then stops after a few hours

**Causes**:
- Out of memory (check with `podman logs`)
- Instance was stopped/rebooted
- Container wasn't set to auto-restart

**Prevention**:
```bash
# Always use --restart flag
podman run -d --restart unless-stopped ...

# Or set up systemd service (shown earlier)
```

---

## Security Hardening

### 1. Run as Non-Root User

Our Dockerfile already does this:
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**Verify**:
```bash
podman exec customer-api whoami
# Should output: appuser (not root)
```

### 2. Use Read-Only Root Filesystem

```bash
podman run -d \
  --read-only \
  --tmpfs /tmp \
  -p 8080:8080 \
  customer-api:1.0.0
```

### 3. Drop Capabilities

```bash
podman run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  -p 8080:8080 \
  customer-api:1.0.0
```

### 4. Scan Image for Vulnerabilities

```bash
# Using Trivy
sudo yum install -y trivy
trivy image customer-api:1.0.0

# Using Podman
podman scan customer-api:1.0.0
```

### 5. Limit Resource Usage

```bash
podman run -d \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --pids-limit=100 \
  -p 8080:8080 \
  customer-api:1.0.0
```

---

## Monitoring and Observability

### Basic Health Monitoring

```bash
#!/bin/bash
# Save as monitor.sh

while true; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health)
    
    if [ "$STATUS" -eq 200 ]; then
        echo "$(date): ✓ Application is healthy"
    else
        echo "$(date): ✗ Application is down (HTTP $STATUS)"
        # Send alert (email, Slack, etc.)
    fi
    
    sleep 60  # Check every minute
done
```

Run in background:
```bash
chmod +x monitor.sh
nohup ./monitor.sh > monitor.log 2>&1 &
```

### Container Stats Logging

```bash
#!/bin/bash
# Save as stats-logger.sh

while true; do
    podman stats --no-stream customer-api >> /var/log/container-stats.log
    sleep 300  # Log every 5 minutes
done
```

### Set Up Alerts

```bash
#!/bin/bash
# Save as alert.sh

WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
MAX_MEMORY=512

MEMORY_MB=$(podman stats --no-stream customer-api | tail -1 | awk '{print $4}' | sed 's/MiB//')

if (( $(echo "$MEMORY_MB > $MAX_MEMORY" | bc -l) )); then
    curl -X POST $WEBHOOK_URL \
        -H 'Content-Type: application/json' \
        -d "{\"text\":\"⚠️ Customer API memory usage high: ${MEMORY_MB}MB\"}"
fi
```

---

## Performance Benchmarking

### Test API Performance

Install Apache Bench:
```bash
sudo yum install -y httpd-tools
```

**Benchmark the health endpoint**:
```bash
ab -n 1000 -c 10 http://localhost:8080/actuator/health

# -n 1000: 1000 requests total
# -c 10: 10 concurrent requests
```

**Expected results**:
```
Requests per second:    500-1000 [#/sec]
Time per request:       10-20 ms [mean]
```

**Benchmark customer creation**:
```bash
ab -n 100 -c 5 -p customer.json -T application/json \
   http://localhost:8080/api/v1/customers
```

Where `customer.json` contains:
```json
{
  "email": "test@example.com",
  "companyName": "Test Corp",
  "contactName": "Test User",
  "phone": "+1-555-0100"
}
```

---

## Next Steps Roadmap

Here's your learning path for the rest of the series:

**✅ Completed**:
- Post 1: Built a Spring Boot REST API
- Post 2: Set up OCI infrastructure
- Post 3: Containerized and deployed to OCI ← **You are here**

**📚 Coming Next**:
- **Post 4**: Vault fundamentals and local integration
- **Post 5**: OCI Instance Principals with Vault (zero-secret auth)
- **Post 6**: Production deployment with Vault on OCI
- **Post 7**: Secret rotation and resilience patterns
- **Post 8**: High availability and scaling
- **Post 9**: CI/CD with GitHub Actions

**🎯 Final Architecture** (Post 9):
```
GitHub (Code) → GitHub Actions (CI/CD)
    ↓
OCI Container Registry (Images)
    ↓
OCI Compute Instances (Multiple)
    ↓
Load Balancer
    ↓
Internet Users
```

---

## Feedback and Community

**Found this helpful?**
- ⭐ Star the [GitHub repository](#)
- 💬 Join our [Discord community](#) for discussions
- 🐦 Follow me on Twitter [@YourHandle](#)
- 📧 Subscribe to the newsletter for updates

**Have questions or issues?**
- Comment below with your specific problem
- Include error messages and logs
- Share your OCI setup details
- Others can learn from your questions too!

**Want to contribute?**
- Submit pull requests to the GitHub repo
- Share your deployment experiences
- Suggest improvements or topics
- Write guest posts about your implementation

---

## Key Takeaways

🔑 **Containerization provides consistency** across environments  
🔑 **Multi-stage builds** optimize image size and security  
🔑 **Podman is Docker-compatible** and rootless by default  
🔑 **OCI Free Tier** is generous enough for real projects  
🔑 **Security lists + firewalld** provide defense in depth  
🔑 **Auto-restart policies** ensure high availability  
🔑 **Systemd integration** makes containers first-class services  

---

*Previous: [Post 2: Understanding OCI - Cloud Infrastructure Fundamentals](#) ←*

*Next: [Post 4: Secrets Management with HashiCorp Vault - Local Setup](#) →*

---

## Tags

#Docker #Podman #SpringBoot #OCI #Containers #CloudDeployment #DevOps #OracleCloud #Microservices #CloudNative #Tutorial #Java #Backend