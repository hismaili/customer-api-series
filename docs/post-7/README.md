# Post 7: Production Hardening & Scaling Guide

**Building Enterprise-Grade Resilience for customer-api-service**

---

**📚 Series: Production-Ready Spring Boot on OCI with Vault**

**Previous:** Post 6 - Production Deployment | **Next:** Post 8 - CI/CD Pipeline with GitHub Actions

---

## Introduction

Congratulations! Your `customer-api-service` is running in production on OCI, secured with HashiCorp Vault and OCI authentication. But deploying to production is just the beginning. To truly be production-ready, your application needs to handle the realities of operating in a 24/7 environment: network failures, service outages, traffic spikes, security threats, and the inevitable unexpected issues.

In this guide, we'll transform your deployment from functional to enterprise-grade by implementing security hardening, resilience patterns, high availability, and performance optimization. These aren't just nice-to-haves—they're essential practices that separate hobby projects from production systems that businesses depend on.

### What You'll Learn

- Harden Vault for production with TLS encryption and seal configuration
- Implement secret rotation strategies for zero-downtime updates
- Build resilience patterns to handle Vault downtime gracefully
- Configure Vault Agent for advanced caching and authentication
- Set up high availability for both Vault and your application
- Optimize performance with connection pooling and caching
- Implement monitoring, alerting, and disaster recovery procedures

### Prerequisites

Before diving into production hardening, ensure you have:

- Completed Posts 1-6 with `customer-api-service` deployed on OCI
- Vault running with OCI authentication configured
- OCI CLI configured with appropriate permissions
- Basic understanding of TLS/SSL certificates
- Access to create OCI load balancers and additional compute instances

---

## 1. Security Hardening

Running Vault in dev mode is convenient for testing, but it's completely unsuitable for production. Dev mode stores everything in memory, has no encryption, and loses all data on restart. Let's fix that.

### 1.1 Production Mode with TLS

First, we need to configure Vault for production mode with persistent storage and TLS encryption.

#### Generate TLS Certificates

For production, you should use certificates from a trusted CA like Let's Encrypt. For this guide, we'll use self-signed certificates:

```bash
# Generate private key
openssl genrsa -out vault-key.pem 2048

# Generate certificate signing request
openssl req -new -key vault-key.pem -out vault.csr \
  -subj "/C=US/ST=State/L=City/O=Org/CN=vault.example.com"

# Self-sign the certificate (valid for 365 days)
openssl x509 -req -in vault.csr -signkey vault-key.pem \
  -out vault-cert.pem -days 365
```

#### Create Production Vault Configuration

Create `/etc/vault.d/vault.hcl` with the following configuration:

```hcl
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault.d/vault-cert.pem"
  tls_key_file  = "/etc/vault.d/vault-key.pem"
}

api_addr = "https://vault.example.com:8200"
cluster_addr = "https://vault.example.com:8201"
ui = true
log_level = "INFO"
```

> ⚠️ **Important:** For production, use integrated storage (Raft) instead of file storage for better performance and HA support. File storage is shown here for simplicity.

### 1.2 Initialize and Unseal Vault

Start Vault and initialize it:

```bash
# Start Vault
sudo systemctl start vault

# Set Vault address
export VAULT_ADDR='https://vault.example.com:8200'

# Initialize Vault (save the output securely!)
vault operator init
```

This generates 5 unseal keys and a root token. You need 3 of the 5 keys to unseal Vault. **Store these keys securely in separate locations—losing them means losing access to all secrets!**

Unseal Vault using three of the keys:

```bash
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>
```

> 💡 **Pro Tip:** For enterprise deployments, use Vault's auto-unseal feature with OCI KMS or another cloud KMS to automatically unseal Vault on startup.

### 1.3 Production Security Checklist

Before declaring Vault production-ready, verify these security measures:

- ✅ TLS enabled on all Vault listeners
- ✅ Root token revoked (create admin tokens instead)
- ✅ Unseal keys stored securely in separate locations
- ✅ OCI security lists restrict Vault access to authorized IPs only
- ✅ Audit logging enabled (covered in Monitoring section)
- ✅ Least-privilege policies for all auth methods
- ✅ Regular security patching schedule established

---

## 2. Secret Rotation Strategies

