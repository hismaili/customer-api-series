# Production-Ready Spring Boot: Multi-Cloud Secrets Management with HashiCorp Vault

## 🎯 What You'll Build

By the end of this comprehensive series, you'll have deployed a **production-grade REST API** (`customer-api-service`) to the cloud with enterprise-level security, high availability, and automated CI/CD—all using **zero-trust authentication** with HashiCorp Vault.

This isn't a toy project. You'll learn the exact patterns and practices used by Fortune 500 companies to secure sensitive data in production environments.

---

## 🚀 Why This Series?

**The Problem:** Most tutorials show you how to build applications, but deploying them securely to production remains a mystery. How do you manage database passwords? API keys? How do you rotate secrets without downtime? How do you prevent credential leaks?

**The Solution:** This series takes you from "Hello World" to production-ready, step-by-step. You'll learn:

- ✅ **Zero-trust security** - No hardcoded credentials, ever
- ✅ **Cloud-native patterns** - Instance principals, managed identities, IAM roles
- ✅ **Production resilience** - High availability, disaster recovery, monitoring
- ✅ **Real-world DevOps** - Automated deployments, secret rotation, scaling

**What makes this different:** We don't just show you the "happy path." You'll learn to handle Vault downtime, debug authentication failures, optimize costs, and implement the boring-but-critical practices that separate hobby projects from production systems.

---

## 📚 Complete Series Overview

### **Core Posts (Cloud-Agnostic)**

