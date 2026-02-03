# Production Deployment: Building a Complete, Secure Stack on Oracle Cloud

*Part 6 of 9 in the series: "Production-Ready Spring Boot on OCI with Vault"*

![Production Infrastructure](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&h=400&fit=crop)

## What You'll Build

This is where everything comes together into a production-ready deployment. You'll take all the pieces from previous posts and assemble them into a complete, secure, scalable application stack running on OCI.

**By the end of this post, you'll have**:
- ✅ Multi-instance architecture with proper networking
- ✅ PostgreSQL database with automated backups
- ✅ Vault managing all secrets securely
- ✅ Load balancer distributing traffic
- ✅ Automated deployment scripts
- ✅ Monitoring and health checks
- ✅ Complete disaster recovery plan
- ✅ Production-grade security hardening

**Time to complete**: 120 minutes  
**Difficulty**: Advanced  
**Cost**: ~$10-20/month (can be reduced with free tier)

---

## Table of Contents

1. [Production Architecture Overview](#production-architecture-overview)
2. [Prerequisites and Planning](#prerequisites-and-planning)
3. [Infrastructure Setup](#infrastructure-setup)
4. [Database Deployment](#database-deployment)
5. [Vault Cluster Setup](#vault-cluster-setup)
6. [Application Deployment](#application-deployment)
7. [Load Balancer Configuration](#load-balancer-configuration)
8. [Monitoring and Alerting](#monitoring-and-alerting)
9. [Backup and Disaster Recovery](#backup-and-disaster-recovery)
10. [Security Hardening](#security-hardening)
11. [Deployment Automation](#deployment-automation)
12. [Testing the Complete Stack](#testing-the-complete-stack)
13. [Operational Runbooks](#operational-runbooks)
14. [What's Next](#whats-next)

---

## Prerequisites

### From Previous Posts

**Required Knowledge**:
- ✅ [Post 2: OCI Infrastructure](link) - VCN, compute, networking
- ✅ [Post 3: Containerization](link) - Docker/Podman deployment
- ✅ [Post 4: Vault Basics](link) - Vault concepts and policies
- ✅ [Post 5: OCI Auth](link) - Instance principals, zero-trust

**You should have**:
- OCI account with available quota
- SSH key pair configured
- Spring Boot application containerized
- Understanding of Vault operations
- Basic database administration knowledge

### Resources Required

**Compute Instances** (minimum):
- 1x Database server (2 OCPU, 16GB RAM recommended)
- 1x Vault server (1 OCPU, 4GB RAM)
- 2x Application servers (1 OCPU, 4GB RAM each)
- Total: 5 OCPUs, 28GB RAM

**Network**:
- VCN with public and private subnets
- Load balancer
- Service gateway for OCI services

**Storage**:
- Block volumes for database (100GB recommended)
- Block volumes for Vault storage (50GB)
- Object storage for backups

**Note**: We'll optimize for free tier where possible!

---

## Production Architecture Overview

### High-Level Architecture

```
                    Internet
                       │
                       ↓
              ┌─────────────────┐
              │  Load Balancer  │
              │   (Public IP)   │
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ↓             ↓             ↓
    ┌────────┐   ┌────────┐   ┌────────┐
    │ App-1  │   │ App-2  │   │ App-3  │
    │ :8080  │   │ :8080  │   │ :8080  │
    └───┬────┘   └───┬────┘   └───┬────┘
        │            │            │
        └────────────┼────────────┘
                     │
                     ↓
              ┌─────────────┐
              │   Vault     │
              │   :8200     │
              └──────┬──────┘
                     │
                     ↓
              ┌─────────────┐
              │ PostgreSQL  │
              │   :5432     │
              └─────────────┘
```

### Network Architecture

```
VCN: production-vcn (10.0.0.0/16)
│
├── Public Subnet (10.0.0.0/24)
│   ├── Load Balancer (10.0.0.5)
│   └── Bastion Host (10.0.0.10) [Optional]
│
├── Application Subnet (10.0.1.0/24) [Private]
│   ├── app-instance-1 (10.0.1.11)
│   ├── app-instance-2 (10.0.1.12)
│   └── app-instance-3 (10.0.1.13)
│
├── Database Subnet (10.0.2.0/24) [Private]
│   ├── vault-instance (10.0.2.10)
│   └── postgres-instance (10.0.2.20)
│
└── Gateways
    ├── Internet Gateway (for public subnet)
    ├── NAT Gateway (for private subnets)
    └── Service Gateway (for OCI services)
```

### Component Responsibilities

**Load Balancer**:
- SSL/TLS termination
- Health checks
- Traffic distribution
- Session persistence

**Application Instances**:
- Run Spring Boot containers
- Stateless design
- Auto-authenticate to Vault
- Connect to shared database

**Vault Instance**:
- Manage all secrets
- OCI authentication
- Audit logging
- Token issuance

**Database Instance**:
- PostgreSQL primary
- Automated backups
- Connection pooling
- Monitoring

---

## Infrastructure Setup

### Step 1: Create VCN with Multiple Subnets

**Using OCI CLI** (recommended for automation):

```bash
# Set variables
export COMPARTMENT_OCID="ocid1.compartment.oc1..YOUR_COMPARTMENT"
export VCN_CIDR="10.0.0.0/16"

# Create VCN
VCN_OCID=$(oci network vcn create \
  --compartment-id $COMPARTMENT_OCID \
  --cidr-block $VCN_CIDR \
  --display-name "production-vcn" \
  --dns-label "prodvcn" \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

echo "VCN OCID: $VCN_OCID"

# Create Internet Gateway
IGW_OCID=$(oci network internet-gateway create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --display-name "production-igw" \
  --is-enabled true \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Create NAT Gateway
NAT_OCID=$(oci network nat-gateway create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --display-name "production-nat" \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Create Service Gateway
SVC_OCID=$(oci network service-gateway create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --services '[{"serviceId": "all-phx-services"}]' \
  --display-name "production-svc-gw" \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)
```

**Or using OCI Console** (easier for beginners):

1. **☰ Menu** → **Networking** → **Virtual Cloud Networks**
2. Click **Start VCN Wizard**
3. Select **Create VCN with Internet Connectivity**
4. Configure:
   - **VCN Name**: `production-vcn`
   - **VCN CIDR**: `10.0.0.0/16`
   - **Public Subnet CIDR**: `10.0.0.0/24`
   - **Private Subnet CIDR**: `10.0.1.0/24`
5. Click **Next** → **Create**

### Step 2: Create Additional Subnets

We need separate subnets for applications and database:

```bash
# Application Subnet (Private)
APP_SUBNET_OCID=$(oci network subnet create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --cidr-block "10.0.1.0/24" \
  --display-name "app-subnet" \
  --dns-label "appsubnet" \
  --prohibit-public-ip-on-vnic true \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Database Subnet (Private)
DB_SUBNET_OCID=$(oci network subnet create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --cidr-block "10.0.2.0/24" \
  --display-name "db-subnet" \
  --dns-label "dbsubnet" \
  --prohibit-public-ip-on-vnic true \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)
```

### Step 3: Configure Route Tables

**Public Subnet Route Table**:
```bash
# Add route to Internet Gateway
oci network route-table update \
  --rt-id $PUBLIC_RT_OCID \
  --route-rules '[
    {
      "destination": "0.0.0.0/0",
      "destinationType": "CIDR_BLOCK",
      "networkEntityId": "'$IGW_OCID'"
    }
  ]' \
  --force
```

**Private Subnets Route Table**:
```bash
# Create route table for private subnets
PRIVATE_RT_OCID=$(oci network route-table create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --display-name "private-rt" \
  --route-rules '[
    {
      "destination": "0.0.0.0/0",
      "destinationType": "CIDR_BLOCK",
      "networkEntityId": "'$NAT_OCID'"
    },
    {
      "destination": "all-phx-services",
      "destinationType": "SERVICE_CIDR_BLOCK",
      "networkEntityId": "'$SVC_OCID'"
    }
  ]' \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Associate with subnets
oci network subnet update \
  --subnet-id $APP_SUBNET_OCID \
  --route-table-id $PRIVATE_RT_OCID \
  --force

oci network subnet update \
  --subnet-id $DB_SUBNET_OCID \
  --route-table-id $PRIVATE_RT_OCID \
  --force
```

### Step 4: Configure Security Lists

**Create separate security lists** for each tier:

```bash
# Application Security List
APP_SL_OCID=$(oci network security-list create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --display-name "app-security-list" \
  --ingress-security-rules '[
    {
      "source": "10.0.0.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 8080, "max": 8080}}
    }
  ]' \
  --egress-security-rules '[
    {
      "destination": "0.0.0.0/0",
      "protocol": "all"
    }
  ]' \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Database Security List
DB_SL_OCID=$(oci network security-list create \
  --compartment-id $COMPARTMENT_OCID \
  --vcn-id $VCN_OCID \
  --display-name "db-security-list" \
  --ingress-security-rules '[
    {
      "source": "10.0.1.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 5432, "max": 5432}}
    },
    {
      "source": "10.0.1.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 8200, "max": 8200}}
    }
  ]' \
  --egress-security-rules '[
    {
      "destination": "0.0.0.0/0",
      "protocol": "all"
    }
  ]' \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)
```

### Step 5: Create IAM Resources

**Dynamic Groups**:

```bash
# Create dynamic group for all production instances
oci iam dynamic-group create \
  --compartment-id $TENANCY_OCID \
  --name "production-instances" \
  --description "All production instances" \
  --matching-rule "instance.compartment.id = '$COMPARTMENT_OCID'"

# Create dynamic group for Vault instances
oci iam dynamic-group create \
  --compartment-id $TENANCY_OCID \
  --name "vault-instances" \
  --description "Vault server instances" \
  --matching-rule "instance.compartment.id = '$COMPARTMENT_OCID' && instance.displayName =~ 'vault'"

# Create dynamic group for app instances
oci iam dynamic-group create \
  --compartment-id $TENANCY_OCID \
  --name "app-instances" \
  --description "Application instances" \
  --matching-rule "instance.compartment.id = '$COMPARTMENT_OCID' && instance.displayName =~ 'app'"
```

**IAM Policies**:

```bash
# Create policies file
cat > production-policies.txt << 'EOF'
Allow dynamic-group vault-instances to {AUTHENTICATION_INSPECT} in tenancy
Allow dynamic-group vault-instances to {GROUP_MEMBERSHIP_INSPECT} in tenancy
Allow dynamic-group app-instances to read secret-family in compartment production
Allow dynamic-group app-instances to use instance-family in compartment production
Allow dynamic-group production-instances to use object-family in compartment production
EOF

# Apply policies (do this in OCI Console for now)
```

---

## Database Deployment

### Step 1: Launch Database Instance

```bash
# Create compute instance for PostgreSQL
DB_INSTANCE_OCID=$(oci compute instance launch \
  --compartment-id $COMPARTMENT_OCID \
  --availability-domain "YOUR-AD-1" \
  --shape "VM.Standard.E4.Flex" \
  --shape-config '{"ocpus": 2, "memoryInGBs": 16}' \
  --display-name "postgres-primary" \
  --subnet-id $DB_SUBNET_OCID \
  --image-id "ocid1.image.oc1.phx.aaaaaaaaxxx" \
  --ssh-authorized-keys-file ~/.ssh/oci_key.pub \
  --wait-for-state RUNNING \
  --query 'data.id' \
  --raw-output)

echo "Database Instance OCID: $DB_INSTANCE_OCID"
```

### Step 2: Create and Attach Block Volume

```bash
# Create block volume for database storage
DB_VOLUME_OCID=$(oci bv volume create \
  --compartment-id $COMPARTMENT_OCID \
  --availability-domain "YOUR-AD-1" \
  --display-name "postgres-data" \
  --size-in-gbs 100 \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Attach to instance
oci compute volume-attachment attach \
  --instance-id $DB_INSTANCE_OCID \
  --type "paravirtualized" \
  --volume-id $DB_VOLUME_OCID \
  --wait-for-state ATTACHED
```

### Step 3: Install and Configure PostgreSQL

SSH into the database instance:

```bash
# Get private IP
DB_PRIVATE_IP=$(oci compute instance get \
  --instance-id $DB_INSTANCE_OCID \
  --query 'data."primary-private-ip"' \
  --raw-output)

# SSH via bastion or configure VPN
ssh -i ~/.ssh/oci_key opc@$DB_PRIVATE_IP
```

**Install PostgreSQL**:

```bash
# Install PostgreSQL 15
sudo dnf module enable postgresql:15
sudo dnf install -y postgresql-server postgresql-contrib

# Initialize database
sudo /usr/bin/postgresql-setup --initdb

# Start and enable
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

**Configure PostgreSQL**:

```bash
# Edit postgresql.conf
sudo vi /var/lib/pgsql/data/postgresql.conf

# Update these settings:
listen_addresses = '*'
max_connections = 100
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 41943kB
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4

# Edit pg_hba.conf
sudo vi /var/lib/pgsql/data/pg_hba.conf

# Add this line (allow from app subnet):
host    all    all    10.0.1.0/24    scram-sha-256

# Restart PostgreSQL
sudo systemctl restart postgresql
```

**Create Database and User**:

```bash
# Switch to postgres user
sudo -u postgres psql

-- Create database
CREATE DATABASE customerdb;

-- Create user
CREATE USER apiuser WITH ENCRYPTED PASSWORD 'CHANGE_ME_STRONG_PASSWORD';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE customerdb TO apiuser;

-- Exit
\q
```

### Step 4: Store Database Credentials in Vault

```bash
# On Vault instance
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='your-root-token'

# Store production database credentials
vault kv put secret/customer-api/production/database \
  host="10.0.2.20" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="CHANGE_ME_STRONG_PASSWORD"

# Verify
vault kv get secret/customer-api/production/database
```

### Step 5: Configure Automated Backups

```bash
# Create backup script
sudo tee /usr/local/bin/postgres-backup.sh > /dev/null << 'EOF'
#!/bin/bash
set -e

# Configuration
BACKUP_DIR="/backup/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
BUCKET_NAME="customer-api-backups"

# Create backup
mkdir -p $BACKUP_DIR
sudo -u postgres pg_dumpall | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# Upload to Object Storage
oci os object put \
  --bucket-name $BUCKET_NAME \
  --file $BACKUP_DIR/backup_$DATE.sql.gz \
  --name postgres/backup_$DATE.sql.gz \
  --auth instance_principal

# Cleanup old local backups (keep last 7 days)
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: backup_$DATE.sql.gz"
EOF

sudo chmod +x /usr/local/bin/postgres-backup.sh

# Create cron job (daily at 2 AM)
echo "0 2 * * * /usr/local/bin/postgres-backup.sh >> /var/log/postgres-backup.log 2>&1" | sudo crontab -
```

---

## Vault Cluster Setup

For production, we'll set up Vault with persistent storage.

### Step 1: Launch Vault Instance

```bash
# Create Vault instance
VAULT_INSTANCE_OCID=$(oci compute instance launch \
  --compartment-id $COMPARTMENT_OCID \
  --availability-domain "YOUR-AD-1" \
  --shape "VM.Standard.E4.Flex" \
  --shape-config '{"ocpus": 1, "memoryInGBs": 4}' \
  --display-name "vault-primary" \
  --subnet-id $DB_SUBNET_OCID \
  --image-id "ocid1.image.oc1.phx.aaaaaaaaxxx" \
  --ssh-authorized-keys-file ~/.ssh/oci_key.pub \
  --wait-for-state RUNNING \
  --query 'data.id' \
  --raw-output)
```

### Step 2: Create Block Volume for Vault Storage

```bash
# Create volume
VAULT_VOLUME_OCID=$(oci bv volume create \
  --compartment-id $COMPARTMENT_OCID \
  --availability-domain "YOUR-AD-1" \
  --display-name "vault-storage" \
  --size-in-gbs 50 \
  --wait-for-state AVAILABLE \
  --query 'data.id' \
  --raw-output)

# Attach
oci compute volume-attachment attach \
  --instance-id $VAULT_INSTANCE_OCID \
  --type "paravirtualized" \
  --volume-id $VAULT_VOLUME_OCID \
  --wait-for-state ATTACHED
```

### Step 3: Configure Vault with File Storage

SSH into Vault instance and set up:

```bash
# Install Vault
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vault

# Mount block volume
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /opt/vault/data
sudo mount /dev/sdb /opt/vault/data
echo "/dev/sdb /opt/vault/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Set permissions
sudo chown -R vault:vault /opt/vault
sudo chmod 700 /opt/vault/data

# Create Vault configuration
sudo mkdir -p /etc/vault.d
sudo tee /etc/vault.d/vault.hcl > /dev/null << 'EOF'
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault.d/tls/vault-cert.pem"
  tls_key_file  = "/etc/vault.d/tls/vault-key.pem"
}

api_addr = "https://PRIVATE_IP:8200"
cluster_addr = "https://PRIVATE_IP:8201"
ui = true
log_level = "Info"
EOF

# Replace PRIVATE_IP
PRIVATE_IP=$(hostname -I | awk '{print $1}')
sudo sed -i "s/PRIVATE_IP/$PRIVATE_IP/g" /etc/vault.d/vault.hcl
```

### Step 4: Generate TLS Certificates

```bash
# Create TLS directory
sudo mkdir -p /etc/vault.d/tls
cd /etc/vault.d/tls

# Generate private key
sudo openssl genrsa -out vault-key.pem 2048

# Generate certificate
sudo openssl req -new -x509 \
  -key vault-key.pem \
  -out vault-cert.pem \
  -days 365 \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=vault.production.local"

# Set permissions
sudo chown vault:vault /etc/vault.d/tls/*
sudo chmod 600 /etc/vault.d/tls/vault-key.pem
sudo chmod 644 /etc/vault.d/tls/vault-cert.pem
```

### Step 5: Initialize and Configure Vault

```bash
# Start Vault
sudo systemctl enable vault
sudo systemctl start vault

# Set environment
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_SKIP_VERIFY=1  # Only for self-signed cert

# Initialize
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  > /tmp/vault-keys.txt

# Save these keys securely!

# Unseal
vault operator unseal  # Key 1
vault operator unseal  # Key 2
vault operator unseal  # Key 3

# Login with root token
vault login

# Configure OCI auth (from Post 5)
vault auth enable oci
vault write auth/oci/config \
  home_tenancy_id=$TENANCY_OCID

# Create production policy
vault policy write customer-api-prod - << EOF
path "secret/data/customer-api/production/*" {
  capabilities = ["read"]
}
path "secret/metadata/customer-api/production/*" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF

# Create OCI role
vault write auth/oci/role/customer-api-prod \
  bound_dynamic_group_id=$APP_DYNAMIC_GROUP_OCID \
  token_policies="customer-api-prod" \
  token_ttl="15m" \
  token_max_ttl="1h"

# Enable secrets engine
vault secrets enable -path=secret kv-v2

# Store secrets (already done in Step 4 of Database section)
```

---

## Application Deployment

### Step 1: Build Production Container Image

On your local machine:

```bash
cd customer-api

# Create production Dockerfile
cat > Dockerfile.production << 'EOF'
FROM maven:3.9-eclipse-temurin-17-alpine AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests -Pproduction

FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN chown -R appuser:appgroup /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
EOF

# Build
docker build -f Dockerfile.production -t customer-api:prod .
```

### Step 2: Push to OCI Container Registry

```bash
# Create repository in OCI
oci artifacts container repository create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name "customer-api"

# Login to OCIR
docker login <region>.ocir.io

# Tag and push
docker tag customer-api:prod \
  <region>.ocir.io/<tenancy-namespace>/customer-api:1.0.0

# Verify
podman ps
podman logs -f customer-api
```

### Step 5: Create Systemd Service

Make the application start on boot:

```bash
# Generate systemd service
podman generate systemd --new --name customer-api \
  --files --restart-policy=always

# Move to systemd
sudo mv container-customer-api.service /etc/systemd/system/

# Enable
sudo systemctl daemon-reload
sudo systemctl enable container-customer-api.service
sudo systemctl start container-customer-api.service

# Check status
sudo systemctl status container-customer-api.service
```

---

## Load Balancer Configuration

### Step 1: Create Load Balancer

```bash
# Create load balancer
LB_OCID=$(oci lb load-balancer create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name "customer-api-lb" \
  --shape-name "flexible" \
  --shape-details '{"minimumBandwidthInMbps": 10, "maximumBandwidthInMbps": 100}' \
  --subnet-ids '["'$PUBLIC_SUBNET_OCID'"]' \
  --is-private false \
  --wait-for-state SUCCEEDED \
  --query 'data.id' \
  --raw-output)

echo "Load Balancer OCID: $LB_OCID"
```

### Step 2: Create Backend Set

```bash
# Create backend set with health check
oci lb backend-set create \
  --load-balancer-id $LB_OCID \
  --name "customer-api-backend" \
  --policy "ROUND_ROBIN" \
  --health-checker-protocol "HTTP" \
  --health-checker-url-path "/actuator/health" \
  --health-checker-port 8080 \
  --health-checker-return-code 200 \
  --health-checker-interval-in-millis 30000 \
  --health-checker-timeout-in-millis 3000 \
  --health-checker-retries 3 \
  --wait-for-state SUCCEEDED
```

### Step 3: Add Backend Servers

```bash
# Get private IPs of app instances
APP1_IP=$(oci compute instance get --instance-id $APP1_OCID \
  --query 'data."primary-private-ip"' --raw-output)
APP2_IP=$(oci compute instance get --instance-id $APP2_OCID \
  --query 'data."primary-private-ip"' --raw-output)
APP3_IP=$(oci compute instance get --instance-id $APP3_OCID \
  --query 'data."primary-private-ip"' --raw-output)

# Add backends
for IP in $APP1_IP $APP2_IP $APP3_IP; do
  oci lb backend create \
    --load-balancer-id $LB_OCID \
    --backend-set-name "customer-api-backend" \
    --ip-address $IP \
    --port 8080 \
    --weight 1 \
    --wait-for-state SUCCEEDED
done
```

### Step 4: Create Listener

```bash
# HTTP listener (we'll add HTTPS in security hardening)
oci lb listener create \
  --load-balancer-id $LB_OCID \
  --name "customer-api-listener" \
  --default-backend-set-name "customer-api-backend" \
  --port 80 \
  --protocol "HTTP" \
  --wait-for-state SUCCEEDED
```

### Step 5: Get Load Balancer IP

```bash
LB_IP=$(oci lb load-balancer get \
  --load-balancer-id $LB_OCID \
  --query 'data."ip-addresses"[0]."ip-address"' \
  --raw-output)

echo "Load Balancer Public IP: $LB_IP"
```

---

## Monitoring and Alerting

### Step 1: Enable OCI Monitoring

```bash
# Create alarm for unhealthy backends
oci monitoring alarm create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name "Unhealthy Backend Alert" \
  --destinations '["'$TOPIC_OCID'"]' \
  --is-enabled true \
  --metric-compartment-id $COMPARTMENT_OCID \
  --namespace "oci_lbaas" \
  --query "UnHealthyBackendServers[1m].mean() > 0" \
  --severity "CRITICAL"
```

### Step 2: Application Monitoring Script

Create monitoring script on each app instance:

```bash
sudo tee /usr/local/bin/monitor-app.sh > /dev/null << 'EOF'
#!/bin/bash

# Check application health
HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health)

if [ "$HEALTH_STATUS" != "200" ]; then
    echo "$(date): Application unhealthy (HTTP $HEALTH_STATUS)" | logger -t customer-api
    # Restart application
    systemctl restart container-customer-api.service
fi

# Check memory usage
MEMORY_USAGE=$(podman stats --no-stream customer-api | tail -1 | awk '{print $4}' | sed 's/%//')
if (( $(echo "$MEMORY_USAGE > 90" | bc -l) )); then
    echo "$(date): High memory usage: ${MEMORY_USAGE}%" | logger -t customer-api
fi

# Check Vault connectivity
if ! podman exec customer-api curl -k -s https://10.0.2.10:8200/v1/sys/health > /dev/null; then
    echo "$(date): Cannot reach Vault" | logger -t customer-api
fi
EOF

sudo chmod +x /usr/local/bin/monitor-app.sh

# Add to cron (every 5 minutes)
echo "*/5 * * * * /usr/local/bin/monitor-app.sh" | sudo crontab -
```

### Step 3: Centralized Logging

Install and configure log aggregation:

```bash
# Install Fluent Bit
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh

# Configure to ship logs to OCI Logging
sudo tee /etc/fluent-bit/fluent-bit.conf > /dev/null << 'EOF'
[SERVICE]
    Flush        5
    Daemon       off
    Log_Level    info

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               container.*

[OUTPUT]
    Name              oci
    Match             *
    Region            us-phoenix-1
    Compartment_OCID  ${COMPARTMENT_OCID}
    Log_Group_OCID    ${LOG_GROUP_OCID}
    Auth              instance_principal
EOF

sudo systemctl enable fluent-bit
sudo systemctl start fluent-bit
```

---

## Backup and Disaster Recovery

### Step 1: Automated Instance Backups

```bash
# Create backup policy
oci bv volume-backup-policy create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name "daily-backup-policy" \
  --schedules '[
    {
      "backupType": "INCREMENTAL",
      "period": "ONE_DAY",
      "retentionSeconds": 2592000,
      "timeZone": "UTC",
      "hourOfDay": 2
    }
  ]'

# Assign to volumes
oci bv volume-backup-policy-assignment create \
  --asset-id $DB_VOLUME_OCID \
  --policy-id $BACKUP_POLICY_OCID
```

### Step 2: Vault Backup Script

```bash
# On Vault instance
sudo tee /usr/local/bin/vault-backup.sh > /dev/null << 'EOF'
#!/bin/bash
set -e

BACKUP_DIR="/backup/vault"
DATE=$(date +%Y%m%d_%H%M%S)
BUCKET_NAME="customer-api-backups"

# Stop Vault
systemctl stop vault

# Create backup
mkdir -p $BACKUP_DIR
tar -czf $BACKUP_DIR/vault-data-$DATE.tar.gz /opt/vault/data

# Start Vault
systemctl start vault

# Wait for Vault to be ready
sleep 10

# Unseal Vault (use automated unsealing in production!)
# This is manual for security

# Upload to Object Storage
oci os object put \
  --bucket-name $BUCKET_NAME \
  --file $BACKUP_DIR/vault-data-$DATE.tar.gz \
  --name vault/vault-data-$DATE.tar.gz \
  --auth instance_principal

# Cleanup old backups
find $BACKUP_DIR -name "vault-data-*.tar.gz" -mtime +7 -delete

echo "Vault backup completed: vault-data-$DATE.tar.gz"
EOF

sudo chmod +x /usr/local/bin/vault-backup.sh

# Schedule weekly (Sunday at 3 AM)
echo "0 3 * * 0 /usr/local/bin/vault-backup.sh >> /var/log/vault-backup.log 2>&1" | sudo crontab -
```

### Step 3: Disaster Recovery Runbook

Create DR documentation:

```markdown
# Disaster Recovery Runbook

## Database Failure

1. Check database status:
   ```
   systemctl status postgresql
   ```

2. If service down, attempt restart:
   ```
   systemctl restart postgresql
   ```

3. If data corruption, restore from backup:
   ```
   # Stop PostgreSQL
   systemctl stop postgresql
   
   # Download latest backup
   oci os object get \
     --bucket-name customer-api-backups \
     --name postgres/backup_YYYYMMDD_HHMMSS.sql.gz \
     --file /tmp/restore.sql.gz \
     --auth instance_principal
   
   # Restore
   gunzip < /tmp/restore.sql.gz | sudo -u postgres psql
   
   # Start PostgreSQL
   systemctl start postgresql
   ```

4. Update application instances to reconnect

## Vault Failure

1. Check Vault status:
   ```
   systemctl status vault
   vault status
   ```

2. If sealed, unseal:
   ```
   vault operator unseal  # Use 3 keys
   ```

3. If data lost, restore:
   ```
   systemctl stop vault
   
   # Download backup
   oci os object get \
     --bucket-name customer-api-backups \
     --name vault/vault-data-YYYYMMDD_HHMMSS.tar.gz \
     --file /tmp/vault-restore.tar.gz
   
   # Restore
   tar -xzf /tmp/vault-restore.tar.gz -C /
   
   # Start and unseal
   systemctl start vault
   vault operator unseal  # 3 times
   ```

## Application Instance Failure

1. Check health:
   ```
   curl http://INSTANCE_IP:8080/actuator/health
   ```

2. Check container:
   ```
   podman ps
   podman logs customer-api
   ```

3. Restart if needed:
   ```
   systemctl restart container-customer-api.service
   ```

4. If instance completely failed:
   - Launch new instance from template
   - Deploy application container
   - Add to load balancer backend set
   - Remove failed instance

## Complete Infrastructure Loss

1. Restore VCN and networking from Terraform
2. Launch instances from images/backups
3. Restore database from Object Storage
4. Restore Vault from Object Storage
5. Deploy applications
6. Configure load balancer
7. Update DNS
```

---

## Security Hardening

### Step 1: Enable HTTPS on Load Balancer

```bash
# Create certificate (or use Let's Encrypt)
openssl req -x509 -newkey rsa:2048 \
  -keyout lb-key.pem \
  -out lb-cert.pem \
  -days 365 -nodes \
  -subj "/CN=api.yourdomain.com"

# Upload certificate to OCI
CERT_OCID=$(oci lb certificate create \
  --certificate-name "api-cert" \
  --load-balancer-id $LB_OCID \
  --ca-certificate-file lb-cert.pem \
  --private-key-file lb-key.pem \
  --public-certificate-file lb-cert.pem \
  --wait-for-state SUCCEEDED \
  --query 'data.id' \
  --raw-output)

# Create HTTPS listener
oci lb listener create \
  --load-balancer-id $LB_OCID \
  --name "customer-api-https" \
  --default-backend-set-name "customer-api-backend" \
  --port 443 \
  --protocol "HTTP" \
  --ssl-certificate-name "api-cert" \
  --wait-for-state SUCCEEDED
```

### Step 2: Implement Rate Limiting

Add to application configuration:

```yaml
# application.yml
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false
  security:
    filter:
      order: 5

bucket4j:
  enabled: true
  filters:
  - cache-name: rate-limit-buckets
    url: /api/.*
    rate-limits:
    - bandwidths:
      - capacity: 100
        time: 1
        unit: minutes
```

### Step 3: Enable Web Application Firewall

```bash
# Create WAF policy
oci waas waas-policy create \
  --compartment-id $COMPARTMENT_OCID \
  --domain "api.yourdomain.com" \
  --display-name "customer-api-waf" \
  --origins '[{
    "uri": "'$LB_IP'",
    "httpPort": 80,
    "httpsPort": 443
  }]' \
  --policy-config '{
    "certificateId": null,
    "isHttpsEnabled": true,
    "isHttpsForced": true
  }'
```

### Step 4: Restrict Network Access

Update security lists to minimum required:

```bash
# Application subnet - only from LB
oci network security-list update \
  --security-list-id $APP_SL_OCID \
  --ingress-security-rules '[
    {
      "source": "10.0.0.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 8080, "max": 8080}}
    },
    {
      "source": "10.0.0.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 22, "max": 22}}
    }
  ]' \
  --force

# Database subnet - only from app subnet
oci network security-list update \
  --security-list-id $DB_SL_OCID \
  --ingress-security-rules '[
    {
      "source": "10.0.1.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 5432, "max": 5432}}
    },
    {
      "source": "10.0.1.0/24",
      "protocol": "6",
      "tcpOptions": {"destinationPortRange": {"min": 8200, "max": 8200}}
    }
  ]' \
  --force
```

---

## Deployment Automation

### Complete Deployment Script

```bash
#!/bin/bash
# deploy-production.sh

set -e

echo "=== Production Deployment Script ==="

# Configuration
export COMPARTMENT_OCID="ocid1.compartment.oc1..YOUR_OCID"
export VCN_OCID="ocid1.vcn.oc1..YOUR_OCID"
export IMAGE_OCID="ocid1.image.oc1.phx.YOUR_OCID"
export CONTAINER_IMAGE="phx.ocir.io/namespace/customer-api:1.0.0"

# Step 1: Deploy database
echo "Step 1: Deploying database..."
./scripts/deploy-database.sh

# Step 2: Deploy Vault
echo "Step 2: Deploying Vault..."
./scripts/deploy-vault.sh

# Step 3: Configure Vault
echo "Step 3: Configuring Vault..."
./scripts/configure-vault.sh

# Step 4: Deploy applications
echo "Step 4: Deploying application instances..."
for i in {1..3}; do
  ./scripts/deploy-app-instance.sh $i
done

# Step 5: Configure load balancer
echo "Step 5: Configuring load balancer..."
./scripts/configure-load-balancer.sh

# Step 6: Run health checks
echo "Step 6: Running health checks..."
./scripts/health-check.sh

echo "=== Deployment Complete ==="
echo "Load Balancer IP: $(cat /tmp/lb-ip.txt)"
echo "Test: curl http://$(cat /tmp/lb-ip.txt)/api/v1/customers"
```

---

## Testing the Complete Stack

### End-to-End Test Script

```bash
#!/bin/bash
# test-production.sh

set -e

LB_IP="YOUR_LOAD_BALANCER_IP"
BASE_URL="http://$LB_IP"

echo "=== Testing Production Deployment ==="

# Test 1: Health check
echo "Test 1: Health check..."
if curl -f "$BASE_URL/actuator/health"; then
  echo "✓ Health check passed"
else
  echo "✗ Health check failed"
  exit 1
fi

# Test 2: Create customer
echo "Test 2: Create customer..."
RESPONSE=$(curl -s -X POST "$BASE_URL/api/v1/customers" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "prod-test@example.com",
    "companyName": "Production Test Corp",
    "contactName": "Test User",
    "phone": "+1-555-PROD"
  }')

CUSTOMER_ID=$(echo $RESPONSE | jq -r '.id')
echo "✓ Customer created with ID: $CUSTOMER_ID"

# Test 3: Retrieve customer
echo "Test 3: Retrieve customer..."
curl -f "$BASE_URL/api/v1/customers/$CUSTOMER_ID"
echo "✓ Customer retrieved"

# Test 4: Load balancer distribution
echo "Test 4: Testing load distribution..."
for i in {1..10}; do
  curl -s "$BASE_URL/api/v1/customers" > /dev/null
done
echo "✓ Load balancer distributing traffic"

# Test 5: Database connectivity
echo "Test 5: Database connectivity..."
CUSTOMER_COUNT=$(curl -s "$BASE_URL/api/v1/customers" | jq '. | length')
echo "✓ Database connected (customers: $CUSTOMER_COUNT)"

echo ""
echo "=== All Tests Passed ==="
```

---

## Operational Runbooks

### Daily Operations Checklist

```markdown
## Daily Checklist

- [ ] Check all instances are running
- [ ] Verify load balancer health
- [ ] Review application logs for errors
- [ ] Check Vault seal status
- [ ] Verify database backups completed
- [ ] Review monitoring dashboards
- [ ] Check for security alerts

## Weekly Tasks

- [ ] Review and rotate logs
- [ ] Update OS patches
- [ ] Review access logs
- [ ] Test disaster recovery procedures
- [ ] Update documentation
- [ ] Review cost optimization

## Monthly Tasks

- [ ] Rotate Vault tokens
- [ ] Update TLS certificates (if needed)
- [ ] Review IAM policies
- [ ] Capacity planning review
- [ ] Security audit
- [ ] DR drill
```

---

## What's Next

🎉 **Congratulations!** You've deployed a complete production-grade stack! Here's what we accomplished:

✅ Multi-tier network architecture  
✅ High-availability application deployment  
✅ Secure database with automated backups  
✅ Vault managing all secrets  
✅ Load balancer with health checks  
✅ Monitoring and alerting  
✅ Disaster recovery procedures  
✅ Security hardening  
✅ Deployment automation  

### Current Architecture

```
Production Stack on OCI:
├── Load Balancer (HTTPS)
├── 3x Application Instances
│   └── Spring Boot + OCI Auth
├── Vault Server
│   └── Zero-trust authentication
├── PostgreSQL Database
│   └── Automated backups
├── Monitoring & Logging
└── Automated DR procedures
```

### Coming Up

**Post 7: Secret Rotation and Resilience**  
- Automatic database credential rotation
- Graceful handling of Vault downtime
- Circuit breakers and retry logic
- Zero-downtime secret updates

**Post 8: High Availability and Scaling**  
- Vault HA cluster with Raft
- Auto-scaling application instances  
- Multi-region deployment
- Performance optimization

**Post 9: CI/CD with GitHub Actions**  
- Automated build and test
- Container registry integration
- Blue-green deployments
- Rollback procedures

---

## Key Takeaways

🔑 **Multi-tier architecture** provides security and scalability  
🔑 **Load balancer** enables high availability and zero-downtime  
🔑 **Automated backups** protect against data loss  
🔑 **Monitoring** catches issues before users notice  
🔑 **Runbooks** ensure consistent operations  
🔑 **Security hardening** protects production data  
🔑 **Deployment automation** reduces human error  

---

*Previous: [Post 5: Zero-Trust Authentication with OCI Instance Principals](#) ←*

*Next: [Post 7: Secret Rotation and Resilience](#) →*

---

## Tags

#Production #OCI #Vault #LoadBalancer #HighAvailability #DevOps #CloudArchitecture #PostgreSQL #Monitoring #DisasterRecovery #SecurityHardening #Automationapi:1.0.0

docker push <region>.ocir.io/<tenancy-namespace>/customer-api:1.0.0
```

### Step 3: Launch Application Instances

Create 3 application instances:

```bash
# Function to create app instance
create_app_instance() {
  local instance_num=$1
  
  oci compute instance launch \
    --compartment-id $COMPARTMENT_OCID \
    --availability-domain "YOUR-AD-1" \
    --shape "VM.Standard.E4.Flex" \
    --shape-config '{"ocpus": 1, "memoryInGBs": 4}' \
    --display-name "app-instance-$instance_num" \
    --subnet-id $APP_SUBNET_OCID \
    --image-id "ocid1.image.oc1.phx.aaaaaaaaxxx" \
    --ssh-authorized-keys-file ~/.ssh/oci_key.pub \
    --wait-for-state RUNNING
}

# Create 3 instances
create_app_instance 1
create_app_instance 2
create_app_instance 3
```

### Step 4: Deploy Application Container

SSH into each app instance and run:

```bash
# Install Podman
sudo yum install -y podman

# Login to OCIR
podman login <region>.ocir.io

# Pull image
podman pull <region>.ocir.io/<tenancy-namespace>/customer-api:1.0.0

# Create application.yml for production
mkdir -p /home/opc/config
cat > /home/opc/config/application.yml << 'EOF'
spring:
  application:
    name: customer-api
  cloud:
    vault:
      uri: https://10.0.2.10:8200
      authentication: OCI
      oci:
        role: customer-api-prod
        auth-type: instance
      kv:
        enabled: true
        backend: secret
        application-name: customer-api
        profiles: production
      ssl:
        trust-store: file:/app/config/truststore.jks
  config:
    import: vault://
  datasource:
    url: jdbc:postgresql://${database.host}:${database.port}/${database.database}
    username: ${database.username}
    password: ${database.password}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

server:
  port: 8080

logging:
  level:
    root: INFO
    com.example.customerapi: INFO
EOF

# Run container
podman run -d \
  --name customer-api \
  --restart unless-stopped \
  -p 8080:8080 \
  -v /home/opc/config:/app/config:ro \
  <region>.ocir.io/<tenancy-namespace>/customer-