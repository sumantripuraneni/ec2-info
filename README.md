# ec2-info

## Overview

A collection of Ansible playbooks for managing AWS EC2 instance information and VPC infrastructure. This repository provides Infrastructure-as-Code templates for automating EC2 metadata retrieval, instance configuration, and multi-environment VPC provisioning.

## Repository Structure

```
ec2-info/
├── ec2-instance-info.yaml           # EC2 instance metadata and MOTD management
├── fedora-lane-vpc-environment.yml   # Multi-environment VPC infrastructure
├── collections/                      # Ansible collections directory
└── README.md
```

## Key Playbooks

### 1. **ec2-instance-info.yaml**

**Purpose**: Retrieve EC2 instance metadata and dynamically create Message of the Day (MOTD) files based on environment tags.

**What It Does**:
- Queries EC2 instances by tag name (e.g., `flane-rhel-srv-01`)
- Extracts instance metadata and tags
- Creates MOTD files on target instances with environment-specific messages
- Supports conditional logic based on Environment tags (production, qa, dev)

**Key Tasks**:
```yaml
- amazon.aws.ec2_instance_info      # Fetch instance facts
- amazon.builtin.copy               # Create MOTD files
- Conditional delegation to instances based on tags
```

**Use Case**: Automatically customize welcome messages or system notifications on EC2 instances based on their deployment environment.

---

### 2. **fedora-lane-vpc-environment.yml**

**Purpose**: Create and manage a complete multi-tier VPC infrastructure for development, QA, and production environments.

**Infrastructure Components Created**:

| Component | Details |
|-----------|---------|
| **VPCs** | 3 VPCs (dev, qa, prod) with CIDR `10.0.0.0/16` each |
| **Internet Gateways** | One per VPC for internet connectivity |
| **Subnets** | Public subnet `10.0.1.0/24` in each VPC |
| **Route Tables** | Default routes pointing to IGW for internet access |
| **Security Groups** | Environment-specific groups with open ports |

**Ports Opened in Security Groups**:
- **Port 22** (SSH) - Remote access
- **Port 2222** (Custom SSH) - Alternative SSH access
- **Port 35443** (HTTPS variant) - Custom application traffic

**Supported Environments**: development, qa, production

---

## Starting Point / Quick Usage

### Prerequisites

```bash
# Install Ansible
pip install ansible

# Install AWS collection for Ansible
ansible-galaxy collection install amazon.aws

# Configure AWS credentials
export AWS_ACCESS_KEY_ID=<your_key>
export AWS_SECRET_ACCESS_KEY=<your_secret>
export AWS_DEFAULT_REGION=us-east-2
```

### Running the VPC Playbook

```bash
# Create infrastructure for all environments
ansible-playbook fedora-lane-vpc-environment.yml

# With debug output
ansible-playbook fedora-lane-vpc-environment.yml -e debug=true
```

### Running the Instance Info Playbook

```bash
# Deploy MOTD files based on instance tags
ansible-playbook ec2-instance-info.yaml
```

---

## Key Features

✅ **Multi-Environment Support** - Separate dev/qa/prod VPC infrastructure  
✅ **Automated Metadata Retrieval** - Query EC2 instances and tags via Ansible  
✅ **Conditional Provisioning** - Apply configurations based on instance environment tags  
✅ **IaC Best Practices** - YAML-based declarative infrastructure  
✅ **Modular Design** - Separate playbooks for different concerns  

---

## Configuration Notes

- **Default Region**: `us-east-2`
- **VPC CIDR Block**: `10.0.0.0/16` (all environments)
- **Subnet CIDR Block**: `10.0.1.0/24`
- **Delegation Pattern**: Tasks delegate to instances via public DNS names

---

## Use Cases

1. **Infrastructure Automation** - Spin up multi-tier VPC environments with a single command
2. **System Customization** - Automatically configure EC2 instances with environment-specific MOTD/settings
3. **Multi-Tenant Infrastructure** - Separate isolated environments for dev/qa/prod workloads
4. **CI/CD Integration** - Integrate with deployment pipelines to auto-provision infrastructure

---

## Ansible Collections Used

- `amazon.aws.*` - AWS resource management (EC2, VPC, security groups, route tables)
- `ansible.builtin.*` - Core Ansible modules (debug, copy, loops)

---

## Typical Workflow

```
1. Run fedora-lane-vpc-environment.yml
   ↓
   Creates 3 VPCs + IGWs + Subnets + Security Groups
   
2. Provision EC2 instances in target VPC
   ↓
   Tag instances (Environment: dev/qa/production)
   
3. Run ec2-instance-info.yaml
   ↓
   Retrieves instance metadata and applies MOTD configurations
```

---

## Next Steps for Integration

If you need to use this repository:

1. **Customize VPC CIDR blocks** if they conflict with existing infrastructure
2. **Modify instance tag names** in `ec2-instance-info.yaml` to match your actual instances
3. **Add more security group rules** for your application ports (beyond 22, 2222, 35443)
4. **Extend MOTD content** with your organization-specific messages
5. **Add variables file** (`.yml`) for environment-specific configurations

---

This is a practical infrastructure-as-code foundation for managing AWS EC2 and VPC resources programmatically via Ansible.