| Post | Title                                                               | What You'll Learn | Time Required |
|------|---------------------------------------------------------------------|-------------------|---------------|
| **1** | [Building a Customer Management API with Spring Boot](#post-1)      | Create a REST API with Spring Boot, JPA, and H2 database | 2-3 hours |
| **4** | [Secrets Management with HashiCorp Vault - Local Setup](#post-4)    | Run Vault locally, store/retrieve secrets, integrate with Spring Boot | 2 hours |
| **7** | [Production Hardening & Scaling Guide](#post-7)                     | Security, HA, performance optimization, disaster recovery | 3-4 hours |
| **8** | [CI/CD Pipeline - Automated Deployment with GitHub Actions](#post-8) | Fully automated deployments with zero downtime | 2-3 hours |

### **Cloud-Specific Tracks**

Choose your cloud provider and follow the corresponding track:

#### **🟠 Oracle Cloud Infrastructure (OCI) Track**

| Post | Title | What You'll Learn | Time Required |
|------|-------|-------------------|---------------|
| **2-OCI** | [Understanding OCI - Cloud Infrastructure Fundamentals](#post-2-oci) | OCI tenancy, compartments, VCN, compute, IAM, dynamic groups | 2 hours |
| **3-OCI** | [Containerizing and Deploying Spring Boot on OCI](#post-3-oci) | Docker, OCI compute deployment, security lists, firewall | 2-3 hours |
| **5-OCI** | [Zero-Trust Authentication - OCI Instance Principals with Vault](#post-5-oci) | OCI auth method, instance principals, cryptographic identity | 3-4 hours |
| **6-OCI** | [Production Deployment - Spring Boot + Vault + OCI Auth](#post-6-oci) | End-to-end secure deployment on OCI | 2-3 hours |

#### **🟡 Amazon Web Services (AWS) Track** *(Coming Soon)*

| Post | Title | What You'll Learn | Status |
|------|-------|-------------------|--------|
| **2-AWS** | Understanding AWS - Cloud Infrastructure Fundamentals | VPC, EC2, IAM roles, instance profiles | Q2 2026 |
| **3-AWS** | Deploying Spring Boot to AWS | EC2 deployment, security groups, ALB | Q2 2026 |
| **5-AWS** | Zero-Trust Authentication - AWS IAM with Vault | AWS auth method, IAM authentication | Q2 2026 |
| **6-AWS** | Production Deployment on AWS | Complete AWS deployment | Q2 2026 |

#### **🔵 Microsoft Azure Track** *(Planned)*

| Post | Title | What You'll Learn | Status |
|------|-------|-------------------|--------|
| **2-Azure** | Understanding Azure - Cloud Infrastructure Fundamentals | VNets, VMs, managed identities, Entra ID | Q3 2026 |
| **3-Azure** | Deploying Spring Boot to Azure | Azure VM deployment, NSGs, load balancer | Q3 2026 |
| **5-Azure** | Zero-Trust Authentication - Azure Managed Identity with Vault | Azure auth, MSI integration | Q3 2026 |
| **6-Azure** | Production Deployment on Azure | Complete Azure deployment | Q3 2026 |

#### **🔴 Google Cloud Platform (GCP) Track** *(Planned)*

| Post | Title | What You'll Learn | Status |
|------|-------|-------------------|--------|
| **2-GCP** | Understanding GCP - Cloud Infrastructure Fundamentals | VPC, Compute Engine, service accounts, IAM | Q4 2026 |
| **3-GCP** | Deploying Spring Boot to GCP | GCE deployment, firewall rules, load balancer | Q4 2026 |
| **5-GCP** | Zero-Trust Authentication - GCP Service Account with Vault | GCP auth method, service account authentication | Q4 2026 |
| **6-GCP** | Production Deployment on GCP | Complete GCP deployment | Q4 2026 |

---

## 🎓 Learning Path & Prerequisites

### **Who This Series Is For**

- ✅ Java developers wanting to learn production deployment
- ✅ DevOps engineers implementing secrets management
- ✅ Backend developers moving to cloud-native architectures
- ✅ Anyone tired of hardcoded credentials and security theater

### **Prerequisites**

**Required:**
- Basic Java knowledge (Spring Boot experience helpful but not required)
- Familiarity with REST APIs
- Comfort with command line/terminal
- A cloud account (OCI, AWS, Azure, or GCP free tier is sufficient)

**Helpful but not required:**
- Docker basics
- Git/GitHub experience
- Basic understanding of databases

### **Recommended Learning Path**

```
Week 1: Foundation
├── Post 1: Build the API (Day 1-2)
└── Post 4: Vault Fundamentals (Day 3-5)

Week 2: Cloud Deployment (Choose Your Track)
├── Post 2: Cloud Fundamentals (Day 1-2)
├── Post 3: Deploy to Cloud (Day 3-4)
└── Post 5: Zero-Trust Auth (Day 5-7)

Week 3: Production Ready
├── Post 6: Production Deployment (Day 1-3)
└── Post 7: Hardening & Scaling (Day 4-7)

Week 4: Automation
└── Post 8: CI/CD Pipeline (Day 1-5)
```

**Total time investment:** 3-4 weeks at 2-3 hours per day

---

## 📋 Progress Tracker

Use this checklist to track your journey:

### **Core Learning**
- [ ] Built customer-api-service REST API
- [ ] Tested API locally with Postman
- [ ] Installed and configured HashiCorp Vault
- [ ] Integrated Spring Boot with Vault (local)
- [ ] Understood secrets management principles

### **Cloud Deployment (Choose One Track)**
- [ ] Learned cloud fundamentals (VPC, compute, IAM)
- [ ] Created cloud account and configured CLI
- [ ] Containerized the application with Docker
- [ ] Deployed container to cloud compute instance
- [ ] Configured cloud networking and security
- [ ] Deployed Vault to the cloud
- [ ] Configured cloud-native authentication (Instance Principals/IAM/MSI)
- [ ] Successfully tested zero-trust authentication
- [ ] Deployed customer-api-service with Vault integration

### **Production Readiness**
- [ ] Configured Vault production mode with TLS
- [ ] Implemented secret rotation strategy
- [ ] Set up high availability (3+ Vault nodes)
- [ ] Deployed multiple app instances with load balancing
- [ ] Optimized database connection pooling
- [ ] Configured monitoring and alerting
- [ ] Implemented automated backup procedures
- [ ] Tested disaster recovery
- [ ] Optimized cloud costs

### **DevOps Automation**
- [ ] Created GitHub repository for code
- [ ] Set up GitHub Actions workflow
- [ ] Automated container builds
- [ ] Implemented automated testing
- [ ] Deployed via CI/CD pipeline
- [ ] Achieved zero-downtime deployments

---

## 🛠️ What You'll Build

### **The Customer Management API**

A production-ready REST API for managing customers and invoices:

**Features:**
- Multi-tenant customer management
- Invoice tracking and reporting
- RESTful endpoints (GET, POST, PUT, DELETE)
- Data validation and error handling
- Database persistence with JPA/Hibernate
- Actuator endpoints for health monitoring

**Technology Stack:**
- **Backend:** Spring Boot 3.x, Java 21
- **Database:** PostgreSQL (production), H2 (local development)
- **Security:** HashiCorp Vault for secrets management
- **Container:** Docker/Podman
- **Cloud:** OCI/AWS/Azure/GCP (your choice)
- **CI/CD:** GitHub Actions
- **Monitoring:** Prometheus, Spring Boot Actuator

**Architecture Evolution:**

```
Phase 1 (Post 1):
┌─────────────────┐
│  Spring Boot    │
│  customer-api   │
│  (H2 database)  │
└─────────────────┘

Phase 2 (Post 4):
┌─────────────────┐      ┌──────────────┐
│  Spring Boot    │─────▶│    Vault     │
│  customer-api   │      │   (local)    │
└─────────────────┘      └──────────────┘

Phase 3 (Post 6):
┌──────────────────────────────┐
│       Cloud Provider         │
│  ┌────────────────────────┐  │
│  │  Compute Instance      │  │
│  │  ┌──────────────────┐  │  │
│  │  │  Spring Boot     │  │  │
│  │  │  customer-api    │  │  │
│  │  └────────┬─────────┘  │  │
│  │           │            │  │
│  │  ┌────────▼─────────┐  │  │
│  │  │  Vault (Cloud)   │  │  │
│  │  │  + Cloud Auth    │  │  │
│  │  └──────────────────┘  │  │
│  └────────────────────────┘  │
└──────────────────────────────┘

Phase 4 (Post 7):
┌─────────────────────────────────────────┐
│         Cloud Provider (HA)             │
│  ┌──────────────────────────────────┐   │
│  │    Load Balancer                 │   │
│  └───────┬──────────┬───────────────┘   │
│          │          │                   │
│  ┌───────▼─────┐ ┌──▼────────────┐      │
│  │ App Instance│ │ App Instance  │      │
│  │      1      │ │      2        │ ...  │
│  └─────────────┘ └───────────────┘      │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │   Vault Cluster (3 nodes)       │    │
│  │   + Raft Storage                │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

---

## 📖 Detailed Post Summaries

### <a name="post-1"></a>**Post 1: Building a Customer Management API with Spring Boot**

**Standalone Value:** A complete, working REST API you can run locally

Learn to build a professional Spring Boot application from scratch:
- Initialize project with Spring Initializr
- Design entities: Customer, Invoice with JPA relationships
- Create REST controllers with full CRUD operations
- Implement request validation and error handling
- Use H2 in-memory database for rapid development
- Test endpoints with Postman and curl
- Package as executable JAR

**Code Repository:** [customer-api-service](https://github.com/yourusername/customer-api-service)

**Key Takeaways:**
- Clean REST API design principles
- Spring Boot project structure
- JPA entity relationships
- Exception handling best practices

**Time:** 2-3 hours

---

### <a name="post-2-oci"></a>**Post 2-OCI: Understanding OCI - Cloud Infrastructure Fundamentals**

**Standalone Value:** Complete crash course on Oracle Cloud Infrastructure

Master the fundamentals of OCI before deploying your application:
- OCI architecture: Tenancy, Compartments, Regions, Availability Domains
- Identity and Access Management (IAM): Users, Groups, Policies
- Networking: VCN, Subnets, Route Tables, Security Lists
- Compute: Instance types, shapes, SSH access
- Dynamic Groups and Instance Principals (critical for zero-trust)
- Metadata Service and how instances prove their identity
- Cost management and free tier optimization

**Key Takeaways:**
- How OCI's IAM differs from AWS/Azure
- When to use instance principals vs. user credentials
- Network security best practices
- Reading OCI documentation effectively

**Time:** 2 hours

---

### <a name="post-3-oci"></a>**Post 3-OCI: Containerizing and Deploying Spring Boot on OCI**

**Standalone Value:** End-to-end deployment guide from code to cloud

Take your local application to production on OCI:
- Write optimized Dockerfile for Spring Boot
- Build container image with Docker/Podman
- Push to Oracle Container Registry (OCIR)
- Provision OCI compute instance (choose the right shape)
- Configure security lists for HTTP/HTTPS traffic
- Deploy and run container on OCI
- Configure firewall rules
- Test API endpoints from the internet
- Monitor logs and troubleshoot deployment issues

**Key Takeaways:**
- Container best practices for Java applications
- OCI networking and security configuration
- Debugging deployment problems
- Performance considerations for compute shapes

**Time:** 2-3 hours

---

### <a name="post-4"></a>**Post 4: Secrets Management with HashiCorp Vault - Local Setup**

**Standalone Value:** Vault fundamentals anyone can follow locally

Understand why secrets management matters and how Vault solves it:
- The dangers of hardcoded credentials
- Vault architecture and core concepts
- Install and run Vault in dev mode
- Store and retrieve secrets via CLI and UI
- Understand Vault paths and policies
- Integrate Spring Boot with Vault using Spring Cloud Vault
- Configure application.yml for Vault
- Handle Vault connection failures gracefully

**Key Takeaways:**
- Why Vault beats environment variables and config files
- How Vault's policy system works
- Spring Cloud Vault configuration patterns
- Local development workflow with Vault

**Time:** 2 hours

---

### <a name="post-5-oci"></a>**Post 5-OCI: Zero-Trust Authentication - OCI Instance Principals with Vault**

**Standalone Value:** Deep dive into zero-trust security patterns

Solve the "Secret Zero" problem with cryptographic authentication:
- The chicken-and-egg problem: How does the app get its first credential?
- How OCI Instance Principals provide cryptographic identity
- Configure Vault's OCI auth method step-by-step
- Create dynamic groups for compute instances
- Write IAM policies for Vault authentication
- Debug authentication failures with Vault logs
- Verify signed certificates and metadata
- Best practices for production OCI auth

**Key Takeaways:**
- Zero-trust security fundamentals
- How cryptographic identity works
- OCI-specific authentication mechanisms
- Troubleshooting authentication issues

**Time:** 3-4 hours

---

### <a name="post-6-oci"></a>**Post 6-OCI: Production Deployment - Spring Boot + Vault + OCI Auth**

**Standalone Value:** Complete secure production deployment

Bring everything together for a production-ready deployment:
- Modify customer-api-service for OCI authentication
- Configure Spring Cloud Vault for OCI auth method
- Deploy Vault on OCI compute instance
- Configure environment variables correctly
- Handle container networking (app ↔ Vault communication)
- Test end-to-end: startup → auth → secret retrieval → database connection
- Debug common issues (auth failures, network timeouts, misconfigurations)
- Verify zero hardcoded credentials

**Key Takeaways:**
- Production Spring Boot configuration
- Container networking patterns
- End-to-end testing strategies
- Common pitfalls and solutions

**Time:** 2-3 hours

---

### <a name="post-7"></a>**Post 7: Production Hardening & Scaling Guide**

**Standalone Value:** Enterprise-grade production practices

Transform your deployment from functional to enterprise-ready:

**Security Hardening:**
- Configure Vault production mode (no more dev mode!)
- Generate and install TLS certificates
- Initialize and unseal Vault safely
- Store unseal keys securely
- Enable audit logging
- Production security checklist

**Secret Rotation:**
- Manual rotation without downtime
- Spring Cloud Vault refresh endpoint
- Vault dynamic database secrets
- Automated rotation strategies

**Resilience:**
- Handle Vault downtime gracefully
- Configure connection retry logic
- Deploy Vault Agent for local caching
- Implement circuit breaker patterns

**High Availability:**
- Vault HA with Raft consensus (3+ nodes)
- Multi-instance application deployment
- OCI load balancer configuration
- Fault domain distribution

**Performance:**
- Database connection pool tuning
- Application-level caching with Caffeine
- Load balancing strategies
- Resource optimization

**Monitoring:**
- Vault audit logs
- Spring Boot metrics with Prometheus
- Critical alerting rules
- Health check configuration

**Disaster Recovery:**
- Automated Vault snapshots
- Database backup procedures
- Recovery testing
- RTO/RPO planning

**Cost Optimization:**
- Right-sizing compute instances
- Using ARM-based instances
- Reserved capacity strategy

**Key Takeaways:**
- Production vs. development configurations
- Enterprise resilience patterns
- Monitoring and observability
- Disaster recovery planning

**Time:** 3-4 hours

---

### <a name="post-8"></a>**Post 8: CI/CD Pipeline - Automated Deployment with GitHub Actions**

**Standalone Value:** Complete DevOps automation** *(Coming Soon)*

Automate the entire deployment pipeline from code push to production:
- GitHub Actions workflow fundamentals
- Build and test customer-api-service automatically
- Build and push Docker images to container registry
- Deploy to OCI compute instances
- Manage multiple environments (dev, staging, prod)
- Store secrets in GitHub Secrets (Vault integration)
- Implement zero-downtime deployments (blue/green)
- Rollback strategies for failed deployments
- Notification integration (Slack, email)

**Key Takeaways:**
- GitHub Actions best practices
- Multi-environment deployment strategies
- Secrets management in CI/CD
- Deployment automation patterns

**Time:** 2-3 hours

---

## 🎁 Bonus Content

### **Comparison Posts** *(Coming Soon)*

- **Cost Comparison:** Running customer-api-service on AWS vs Azure vs GCP vs OCI
- **Performance Benchmarks:** Response times and throughput across clouds
- **Security Comparison:** Native secrets managers vs Vault
- **Which Cloud for Spring Boot?** A decision framework for Java developers

### **Advanced Topics** *(Planned)*

- Multi-region Vault deployment for global applications
- Kubernetes deployment with Vault Agent Injector
- Vault secrets in serverless functions (Lambda, Cloud Functions, Azure Functions)
- Compliance automation (SOC2, HIPAA, PCI-DSS)
- Vault Enterprise features: namespaces, replication, HSM integration

---

## 💬 Community & Support

### **Get Help**

- **GitHub Discussions:** [Ask questions and share your progress](https://github.com/yourusername/customer-api-service/discussions)
- **Issues:** [Report bugs or request clarifications](https://github.com/yourusername/customer-api-service/issues)
- **Twitter/X:** Share your wins with [#ProductionReadySpringBoot](https://twitter.com/search?q=%23ProductionReadySpringBoot)

### **Contribute**

Found a typo? Have a better approach? Contributions welcome!
- Submit pull requests for documentation improvements
- Share your deployment experiences
- Suggest additional cloud providers or topics

### **Stay Updated**

- **Newsletter:** [Subscribe for new posts and updates](https://yourwebsite.com/newsletter)
- **RSS Feed:** [https://yourwebsite.com/feed.xml](https://yourwebsite.com/feed.xml)
- **GitHub Watch:** Star the repo to get notified of updates

---

## 📚 Additional Resources

### **Official Documentation**

- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [Spring Cloud Vault Reference](https://docs.spring.io/spring-cloud-vault/docs/current/reference/html/)
- [OCI Documentation](https://docs.oracle.com/en-us/iaas/Content/home.htm)
- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)

### **Related Learning**

- [Vault Production Hardening Guide](https://learn.hashicorp.com/tutorials/vault/production-hardening)
- [Spring Boot Production Best Practices](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html)
- [OCI Architecture Center](https://docs.oracle.com/solutions/)
- [Twelve-Factor App Methodology](https://12factor.net/)

### **Tools You'll Use**

- [Spring Initializr](https://start.spring.io/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Postman](https://www.postman.com/)
- [OCI CLI](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/cliconcepts.htm)

---

## 🚀 Ready to Start?

Choose your path:

### **🟠 Start with OCI** (Recommended - Complete Track Available)
Begin with [Post 1: Building a Customer Management API with Spring Boot](#post-1)

### **🟡 Prefer AWS?** (Coming Q2 2026)
[Join the waitlist for AWS track updates](https://yourwebsite.com/aws-waitlist)

### **🔵 Azure Fan?** (Planned Q3 2026)
[Join the waitlist for Azure track updates](https://yourwebsite.com/azure-waitlist)

### **🔴 GCP User?** (Planned Q4 2026)
[Join the waitlist for GCP track updates](https://yourwebsite.com/gcp-waitlist)

---

## 📝 About This Series

This series was created to fill a gap in production deployment education for Java developers. Most tutorials stop at "it works on my laptop"—this series takes you all the way to "it's running in production with enterprise-grade security."

**Author:** [Your Name]  
**Last Updated:** February 2026  
**Series Status:** OCI track complete, AWS track in progress

### **Why I Created This**

After years of seeing Spring Boot applications deployed with hardcoded credentials and no proper secrets management, I wanted to create a comprehensive guide showing the *right* way to do production deployments. The kind of deployment you'd be proud to show security auditors.

This isn't theoretical—these are the exact patterns I've used in production systems serving millions of requests per day.

---

## ⭐ Support This Series

If you find this series valuable:

- ⭐ **Star the GitHub repository**
- 🐦 **Share on Twitter/LinkedIn**
- 💬 **Write a blog post** about your experience
- 🎁 **Buy me a coffee** (link)
- 📧 **Recommend to colleagues**

Your support helps me create more free, in-depth content!

---

## 📜 License

All code examples are released under the MIT License. Tutorial content is licensed under Creative Commons BY-NC-SA 4.0.

---

**Let's build production-ready applications together! Start with [Post 1](#post-1) →**