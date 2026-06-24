# ec2-info

---

## 🎯 Overview

The **`terraform-vpc`** branch evolves the Ansible-based infrastructure from `main` into a **Terraform-first Infrastructure-as-Code** approach, while introducing a multi-tier **RHEL Image Mode (bootc) Control Plane** environment for building, testing, and distributing containerized RHEL appliances.

This branch contains two major deployment patterns:

1. **Legacy Ansible Playbooks** (preserved from main branch)
   - EC2 instance metadata management
   - VPC environment provisioning via Ansible

2. **New Terraform-based Infrastructure** (image-mode-hack-a-thon/)
   - Complete RHEL 9/10 control plane provisioning
   - 6-tier architecture: Bastion, Satellite, IdM, Quay, Jenkins, Ansible AAP
   - Identity Management (FreeIPA) automation
   - Production-grade security and networking

---

## 📁 Repository Structure

```
terraform-vpc/ (branch)
├── ec2-instance-info.yaml                    # [Legacy] Instance MOTD management
├── fedora-lane-vpc-environment.yml            # [Legacy] Ansible VPC provisioning
├── playbooks/                                 # NEW: Advanced playbook directory
│   ├── README-idm-install.md                 # Comprehensive IdM setup documentation
│   ├── deploy-idm-pre-reqs.yml               # IdM/FreeIPA prerequisites & installation
│   └── inventory/                            # Ansible inventory structures
├── image-mode-hack-a-thon/                   # NEW: Terraform control plane
│   ├── README.md                             # Main architecture documentation
│   ├── README-HOWTO-CONNECT.md               # SSH tunneling & connectivity guide
│   ├── create-environment.sh                 # Automated deployment script
│   ├── provider.tf                           # Terraform provider configuration
│   ├── network.tf                            # VPC, subnets, NAT gateway (terraform-aws-modules/vpc)
│   ├── security.tf                           # Security groups, key pairs
│   ├── instances.tf                          # 6-tier EC2 instance provisioning
│   └── outputs.tf                            # Terraform outputs (IP addresses, etc)
└── .gitignore                                # Terraform-specific ignores (.terraform/, .tfstate, etc)
```

---

## 🏗️ Part 1: Legacy Ansible Components (from main)

### 1.1 EC2 Instance Info Playbook

**File:** `ec2-instance-info.yaml`

Queries EC2 instances by tag and creates MOTD files based on environment classification.

```yaml
# Sample Task
- amazon.aws.ec2_instance_info:
    region: us-east-2
    filters:
      tag:Name: flane-rhel-srv-01

- ansible.builtin.copy:
    dest: /etc/motd.d/flane-motd
    content: "Welcome to {{ environment }} host"
```

### 1.2 Fedora Lane VPC Playbook

**File:** `fedora-lane-vpc-environment.yml`

Creates 3 isolated VPC environments (dev, qa, production) with:
- VPCs (CIDR: `10.0.0.0/16`)
- Internet Gateways
- Subnets (`10.0.1.0/24`)
- Security Groups (ports 22, 2222, 35443)
- Route tables and default routes

---

## 🚀 Part 2: NEW Terraform Control Plane Architecture

### 2.1 Purpose & Use Case

**Target:** Deploy a **complete RHEL Image Mode engineering environment** in AWS `us-west-2`.

**Image Mode (bootc)** = Container-native RHEL appliances built, tested, and versioned as OCI images.

This infrastructure serves as the foundational **"OS Factory"** for:
- Building custom RHEL bootc appliances
- Managing identity/authentication (IdM/FreeIPA)
- Housing container registries (Quay)
- Running CI/CD pipelines (Jenkins)
- Orchestrating infrastructure (Ansible AAP)
- Caching package repositories (Red Hat Satellite)

---

### 2.2 Network Architecture

