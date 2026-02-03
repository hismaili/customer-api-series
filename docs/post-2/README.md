# Understanding Oracle Cloud Infrastructure: A Complete Guide for Developers

*Part 2 of 9 in the series: "Production-Ready Spring Boot on OCI with Vault"*

![Oracle Cloud Infrastructure](https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=1200&h=400&fit=crop)

## What You'll Learn

Before deploying applications to the cloud, you need to understand the infrastructure layer. In this comprehensive guide, you'll master Oracle Cloud Infrastructure (OCI) fundamentals from a developer's perspective.

By the end of this post, you'll understand:
- ✅ OCI's unique architecture and core concepts
- ✅ How to set up networking for secure cloud applications
- ✅ Compute instances and how to access them
- ✅ Identity and Access Management (IAM) essentials
- ✅ Dynamic groups and instance principals for zero-trust authentication
- ✅ The metadata service that enables secure application identity

**Time to complete**: 60 minutes  
**Difficulty**: Beginner to Intermediate  
**Cost**: Free tier eligible (no credit card required for most features)

---

## Table of Contents

1. [Why Oracle Cloud Infrastructure?](#why-oracle-cloud-infrastructure)
2. [OCI Architecture Overview](#oci-architecture-overview)
3. [Setting Up Your OCI Account](#setting-up-your-oci-account)
4. [Understanding Tenancy and Compartments](#understanding-tenancy-and-compartments)
5. [Networking Fundamentals](#networking-fundamentals)
6. [Compute Instances Deep Dive](#compute-instances-deep-dive)
7. [Identity and Access Management](#identity-and-access-management)
8. [Dynamic Groups and Instance Principals](#dynamic-groups-and-instance-principals)
9. [The Metadata Service](#the-metadata-service)
10. [Security Best Practices](#security-best-practices)
11. [What's Next](#whats-next)

---

## Prerequisites

**From Previous Posts:**
- [Post 1: Building a Customer Management API with Spring Boot](#) - We'll deploy this application in Post 3

**For This Tutorial:**
- Email address for OCI account signup
- Basic understanding of networking concepts (IP addresses, subnets)
- SSH client installed (OpenSSH, PuTTY, or built-in terminal)

**No Credit Card Required**: OCI offers a generous free tier that's sufficient for this entire series.

---

## Why Oracle Cloud Infrastructure?

Before diving in, let's understand what makes OCI special:

### 🚀 Performance
- Bare metal compute instances with no virtualization overhead
- Non-blocking, full-bandwidth network with 25/50/100 Gbps options
- NVMe-based block storage with consistent IOPS

### 💰 Cost-Effective
- **Free Tier**: 2 AMD Compute VMs (1/8 OCPU, 1GB RAM each) forever
- Up to 4 Arm Ampere A1 cores and 24 GB memory free forever
- 200 GB Block Volume, 10 GB Object Storage - all free
- Predictable pricing with no hidden costs

### 🔒 Security-First Design
- Isolated network virtualization
- Built-in DDoS protection
- Instance principals for zero-secret authentication
- Compartment-based resource isolation

### 🎯 Developer-Friendly
- Extensive free tier for learning and development
- Clean, modern API with comprehensive SDKs
- Strong integration with HashiCorp tools (Terraform, Vault, Consul)
- Active open-source community

**Perfect for our series**: We'll use only free tier resources throughout!

---

## OCI Architecture Overview

### The Hierarchy

OCI resources are organized in a clear hierarchy:

```
Tenancy (Your Cloud Account)
    └── Regions (Geographic locations)
        └── Availability Domains (Isolated data centers)
            └── Fault Domains (Hardware isolation within AD)
                └── Compartments (Logical resource containers)
                    └── Resources (VMs, Networks, Storage, etc.)
```

### Key Concepts

**Tenancy**: Your entire OCI account. Think of it as your cloud "house."

**Region**: Physical geographic location (e.g., US East, EU Frankfurt). Each region is completely independent.

**Availability Domain (AD)**: Isolated data center within a region. ADs don't share physical infrastructure (power, cooling, network).

**Fault Domain**: Hardware grouping within an AD. Ensures resources aren't on the same physical hardware.

**Compartment**: Logical container to organize and isolate resources. Essential for access control and billing.

### Understanding OCIDs

Every resource in OCI has a unique identifier called an **OCID** (Oracle Cloud Identifier):

```
ocid1.instance.oc1.phx.anyhqljs2i3lvbicxyz...
  │      │        │   │    └─ Unique resource ID
  │      │        │   └─ Availability Domain
  │      │        └─ Region
  │      └─ Resource type
  └─ Version
```

OCIDs are crucial - you'll use them everywhere in OCI!

---

## Setting Up Your OCI Account

### Step 1: Sign Up for OCI Free Tier

1. Go to [oracle.com/cloud/free](https://www.oracle.com/cloud/free/)
2. Click **Start for free**
3. Fill in your details:
   - Email address
   - Country/Territory
   - Full name
4. Verify your email
5. Complete the registration:
   - Choose your **Home Region** (cannot be changed later!)
   - For this series, any region works
   - I recommend **US East (Ashburn)** or **EU Frankfurt** for low latency
6. Choose **Account Type**: Select "Individual" for learning

**Important**: Once you select a home region, you cannot change it. All IAM resources will be in this region.

### Step 2: Navigate the OCI Console

After signup, you'll land in the OCI Console:

**Key Areas**:
- **☰ Menu** (top-left): Access all OCI services
- **Region selector** (top-right): Switch between regions
- **Profile menu** (top-right): Account settings, tenancy info
- **Breadcrumbs** (top): Navigate back through pages
- **Search bar** (top): Quick access to resources and docs

**Pro Tip**: Bookmark these direct links:
- Console: https://cloud.oracle.com/
- Documentation: https://docs.oracle.com/iaas/
- Free Tier: https://cloud.oracle.com/free-tier

### Step 3: Note Your Tenancy Information

You'll need these values throughout the series:

1. Click **Profile Icon** → **Tenancy: [your-tenancy-name]**
2. Copy and save:
   - **Tenancy OCID**: `ocid1.tenancy.oc1..aaaaaaaaxyz...`
   - **Tenancy Name**: Your chosen name
   - **Home Region**: e.g., `us-ashburn-1`

Save these in a secure note - you'll use them in every post!

---

## Understanding Tenancy and Compartments

### Tenancy: Your Cloud Foundation

Your **tenancy** is your entire OCI account. It includes:
- All regions (but you're billed per region)
- Root compartment (same as tenancy)
- IAM resources (users, groups, policies)
- Service limits and quotas

**Key Facts**:
- One tenancy per organization (typically)
- Tenancy OCID never changes
- Root compartment = tenancy (used interchangeably)

### Compartments: Organizing Resources

**Compartments** are logical containers for resources. Think of them like folders:

```
Root Compartment (Tenancy)
    ├── Development
    │   ├── dev-network
    │   ├── dev-compute
    │   └── dev-databases
    ├── Production
    │   ├── prod-network
    │   ├── prod-compute
    │   └── prod-databases
    └── Shared-Services
        ├── monitoring
        └── security
```

**Why Use Compartments?**
1. **Access Control**: Grant different teams access to different compartments
2. **Cost Tracking**: See spending per compartment
3. **Resource Organization**: Keep resources logically grouped
4. **Policy Scope**: Apply security policies per compartment

### Create Your First Compartment

For this series, let's create a dedicated compartment:

1. **☰ Menu** → **Identity & Security** → **Compartments**
2. Click **Create Compartment**
3. Fill in:
   - **Name**: `customer-api-dev`
   - **Description**: `Development environment for Customer API series`
   - **Parent Compartment**: (root) - your tenancy name
4. Click **Create Compartment**

**Save the Compartment OCID** - you'll need it for networking and compute!

**Pro Tip**: For learning, you can use the root compartment, but compartments are a best practice for organization.

---

## Networking Fundamentals

OCI networking is powerful and secure by default. Let's build a network for our application.

### Virtual Cloud Network (VCN) Basics

A **VCN** is your private network in OCI. It's like your own private AWS VPC or Azure VNet.

**Key Components**:
- **CIDR Block**: IP range for your VCN (e.g., `10.0.0.0/16`)
- **Subnets**: Segments within your VCN
- **Route Tables**: Control traffic routing
- **Security Lists**: Firewall rules for subnets
- **Internet Gateway**: Access to/from internet
- **Service Gateway**: Access to OCI services without internet

### Create a VCN (Quick Method)

OCI provides a VCN wizard that sets up everything:

1. **☰ Menu** → **Networking** → **Virtual Cloud Networks**
2. Click **Start VCN Wizard**
3. Select **Create VCN with Internet Connectivity**
4. Click **Start VCN Wizard**
5. Fill in:
   - **VCN Name**: `customer-api-vcn`
   - **Compartment**: Select `customer-api-dev`
   - **VCN CIDR Block**: `10.0.0.0/16` (65,536 IPs)
   - **Public Subnet CIDR**: `10.0.0.0/24` (256 IPs)
   - **Private Subnet CIDR**: `10.0.1.0/24` (256 IPs)
6. Click **Next** → **Create**

**What Got Created?**
- ✅ VCN with CIDR `10.0.0.0/16`
- ✅ Public subnet (for resources with public IPs)
- ✅ Private subnet (for internal resources)
- ✅ Internet Gateway (for public internet access)
- ✅ NAT Gateway (for private subnet outbound access)
- ✅ Service Gateway (for OCI services access)
- ✅ Route tables (pre-configured)
- ✅ Security lists (with basic rules)

**Save the VCN OCID** - you'll need it when creating compute instances!

### Understanding Subnets

**Public Subnet** (`10.0.0.0/24`):
- Resources get public IP addresses
- Accessible from the internet (with proper security rules)
- Use for: Web servers, load balancers, bastion hosts

**Private Subnet** (`10.0.1.0/24`):
- Resources have only private IPs
- Not directly accessible from internet
- Can access internet via NAT Gateway
- Use for: Databases, internal services, backend apps

For our series, we'll use the **public subnet** to make testing easier.

### Security Lists Explained

Security Lists are stateful firewalls that control traffic to/from subnets.

**View Your Security List**:
1. Navigate to your VCN
2. Click **Security Lists** (left menu)
3. Click **Default Security List for customer-api-vcn**

**Default Rules**:

**Ingress (Incoming)**:
- TCP 22 (SSH) from `0.0.0.0/0` - allows SSH from anywhere
- ICMP from `10.0.0.0/16` - allows ping within VCN

**Egress (Outgoing)**:
- All protocols to `0.0.0.0/0` - allows all outbound traffic

### Add HTTP/HTTPS Rules

For our API, we need to allow HTTP traffic:

1. In the Default Security List, click **Add Ingress Rules**
2. Add Rule #1 (HTTP):
   - **Source CIDR**: `0.0.0.0/0`
   - **IP Protocol**: TCP
   - **Destination Port Range**: `8080`
   - **Description**: `Allow HTTP traffic to Spring Boot`
3. Click **Add Ingress Rules**

Repeat for HTTPS if needed (port 443).

**Security Note**: In production, restrict source CIDR to specific IPs or ranges. `0.0.0.0/0` means "allow from anywhere" - good for testing, not for production!

### Service Gateway - The Hidden Gem

The **Service Gateway** allows your instances to access OCI services (Object Storage, Vault, etc.) without going through the internet.

**Why It Matters**:
- ✅ Free data transfer (no egress charges)
- ✅ Better security (traffic stays in OCI network)
- ✅ Lower latency
- ✅ Required for Instance Principals to work properly

The VCN wizard already created this for you. We'll use it extensively when integrating with Vault!

---

## Compute Instances Deep Dive

Time to create your first compute instance!

### Understanding Instance Types

OCI offers several compute shapes:

**VM (Virtual Machine)**:
- Standard: General purpose, balanced resources
- DenseIO: High local NVMe storage
- GPU: For ML/AI workloads

**Bare Metal**:
- Dedicated physical servers
- No virtualization overhead
- Maximum performance

**For This Series**: We'll use **VM.Standard.E2.1.Micro** (Always Free tier eligible).

### Create a Compute Instance

#### Step 1: Generate SSH Keys

Before creating an instance, you need SSH keys:

**On macOS/Linux**:
```bash
# Create .ssh directory if it doesn't exist
mkdir -p ~/.ssh

# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/oci_key -N ""

# View public key (you'll need this)
cat ~/.ssh/oci_key.pub
```

**On Windows** (PowerShell):
```powershell
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f $env:USERPROFILE\.ssh\oci_key

# View public key
Get-Content $env:USERPROFILE\.ssh\oci_key.pub
```

**Copy the entire public key** (starts with `ssh-rsa ...`).

#### Step 2: Launch the Instance

1. **☰ Menu** → **Compute** → **Instances**
2. Click **Create Instance**
3. Configure:

**Name**: `customer-api-instance`

**Compartment**: `customer-api-dev`

**Placement**:
- **Availability Domain**: Select any (AD-1, AD-2, or AD-3)
- Leave fault domain as default

**Image and Shape**:
- **Image**: Oracle Linux 8 (default is good)
- Click **Change Shape**
  - **Instance Type**: Virtual Machine
  - **Shape Series**: AMD
  - **Shape Name**: VM.Standard.E2.1.Micro
  - Click **Select Shape**

**Networking**:
- **VCN**: `customer-api-vcn`
- **Subnet**: Public Subnet (e.g., `Public Subnet-customer-api-vcn`)
- **Assign a public IPv4 address**: ✅ Checked

**Add SSH Keys**:
- **SSH Key**: Paste your public key from Step 1
- Or click **Choose files** and upload your `.pub` file

**Boot Volume**:
- Leave defaults (50 GB)

4. Click **Create**

**Wait ~2 minutes** for provisioning. Status will change from "Provisioning" to "Running".

#### Step 3: Note Instance Details

Once running, copy these values:

1. Click your instance name
2. Save:
   - **Public IP Address**: e.g., `144.24.xxx.xxx`
   - **Instance OCID**: `ocid1.instance.oc1.phx.anyhqljs...`
   - **Availability Domain**: e.g., `fMgF:PHX-AD-1`

---

## Accessing Your Instance

### Connect via SSH

**On macOS/Linux**:
```bash
# Set proper permissions on private key
chmod 600 ~/.ssh/oci_key

# Connect (replace with your public IP)
ssh -i ~/.ssh/oci_key opc@144.24.xxx.xxx
```

**On Windows** (PowerShell):
```powershell
ssh -i $env:USERPROFILE\.ssh\oci_key opc@144.24.xxx.xxx
```

**Default username**: `opc` (Oracle Public Cloud)

**First time connecting**:
- You'll see: `The authenticity of host ... can't be established`
- Type `yes` to continue

**Troubleshooting**:

❌ **Connection Timeout**:
- Check security list allows port 22 from your IP
- Verify public IP is correct
- Wait a bit longer (provisioning might not be complete)

❌ **Permission Denied**:
- Verify you're using the correct private key
- Check key permissions: `chmod 600 ~/.ssh/oci_key`
- Ensure username is `opc`, not `root` or `ec2-user`

### Verify Instance Setup

Once connected, run:

```bash
# Check OS version
cat /etc/os-release

# Check instance metadata access
curl -s http://169.254.169.254/opc/v2/instance/ | head -20

# Update system packages
sudo yum update -y

# Install useful tools
sudo yum install -y git wget curl jq
```

**Success!** You now have a running compute instance in OCI.

---

## Identity and Access Management

IAM is the foundation of OCI security. Let's understand how it works.

### Core IAM Concepts

**Users**: Individual people who access OCI
**Groups**: Collections of users with similar access needs
**Policies**: Rules that grant permissions to groups
**Dynamic Groups**: Groups of compute instances (not people!)
**Compartments**: Scope where policies apply

### The Policy Language

OCI uses a simple policy language:

```
Allow <subject> to <verb> <resource-type> in <location>
```

**Examples**:
```
Allow group Developers to manage instances in compartment Development
Allow group DBAdmins to read databases in tenancy
Allow dynamic-group AppServers to use secret-family in compartment Production
```

**Verbs** (in order of permissions):
- `inspect` - List resources, see metadata
- `read` - View resource details
- `use` - Use but not modify
- `manage` - Full control (create, update, delete)

### Create a Policy for Your User

Let's give your user full access to the `customer-api-dev` compartment:

1. **☰ Menu** → **Identity & Security** → **Policies**
2. **Compartment**: Select **(root)** - your tenancy
   - **Note**: Policies are always created in root or a parent compartment
3. Click **Create Policy**
4. Fill in:
   - **Name**: `customer-api-dev-admin-policy`
   - **Description**: `Full access to customer-api-dev compartment`
   - **Compartment**: (root)
   - Toggle **Show manual editor**
   - **Policy Builder**: Add this statement:
     ```
     Allow group Administrators to manage all-resources in compartment customer-api-dev
     ```
5. Click **Create**

**What this does**: Grants your group (Administrators) full control over all resources in the `customer-api-dev` compartment.

---

## Dynamic Groups and Instance Principals

This is where OCI shines for secure, zero-secret authentication!

### What Are Dynamic Groups?

**Dynamic Groups** are groups of compute instances (not users) that match specific rules.

**Why They Matter**:
- Instances can authenticate to OCI services without API keys
- No secrets stored on the instance
- Automatically managed by OCI
- Foundation for zero-trust architecture

### How Instance Principals Work

```
┌─────────────────────────────────────────────┐
│         Compute Instance                    │
│  ┌────────────────────────────────────┐    │
│  │  1. App needs to call OCI API      │    │
│  │  2. Requests token from metadata   │    │
│  │     service (169.254.169.254)      │    │
│  │  3. Gets cryptographically signed  │    │
│  │     token proving instance identity│    │
│  │  4. Uses token to call OCI API     │    │
│  └────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────┐
│         OCI IAM Service                      │
│  ┌────────────────────────────────────┐    │
│  │  1. Verifies token signature       │    │
│  │  2. Checks if instance is in       │    │
│  │     dynamic group                  │    │
│  │  3. Checks policy permissions      │    │
│  │  4. Grants or denies access        │    │
│  └────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

**The Magic**: No API keys, no passwords, no secrets on disk!

### Create a Dynamic Group

Let's create a dynamic group for our instance:

1. **☰ Menu** → **Identity & Security** → **Dynamic Groups**
2. Click **Create Dynamic Group**
3. Fill in:
   - **Name**: `customer-api-instances`
   - **Description**: `All instances in customer-api-dev compartment`
   - **Matching Rules**: Add one of these rules:

**Option 1: Match by Instance OCID** (most specific):
```
instance.id = 'ocid1.instance.oc1.phx.anyhqljs2i3lvbic...'
```
Replace with your actual instance OCID.

**Option 2: Match by Compartment** (all instances in compartment):
```
instance.compartment.id = 'ocid1.compartment.oc1..aaaaaaaa...'
```
Replace with your `customer-api-dev` compartment OCID.

**I recommend Option 2** for flexibility - any instance you create in this compartment will automatically be in the group.

4. Click **Create**

**Save the Dynamic Group OCID** - you'll need it for Vault setup in Post 5!

### Create a Policy for the Dynamic Group

Now let's give our instances permission to access their own metadata:

1. **☰ Menu** → **Identity & Security** → **Policies**
2. **Compartment**: **(root)**
3. Click **Create Policy**
4. Fill in:
   - **Name**: `customer-api-instance-principals-policy`
   - **Description**: `Allow instances to authenticate via instance principals`
   - **Compartment**: (root)
   - Toggle **Show manual editor**
   - **Policy Builder**: Add these statements:
     ```
     Allow dynamic-group customer-api-instances to {AUTHENTICATION_INSPECT} in tenancy
     Allow dynamic-group customer-api-instances to {GROUP_MEMBERSHIP_INSPECT} in tenancy
     ```
5. Click **Create**

**What this does**:
- `AUTHENTICATION_INSPECT`: Allows instances to verify authentication requests
- `GROUP_MEMBERSHIP_INSPECT`: Allows checking group membership

These are the **minimum required permissions** for instance principals to work with Vault!

---

## The Metadata Service

The metadata service is how instances learn about themselves and get credentials.

### What Is It?

Every OCI compute instance has access to a special HTTP endpoint:

```
http://169.254.169.254/opc/v2/
```

This link-local address is **only accessible from within the instance** - not from the internet!

### Test the Metadata Service

SSH into your instance and run:

```bash
# Get instance information
curl -s -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq '.'
```

**Output** (example):
```json
{
  "id": "ocid1.instance.oc1.phx.anyhqljs2i3lvbic...",
  "displayName": "customer-api-instance",
  "compartmentId": "ocid1.compartment.oc1..aaaaaaaa...",
  "shape": "VM.Standard.E2.1.Micro",
  "region": "phx",
  "availabilityDomain": "fMgF:PHX-AD-1",
  "faultDomain": "FAULT-DOMAIN-1",
  "timeCreated": "2026-01-11T20:00:00.000Z",
  ...
}
```

**Key Metadata Fields**:
- `id`: Instance OCID (unique identifier)
- `compartmentId`: Which compartment the instance is in
- `region`: Geographic region
- `shape`: Instance type

### Why This Matters for Vault

In Post 5, when we configure Vault with OCI authentication:

1. Our Spring Boot app will request a token from this metadata service
2. The metadata service will return a cryptographically signed token
3. The app sends this token to Vault
4. Vault verifies the token with OCI
5. If valid, Vault grants access

**Zero secrets needed!** The metadata service proves the instance's identity.

### Metadata Service Versions

OCI has two metadata API versions:

**v1** (`/opc/v1/`):
- No authentication required
- Basic instance info
- Legacy, but still works

**v2** (`/opc/v2/`):
- Requires `Authorization: Bearer Oracle` header
- More secure
- Additional information
- **Recommended for new applications**

Always use **v2** for security!

---

## Security Best Practices

### 1. Use Compartments

❌ Don't put everything in root compartment  
✅ Create compartments per environment/project

### 2. Principle of Least Privilege

❌ Don't grant `manage all-resources`  
✅ Grant minimum required permissions

Example:
```
# Bad
Allow group Developers to manage all-resources in tenancy

# Good
Allow group Developers to manage instance-family in compartment Development
Allow group Developers to read virtual-network-family in compartment Development
```

### 3. Use Dynamic Groups for Instances

❌ Don't create API keys for instances  
✅ Use dynamic groups and instance principals

### 4. Restrict Security List Rules

❌ Don't allow `0.0.0.0/0` for all ports  
✅ Allow only required ports from specific sources

Example:
```
# Bad
Allow TCP 0-65535 from 0.0.0.0/0

# Good
Allow TCP 8080 from 203.0.113.0/24  # Your office IP range
Allow TCP 22 from 203.0.113.5/32    # Your specific IP
```

### 5. Regularly Rotate SSH Keys

Update your instance SSH keys every 90 days:

```bash
# On your local machine, generate new key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/oci_key_new

# In OCI Console: Instance Details → Resources → Instance access
# Add the new public key
# Remove the old key after verifying the new one works
```

### 6. Enable MFA

Enable Multi-Factor Authentication for your OCI account:

1. **Profile Icon** → **My Profile**
2. **Auth Tokens** → **Enable Multi-Factor Authentication**
3. Follow the setup wizard

### 7. Monitor with Audit Logs

OCI automatically logs all API calls:

1. **☰ Menu** → **Observability & Management** → **Audit**
2. Review who did what and when
3. Set up alerts for suspicious activity

---

## What's Next

Congratulations! You now understand OCI fundamentals and have:

✅ Created an OCI account  
✅ Set up networking (VCN, subnets, security lists)  
✅ Launched a compute instance  
✅ Configured IAM (users, groups, policies)  
✅ Created dynamic groups for instance principals  
✅ Tested the metadata service  

### Coming Up in This Series

**Post 3: Containerizing and Deploying Spring Boot on OCI**  
Take the Customer API from Post 1 and deploy it to the compute instance we just created. Make it accessible from the internet!

**Post 4: Secrets Management with HashiCorp Vault - Local Setup**  
Learn Vault fundamentals and integrate it with our Spring Boot app locally.

**Post 5: Zero-Trust Authentication - OCI Instance Principals with Vault**  
Deploy Vault on OCI and configure it to use instance principals for authentication - no secrets required!

---

## Complete Architecture So Far

Here's what we've built:

```
OCI Tenancy
  └── customer-api-dev (Compartment)
      └── customer-api-vcn (VCN: 10.0.0.0/16)
          ├── Public Subnet (10.0.0.0/24)
          │   └── customer-api-instance (VM.Standard.E2.1.Micro)
          │       ├── Public IP: 144.24.xxx.xxx
          │       ├── SSH Access on port 22
          │       └── Future: Spring Boot API on port 8080
          └── Private Subnet (10.0.1.0/24)
              └── (Reserved for databases in future posts)

IAM Configuration:
  ├── customer-api-instances (Dynamic Group)
  └── Policies:
      ├── customer-api-dev-admin-policy
      └── customer-api-instance-principals-policy
```

---

## Quick Reference: OCI CLI Commands

Install the OCI CLI on your instance:

```bash
# Install OCI CLI
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

# Configure (only needed for API key auth, not instance principals)
oci setup config

# Test instance principal authentication
oci os ns get --auth instance_principal

# List compartments
oci iam compartment list --auth instance_principal

# Get instance metadata via CLI
oci compute instance get \
  --instance-id $(curl -s http://169.254.169.254/opc/v1/instance/id) \
  --auth instance_principal
```

---

## Troubleshooting Common Issues

### Can't SSH to Instance

**Check Security List**:
- Ensure port 22 is open from your IP
- Rule should be: TCP 22 from `0.0.0.0/0` (or your specific IP)

**Check Instance State**:
- Instance status must be "Running"
- Public IP must be assigned

**Check SSH Key**:
- Verify you're using the correct private key
- Check permissions: `chmod 600 ~/.ssh/oci_key`
- Username is `opc`, not `root`

### Metadata Service Returns 403

The metadata service is always accessible from within an instance. A 403 typically means:
- You're not running the command from within the OCI instance
- Or metadata service is disabled (rare, usually enabled by default)

### Dynamic Group Not Working

**Check Matching Rules**:
- Verify your instance OCID or compartment OCID is correct
- Rules are case-sensitive
- Use exact format: `instance.id = 'ocid1...'` (with quotes)

**Check Policies**:
- Ensure policy is in root compartment
- Verify policy allows the actions you need
- Wait 30 seconds for policy changes to propagate

### Policy Not Taking Effect

- Policies can take up to 60 seconds to propagate
- Clear your browser cache
- Sign out and sign back in to OCI Console
- Verify policy is in the correct compartment

---

## Resource Cleanup

**Want to keep your instance?** Great! It's free tier eligible.

**Want to clean up?** Here's how:

```bash
# 1. Terminate Instance
# OCI Console → Compute → Instances →