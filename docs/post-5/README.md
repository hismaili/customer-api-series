# Zero-Trust Authentication: Eliminating Secrets with OCI Instance Principals and Vault

*Part 5 of 9 in the series: "Production-Ready Spring Boot on OCI with Vault"*

![Zero Trust Security](https://images.unsplash.com/photo-1563986768609-322da13575f3?w=1200&h=400&fit=crop)

## What You'll Build

This is where everything comes together. You'll deploy HashiCorp Vault to OCI and configure it to use OCI Instance Principals for authentication - achieving true zero-secret architecture. No passwords, no tokens, no API keys stored anywhere.

**By the end of this post, you'll have**:
- ✅ Vault running in production mode on OCI
- ✅ OCI auth method configured in Vault
- ✅ Dynamic groups and IAM policies for instance authentication
- ✅ Spring Boot automatically authenticating to Vault without credentials
- ✅ Complete zero-trust security architecture
- ✅ Understanding of the Secret Zero Paradox and how to solve it

**Time to complete**: 90 minutes  
**Difficulty**: Advanced  
**Cost**: $0 (using OCI Free Tier)

---

## Table of Contents

1. [The Secret Zero Paradox](#the-secret-zero-paradox)
2. [Understanding Zero-Trust Architecture](#understanding-zero-trust-architecture)
3. [How OCI Instance Principals Work](#how-oci-instance-principals-work)
4. [Architecture Overview](#architecture-overview)
5. [Prerequisites and Planning](#prerequisites-and-planning)
6. [Setting Up OCI IAM](#setting-up-oci-iam)
7. [Deploying Vault in Production Mode](#deploying-vault-in-production-mode)
8. [Configuring OCI Auth Method](#configuring-oci-auth-method)
9. [Testing Instance Principal Authentication](#testing-instance-principal-authentication)
10. [Deploying Spring Boot with OCI Auth](#deploying-spring-boot-with-oci-auth)
11. [Troubleshooting Common Issues](#troubleshooting-common-issues)
12. [Security Hardening](#security-hardening)
13. [What's Next](#whats-next)

---

## Prerequisites

### From Previous Posts

**Required**:
- ✅ [Post 2: OCI Infrastructure](link) - VCN, compute instance, IAM basics
- ✅ [Post 3: Container Deployment](link) - Docker/Podman skills
- ✅ [Post 4: Vault Fundamentals](link) - Vault concepts and CLI

**You should have**:
- OCI compute instance running (from Post 2)
- SSH access to your instance
- Understanding of Vault policies and secrets engines
- Spring Boot Customer API source code

### What You'll Need

- OCI tenancy with admin access
- Ability to create dynamic groups and policies
- Two compute instances (or use one for both Vault and app)
- Basic understanding of cryptographic signatures

---

## The Secret Zero Paradox

Before we dive in, let's understand the fundamental problem we're solving.

### The Classic Chicken-and-Egg Problem

**Scenario**: Your application needs to authenticate to Vault to get secrets.

**The Problem**:
```
To get secrets from Vault...
    ↓
You need to authenticate to Vault...
    ↓
To authenticate, you need credentials...
    ↓
But where do you store those credentials?
    ↓
🔄 Back to square one!
```

### Traditional "Solutions" (All Bad)

**Option 1: Hardcoded Token**
```yaml
# application.yml
spring:
  cloud:
    vault:
      token: hvs.CAESIJ8...  # ❌ Token in source code!
```
**Problem**: The token IS a secret. We're back to storing secrets!

**Option 2: Environment Variable**
```bash
export VAULT_TOKEN=hvs.CAESIJ8...  # ❌ Secret in environment!
```
**Problem**: Still a secret that needs to be managed and rotated.

**Option 3: Config File**
```bash
# /etc/vault/token
hvs.CAESIJ8...  # ❌ Secret on disk!
```
**Problem**: Anyone with file access has the token.

**Option 4: Secret Management Tool**
```bash
# Use AWS Secrets Manager to store Vault token
aws secretsmanager get-secret-value --secret-id vault-token
```
**Problem**: Now you need AWS credentials! We've just moved the problem.

### The Real Solution: Cryptographic Identity

Instead of storing credentials, we use **cryptographic proof of identity**:

```
1. OCI cryptographically signs instance identity
   ↓
2. Instance presents signed proof to Vault
   ↓
3. Vault verifies signature with OCI
   ↓
4. Vault grants access based on identity
   ↓
✅ No secrets stored anywhere!
```

**This is what OCI Instance Principals achieve.**

---

## Understanding Zero-Trust Architecture

### What is Zero-Trust?

**Traditional Security Model** (Perimeter-based):
```
Outside (Untrusted)
    ↓
Firewall ← "The Wall"
    ↓
Inside (Trusted)
```
**Problem**: Once inside, everything is trusted. Breach = game over.

**Zero-Trust Model**:
```
Every Request Must Be:
├── Authenticated (Who are you?)
├── Authorized (What can you do?)
├── Encrypted (Secure communication)
└── Audited (Log everything)
```
**Principle**: "Never trust, always verify"

### Zero-Trust Applied to Secrets

**Components**:
1. **Identity**: OCI Instance Principal (cryptographically verified)
2. **Authentication**: OCI signs identity, Vault verifies
3. **Authorization**: Vault policies control access
4. **Encryption**: TLS for all communication
5. **Audit**: Vault logs all access attempts

### The Complete Flow

```
┌─────────────────────────────────────────────┐
│         Spring Boot Application              │
│  "I need database credentials"               │
└─────────────────┬───────────────────────────┘
                  │
                  │ 1. Request metadata token
                  ↓
┌─────────────────────────────────────────────┐
│    OCI Metadata Service (169.254.169.254)   │
│  "Here's your cryptographically signed       │
│   instance identity"                         │
└─────────────────┬───────────────────────────┘
                  │
                  │ 2. Present signed identity
                  ↓
┌─────────────────────────────────────────────┐
│           HashiCorp Vault                    │
│  Step 1: Verify signature with OCI          │
│  Step 2: Check dynamic group membership     │
│  Step 3: Apply policies                     │
│  Step 4: Return Vault token                 │
└─────────────────┬───────────────────────────┘
                  │
                  │ 3. Use Vault token
                  ↓
┌─────────────────────────────────────────────┐
│  "Here are your database credentials"        │
└─────────────────────────────────────────────┘
```

**Key Point**: No secrets stored at any step!

---

## How OCI Instance Principals Work

### The Cryptographic Magic

**What is an Instance Principal?**

An instance principal is an IAM identity that represents a compute instance. It's based on:
1. **Instance OCID** - Unique identifier
2. **Compartment membership** - Which compartment it belongs to
3. **Cryptographic certificate** - Signed by OCI
4. **Metadata service** - Provides secure token endpoint

### The Authentication Flow (Deep Dive)

**Step 1: Instance Requests Token**
```bash
# Application calls:
curl -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/
```

**Step 2: Metadata Service Returns Signed Token**
```json
{
  "id": "ocid1.instance.oc1.phx.anyhqljs...",
  "compartmentId": "ocid1.compartment...",
  "tenantId": "ocid1.tenancy...",
  "signature": "BASE64_ENCODED_SIGNATURE",
  "certificate": "-----BEGIN CERTIFICATE-----..."
}
```

The signature proves:
- This request came from this specific instance
- At this specific time
- In this specific tenancy

**Step 3: Vault Verifies with OCI**
```
Vault → OCI API: "Is this signature valid?"
OCI → Vault: "Yes, it's from instance X in compartment Y"
```

**Step 4: Vault Checks Policies**
```
- Is instance X in dynamic group Z?
- Does dynamic group Z have policy for this Vault role?
- What permissions does the policy grant?
```

**Step 5: Vault Issues Token**
```
Vault → Application: "Here's your token: hvs.ABC..."
Application → Vault: "Give me secret/database"
Vault → Application: "Here are your credentials"
```

### Why This is Secure

**Cryptographic Proof**:
- Signatures use PKI (Public Key Infrastructure)
- Private keys never leave OCI infrastructure
- Impossible to forge without OCI's private key

**Time-Limited**:
- Tokens expire (typically 1 hour)
- Must re-authenticate periodically
- Compromised token has limited lifetime

**Auditable**:
- Every authentication attempt is logged
- Track which instances accessed what
- Detect unusual patterns

**No Stored Secrets**:
- No passwords, tokens, or keys on disk
- No environment variables to leak
- No secrets in source code or images

---

## Architecture Overview

Here's what we're building:

```
┌────────────────────────────────────────────────────┐
│                OCI Tenancy                          │
│                                                     │
│  ┌──────────────────────────────────────────────┐ │
│  │  Dynamic Group: vault-servers                 │ │
│  │  Rule: instance.compartment.id = 'xxx'        │ │
│  └──────────────────────────────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐ │
│  │  Dynamic Group: customer-api-instances        │ │
│  │  Rule: instance.compartment.id = 'xxx'        │ │
│  └──────────────────────────────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐ │
│  │  IAM Policies:                                │ │
│  │  - vault-servers can verify authentication    │ │
│  │  - customer-api-instances can authenticate    │ │
│  └──────────────────────────────────────────────┘ │
│                                                     │
│  ┌───────────────┐         ┌───────────────────┐  │
│  │ Vault Server  │         │ Spring Boot App   │  │
│  │ Instance      │◄────────│ Instance          │  │
│  │               │ Verify  │                   │  │
│  │ - Vault       │ Auth    │ - Customer API    │  │
│  │ - OCI Auth    │         │ - No credentials! │  │
│  │ - Policies    │         │                   │  │
│  └───────────────┘         └───────────────────┘  │
│         ↓                           ↓              │
│         ↓                           ↓              │
│  [Metadata Service: 169.254.169.254]              │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Network Architecture

```
VCN: customer-api-vcn (10.0.0.0/16)
    │
    ├── Public Subnet (10.0.0.0/24)
    │   ├── vault-instance (10.0.0.10)
    │   │   └── Vault Server on port 8200
    │   │
    │   └── app-instance (10.0.0.11)
    │       └── Spring Boot on port 8080
    │
    └── Security Lists
        ├── Allow 8200 (Vault API)
        ├── Allow 8080 (Spring Boot)
        └── Allow 22 (SSH)
```

---

## Prerequisites and Planning

### Instance Requirements

For this tutorial, you have two options:

**Option 1: Two Instances** (Recommended for production learning)
- **vault-instance**: Runs Vault server
- **app-instance**: Runs Spring Boot application

**Option 2: Single Instance** (Simpler for learning)
- One instance runs both Vault and Spring Boot

We'll use **Option 2** for simplicity. You can adapt for Option 1.

### Gather Required Information

Before starting, collect these OCIDs:

```bash
# SSH into your instance
ssh -i ~/.ssh/oci_key opc@YOUR_INSTANCE_IP

# Get instance information
INSTANCE_OCID=$(curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.id')

COMPARTMENT_OCID=$(curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.compartmentId')

TENANCY_OCID=$(curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.compartmentId')

echo "Instance OCID: $INSTANCE_OCID"
echo "Compartment OCID: $COMPARTMENT_OCID"
echo "Tenancy OCID: $TENANCY_OCID"
```

**Save these values** - you'll need them throughout this tutorial!

---

## Setting Up OCI IAM

### Step 1: Create Dynamic Group for Vault Server

1. **Go to OCI Console** → **Identity & Security** → **Dynamic Groups**
2. Click **Create Dynamic Group**
3. Fill in:
   - **Name**: `vault-servers`
   - **Description**: `Vault server instances that can verify authentication`
   - **Matching Rules**:
     ```
     instance.compartment.id = 'ocid1.compartment.oc1..YOUR_COMPARTMENT_OCID'
     ```
     Or for specific instance:
     ```
     instance.id = 'ocid1.instance.oc1.phx.YOUR_INSTANCE_OCID'
     ```
4. Click **Create**
5. **Copy the Dynamic Group OCID** - save it!

### Step 2: Create Dynamic Group for Application Instances

1. Click **Create Dynamic Group**
2. Fill in:
   - **Name**: `customer-api-instances`
   - **Description**: `Application instances that authenticate to Vault`
   - **Matching Rules**:
     ```
     instance.compartment.id = 'ocid1.compartment.oc1..YOUR_COMPARTMENT_OCID'
     ```
3. Click **Create**
4. **Copy this Dynamic Group OCID** too!

### Step 3: Create IAM Policy for Vault Servers

**Critical**: Vault needs permission to verify OCI authentication requests.

1. **Go to** → **Identity & Security** → **Policies**
2. **Compartment**: Select **(root)** - your tenancy
3. Click **Create Policy**
4. Fill in:
   - **Name**: `vault-oci-auth-policy`
   - **Description**: `Allow Vault to verify OCI authentication`
   - **Compartment**: (root)
   - Toggle **Show manual editor**
   - **Statements**:
     ```
     Allow dynamic-group vault-servers to {AUTHENTICATION_INSPECT} in tenancy
     Allow dynamic-group vault-servers to {GROUP_MEMBERSHIP_INSPECT} in tenancy
     ```
5. Click **Create**

**What these permissions do**:
- `AUTHENTICATION_INSPECT`: Verify authentication requests are valid
- `GROUP_MEMBERSHIP_INSPECT`: Check if instances belong to dynamic groups

### Step 4: Create IAM Policy for Application Instances

**Optional**: If your application needs to call other OCI services:

```
Allow dynamic-group customer-api-instances to read object-family in tenancy
Allow dynamic-group customer-api-instances to use secret-family in tenancy
```

For our tutorial, the application only needs to access Vault, so **this step is optional**.

### Verify Dynamic Groups

```bash
# If you have OCI CLI installed
oci iam dynamic-group list --all

# Or check in OCI Console:
# Identity & Security → Dynamic Groups
# You should see:
# - vault-servers
# - customer-api-instances
```

---

## Deploying Vault in Production Mode

Now let's deploy Vault in production mode (not dev mode!).

### Step 1: Install Vault on OCI Instance

SSH into your instance:

```bash
ssh -i ~/.ssh/oci_key opc@YOUR_INSTANCE_IP

# Install Vault
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vault

# Verify
vault version
```

### Step 2: Create Vault Configuration

Create `/etc/vault.d/vault.hcl`:

```bash
sudo mkdir -p /etc/vault.d
sudo tee /etc/vault.d/vault.hcl > /dev/null << 'EOF'
# Vault server configuration

# Storage backend (file storage for single instance)
storage "file" {
  path = "/opt/vault/data"
}

# Listener (API endpoint)
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1  # For learning - enable TLS in production!
}

# API address
api_addr = "http://YOUR_PRIVATE_IP:8200"

# UI
ui = true

# Log level
log_level = "Info"
EOF
```

**Replace `YOUR_PRIVATE_IP`** with your instance's private IP:

```bash
PRIVATE_IP=$(hostname -I | awk '{print $1}')
sudo sed -i "s/YOUR_PRIVATE_IP/$PRIVATE_IP/g" /etc/vault.d/vault.hcl
```

### Step 3: Create Storage Directory

```bash
sudo mkdir -p /opt/vault/data
sudo chown -R vault:vault /opt/vault
sudo chmod 700 /opt/vault/data
```

### Step 4: Start Vault as a Service

```bash
# Start Vault
sudo systemctl enable vault
sudo systemctl start vault

# Check status
sudo systemctl status vault

# Should show "active (running)"
```

### Step 5: Initialize Vault

**⚠️ IMPORTANT**: This is done ONCE. Save the output securely!

```bash
# Set Vault address
export VAULT_ADDR='http://127.0.0.1:8200'

# Initialize Vault
vault operator init -key-shares=5 -key-threshold=3

# Output will look like:
Unseal Key 1: AAAA...
Unseal Key 2: BBBB...
Unseal Key 3: CCCC...
Unseal Key 4: DDDD...
Unseal Key 5: EEEE...

Initial Root Token: hvs.XXXXXX...
```

**⚠️ CRITICAL**: 
- **Save these keys offline** (not on the instance!)
- You need 3 keys to unseal Vault
- Root token is for initial setup only
- Never lose these keys!

### Step 6: Unseal Vault

```bash
# Unseal with 3 different keys
vault operator unseal  # Enter Unseal Key 1
vault operator unseal  # Enter Unseal Key 2
vault operator unseal  # Enter Unseal Key 3

# Check status
vault status

# Should show:
# Sealed: false
```

### Step 7: Login with Root Token

```bash
# Login
vault login

# Enter your Initial Root Token

# Verify
vault status
```

---

## Configuring OCI Auth Method

Now we configure Vault to accept OCI authentication.

### Step 1: Enable OCI Auth Method

```bash
# Enable OCI auth
vault auth enable oci

# Verify
vault auth list
# Should show "oci/" in the list
```

### Step 2: Configure OCI Auth with Tenancy

```bash
# Get your tenancy OCID
TENANCY_OCID=$(curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.compartmentId')

# Configure Vault with your tenancy
vault write auth/oci/config \
  home_tenancy_id=$TENANCY_OCID

# Verify
vault read auth/oci/config
```

### Step 3: Create Vault Policy for Application

Create a policy file:

```bash
cat > /tmp/customer-api-policy.hcl << 'EOF'
# Read customer-api secrets
path "secret/data/customer-api/*" {
  capabilities = ["read", "list"]
}

# Read metadata
path "secret/metadata/customer-api/*" {
  capabilities = ["read", "list"]
}

# Renew own token
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Lookup own token
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF

# Upload policy
vault policy write customer-api-policy /tmp/customer-api-policy.hcl

# Verify
vault policy read customer-api-policy
```

### Step 4: Create OCI Auth Role

This is the crucial step - linking dynamic groups to Vault policies!

```bash
# Get dynamic group OCID
DYNAMIC_GROUP_OCID="ocid1.dynamicgroup.oc1..YOUR_DYNAMIC_GROUP_OCID"

# Create role
vault write auth/oci/role/customer-api-role \
  bound_dynamic_group_id=$DYNAMIC_GROUP_OCID \
  token_policies="customer-api-policy" \
  token_ttl="1h" \
  token_max_ttl="24h"

# Verify
vault read auth/oci/role/customer-api-role
```

**What this does**:
- Instances in `customer-api-instances` dynamic group
- Can authenticate using the `customer-api-role` role
- Will receive tokens with `customer-api-policy` attached
- Tokens last 1 hour, renewable up to 24 hours

### Step 5: Enable KV Secrets Engine

```bash
# Enable KV v2
vault secrets enable -path=secret kv-v2

# Verify
vault secrets list
```

### Step 6: Store Secrets

```bash
# Store database credentials
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="SecurePassword123!"

# Store API keys
vault kv put secret/customer-api/api-keys \
  stripe_key="sk_live_abc123xyz789" \
  sendgrid_key="SG.abc123.xyz789"

# Store configuration
vault kv put secret/customer-api/config \
  jwt_secret="your-jwt-secret-key-here" \
  encryption_key="your-encryption-key" \
  environment="production"

# Verify
vault kv list secret/customer-api/
vault kv get secret/customer-api/database
```

---

## Testing Instance Principal Authentication

Before deploying Spring Boot, let's test that OCI auth works.

### Test from the Command Line

```bash
# Test OCI authentication
vault login -method=oci \
  auth_type=instance \
  role=customer-api-role

# Expected output:
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.CAESIJ...
token_accessor       abc123...
token_duration       1h
token_renewable      true
token_policies       ["customer-api-policy" "default"]
```

**✅ If this works, OCI authentication is configured correctly!**

### Test Reading Secrets

```bash
# Read a secret
vault kv get secret/customer-api/database

# Should work! You're authenticated via instance principal.
```

### Troubleshooting Authentication

If authentication fails:

**Error: "permission denied"**
```bash
# Check IAM policies
# Ensure vault-servers dynamic group has AUTHENTICATION_INSPECT

# Check if instance is in dynamic group
# OCI Console → Dynamic Groups → customer-api-instances
# Check "Matching Resources" section
```

**Error: "role not found"**
```bash
# List roles
vault list auth/oci/role

# Verify role exists
vault read auth/oci/role/customer-api-role
```

**Error: "invalid dynamic group"**
```bash
# Verify dynamic group OCID
vault read auth/oci/role/customer-api-role
# Check bound_dynamic_group_id matches your actual dynamic group
```

---

## Deploying Spring Boot with OCI Auth

Now let's deploy our Spring Boot application to use OCI authentication!

### Step 1: Update Spring Boot Configuration

Update `src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: customer-api
  
  # Vault configuration with OCI auth
  cloud:
    vault:
      uri: http://localhost:8200
      authentication: OCI
      oci:
        role: customer-api-role
        auth-type: instance
      kv:
        enabled: true
        backend: secret
        application-name: customer-api
      config:
        lifecycle:
          enabled: true
          min-renewal: 10s
  
  # Import from Vault
  config:
    import: vault://
  
  # Database configuration (values from Vault)
  datasource:
    url: jdbc:postgresql://${database.host}:${database.port}/${database.database}
    username: ${database.username}
    password: ${database.password}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
  
  # JPA
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    database-platform: org.hibernate.dialect.PostgreSQLDialect

# Server
server:
  port: 8080

# Logging
logging:
  level:
    root: INFO
    com.example.customerapi: DEBUG
    org.springframework.cloud.vault: INFO
```

**Key changes**:
- `authentication: OCI` - Use OCI instead of TOKEN
- `oci.role: customer-api-role` - Vault role name
- `oci.auth-type: instance` - Instance principal type
- **No token specified!** - Automatic authentication

### Step 2: Build the Application

```bash
# On your local machine
cd customer-api

# Build the JAR
./mvnw clean package -DskipTests

# The JAR is in target/customer-api-0.0.1-SNAPSHOT.jar
```

### Step 3: Transfer to OCI Instance

```bash
# Copy JAR to OCI instance
scp -i ~/.ssh/oci_key \
  target/customer-api-0.0.1-SNAPSHOT.jar \
  opc@YOUR_INSTANCE_IP:~/customer-api.jar
```

### Step 4: Start PostgreSQL Database

```bash
# SSH into your instance
ssh -i ~/.ssh/oci_key opc@YOUR_INSTANCE_IP

# Start PostgreSQL
podman run -d \
  --name customer-api-db \
  -e POSTGRES_DB=customerdb \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=SecurePassword123! \
  -p 5432:5432 \
  postgres:15-alpine

# Verify
podman ps | grep customer-api-db
```

### Step 5: Run Spring Boot Application

```bash
# Run the application
java -jar customer-api.jar

# Watch the logs...
```

**Expected log output**:
```
2026-01-12 15:00:00.123  INFO --- Vault OCI authentication
2026-01-12 15:00:01.456  INFO --- Using OCI instance principal
2026-01-12 15:00:02.789  INFO --- Fetching config from server at: http://localhost:8200
2026-01-12 15:00:03.012  INFO --- Located property source: VaultPropertySource
2026-01-12 15:00:04.345  INFO --- Using database: customerdb
2026-01-12 15:00:05.678  INFO --- Started CustomerApiApplication in 6.789 seconds
```

**✅ Success indicators**:
- "Vault OCI authentication"
- "Using OCI instance principal"
- "Located property source: VaultPropertySource"
- No errors about authentication

### Step 6: Test the Application

```bash
# In another SSH session or terminal

# Test health endpoint
curl http://localhost:8080/actuator/health

# Create a customer
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -d '{
    "email": "zero-trust@example.com",
    "companyName": "Zero Trust Corp",
    "contactName": "Secure User",
    "phone": "+1-555-ZERO"
  }'

# Get all customers
curl http://localhost:8080/api/v1/customers
```

**🎉 If this works, you have achieved zero-secret architecture!**

---

## Troubleshooting Common Issues

### Issue 1: "403 Forbidden" from Metadata Service

**Error**:
```
Failed to retrieve instance metadata
HTTP 403 Forbidden
```

**Cause**: Metadata service not accessible

**Fix**:
```bash
# Test metadata service
curl -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/

# If this fails, check:
# 1. Are you running ON the OCI instance?
# 2. Is metadata service enabled for the instance?

# Check in OCI Console:
# Compute → Instances → Your Instance
# Look for "Instance metadata service" status
```

### Issue 2: "Service error: NotAuthorizedOrNotFound"

**Error**:
```
Authorization failed for authenticated request
http status code: 404
```

**Cause**: Instance not in dynamic group or IAM policies missing

**Fix**:
```bash
# Step 1: Verify instance OCID
INSTANCE_OCID=$(curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.id')
echo $INSTANCE_OCID

# Step 2: Check dynamic group matching rules
# OCI Console → Dynamic Groups → customer-api-instances
# Verify matching rule includes your instance

# Step 3: Verify IAM policies
# OCI Console → Policies
# Ensure "vault-oci-auth-policy" exists with:
# - AUTHENTICATION_INSPECT
# - GROUP_MEMBERSHIP_INSPECT
```

### Issue 3: Application Can't Connect to Vault

**Error**:
```
Connection refused: connect to http://localhost:8200
```

**Fix**:
```bash
# Check if Vault is running
sudo systemctl status vault

# Check if Vault is unsealed
export VAULT_ADDR='http://127.0.0.1:8200'
vault status
# Sealed should be: false

# If sealed, unseal it:
vault operator unseal  # Enter key 1
vault operator unseal  # Enter key 2
vault operator unseal  # Enter key 3
```

### Issue 4: "Invalid role" Error

**Error**:
```
role "customer-api-role" not found
```

**Fix**:
```bash
# List existing roles
vault list auth/oci/role

# If role doesn't exist, create it:
vault write auth/oci/role/customer-api-role \
  bound_dynamic_group_id=$DYNAMIC_GROUP_OCID \
  token_policies="customer-api-policy" \
  token_ttl="1h" \
  token_max_ttl="24h"
```

### Issue 5: Secrets Not Found

**Error**:
```
No value found at secret/data/customer-api/database
```

**Fix**:
```bash
# Verify secrets exist
vault kv list secret/customer-api/

# If empty, create secrets:
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="SecurePassword123!"
```

### Issue 6: Token Expired

**Error**:
```
permission denied
```

**Cause**: Vault token expired

**Fix**: Spring Cloud Vault automatically renews tokens, but if manual:
```bash
# Check token status
vault token lookup

# Token should auto-renew, but if needed:
vault token renew
```

---

## Security Hardening

Now that it's working, let's harden security.

### 1. Enable TLS for Vault

**Generate certificates**:
```bash
# Using self-signed cert (for learning)
sudo mkdir -p /opt/vault/tls
cd /opt/vault/tls

# Generate private key
sudo openssl genrsa -out vault-key.pem 2048

# Generate certificate
sudo openssl req -new -x509 -key vault-key.pem -out vault-cert.pem -days 365 \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=vault.example.com"

# Set permissions
sudo chown vault:vault /opt/vault/tls/*
sudo chmod 600 /opt/vault/tls/vault-key.pem
```

**Update Vault configuration**:
```bash
sudo tee /etc/vault.d/vault.hcl > /dev/null << 'EOF'
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 0
  tls_cert_file = "/opt/vault/tls/vault-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-key.pem"
}

api_addr = "https://YOUR_PRIVATE_IP:8200"
ui = true
log_level = "Info"
EOF

# Replace YOUR_PRIVATE_IP
PRIVATE_IP=$(hostname -I | awk '{print $1}')
sudo sed -i "s/YOUR_PRIVATE_IP/$PRIVATE_IP/g" /etc/vault.d/vault.hcl

# Restart Vault
sudo systemctl restart vault
```

**Update Spring Boot**:
```yaml
spring:
  cloud:
    vault:
      uri: https://localhost:8200
      ssl:
        trust-store: file:/path/to/truststore.jks
        trust-store-password: changeit
```

### 2. Restrict Network Access

**Update Security Lists**:
- Port 8200: Only from application instances (not 0.0.0.0/0)
- Port 22: Only from your IP (not 0.0.0.0/0)

```bash
# Example: Update security list to allow Vault access only from specific subnet
# OCI Console → Networking → Security Lists
# Edit Ingress Rules:
# Source: 10.0.0.0/24 (your VCN subnet)
# Destination Port: 8200
```

### 3. Rotate Root Token

After initial setup, revoke the root token:

```bash
# Create a new admin token first
vault token create -policy=root -ttl=24h

# Save the new token

# Revoke the initial root token
vault token revoke <INITIAL_ROOT_TOKEN>
```

### 4. Enable Audit Logging

```bash
# Enable file audit
vault audit enable file file_path=/var/log/vault/audit.log

# Create log directory
sudo mkdir -p /var/log/vault
sudo chown vault:vault /var/log/vault

# Restart Vault
sudo systemctl restart vault

# View audit logs
sudo tail -f /var/log/vault/audit.log
```

### 5. Implement Token TTL Best Practices

```bash
# Update role with shorter TTL
vault write auth/oci/role/customer-api-role \
  bound_dynamic_group_id=$DYNAMIC_GROUP_OCID \
  token_policies="customer-api-policy" \
  token_ttl="15m" \
  token_max_ttl="1h" \
  token_period="15m"

# Spring Cloud Vault will auto-renew every 15 minutes
```

### 6. Implement Least Privilege Policies

**Current policy** (read everything):
```hcl
path "secret/data/customer-api/*" {
  capabilities = ["read", "list"]
}
```

**Better policy** (specific paths):
```hcl
# Only read database credentials
path "secret/data/customer-api/database" {
  capabilities = ["read"]
}

# Only read API keys
path "secret/data/customer-api/api-keys" {
  capabilities = ["read"]
}

# Deny access to production secrets (if on same Vault)
path "secret/data/customer-api/production/*" {
  capabilities = ["deny"]
}
```

### 7. Monitor and Alert

**Set up monitoring**:
```bash
# Install Prometheus exporter
# Vault exposes metrics at /v1/sys/metrics

# Monitor key metrics:
# - vault_core_unsealed (should be 1)
# - vault_token_count (watch for token leak)
# - vault_policy_get_request_count (track access patterns)
```

---

## Advanced Configurations

### Using Vault Agent for Secret Templating

Instead of Spring Cloud Vault, use Vault Agent:

**Create agent config**:
```hcl
# /etc/vault-agent.d/config.hcl
vault {
  address = "http://127.0.0.1:8200"
}

auto_auth {
  method {
    type = "oci"
    config = {
      role = "customer-api-role"
      type = "instance"
    }
  }
  
  sink {
    type = "file"
    config = {
      path = "/opt/vault-agent/token"
    }
  }
}

template {
  source      = "/opt/vault-agent/templates/application.properties.tpl"
  destination = "/opt/app/config/application.properties"
  command     = "systemctl restart customer-api"
}
```

**Template file**:
```properties
# /opt/vault-agent/templates/application.properties.tpl
{{ with secret "secret/data/customer-api/database" }}
spring.datasource.url=jdbc:postgresql://{{ .Data.data.host }}:{{ .Data.data.port }}/{{ .Data.data.database }}
spring.datasource.username={{ .Data.data.username }}
spring.datasource.password={{ .Data.data.password }}
{{ end }}
```

**Benefits**:
- Application has no Vault dependency
- Secrets rendered as plain config files
- Automatic updates on secret rotation

### Multiple Environments

**Separate Vault namespaces**:
```bash
# Create namespace (Vault Enterprise feature)
vault namespace create development
vault namespace create production

# Or use different Vault paths:
secret/customer-api/development/*
secret/customer-api/production/*
```

**Different roles per environment**:
```bash
# Development role (read/write)
vault write auth/oci/role/customer-api-dev \
  bound_dynamic_group_id=$DEV_DYNAMIC_GROUP_OCID \
  token_policies="customer-api-dev-policy"

# Production role (read-only)
vault write auth/oci/role/customer-api-prod \
  bound_dynamic_group_id=$PROD_DYNAMIC_GROUP_OCID \
  token_policies="customer-api-prod-policy"
```

### High Availability Setup

For production, run multiple Vault instances:

```bash
# vault.hcl for HA
storage "raft" {
  path = "/opt/vault/data"
  node_id = "vault-1"
  
  retry_join {
    leader_api_addr = "http://vault-2:8200"
  }
  retry_join {
    leader_api_addr = "http://vault-3:8200"
  }
}

ha_storage "raft" {
  path = "/opt/vault/data"
}
```

We'll cover HA in detail in Post 8!

---

## Complete Deployment Script

Here's a complete automation script:

```bash
#!/bin/bash
# deploy-vault-oci-auth.sh

set -e

echo "=== Deploying Vault with OCI Instance Principals ==="

# Configuration
VAULT_VERSION="1.15.4"
COMPARTMENT_OCID="ocid1.compartment.oc1..YOUR_COMPARTMENT_OCID"
DYNAMIC_GROUP_NAME="customer-api-instances"

# Step 1: Install Vault
echo "Step 1: Installing Vault..."
if ! command -v vault &> /dev/null; then
    sudo yum install -y yum-utils
    sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
    sudo yum -y install vault-${VAULT_VERSION}
fi

# Step 2: Configure Vault
echo "Step 2: Configuring Vault..."
sudo mkdir -p /opt/vault/data /etc/vault.d
PRIVATE_IP=$(hostname -I | awk '{print $1}')

sudo tee /etc/vault.d/vault.hcl > /dev/null << EOF
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://${PRIVATE_IP}:8200"
ui = true
log_level = "Info"
EOF

sudo chown -R vault:vault /opt/vault
sudo chmod 700 /opt/vault/data

# Step 3: Start Vault
echo "Step 3: Starting Vault..."
sudo systemctl enable vault
sudo systemctl start vault

# Wait for Vault to start
sleep 5

# Step 4: Check if already initialized
export VAULT_ADDR='http://127.0.0.1:8200'
if vault status 2>&1 | grep -q "not initialized"; then
    echo "Step 4: Initializing Vault..."
    vault operator init -key-shares=5 -key-threshold=3 > /tmp/vault-init.txt
    
    echo "⚠️  IMPORTANT: Vault keys saved to /tmp/vault-init.txt"
    echo "⚠️  Copy this file to a secure location and DELETE from server!"
    
    # Extract keys and unseal
    KEY1=$(grep 'Unseal Key 1:' /tmp/vault-init.txt | awk '{print $NF}')
    KEY2=$(grep 'Unseal Key 2:' /tmp/vault-init.txt | awk '{print $NF}')
    KEY3=$(grep 'Unseal Key 3:' /tmp/vault-init.txt | awk '{print $NF}')
    ROOT_TOKEN=$(grep 'Initial Root Token:' /tmp/vault-init.txt | awk '{print $NF}')
    
    echo "Step 5: Unsealing Vault..."
    vault operator unseal $KEY1
    vault operator unseal $KEY2
    vault operator unseal $KEY3
    
    export VAULT_TOKEN=$ROOT_TOKEN
else
    echo "Vault already initialized"
    echo "Please set VAULT_TOKEN manually"
    exit 1
fi

# Step 6: Get instance metadata
echo "Step 6: Getting instance metadata..."
TENANCY_OCID=$(curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.compartmentId')

# Step 7: Enable OCI auth
echo "Step 7: Configuring OCI authentication..."
vault auth enable oci 2>/dev/null || echo "OCI auth already enabled"

vault write auth/oci/config \
  home_tenancy_id=$TENANCY_OCID

# Step 8: Create policy
echo "Step 8: Creating application policy..."
vault policy write customer-api-policy - << EOF
path "secret/data/customer-api/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/customer-api/*" {
  capabilities = ["read", "list"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF

# Step 9: Create role
echo "Step 9: Creating OCI auth role..."
echo "⚠️  Enter your dynamic group OCID:"
read DYNAMIC_GROUP_OCID

vault write auth/oci/role/customer-api-role \
  bound_dynamic_group_id=$DYNAMIC_GROUP_OCID \
  token_policies="customer-api-policy" \
  token_ttl="1h" \
  token_max_ttl="24h"

# Step 10: Enable secrets engine
echo "Step 10: Enabling KV secrets engine..."
vault secrets enable -path=secret kv-v2 2>/dev/null || echo "KV already enabled"

# Step 11: Store sample secrets
echo "Step 11: Storing sample secrets..."
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="ChangeMe123!"

vault kv put secret/customer-api/config \
  environment="production"

# Step 12: Test authentication
echo "Step 12: Testing OCI authentication..."
if vault login -method=oci auth_type=instance role=customer-api-role; then
    echo "✅ OCI authentication successful!"
else
    echo "❌ OCI authentication failed"
    exit 1
fi

echo ""
echo "=== Deployment Complete ==="
echo ""
echo "Vault Address: http://127.0.0.1:8200"
echo "Root Token: $ROOT_TOKEN"
echo ""
echo "⚠️  IMPORTANT NEXT STEPS:"
echo "1. Copy /tmp/vault-init.txt to secure location"
echo "2. Delete /tmp/vault-init.txt from server"
echo "3. Never commit vault keys to Git"
echo "4. Consider enabling TLS for production"
echo ""
echo "Test command:"
echo "vault login -method=oci auth_type=instance role=customer-api-role"
```

Make executable and run:
```bash
chmod +x deploy-vault-oci-auth.sh
./deploy-vault-oci-auth.sh
```

---

## What's Next

🎉 **Congratulations!** You've achieved true zero-trust authentication! Let's recap what we accomplished:

✅ Deployed Vault in production mode on OCI  
✅ Configured OCI Instance Principal authentication  
✅ Set up dynamic groups and IAM policies  
✅ Created Vault roles and policies  
✅ Deployed Spring Boot with automatic Vault authentication  
✅ **Eliminated ALL stored secrets - true zero-secret architecture!**  

### Current Architecture

```
┌─────────────────────────────────────────────┐
│         Zero-Trust Architecture              │
│                                              │
│  Spring Boot Application                    │
│    ↓ (no credentials!)                      │
│  OCI Metadata Service                       │
│    ↓ (cryptographic proof)                  │
│  HashiCorp Vault                            │
│    ↓ (verifies with OCI)                    │
│  OCI IAM                                    │
│    ↓ (checks dynamic groups & policies)    │
│  Vault Issues Token                         │
│    ↓ (time-limited, policy-bound)          │
│  Application Gets Secrets                   │
│                                              │
│  🔐 No secrets stored anywhere!             │
└─────────────────────────────────────────────┘
```

### Coming Up in This Series

**Post 6: Production Deployment - Complete Stack on OCI**  
Bring everything together with production-grade deployment:
- Load balancer configuration
- Multiple application instances
- Database setup with connection pooling
- Monitoring and alerting
- Complete end-to-end testing

**Post 7: Secret Rotation and Resilience**  
Build resilient applications:
- Handling Vault downtime gracefully
- Automatic secret rotation
- Database credential rotation
- Backup and recovery strategies

**Post 8: High Availability and Scaling**  
Scale to enterprise-grade:
- Vault HA cluster with Raft storage
- Multiple application instances
- Auto-scaling configurations
- Load testing and optimization

**Post 9: CI/CD with GitHub Actions**  
Automate everything:
- Build and test pipeline
- Automated deployment to OCI
- Secret management in CI/CD
- Zero-downtime deployments

---

## Key Takeaways

🔑 **Secret Zero Paradox solved** with cryptographic identity  
🔑 **OCI Instance Principals** provide automatic, credential-free authentication  
🔑 **Dynamic groups + IAM policies** enable fine-grained access control  
🔑 **Vault OCI auth method** verifies instance identity with OCI  
🔑 **Spring Cloud Vault** seamlessly integrates with OCI authentication  
🔑 **Zero stored secrets** - no passwords, tokens, or keys anywhere  
🔑 **Automatic token renewal** keeps applications running without intervention  
🔑 **Audit trail** tracks every secret access for compliance  

---

## Real-World Production Checklist

Before going to production, ensure:

**Infrastructure**:
- [ ] Vault HA cluster (minimum 3 nodes)
- [ ] TLS enabled for all Vault connections
- [ ] Separate Vault instances from application instances
- [ ] Load balancer for Vault cluster
- [ ] Backup strategy for Vault data

**Security**:
- [ ] Root token revoked (use admin tokens instead)
- [ ] Audit logging enabled
- [ ] Network security lists restrict access
- [ ] Separate dynamic groups per environment
- [ ] Least privilege IAM policies

**Monitoring**:
- [ ] Vault health checks configured
- [ ] Alerts for Vault seal status
- [ ] Token expiration monitoring
- [ ] Secret access pattern monitoring
- [ ] Anomaly detection for unusual access

**Operations**:
- [ ] Unseal keys stored securely offline
- [ ] Disaster recovery plan documented
- [ ] Secret rotation schedule defined
- [ ] On-call runbooks created
- [ ] Team trained on Vault operations

**Application**:
- [ ] Token renewal logic tested
- [ ] Graceful handling of Vault downtime
- [ ] Connection pooling configured
- [ ] Retry logic implemented
- [ ] Logging (without exposing secrets!)

---

## Troubleshooting Quick Reference

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| 403 from metadata service | Not on OCI instance | Must run on OCI compute |
| NotAuthorizedOrNotFound | Not in dynamic group | Check matching rules |
| Invalid role | Role doesn't exist | Create role in Vault |
| Connection refused to Vault | Vault not running | Check `systemctl status vault` |
| Vault sealed | Needs unsealing | Run `vault operator unseal` 3x |
| Token expired | TTL elapsed | Should auto-renew; check logs |
| Permission denied | Policy too restrictive | Update Vault policy |
| Secrets not found | Wrong path | Verify secret path matches config |

---

## Additional Resources

- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [OCI Instance Principals](https://docs.oracle.com/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm)
- [Spring Cloud Vault OCI Auth](https://docs.spring.io/spring-cloud-vault/docs/current/reference/html/#vault.config.backends.oci)
- [Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [Vault Production Hardening](https://learn.hashicorp.com/tutorials/vault/production-hardening)

---

## Source Code Repository

Complete code for this post:

📦 **[github.com/your-username/customer-api-series](https://github.com/your-username/customer-api-series)**

**Branch**: `post-5-oci-instance-principals`

Includes:
- Vault configuration files
- OCI IAM policy templates
- Spring Boot application with OCI auth
- Deployment automation scripts
- Troubleshooting guides

```bash
git clone https://github.com/your-username/customer-api-series.git
cd customer-api-series
git checkout post-5-oci-instance-principals
```

---

## Community Feedback

**Achieved zero-trust security?** Share your success:

- 🎉 Tweet your setup: `@YourHandle #Vault #ZeroTrust #OCI #CloudSecurity`
- ⭐ Star the GitHub repository
- 💬 Comment below with questions or success stories
- 📧 Subscribe for Post 6 notification

**Questions?** Common discussion topics:
- Vault HA setup strategies
- Secret rotation best practices
- Multi-cloud Vault architectures
- Compliance requirements (SOC2, HIPAA, etc.)

---

*Previous: [Post 4: Secrets Management with HashiCorp Vault - Local Setup](#) ←*

*Next: [Post 6: Production Deployment - Complete Stack on OCI](#) →*

---

## Tags

#HashiCorpVault #ZeroTrust #OCI #InstancePrincipals #CloudSecurity #SpringBoot #SecretManagement #DevSecOps #OracleCloud #CloudNative #SecurityBestPractices #NoSecrets #Tutorial