Secrets don't last forever. Database passwords, API keys, and certificates all need periodic rotation for security. Let's implement rotation without causing downtime for `customer-api-service`.

### 2.1 Manual Secret Rotation

The simplest approach is manual rotation with versioned secrets:

```bash
# Update the database password in Vault
vault kv put secret/customer-api/db \
  password="new_secure_password" \
  username="app_user"

# Restart the application to pick up the new secret
kubectl rollout restart deployment/customer-api-service
```

This works for small deployments, but causes brief downtime. For zero-downtime rotation, we need more sophisticated approaches.

### 2.2 Runtime Secret Refresh with Spring Cloud Vault

Spring Cloud Vault can automatically refresh secrets without restarting the application. Add this to `application.yml`:

```yaml
spring:
  cloud:
    vault:
      config:
        lifecycle:
          enabled: true
          min-renewal: 10s     # Check for secret changes

management:
  endpoints:
    web:
      exposure:
        include: refresh      # Enable /actuator/refresh endpoint
```

Now you can trigger a secret refresh without restarting:

```bash
curl -X POST http://localhost:8080/actuator/refresh
```

> ⚠️ **Limitation:** This only works for `@RefreshScope` beans. Database connection pools typically don't support runtime refresh and require a restart.

### 2.3 Vault Dynamic Secrets (Advanced)

For true zero-downtime rotation, use Vault's dynamic database secrets. Vault generates short-lived credentials on-demand and automatically rotates them:

```bash
# Enable database secrets engine
vault secrets enable database

# Configure database connection
vault write database/config/customer-db \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/customers" \
  username="vault" \
  password="vault_password"

# Create a role with 1-hour TTL
vault write database/roles/customer-api \
  db_name=customer-db \
  creation_statements="CREATE USER '{{name}}' WITH PASSWORD '{{password}}'
    VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA public TO '{{name}}';" \
  default_ttl="1h" \
  max_ttl="24h"
```

Now `customer-api-service` requests new credentials every hour, and Vault automatically revokes expired ones. This is the gold standard for secret rotation, but requires application-level support for credential refresh.

---

## 3. Handling Vault Downtime

What happens if Vault becomes unavailable? Your application shouldn't crash. Let's implement resilience patterns to gracefully handle Vault failures.

### 3.1 Configure Connection Retry Logic

Spring Cloud Vault supports retry logic for transient failures:

```yaml
spring:
  cloud:
    vault:
      fail-fast: false         # Don't crash on startup if Vault is down
      connection-timeout: 5000  # 5 second timeout
      read-timeout: 15000       # 15 second read timeout
```

With `fail-fast: false`, the application starts even if Vault is temporarily unavailable, and will fetch secrets when Vault comes back online.

### 3.2 Vault Agent for Local Caching

Vault Agent is a daemon that runs alongside your application, automatically handling authentication and caching secrets locally. This provides resilience and reduces Vault load.

Create `vault-agent.hcl`:

```hcl
pid_file = "./pidfile"

auto_auth {
  method {
    type = "oci"
    config = {
      role = "customer-api-role"
      type = "instance"
    }
  }
}

cache {
  use_auto_auth_token = true
}

listener "tcp" {
  address = "127.0.0.1:8200"
  tls_disable = true
}

vault {
  address = "https://vault.example.com:8200"
}
```

Now your application connects to `localhost:8200`, and Vault Agent handles all communication with the remote Vault server, providing caching and automatic re-authentication.

Run Vault Agent:

```bash
vault agent -config=vault-agent.hcl
```

---

## 4. High Availability Architecture

Single points of failure are production's enemy. Let's eliminate them by implementing HA for both Vault and `customer-api-service`.

### 4.1 Vault HA with Integrated Storage (Raft)

Vault's integrated storage (Raft consensus) provides built-in HA without external dependencies. You need at least 3 Vault nodes for fault tolerance.

Update `vault.hcl` on each node:

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"  # Change for each node: vault-1, vault-2, vault-3
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault.d/vault-cert.pem"
  tls_key_file  = "/etc/vault.d/vault-key.pem"
}

api_addr = "https://vault-1.example.com:8200"  # Change for each node
cluster_addr = "https://vault-1.example.com:8201"  # Change for each node
```

Initialize the first node, then join the others:

```bash
# On vault-1 (first node)
vault operator init