```
┌─────────────────────────────────────────────────┐
│  AWS Region: us-west-2 (3 AZs)                 │
│  CIDR: 10.20.0.0/16                            │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─ PUBLIC SUBNET TIER ─────────┐              │
│  │  10.20.101-103.0/24           │              │
│  │  + NAT Outbound Gateway       │              │
│  │                               │              │
│  │  [BASTION (t3.small)]         │ ← Public IP  │
│  │  200GB / RHEL9                │              │
│  └───────────────────────────────┘              │
│          ↓ (SSH ProxyJump)                      │
│  ┌─ PRIVATE SUBNET TIER ────────┐              │
│  │  10.20.1-3.0/24               │              │
│  │  + NAT Outbound Only          │              │
│  │                               │              │
│  │  [SATELLITE]     [JENKINS]    │              │
│  │  m6i.2xlarge     c6i.xlarge   │              │
│  │  500GB           150GB        │              │
│  │                               │              │
│  │  [IDM]           [ANSIBLE]    │              │
│  │  m6i.xlarge      m6i.xlarge   │              │
│  │  50GB            100GB        │              │
│  │                               │              │
│  │  [QUAY]                       │              │
│  │  m6i.xlarge                   │              │
│  │  100GB                        │              │
│  └───────────────────────────────┘              │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

### 2.3 Infrastructure Inventory

| Host | Instance Type | AMI | Disk | Purpose | Network |
|------|---------------|-----|------|---------|---------|
| **bastion** | `t3.small` | RHEL 9 | 200GB gp3 | SSH proxy jump, admin gateway | **PUBLIC** (has public IP) |
| **satellite** | `m6i.2xlarge` | RHEL 9 | 500GB gp3 (IOPS 3000) | Red Hat Satellite 6.16 (package repos) | Private |
| **idm** | `m6i.xlarge` | RHEL 10 | 50GB gp3 | FreeIPA (Kerberos/LDAP identity) | Private |
| **quay** | `m6i.xlarge` | RHEL 9 | 100GB gp3 | Red Hat Quay (OCI registry) | Private |
| **jenkins** | `c6i.xlarge` | RHEL 9 | 150GB gp3 | Jenkins CI/CD (bootc-image-builder) | Private |
| **ansible** | `m6i.xlarge` | RHEL 9 | 100GB gp3 | Ansible Automation Platform (AAP) | Private |

**Key Design:**
- ✅ **Bastion** = Only public-facing node (high isolation)
- ✅ All private nodes use **SSH ProxyJump** through Bastion
- ✅ All private nodes exit to internet via **NAT Gateway** (for `dnf update`, etc)
- ✅ Security group allows SSH (22) + VPC mesh traffic
- ✅ All disks encrypted at rest (`encrypted = true`)

---

### 2.4 Terraform Configuration Breakdown

#### **provider.tf**
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}
```

**Key Points:**
- Targets AWS Provider v6.x (latest)
- Region locked to `us-west-2`

---

#### **network.tf**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "us-west-imagemode-vpc"
  cidr = "10.20.0.0/16"

  azs             = slice(data.aws_availability_zones.available.names, 0, 3)
  private_subnets = ["10.20.1.0/24", "10.20.2.0/24", "10.20.3.0/24"]
  public_subnets  = ["10.20.101.0/24", "10.20.102.0/24", "10.20.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true  # Cost optimization: shared NAT for all AZs
  enable_dns_hostnames = true
  enable_dns_support   = true
}
```

**Key Points:**
- Uses **terraform-aws-modules/vpc** (community module)
- 3 AZs for HA
- Single NAT Gateway (cost-effective for non-production)
- DNS resolution enabled for service discovery

---

#### **security.tf**
```hcl
# SSH Key from local filesystem
resource "aws_key_pair" "bastion_key" {
  key_name   = "us-west-2-bastion-key"
  public_key = file("~/.ssh/aws_bastion_key.pub")
}

