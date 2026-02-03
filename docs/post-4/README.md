# Mastering HashiCorp Vault: Secure Secrets Management for Spring Boot

*Part 4 of 9 in the series: "Production-Ready Spring Boot on OCI with Vault"*

![HashiCorp Vault Security](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=1200&h=400&fit=crop)

## What You'll Learn

Hardcoded passwords and API keys are security nightmares waiting to happen. In this comprehensive guide, you'll learn how to use HashiCorp Vault to securely manage secrets for your Spring Boot application.

**By the end of this post, you'll be able to**:
- ✅ Understand Vault's architecture and core concepts
- ✅ Run Vault locally in development mode
- ✅ Store and retrieve secrets via CLI and API
- ✅ Create and manage Vault policies for access control
- ✅ Integrate Spring Boot with Vault using Spring Cloud Vault
- ✅ Replace hardcoded credentials with dynamic secrets
- ✅ Implement secret versioning and rollback

**Time to complete**: 60 minutes  
**Difficulty**: Intermediate  
**Prerequisites**: Posts 1-3 completed

---

## Table of Contents

1. [The Secrets Management Problem](#the-secrets-management-problem)
2. [Why HashiCorp Vault?](#why-hashicorp-vault)
3. [Vault Architecture and Concepts](#vault-architecture-and-concepts)
4. [Installing Vault Locally](#installing-vault-locally)
5. [Starting Vault in Dev Mode](#starting-vault-in-dev-mode)
6. [Vault CLI Basics](#vault-cli-basics)
7. [Storing Secrets in Vault](#storing-secrets-in-vault)
8. [Vault Policies Explained](#vault-policies-explained)
9. [Integrating Spring Boot with Vault](#integrating-spring-boot-with-vault)
10. [Testing the Integration](#testing-the-integration)
11. [Secret Versioning and Rollback](#secret-versioning-and-rollback)
12. [Best Practices](#best-practices)
13. [What's Next](#whats-next)

---

## Prerequisites

### From Previous Posts

- ✅ [Post 1: Spring Boot Customer API](link) - The application we'll secure
- ✅ [Post 3: Deployed on OCI](link) - Understanding of deployment

**You should have**:
- Spring Boot application source code
- Local development environment set up
- Basic understanding of REST APIs
- Terminal/command line familiarity

### Tools Required

- **Java 17+** (from Post 1)
- **Maven 3.8+** (from Post 1)
- **curl** or **Postman** for API testing
- **jq** (optional, for JSON parsing)

---

## The Secrets Management Problem

Let's look at how most applications handle secrets today.

### The Traditional Approach (❌ Bad)

**Hardcoded in code**:
```java
// DON'T DO THIS!
@Configuration
public class DatabaseConfig {
    private static final String DB_PASSWORD = "MyP@ssw0rd123!";
    private static final String API_KEY = "sk_live_abc123xyz789";
}
```

**Problems**:
- 🔴 Secrets in version control (Git history forever)
- 🔴 Anyone with code access has production secrets
- 🔴 Can't rotate without code changes
- 🔴 Audit trail? What audit trail?

**Environment variables** (slightly better):
```bash
export DB_PASSWORD="MyP@ssw0rd123!"
export API_KEY="sk_live_abc123xyz789"
```

**Problems**:
- 🟡 Still in plaintext somewhere (.bashrc, .env files)
- 🟡 Process inspection can reveal them
- 🟡 Difficult to rotate across multiple servers
- 🟡 No access control or audit logging

**Configuration files**:
```properties
# application.properties
spring.datasource.password=MyP@ssw0rd123!
stripe.api.key=sk_live_abc123xyz789
```

**Problems**:
- 🔴 Often committed to Git
- 🔴 Plaintext on disk
- 🔴 No encryption at rest
- 🔴 Rotation requires file updates everywhere

### The Reality Check

**What happens when**:
- An employee leaves?
- A laptop is stolen?
- A Git repository is leaked?
- Compliance audit happens?
- You need to rotate credentials?

**Answer**: Panic, scrambling, and security incidents.

---

## Why HashiCorp Vault?

Vault solves these problems with a centralized secrets management system.

### What is Vault?

**HashiCorp Vault** is a tool for securely accessing secrets. A secret is anything you want to tightly control access to:
- Database credentials
- API keys
- TLS certificates
- Encryption keys
- Cloud provider credentials

### Key Features

**🔐 Centralized Storage**
- Single source of truth for all secrets
- Encrypted at rest and in transit
- High availability and redundancy

**🔑 Dynamic Secrets**
- Generate credentials on-demand
- Automatic expiration and renewal
- Unique credentials per client

**🛡️ Access Control**
- Fine-grained policies
- Multiple authentication methods
- Audit logging of all access

**🔄 Secret Rotation**
- Automated rotation
- Zero-downtime updates
- Versioned secrets with rollback

**📊 Audit Trail**
- Who accessed what and when
- Failed access attempts
- Complete compliance trail

### Vault vs. Alternatives

| Feature | Vault | AWS Secrets Manager | Azure Key Vault | Kubernetes Secrets |
|---------|-------|---------------------|-----------------|-------------------|
| Multi-cloud | ✅ | ❌ | ❌ | ✅ |
| Dynamic secrets | ✅ | ❌ | ❌ | ❌ |
| Encryption as a service | ✅ | ❌ | Limited | ❌ |
| Fine-grained policies | ✅ | Limited | Limited | Limited |
| Open source | ✅ | ❌ | ❌ | ✅ |
| On-premise | ✅ | ❌ | ❌ | ✅ |

**Why Vault for our series?**
- Works on OCI (cloud-agnostic)
- Perfect for learning secrets management
- Industry standard
- Excellent OCI integration via Instance Principals

---

## Vault Architecture and Concepts

Before installing, let's understand how Vault works.

### Core Components

```
┌─────────────────────────────────────────────┐
│              Vault Server                    │
│                                              │
│  ┌────────────┐  ┌──────────────────────┐  │
│  │   API      │  │   Storage Backend    │  │
│  │  (HTTP)    │←→│  (Encrypted Data)    │  │
│  └────────────┘  └──────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │        Secrets Engines                 │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │ │
│  │  │ KV   │ │ DB   │ │ AWS  │ │ PKI  │ │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │        Auth Methods                     │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │ │
│  │  │Token │ │ LDAP │ │ OCI  │ │ K8s  │ │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### Key Concepts

**Secrets Engine**:
- Plugin that stores, generates, or encrypts data
- Examples: Key/Value, Database, AWS, PKI
- We'll use **KV (Key/Value) v2** for static secrets

**Auth Method**:
- How clients authenticate to Vault
- Examples: Tokens, Username/Password, OCI, Kubernetes
- We'll use **Token** (Post 4) and **OCI** (Post 5)

**Policy**:
- Rules that govern access to secrets
- Written in HCL (HashiCorp Configuration Language)
- Defines who can do what

**Token**:
- Client credential for accessing Vault
- Can have expiration, limited uses, and policies
- Root token = admin access (don't use in production!)

**Path**:
- Secrets are stored at paths like filesystems
- Example: `secret/data/customer-api/database`
- Policies control access to paths

### Vault Workflow

```
1. Application authenticates to Vault
   ↓
2. Vault verifies credentials (auth method)
   ↓
3. Vault checks policies (authorization)
   ↓
4. Vault returns a token
   ↓
5. Application uses token to read/write secrets
   ↓
6. Token expires → Application re-authenticates
```

---

## Installing Vault Locally

### Option 1: Binary Installation (Recommended)

**macOS** (via Homebrew):
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault

# Verify installation
vault version
# Expected: Vault v1.15.x or higher
```

**Linux** (Ubuntu/Debian):
```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install
sudo apt update && sudo apt install vault

# Verify
vault version
```

**Linux** (RHEL/Oracle Linux):
```bash
# Add HashiCorp repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Vault
sudo yum -y install vault

# Verify
vault version
```

**Windows** (via Chocolatey):
```powershell
choco install vault

# Verify
vault version
```

### Option 2: Docker Container

```bash
# Pull Vault image
docker pull hashicorp/vault:latest

# Run in dev mode (we'll use this approach)
docker run -d \
  --name vault-dev \
  --cap-add=IPC_LOCK \
  -p 8200:8200 \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  hashicorp/vault:latest

# Verify
docker ps | grep vault-dev
```

**For this tutorial, we'll use binary installation** for simplicity.

### Enable Command Autocompletion

```bash
# For bash
vault -autocomplete-install
source ~/.bashrc

# For zsh
vault -autocomplete-install
source ~/.zshrc
```

---

## Starting Vault in Dev Mode

Vault has two modes:
- **Dev Mode**: For learning and testing (NOT for production!)
- **Production Mode**: Sealed, persisted storage, HA

We'll start with **Dev Mode** for learning.

### Start Vault Dev Server

```bash
# Start Vault in dev mode
vault server -dev

# You'll see output like:
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.21.5
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.15.4

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: abcdef1234567890...
Root Token: hvs.ABCD1234EFGH5678

Development mode should NOT be used in production installations!
```

**⚠️ IMPORTANT**: Copy the **Root Token** (starts with `hvs.`). You'll need it!

### Configure Environment (New Terminal)

Keep the Vault server running in one terminal, open a **new terminal** for commands:

```bash
# Set Vault address
export VAULT_ADDR='http://127.0.0.1:8200'

# Set root token (replace with YOUR token)
export VAULT_TOKEN='hvs.ABCD1234EFGH5678'

# Verify connection
vault status
```

**Expected output**:
```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.15.4
Storage Type    inmem
Cluster Name    vault-cluster-abc123
Cluster ID      abcd-efgh-1234-5678
HA Enabled      false
```

**Key points**:
- **Sealed: false** - Vault is ready to use
- **Storage Type: inmem** - Data is in memory (lost on restart)
- **HA Enabled: false** - Single instance (dev mode)

---

## Vault CLI Basics

Let's explore Vault's command-line interface.

### Essential Commands

```bash
# Get help
vault --help

# Get help for a specific command
vault kv --help

# Check server status
vault status

# List enabled secrets engines
vault secrets list

# List enabled auth methods
vault auth list

# List policies
vault policy list

# Read your token info
vault token lookup
```

### Understanding Paths

Vault organizes everything as paths:

```
/auth/          - Authentication methods
/secret/        - Secrets engines
/sys/           - System backend (admin operations)
```

**Example paths**:
```
secret/data/customer-api/database
secret/data/customer-api/stripe
secret/metadata/customer-api/database
```

### KV Secrets Engine v2

Dev mode automatically enables KV v2 at `secret/`:

```bash
# List what's in secret/
vault kv list secret/

# Currently empty, you'll see:
No value found at secret/metadata/
```

---

## Storing Secrets in Vault

Let's store secrets for our Customer API.

### Create a Secret - Database Credentials

```bash
# Store database credentials
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="SuperSecret123!"

# Expected output:
===== Secret Path =====
secret/data/customer-api/database

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-12T10:00:00.123456Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

### Read the Secret

```bash
# Read the secret
vault kv get secret/customer-api/database

# Output:
===== Secret Path =====
secret/data/customer-api/database

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-12T10:00:00.123456Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
database    customerdb
host        localhost
password    SuperSecret123!
port        5432
username    apiuser
```

**JSON output** (easier for parsing):
```bash
vault kv get -format=json secret/customer-api/database | jq .

# Or just the data:
vault kv get -format=json secret/customer-api/database | jq -r .data.data
```

### Store More Secrets

```bash
# API Keys
vault kv put secret/customer-api/api-keys \
  stripe_key="sk_test_abc123xyz789" \
  sendgrid_key="SG.abc123.xyz789" \
  maps_api_key="AIza..."

# Application Configuration
vault kv put secret/customer-api/config \
  jwt_secret="your-256-bit-secret-key-here" \
  encryption_key="another-secret-key" \
  environment="development"

# Verify all secrets
vault kv list secret/customer-api/
```

**Output**:
```
Keys
----
api-keys
config
database
```

### Update a Secret

```bash
# Update just one field (other fields are preserved)
vault kv patch secret/customer-api/database \
  password="EvenBetterPassword456!"

# This creates version 2 of the secret
```

### Delete a Secret

```bash
# Soft delete (can be recovered)
vault kv delete secret/customer-api/database

# Undelete (restore)
vault kv undelete -versions=2 secret/customer-api/database

# Hard delete (permanent)
vault kv destroy -versions=2 secret/customer-api/database

# Delete all versions and metadata
vault kv metadata delete secret/customer-api/database
```

For now, let's recreate our database secret:
```bash
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="SuperSecret123!"
```

---

## Vault Policies Explained

Policies control who can access what in Vault.

### Understanding Policy Syntax

Policies are written in HCL (HashiCorp Configuration Language):

```hcl
# Policy structure
path "secret/data/customer-api/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

**Capabilities**:
- `create` - Create new secrets
- `read` - Read existing secrets
- `update` - Update existing secrets (also used for create)
- `delete` - Delete secrets
- `list` - List secrets at a path
- `sudo` - Admin operations
- `deny` - Explicitly deny (highest priority)

### Create an Application Policy

Create a file `customer-api-policy.hcl`:

```hcl
# Read-only access to customer-api secrets
path "secret/data/customer-api/*" {
  capabilities = ["read", "list"]
}

# Allow reading secret metadata
path "secret/metadata/customer-api/*" {
  capabilities = ["read", "list"]
}

# Allow token self-renewal
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Allow token self-lookup
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
```

**Upload the policy**:
```bash
vault policy write customer-api-policy customer-api-policy.hcl

# Verify
vault policy read customer-api-policy
```

### Create a Token with the Policy

```bash
# Create a token for the application
vault token create \
  -policy=customer-api-policy \
  -ttl=24h \
  -renewable=true \
  -display-name="customer-api-app"

# Output:
Key                  Value
---                  -----
token                hvs.CAESIJ8...
token_accessor       abc123...
token_duration       24h
token_renewable      true
token_policies       ["customer-api-policy" "default"]
identity_policies    []
policies             ["customer-api-policy" "default"]
```

**Save this token** - we'll use it in Spring Boot!

### Test the Policy

```bash
# Save the app token
export APP_TOKEN="hvs.CAESIJ8..."

# Try to read a secret (should work)
VAULT_TOKEN=$APP_TOKEN vault kv get secret/customer-api/database
# ✓ Works!

# Try to create a secret (should fail - read-only policy)
VAULT_TOKEN=$APP_TOKEN vault kv put secret/customer-api/test foo=bar
# ✗ Error: 1 error occurred:
#     * permission denied

# Try to read a different path (should fail)
VAULT_TOKEN=$APP_TOKEN vault kv get secret/other-app/secrets
# ✗ Error: permission denied
```

**Perfect!** The policy is working as expected.

### Policy Best Practices

**Principle of Least Privilege**:
```hcl
# Bad - too permissive
path "secret/*" {
  capabilities = ["create", "read", "update", "delete"]
}

# Good - specific and minimal
path "secret/data/customer-api/*" {
  capabilities = ["read"]
}
```

**Use Wildcards Carefully**:
```hcl
# Allow read for all customer-api secrets
path "secret/data/customer-api/*" {
  capabilities = ["read"]
}

# But deny access to sensitive sub-path
path "secret/data/customer-api/production/*" {
  capabilities = ["deny"]
}
```

**Template Policies** (for multiple similar apps):
```hcl
# This will be filled in by Vault
path "secret/data/{{identity.entity.metadata.app_name}}/*" {
  capabilities = ["read"]
}
```

---

## Integrating Spring Boot with Vault

Now let's connect our Customer API to Vault!

### Step 1: Add Spring Cloud Vault Dependency

Update `pom.xml`:

```xml
<dependencies>
    <!-- Existing dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Add Spring Cloud Vault -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>
    
    <!-- PostgreSQL Driver (replacing H2 for real DB) -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>

<!-- Add Spring Cloud BOM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Step 2: Configure Spring Boot for Vault

Create `src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: customer-api
  
  # Vault configuration
  cloud:
    vault:
      uri: http://127.0.0.1:8200
      token: hvs.CAESIJ8...  # Use your APP_TOKEN
      kv:
        enabled: true
        backend: secret
        application-name: customer-api
  
  # Import configuration from Vault
  config:
    import: vault://
  
  # Database configuration (values will come from Vault)
  datasource:
    url: jdbc:postgresql://${database.host}:${database.port}/${database.database}
    username: ${database.username}
    password: ${database.password}
    driver-class-name: org.postgresql.Driver
  
  # JPA configuration
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    database-platform: org.hibernate.dialect.PostgreSQLDialect

# Server configuration
server:
  port: 8080

# Logging
logging:
  level:
    org.springframework.cloud.vault: DEBUG
    com.example.customerapi: DEBUG
```

### Step 3: Use Secrets in Your Code

Spring automatically injects Vault secrets as properties!

**Example - Using API Keys**:

```java
package com.example.customerapi.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class PaymentService {

    @Value("${stripe_key}")
    private String stripeApiKey;
    
    @Value("${sendgrid_key}")
    private String sendgridApiKey;
    
    public void processPayment(Long invoiceId, Double amount) {
        log.info("Processing payment for invoice: {}", invoiceId);
        
        // Use Stripe API with the key from Vault
        // stripeApiKey is automatically injected!
        log.debug("Using Stripe key: {}...", 
            stripeApiKey.substring(0, 10));
        
        // Payment processing logic here
    }
    
    public void sendInvoiceEmail(String email, String invoiceNumber) {
        log.info("Sending invoice {} to {}", invoiceNumber, email);
        
        // Use SendGrid with key from Vault
        log.debug("Using SendGrid key: {}...", 
            sendgridApiKey.substring(0, 10));
        
        // Email sending logic here
    }
}
```

**Example - Using JWT Secret**:

```java
package com.example.customerapi.security;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

@Component
public class JwtTokenProvider {

    @Value("${jwt_secret}")
    private String jwtSecret;
    
    @Value("${environment}")
    private String environment;
    
    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public void logEnvironment() {
        System.out.println("Running in: " + environment);
    }
}
```

### Step 4: Run PostgreSQL Locally

Before testing, start a PostgreSQL database:

```bash
# Using Docker
docker run -d \
  --name customer-api-db \
  -e POSTGRES_DB=customerdb \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=SuperSecret123! \
  -p 5432:5432 \
  postgres:15-alpine

# Verify it's running
docker ps | grep customer-api-db
```

Or use H2 for now (in-memory):

```yaml
# In application.yml, comment out PostgreSQL and use:
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
```

---

## Testing the Integration

### Start the Application

```bash
# Make sure Vault is running in dev mode
# Make sure PostgreSQL is running (or using H2)

# Start Spring Boot
./mvnw spring-boot:run
```

**Watch the logs** - you should see:

```
2026-01-12 11:00:00.123  INFO --- Vault location [vault://secret/customer-api]
2026-01-12 11:00:00.456  INFO --- Fetching config from server at: http://127.0.0.1:8200
2026-01-12 11:00:01.789  INFO --- Located property source: VaultPropertySource
2026-01-12 11:00:02.012  INFO --- Using database: customerdb
2026-01-12 11:00:02.345  INFO --- Started CustomerApiApplication in 5.678 seconds
```

**✅ Success indicators**:
- "Located property source: VaultPropertySource"
- No errors about missing properties
- Application starts successfully

### Verify Secrets are Loaded

```bash
# Test the API
curl http://localhost:8080/api/v1/customers

# Create a customer (this will use DB credentials from Vault)
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -d '{
    "email": "vault@test.com",
    "companyName": "Vault Corp",
    "contactName": "Secure User",
    "phone": "+1-555-VAULT"
  }'

# Check the database
docker exec -it customer-api-db psql -U apiuser -d customerdb -c "SELECT * FROM customers;"
```

### Test with Different Secrets

Let's verify all secrets are accessible:

Create a test controller:

```java
package com.example.customerapi.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/config")
public class ConfigTestController {

    @Value("${database.host}")
    private String dbHost;
    
    @Value("${database.database}")
    private String dbName;
    
    @Value("${stripe_key}")
    private String stripeKey;
    
    @Value("${environment}")
    private String environment;
    
    @GetMapping("/test")
    public Map<String, String> testVaultIntegration() {
        Map<String, String> config = new HashMap<>();
        config.put("dbHost", dbHost);
        config.put("dbName", dbName);
        config.put("stripeKeyPrefix", stripeKey.substring(0, 10) + "...");
        config.put("environment", environment);
        config.put("vaultStatus", "✓ Connected");
        return config;
    }
}
```

Test it:
```bash
curl http://localhost:8080/api/v1/config/test

# Expected output:
{
  "dbHost": "localhost",
  "dbName": "customerdb",
  "stripeKeyPrefix": "sk_test_ab...",
  "environment": "development",
  "vaultStatus": "✓ Connected"
}
```

---

## Secret Versioning and Rollback

KV v2 keeps versions of your secrets - let's explore this feature.

### View Secret History

```bash
# Get metadata (shows all versions)
vault kv metadata get secret/customer-api/database

# Output shows version history:
========== Metadata ==========
Key                     Value
---                     -----
created_time            2026-01-12T10:00:00.123Z
current_version         3
max_versions            10
oldest_version          0
updated_time            2026-01-12T11:30:00.456Z
versions                map[1:map[created_time:...] 2:map[created_time:...] 3:map[created_time:...]]
```

### Update a Secret (Creates New Version)

```bash
# Update the database password
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="NewPassword789!"

# This creates version 4
```

### Read Specific Versions

```bash
# Read current version (latest)
vault kv get secret/customer-api/database

# Read version 3 (previous password)
vault kv get -version=3 secret/customer-api/database

# Read version 2
vault kv get -version=2 secret/customer-api/database
```

### Rollback to Previous Version

```bash
# Get the old secret data
OLD_DATA=$(vault kv get -version=3 -format=json secret/customer-api/database | jq -r .data.data)

# Write it back (creates a new version with old data)
echo $OLD_DATA | vault kv put secret/customer-api/database -

# Or manually:
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="SuperSecret123!"  # Back to version 3's password
```

### Configure Version Limits

```bash
# Set max versions to keep (default is 10)
vault kv metadata put \
  -max-versions=5 \
  secret/customer-api/database

# After 5 versions, oldest ones are deleted automatically
```

### Delete Specific Versions

```bash
# Soft delete version 2 (can be undeleted)
vault kv delete -versions=2 secret/customer-api/database

# Undelete it
vault kv undelete -versions=2 secret/customer-api/database

# Permanently destroy version 2
vault kv destroy -versions=2 secret/customer-api/database

# Destroy multiple versions
vault kv destroy -versions=2,3,4 secret/customer-api/database
```

---

## Best Practices

### 1. Never Store Vault Tokens in Code

❌ **Bad**:
```yaml
# application.yml
spring:
  cloud:
    vault:
      token: hvs.CAESIJ8...  # Hardcoded token!
```

✅ **Good**:
```bash
# Use environment variable
export SPRING_CLOUD_VAULT_TOKEN=hvs.CAESIJ8...

# Or in application.yml:
spring:
  cloud:
    vault:
      token: ${VAULT_TOKEN}  # Read from environment
```

### 2. Use Short-Lived Tokens

```bash
# Create token with TTL
vault token create \
  -policy=customer-api-policy \
  -ttl=1h \
  -renewable=true

# Spring Cloud Vault automatically renews tokens!
```

### 3. Organize Secrets Hierarchically

✅ **Good structure**:
```
secret/
  ├── customer-api/
  │   ├── database
  │   ├── api-keys
  │   ├── config
  │   └── production/
  │       ├── database
  │       └── api-keys
  └── other-app/
      └── ...
```

❌ **Bad structure**:
```
secret/
  ├── db_password
  ├── stripe_key
  ├── jwt_secret
  └── ...  (flat, no organization)
```

### 4. Use Different Policies Per Environment

```hcl
# development-policy.hcl
path "secret/data/customer-api/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# production-policy.hcl (read-only)
path "secret/data/customer-api/production/*" {
  capabilities = ["read"]
}
```

### 5. Enable Audit Logging

```bash
# Enable file audit (in production mode)
vault audit enable file file_path=/var/log/vault_audit.log

# All secret access is logged!
```

### 6. Rotate Secrets Regularly

```bash
# Script to rotate database password
#!/bin/bash

# Generate new password
NEW_PASSWORD=$(openssl rand -base64 32)

# Update in Vault
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="$NEW_PASSWORD"

# Update actual database
psql -c "ALTER USER apiuser WITH PASSWORD '$NEW_PASSWORD';"

# Restart application to pick up new password
kubectl rollout restart deployment customer-api
```

### 7. Use Vault Agent for Production

Instead of tokens in environment variables, use Vault Agent:

```hcl
# agent-config.hcl
vault {
  address = "https://vault.example.com:8200"
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
      path = "/vault/secrets/.token"
    }
  }
}

template {
  source      = "/vault/templates/application.properties.tpl"
  destination = "/app/config/application.properties"
}
```

We'll cover this in detail in Post 6!

---

## Troubleshooting

### Issue: "connection refused" to Vault

**Problem**: Spring Boot can't reach Vault

**Check**:
```bash
# Verify Vault is running
vault status

# Verify VAULT_ADDR
echo $VAULT_ADDR
# Should be: http://127.0.0.1:8200

# Test connection
curl http://127.0.0.1:8200/v1/sys/health
```

**Solution**:
```yaml
# In application.yml
spring:
  cloud:
    vault:
      uri: http://127.0.0.1:8200  # Correct address
      # NOT https, NOT localhost:8200
```

### Issue: "permission denied" reading secrets

**Problem**: Token doesn't have access

**Debug**:
```bash
# Check token policies
vault token lookup $APP_TOKEN

# Test access manually
VAULT_TOKEN=$APP_TOKEN vault kv get secret/customer-api/database
```

**Solution**: Update policy to grant access:
```hcl
path "secret/data/customer-api/*" {
  capabilities = ["read", "list"]  # Add "read"!
}
```

### Issue: "secret not found"

**Problem**: Secret doesn't exist or wrong path

**Debug**:
```bash
# List secrets
vault kv list secret/customer-api/

# Check exact path
vault kv get secret/customer-api/database
```

**Solution**: Ensure path matches exactly:
```yaml
# Spring Cloud Vault looks for:
# secret/data/{application-name}/*

spring:
  application:
    name: customer-api  # Must match Vault path!
  cloud:
    vault:
      kv:
        application-name: customer-api  # Override if different
```

### Issue: Application starts but properties are null

**Problem**: Vault integration not configured correctly

**Check logs** for:
```
INFO --- Fetching config from server at: http://127.0.0.1:8200
INFO --- Located property source: VaultPropertySource
```

If missing:
```yaml
# Ensure you have:
spring:
  config:
    import: vault://  # This is REQUIRED!
```

### Issue: Token expired

**Problem**: Token TTL ran out

**Check**:
```bash
vault token lookup $APP_TOKEN
# Look at: ttl, renewable

# If expired:
# Error: permission denied
```

**Solution**:
```bash
# Create renewable token
vault token create \
  -policy=customer-api-policy \
  -ttl=24h \
  -renewable=true \
  -period=24h  # Auto-renew every 24h
```

Spring Cloud Vault automatically renews tokens!

### Issue: Dev mode data lost after restart

**Problem**: Vault dev mode uses in-memory storage

**This is expected!** Dev mode data is not persisted.

**Solution for persistence**:
Use production mode with file storage (covered in Post 5).

---

## Advanced Configuration

### Multiple Vault Paths

Read from multiple paths in Vault:

```yaml
spring:
  cloud:
    vault:
      kv:
        enabled: true
        backend: secret
        application-name: customer-api
        profiles: production  # Also read from production/
```

This reads from:
- `secret/customer-api/`
- `secret/customer-api/production/`

### Generic Backend

For non-KV secrets engines:

```yaml
spring:
  cloud:
    vault:
      generic:
        enabled: true
        backend: custom-engine
        application-name: customer-api
```

### Custom Secret Resolution

```java
@Configuration
public class VaultConfig {
    
    @Bean
    public VaultPropertySource customVaultPropertySource(
            VaultTemplate vaultTemplate) {
        
        return new VaultPropertySource(
            vaultTemplate,
            "secret/custom-path"
        );
    }
}
```

### Programmatic Secret Access

```java
@Service
public class SecretService {
    
    @Autowired
    private VaultTemplate vaultTemplate;
    
    public String getSecret(String path, String key) {
        VaultResponse response = vaultTemplate
            .read("secret/data/" + path);
        
        if (response != null && response.getData() != null) {
            return (String) response.getData().get(key);
        }
        return null;
    }
    
    public void writeSecret(String path, Map<String, Object> data) {
        vaultTemplate.write("secret/data/" + path, data);
    }
}
```

---

## Vault UI (Bonus)

Vault has a web UI you can enable!

### Access the UI

Vault dev mode automatically enables the UI:

1. Open browser: http://127.0.0.1:8200/ui
2. Enter your **Root Token** (from dev mode startup)
3. Explore the UI!

**What you can do**:
- Browse secrets
- Create/update/delete secrets
- Manage policies
- View audit logs
- Configure auth methods
- Monitor Vault health

### UI Tour

**Secrets Engine**:
- Navigate to "Secrets" → "secret/"
- Click "customer-api/"
- See all your secrets
- Click to view/edit

**Policies**:
- Navigate to "Policies"
- Click "customer-api-policy"
- View/edit policy rules

**Access**:
- Navigate to "Access" → "Auth Methods"
- See enabled auth methods
- Configure new methods

**Note**: UI is great for learning, but use CLI/API for automation!

---

## Performance Considerations

### Caching Secrets

Spring Cloud Vault caches secrets by default:

```yaml
spring:
  cloud:
    vault:
      config:
        lifecycle:
          enabled: true  # Enable lease renewal
          min-renewal: 10s
          expiry-threshold: 60s
```

### Connection Pooling

```yaml
spring:
  cloud:
    vault:
      config:
        timeout: 5s
        read-timeout: 15s
      session:
        timeout: 5s
```

### Async Secret Loading

```java
@Service
public class AsyncSecretService {
    
    @Async
    public CompletableFuture<String> getSecretAsync(String path) {
        // Load secret asynchronously
        // Useful for multiple secrets in parallel
    }
}
```

---

## What's Next

Congratulations! You've mastered Vault fundamentals and integrated it with Spring Boot. Here's what we accomplished:

✅ Understood the secrets management problem  
✅ Installed and configured HashiCorp Vault  
✅ Stored and retrieved secrets via CLI  
✅ Created fine-grained access policies  
✅ Integrated Spring Boot with Vault  
✅ Replaced hardcoded credentials with Vault secrets  
✅ Learned secret versioning and rollback  
✅ Implemented security best practices  

### Current Architecture

```
Spring Boot Application
    ↓ (reads secrets via Spring Cloud Vault)
Vault Server (Dev Mode)
    ├── secret/customer-api/database
    ├── secret/customer-api/api-keys
    └── secret/customer-api/config
```

**Limitation**: Using token authentication (static credential)

### Coming Up in This Series

**Post 5: Zero-Trust Authentication - OCI Instance Principals with Vault** ⭐  
The game-changer! We'll deploy Vault to OCI and configure it to use OCI Instance Principals for authentication. This eliminates the need for any tokens or passwords - true zero-secret architecture!

**What you'll learn**:
- Deploy Vault in production mode on OCI
- Configure OCI auth method in Vault
- Set up dynamic groups and IAM policies
- Enable instance principal authentication
- Deploy Spring Boot with automatic Vault authentication
- **No tokens, no passwords, no secrets!**

**Post 6: Production Deployment with Vault + Spring Boot on OCI**  
Bring it all together - deploy the complete stack to OCI with enterprise-grade security.

**Post 7: Secret Rotation and Resilience Patterns**  
Handle Vault downtime gracefully, implement automatic secret rotation, and build resilient applications.

---

## Complete Code Examples

### Full application.yml

```yaml
spring:
  application:
    name: customer-api
  
  # Vault configuration
  cloud:
    vault:
      uri: ${VAULT_ADDR:http://127.0.0.1:8200}
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        application-name: customer-api
      config:
        lifecycle:
          enabled: true
          min-renewal: 10s
          expiry-threshold: 1m
  
  # Import from Vault
  config:
    import: vault://
  
  # Database (values from Vault)
  datasource:
    url: jdbc:postgresql://${database.host:localhost}:${database.port:5432}/${database.database:customerdb}
    username: ${database.username}
    password: ${database.password}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
  
  # JPA
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    properties:
      hibernate:
        format_sql: true

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

### Complete Policy File

```hcl
# customer-api-policy.hcl

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

# Revoke own token (for clean shutdown)
path "auth/token/revoke-self" {
  capabilities = ["update"]
}
```

### Startup Script

```bash
#!/bin/bash
# start-local-dev.sh

set -e

echo "Starting local development environment..."

# Start Vault in dev mode (background)
vault server -dev -dev-root-token-id=myroot > vault.log 2>&1 &
VAULT_PID=$!
echo "Vault started with PID: $VAULT_PID"

# Wait for Vault to be ready
sleep 3

# Configure Vault
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='myroot'

echo "Creating secrets..."
vault kv put secret/customer-api/database \
  host="localhost" \
  port="5432" \
  database="customerdb" \
  username="apiuser" \
  password="SuperSecret123!"

vault kv put secret/customer-api/api-keys \
  stripe_key="sk_test_abc123" \
  sendgrid_key="SG.abc123"

vault kv put secret/customer-api/config \
  jwt_secret="my-secret-key" \
  encryption_key="encryption-key" \
  environment="development"

echo "Creating policy..."
vault policy write customer-api-policy - <<EOF
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

echo "Creating app token..."
APP_TOKEN=$(vault token create \
  -policy=customer-api-policy \
  -ttl=24h \
  -renewable=true \
  -format=json | jq -r .auth.client_token)

echo "App token: $APP_TOKEN"

# Start PostgreSQL
echo "Starting PostgreSQL..."
docker run -d \
  --name customer-api-db \
  -e POSTGRES_DB=customerdb \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=SuperSecret123! \
  -p 5432:5432 \
  postgres:15-alpine

# Wait for PostgreSQL
sleep 5

echo ""
echo "=== Environment Ready ==="
echo "Vault UI: http://127.0.0.1:8200/ui"
echo "Vault Token: myroot"
echo "App Token: $APP_TOKEN"
echo ""
echo "Export these variables:"
echo "export VAULT_ADDR='http://127.0.0.1:8200'"
echo "export VAULT_TOKEN='$APP_TOKEN'"
echo ""
echo "Start Spring Boot:"
echo "./mvnw spring-boot:run"
```

Make executable and run:
```bash
chmod +x start-local-dev.sh
./start-local-dev.sh
```

---

## Quick Reference Commands

```bash
# Vault Server
vault server -dev                          # Start dev mode
vault status                               # Check status
vault operator seal                        # Seal Vault
vault operator unseal                      # Unseal Vault

# Secrets
vault kv put secret/path key=value         # Create/update
vault kv get secret/path                   # Read
vault kv get -version=2 secret/path        # Read specific version
vault kv list secret/                      # List secrets
vault kv delete secret/path                # Soft delete
vault kv undelete -versions=2 secret/path  # Undelete
vault kv destroy -versions=2 secret/path   # Hard delete
vault kv metadata get secret/path          # Get metadata

# Policies
vault policy write name policy.hcl         # Create policy
vault policy read name                     # Read policy
vault policy list                          # List policies
vault policy delete name                   # Delete policy

# Tokens
vault token create -policy=name            # Create token
vault token lookup                         # Lookup own token
vault token lookup TOKEN                   # Lookup specific token
vault token renew                          # Renew own token
vault token revoke TOKEN                   # Revoke token

# Auth Methods
vault auth list                            # List auth methods
vault auth enable oci                      # Enable auth method
vault auth disable oci                     # Disable auth method
```

---

## Resources and Further Reading

- [Vault Documentation](https://www.vaultproject.io/docs)
- [Spring Cloud Vault Reference](https://docs.spring.io/spring-cloud-vault/docs/current/reference/html/)
- [Vault Best Practices](https://learn.hashicorp.com/collections/vault/best-practices)
- [KV Secrets Engine v2](https://www.vaultproject.io/docs/secrets/kv/kv-v2)
- [Vault Policies](https://www.vaultproject.io/docs/concepts/policies)

---

## Community and Support

**Found this helpful?**
- ⭐ Star the [GitHub repository](https://github.com/your-username/customer-api-series)
- 💬 Ask questions in the comments below
- 🐦 Share on Twitter with #Vault #SpringBoot #Security
- 📧 Subscribe for Post 5 notifications

**Having issues?**
- Check the [Troubleshooting](#troubleshooting) section
- Review application logs: `./mvnw spring-boot:run`
- Review Vault logs: `vault.log` or `docker logs vault-dev`
- Comment below with your specific error

---

## Key Takeaways

🔑 **Vault centralizes secrets management** across all applications  
🔑 **KV v2 provides versioning** for safe secret rotation  
🔑 **Policies enable fine-grained access control** following least privilege  
🔑 **Spring Cloud Vault seamlessly integrates** with minimal code changes  
🔑 **Dev mode is great for learning** but never use in production  
🔑 **Token authentication works** but instance principals are better (Post 5!)  
🔑 **Secret rotation is built-in** with version history and rollback  

---

*Previous: [Post 3: Containerizing and Deploying Spring Boot on OCI](#) ←*

*Next: [Post 5: Zero-Trust Authentication - OCI Instance Principals with Vault](#) →*

---

## Tags

#HashiCorpVault #SpringBoot #Security #SecretsManagement #SpringCloud #Java #DevOps #CloudSecurity #BestPractices #Tutorial #Vault #SecureConfiguration