# On vault-2 and vault-3
vault operator raft join https://vault-1.example.com:8200
```

Now you have a 3-node Vault cluster. If one node fails, the cluster continues operating. Use an OCI load balancer to distribute traffic across healthy nodes.

### 4.2 Application High Availability

Deploy multiple instances of `customer-api-service` behind an OCI load balancer. Each instance independently authenticates to Vault using its own OCI instance principal.

Architecture overview:

- 3+ compute instances running `customer-api-service`
- OCI Load Balancer distributing traffic (round-robin or least connections)
- Health checks on `/actuator/health` endpoint
- Each instance in a separate OCI Fault Domain for hardware fault isolation

This ensures zero downtime during deployments, instance failures, or maintenance windows.

---

## 5. Performance Optimization

Performance matters. Let's optimize `customer-api-service` to handle production load efficiently.

### 5.1 Database Connection Pooling

Creating database connections is expensive. Use HikariCP (Spring Boot's default) with proper tuning:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10          # Max connections per instance
      minimum-idle: 5                # Keep 5 connections ready
      connection-timeout: 30000      # 30s timeout
      idle-timeout: 600000           # Close idle after 10min
      max-lifetime: 1800000          # Recycle connections after 30min
      leak-detection-threshold: 60000 # Detect leaks after 60s
```

> 💡 **Rule of thumb:** `maximum-pool-size = (number of CPU cores × 2) + number of disk spindles`. For cloud VMs with SSDs, use CPU cores × 2.

### 5.2 Application-Level Caching

Cache frequently accessed data to reduce database load. Spring Boot makes this easy:

```java
// Add @Cacheable to frequently accessed methods
@Cacheable(value = "customers", key = "#id")
public Customer getCustomer(Long id) {
  return customerRepository.findById(id).orElseThrow();
}
```

Configure cache in `application.yml`:

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=600s  # 1000 items, 10min TTL
```

For distributed caching across multiple instances, consider Redis or Hazelcast.

### 5.3 Optimal Load Balancing

Configure your OCI load balancer with these settings:

- **Algorithm:** Least Connections (better than round-robin for variable request times)
- **Session Persistence:** None (stateless API)
- **Health Check:** HTTP GET `/actuator/health` every 10s
- **Healthy Threshold:** 2 consecutive successes
- **Unhealthy Threshold:** 3 consecutive failures

---

## 6. Monitoring and Alerting

You can't fix what you can't see. Production systems need comprehensive monitoring and proactive alerting.

### 6.1 Enable Vault Audit Logging

Audit logs are critical for security compliance and debugging:

```bash
# Enable file-based audit logging
vault audit enable file file_path=/var/log/vault/audit.log

# Optionally enable syslog for centralized logging
vault audit enable syslog
```

> ⚠️ **Important:** Vault will refuse all requests if audit logging fails. Enable multiple audit backends for redundancy.

### 6.2 Spring Boot Metrics with Micrometer

Spring Boot Actuator exposes metrics automatically. Add Prometheus support:

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Configure in `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

Now Prometheus can scrape metrics from `/actuator/prometheus`. Track key metrics like request latency, database connection pool usage, and Vault authentication success rates.

### 6.3 Critical Alert Rules

Set up alerts for these critical conditions:

- Vault sealed status (immediate alert)
- Vault authentication failures > 5% in 5 minutes
- Application health check failures > 3 consecutive
- API error rate > 1% in 5 minutes
- Database connection pool exhaustion (all connections in use)
- Response time P95 > 2 seconds
- Certificate expiration < 30 days

---

## 7. Backup and Disaster Recovery

Hope for the best, plan for the worst. Let's ensure you can recover from catastrophic failures.

### 7.1 Automated Vault Snapshots

Take regular snapshots of Vault's Raft storage:

```bash
# Create a snapshot
vault operator raft snapshot save backup-$(date +%Y%m%d-%H%M%S).snap

# Restore from snapshot
vault operator raft snapshot restore backup-20260129-120000.snap
```

Automate this with a cron job:

```bash
# Backup script: /usr/local/bin/vault-backup.sh
#!/bin/bash
BACKUP_DIR="/opt/vault/backups"
SNAPSHOT_FILE="$BACKUP_DIR/vault-$(date +%Y%m%d-%H%M%S).snap"

vault operator raft snapshot save "$SNAPSHOT_FILE"

# Upload to OCI Object Storage
oci os object put -bn vault-backups --file "$SNAPSHOT_FILE"

# Retain only last 30 days
find "$BACKUP_DIR" -name "vault-*.snap" -mtime +30 -delete
```

Schedule daily backups:

```bash
0 2 * * * /usr/local/bin/vault-backup.sh  # Daily at 2 AM
```

### 7.2 Database Backup Strategy

Don't forget your application database! For PostgreSQL:

```bash
# Full backup
pg_dump -U postgres -d customers > customers-backup-$(date +%Y%m%d).sql

# Restore
psql -U postgres -d customers < customers-backup-20260129.sql
```

For production, use OCI Database Service's automated backup feature with point-in-time recovery.

### 7.3 Test Your Recovery Procedures

Backups are worthless if you can't restore from them. Schedule quarterly disaster recovery drills:

1. Destroy a test Vault instance
2. Restore from snapshot
3. Unseal Vault
4. Verify application can authenticate and retrieve secrets
5. Document recovery time and any issues encountered

Measure your Recovery Time Objective (RTO) and Recovery Point Objective (RPO). Most businesses need RTO < 4 hours and RPO < 1 hour for critical systems.

---

## 8. Cost Optimization

Production resilience doesn't have to break the bank. Let's optimize costs without sacrificing reliability.

### Right-Sizing Compute Instances

OCI cost optimization strategies:

- Use ARM-based Ampere instances (up to 40% cheaper than x86)
- Start small (VM.Standard.A1.Flex with 2 OCPUs, 12GB RAM)
- Monitor CPU/memory utilization - scale up only when needed
- Use autoscaling for variable traffic patterns
- Reserve capacity for production (significant discount for 1-3 year commits)

### Vault Licensing Considerations

Vault open-source is free and production-ready for most use cases. Vault Enterprise adds:

- Multi-region replication
- HSM integration for seal keys
- FIPS 140-2 compliance
- Enterprise support

For most organizations, open-source Vault with Raft storage provides excellent HA at zero licensing cost.

---

## Conclusion

Congratulations! You've transformed `customer-api-service` from a basic deployment into an enterprise-grade production system. You now have:

- ✅ Hardened Vault with TLS and production-mode configuration
- ✅ Secret rotation strategies for zero-downtime updates
- ✅ Resilience patterns to handle Vault failures gracefully
- ✅ High availability for both Vault and your application
- ✅ Performance optimizations with connection pooling and caching
- ✅ Comprehensive monitoring and alerting
- ✅ Disaster recovery procedures and cost optimization

This is what production-ready looks like. Your system can now handle real-world conditions: traffic spikes, infrastructure failures, security incidents, and planned maintenance—all while maintaining the trust of your users and stakeholders.

### Production Readiness Checklist

Before going live, verify:

- ☐ Vault running in production mode with TLS
- ☐ Unseal keys stored securely in separate locations
- ☐ Root token revoked
- ☐ Audit logging enabled with redundant backends
- ☐ At least 3 Vault nodes in HA configuration
- ☐ Multiple application instances behind load balancer
- ☐ Health checks configured and tested
- ☐ Database connection pooling optimized
- ☐ Monitoring and alerting operational
- ☐ Automated backup procedures in place
- ☐ Disaster recovery tested successfully
- ☐ Runbooks documented for common operations

### What's Next?

In **Post 8**, we'll automate the entire deployment pipeline with GitHub Actions, enabling continuous delivery with zero-downtime deployments. You'll learn how to build, test, and deploy `customer-api-service` automatically whenever you push code—the final piece of a truly modern DevOps workflow.

### Additional Resources

- [HashiCorp Vault Production Hardening](https://learn.hashicorp.com/tutorials/vault/production-hardening)
- [OCI High Availability Best Practices](https://docs.oracle.com/en-us/iaas/Content/Resources/Assets/whitepapers/high-availability-on-oci.pdf)
- [Spring Boot Production Best Practices](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html)
- [Vault Agent Documentation](https://www.vaultproject.io/docs/agent)

---

**📚 Part of the Production-Ready Spring Boot on OCI with Vault series**

*Questions? Found this helpful? Share your experience in the comments!*