# Security Group: Bastion + VPC mesh
resource "aws_security_group" "bastion_sg" {
  name        = "us-west-2-bastion-sg"
  description = "Security baseline for Image Mode infrastructure"
  vpc_id      = module.vpc.vpc_id

  # Inbound
  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ⚠️ Restrict to your IP in production
  }

  ingress {
    description = "VPC-internal mesh"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    self        = true
  }

  # Outbound
  egress {
    description = "Unrestricted outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Key Points:**
- SSH key loaded from `~/.ssh/aws_bastion_key.pub`
- Single security group (reused for all instances)
- Port 22 + all internal VPC traffic allowed
- ⚠️ SSH open to `0.0.0.0/0` (restrict before production use)

---

#### **instances.tf** (excerpt)

```hcl
# Bastion Host
resource "aws_instance" "bastion" {
  ami                         = data.aws_ami.rhel9.id
  instance_type               = "t3.small"
  subnet_id                   = module.vpc.public_subnets[0]
  associate_public_ip_address = true  # Force public IP
  key_name                    = aws_key_pair.bastion_key.key_name
  
  root_block_device {
    volume_size = 200
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name        = "bastion"
    Environment = "hack-a-thon"
  }
}

# Private node example: Satellite
resource "aws_instance" "satellite" {
  ami                    = data.aws_ami.rhel9.id
  instance_type          = "m6i.2xlarge"
  subnet_id              = module.vpc.private_subnets[0]
  key_name               = aws_key_pair.bastion_key.key_name
  vpc_security_group_ids = [aws_security_group.bastion_sg.id]

  root_block_device {
    volume_size = 500
    volume_type = "gp3"
    iops        = 3000
    throughput  = 125
    encrypted   = true
  }

  tags = {
    Name        = "satellite"
    Environment = "hack-a-thon"
  }
}
```

**Key Pattern:**
- AMI data sources fetch **latest RHEL 9/10** from Red Hat (owner: `309956199498`)
- All instances use KMS encryption
- gp3 volumes for performance control
- Satellite gets high IOPS for package repository workload

---

#### **outputs.tf**

```hcl
output "public_bastion_ip" {
  value       = aws_instance.bastion.public_ip
  description = "Public IPv4 for SSH proxy jumping"
}

output "internal_control_plane_ips" {
  value = {
    satellite = aws_instance.satellite.private_ip
    idm       = aws_instance.idm.private_ip
    quay      = aws_instance.quay.private_ip
    jenkins   = aws_instance.jenkins.private_ip
    ansible   = aws_instance.ansible.private_ip
  }
  description = "Private VPC routing targets"
}
```

---

## 🧠 Part 3: Identity Management (IdM) Automation

### 3.1 IdM Prerequisites Playbook

**File:** `playbooks/deploy-idm-pre-reqs.yml`

Prepares a RHEL host for **FreeIPA/Identity Management** deployment (8 phases).

**Key Automations:**

| Phase | Task | Why |
|-------|------|-----|
| 1 | Install AWS RHUI client packages | Ensure yum repositories configured for AWS-hosted RHEL |
| 2 | Set FQDN (`idm.aws.redhat.local`), pin hostname in cloud-init | Prevent dhcp from overriding identity |
| 3 | Strip `myhostname` from nsswitch.conf, restart NetworkManager | Fix DNS lookup conflicts on AWS network overlay |
| 4 | Enable IPv6 on loopback, configure sysctl | FreeIPA's embedded Dogtag CA requires IPv6 |
| 5 | Pre-inject LDAP schema optimizations | Performance tuning for 389 Directory Server |
| 6 | Install `ipa-server` + `ipa-server-dns` packages | Core Identity Management binaries |
| 7 | Configure firewalld, open ports 22/80/443/389/636/88/464/53 (TCP+UDP) | Identity/Kerberos/DNS service accessibility |
| 8 | Run `ipa-server-install --unattended` with vault passwords | Automated cluster initialization |

**Playbook Usage:**
```bash
# Create encrypted vault with domain credentials
ansible-vault create vars-vault.yml
# (enter: idm_vault_dm_password, idm_vault_admin_password)

# Run playbook
ansible-playbook \
  -i inventory/hosts \
  playbooks/deploy-idm-pre-reqs.yml \
  --ask-vault-pass
```

**Result:**
```
✅ FreeIPA server operational at idm.aws.redhat.local
✅ Kerberos realm: AWS.REDHAT.LOCAL
✅ Admin principal: admin@AWS.REDHAT.LOCAL
✅ LDAP directory: cn=accounts,dc=aws,dc=redhat,dc=local
```

---

### 3.2 IdM Post-Deployment Verification

```bash
# SSH into IdM host
ssh -J ec2-user@<BASTION_IP> ec2-user@<IDM_PRIVATE_IP>

# Test Kerberos authentication
KRB5_TRACE=/dev/stdout kinit admin
# Enter password: Admin_Secret_Pass123

# List acquired tickets
klist
# Expected: krbtgt/AWS.REDHAT.LOCAL@AWS.REDHAT.LOCAL
```

---

## 🚀 Part 4: Deployment Workflows

### 4.1 Option A: Terraform Deployment (Image Mode Control Plane)

**Directory:** `image-mode-hack-a-thon/`

#### Step 1: Prerequisites

```bash
# Ensure SSH key exists
ls -l ~/.ssh/aws_bastion_key.pub

# Install Terraform
terraform version  # Should be >= 1.5.0
```

#### Step 2: Initialize & Plan

```bash
cd image-mode-hack-a-thon/

# Download modules and provider
terraform init --upgrade

# Validate syntax
terraform validate

# Preview resources
terraform plan
```

#### Step 3: Deploy Infrastructure

```bash
# Option A: Manual approval
terraform apply

# Option B: Auto-approve (scripted)
terraform apply -auto-approve

# OR use the provided shell script
bash create-environment.sh
```

#### Step 4: Capture Outputs

```bash
# Print IPs and connection details
terraform output

# Example output:
# public_bastion_ip = "18.236.159.51"
# internal_control_plane_ips = {
#   ansible   = "10.20.2.68"
#   idm       = "10.20.2.100"
#   jenkins   = "10.20.1.59"
#   quay      = "10.20.3.59"
#   satellite = "10.20.1.103"
# }
```

#### Step 5: Connect via SSH ProxyJump

```bash
# Direct SSH (simple terminal access)
ssh -J ec2-user@18.236.159.51 ec2-user@10.20.2.100 -i ~/.ssh/aws_bastion_key

# OR configure ~/.ssh/config for aliases (see connectivity guide)
ssh idm
```

#### Step 6: HTTPS Tunneling (for web consoles)

```bash
# In one terminal, create forwarding tunnel
ssh -N -L 7443:10.20.2.100:443 ec2-user@18.236.159.51 -i ~/.ssh/aws_bastion_key

# In browser, visit:
# https://127.0.0.1:7443  (IdM WebUI)
```

---

### 4.2 Option B: Ansible Playbook Deployment (Legacy VPC)

**File:** `fedora-lane-vpc-environment.yml`

```bash
# Create VPC infrastructure
ansible-playbook fedora-lane-vpc-environment.yml

# With debug output
ansible-playbook fedora-lane-vpc-environment.yml -e debug=true
```

---

## 🔐 Security Best Practices (terraform-vpc)

✅ **Implemented:**
- KMS encryption on all EBS volumes
- Private subnets for sensitive workloads (IdM, Satellite)
- VPC-isolated NAT for outbound internet
- Security group-based firewall rules
- SSH key-pair authentication only

⚠️ **Needs Hardening:**
- SSH open to `0.0.0.0/0` → Restrict to your public IP before production
- Single NAT Gateway → Add HA NAT for production workloads
- No CloudWatch/VPC Flow Logs → Add monitoring
- Terraform state stored locally → Use S3 backend for team environments

---

## 📋 Quick Reference: File Purposes

| File | Branch | Purpose |
|------|--------|---------|
| `ec2-instance-info.yaml` | Both | Query EC2 instances, create MOTD files |
| `fedora-lane-vpc-environment.yml` | Both | Create 3 multi-tier VPCs (Ansible) |
| `playbooks/deploy-idm-pre-reqs.yml` | terraform-vpc | FreeIPA prerequisites + automated install |
| `image-mode-hack-a-thon/provider.tf` | terraform-vpc | Terraform AWS provider config |
| `image-mode-hack-a-thon/network.tf` | terraform-vpc | VPC + subnets + NAT (terraform-aws-modules) |
| `image-mode-hack-a-thon/security.tf` | terraform-vpc | SSH keys + security groups |
| `image-mode-hack-a-thon/instances.tf` | terraform-vpc | 6 EC2 instances (bastion + 5 private) |
| `image-mode-hack-a-thon/outputs.tf` | terraform-vpc | Terraform output values (IPs) |

---

## 🎓 Learning Path / Getting Started

**For Terraform approach (recommended):**

1. **Review architecture diagram** in README.md
2. **Set up SSH key:** `ls ~/.ssh/aws_bastion_key.pub`
3. **Run Terraform:** `cd image-mode-hack-a-thon && terraform init && terraform plan`
4. **Deploy:** `terraform apply`
5. **Connect:** Use SSH ProxyJump or configure `~/.ssh/config`
6. **Optional:** Deploy IdM prerequisites playbook into IdM instance

**For Ansible approach (legacy):**

1. Configure AWS credentials in environment
2. Run `fedora-lane-vpc-environment.yml` to create VPCs
3. Tag EC2 instances appropriately
4. Run `ec2-instance-info.yaml` for MOTD configuration

---

## 📌 Key Differences: main vs terraform-vpc

| Aspect | main (Ansible) | terraform-vpc |
|--------|---|---|
| **Infrastructure Code** | YAML playbooks | HCL (Terraform) |
| **Deployment Model** | Imperative (procedural) | Declarative (state-managed) |
| **VPC Setup** | 3 simple VPCs created inline | 1 complex multi-AZ VPC (terraform-aws-modules) |
| **Instance Management** | Tag-based filtering | Explicit resource declarations |
| **Control Plane** | None (basic setup only) | Full RHEL Image Mode factory (Satellite, Quay, Jenkins, IdM, Ansible AAP) |
| **Identity Mgmt** | Not included | FreeIPA/IdM automated deployment |
| **SSH Access Pattern** | Direct (assumes public access) | ProxyJump via Bastion (production-grade) |
| **State Management** | Stateless (idempotent playbooks) | Stateful (terraform.tfstate) |

---

## 🚨 Common Issues & Troubleshooting

### Issue: "File not found: ~/.ssh/aws_bastion_key.pub"
**Solution:** Generate key pair first:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/aws_bastion_key
```

### Issue: SSH ProxyJump connection timeout
**Solution:** Verify Bastion security group allows SSH (22) from your IP:
```bash
# In AWS Console, check security group inbound rules
# Or run from Bastion to test private connectivity
ssh -J ec2-user@BASTION_IP ec2-user@PRIVATE_IP
```

### Issue: Terraform plan fails with "InvalidKeyPair.NotFound"
**Solution:** SSH key must already be created locally; Terraform only imports it:
```bash
terraform destroy  # Clean up
ssh-keygen -t rsa -b 4096 -f ~/.ssh/aws_bastion_key
terraform apply
```

### Issue: IdM playbook fails on "ipa-server-install"
**Solution:** Ensure instance has minimum specs (m6i.xlarge, 50GB disk) and proper firewall rules are open.

---

## 📚 Additional Resources

- **Terraform AWS Provider:** https://registry.terraform.io/providers/hashicorp/aws/latest
- **Terraform VPC Module:** https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest
- **FreeIPA Documentation:** https://freeipa.org/page/Documentation
- **Red Hat Satellite:** https://access.redhat.com/products/red-hat-satellite
- **RHEL Image Mode (bootc):** https://containers.redhat.com/blog/

---

## 🎯 Summary

The **terraform-vpc** branch transitions from basic Ansible VPC provisioning to a **production-grade, Terraform-managed, RHEL Image Mode control plane** with integrated identity management, package caching, container registry, and CI/CD infrastructure.

**Best for:** Organizations building containerized RHEL appliances, needing centralized identity management (FreeIPA), and wanting Infrastructure-as-Code with state management.

**Start here:** Run `cd image-mode-hack-a-thon && terraform apply` to get a fully operational 6-node control plane in ~10 minutes.
