# Enterprise VPC Terraform Template

[![Terraform](https://img.shields.io/badge/Terraform-1.5%2B-623CE4?logo=terraform)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/AWS-VPC-FF9900?logo=amazon-aws)](https://aws.amazon.com/vpc/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)


> **⚠️ Demo Template Notice**  
> This is a demonstration template showcasing modular VPC design patterns. For production-grade implementations with organization-specific configurations, Transit Gateway integration, multi-account patterns, and advanced security controls, please [click here](https://sales.com).

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture Components](#architecture-components)
- [VPC Endpoint Types](#vpc-endpoint-types)
- [Quick Start](#quick-start)
- [Module Structure](#module-structure)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Use Cases](#use-cases)
- [Production Considerations](#production-considerations)
- [Contact](#contact)

---

## 🎯 Overview

This template provides a foundation for deploying AWS VPC infrastructure using Terraform, demonstrating:

- **Modular Design**: Separate modules for VPC, subnets, route tables, security groups, and endpoints
- **Flexible Endpoint Configuration**: Toggle Gateway and Interface VPC Endpoints via variables
- **Enterprise Patterns**: Structure reflects multi-VPC architectures used in large organizations
- **Clean Abstractions**: Easy to extend and customize for specific requirements

### What This Template Includes

✅ VPC with configurable CIDR blocks  
✅ Public and Private subnets across multiple AZs  
✅ Gateway VPC Endpoints (S3, DynamoDB)  
✅ Interface VPC Endpoints (SageMaker Runtime, ECR, etc.)  
✅ Route tables with proper associations  
✅ Security groups for VPC endpoints  
✅ Internet Gateway and NAT Gateway support  

### What This Template Does NOT Include

❌ Transit Gateway configuration  
❌ Multi-account organization setup  
❌ VPC Peering or PrivateLink configurations  
❌ Production-grade IAM roles and policies  
❌ Centralized logging and monitoring  
❌ Cost optimization and tagging strategies  
❌ Compliance and audit controls  

---

## 🏗️ Architecture Components

Modern enterprise AWS architectures typically follow a **hub-and-spoke** pattern with specialized VPCs. While this template focuses on a single VPC, understanding the broader context is essential for scalable cloud design.

### 1. **Network VPC (Hub)**
The central connectivity hub in a multi-VPC architecture.

**Purpose:**
- Centralized network transit and routing
- Hosts Transit Gateway attachments
- Provides shared networking services

**Key Components:**
- Transit Gateway
- Route 53 Resolver endpoints
- Centralized NAT Gateways (optional)
- Network inspection (AWS Network Firewall)

**Use Case:** A financial services company uses a Network VPC to route traffic between 50+ application VPCs across dev, staging, and production environments.

---

### 2. **Shared Services VPC**
Centralized infrastructure shared across multiple workloads.

**Purpose:**
- Host common services used by application VPCs
- Reduce duplication and costs
- Centralize management and updates

**Common Services:**
- Active Directory (AWS Managed Microsoft AD)
- DNS servers (Route 53 Resolver)
- Patch management systems
- Centralized logging (Kinesis, S3)
- Container registries (ECR)
- Artifact repositories

**Use Case:** A SaaS company runs all their CI/CD tools, container registries, and shared databases in a Shared Services VPC, accessible to 20+ application VPCs via Transit Gateway.

---

### 3. **Audit/Logging VPC**
Dedicated VPC for security, compliance, and audit workloads.

**Purpose:**
- Isolate security and audit infrastructure
- Centralized log aggregation
- Security monitoring and alerting

**Key Components:**
- CloudTrail log aggregation
- VPC Flow Logs collection
- Security tools (GuardDuty, Security Hub)
- SIEM integration endpoints
- Compliance scanning tools

**Use Case:** A healthcare provider maintains an Audit VPC that collects logs from all production VPCs to meet HIPAA compliance requirements, with restricted access limited to the security team.

---

### 4. **Transit Gateway**
AWS-managed network transit hub enabling VPC-to-VPC and VPC-to-on-premises connectivity.

**Purpose:**
- Simplified network topology
- Centralized routing policies
- Scalable VPC connectivity

**Key Features:**
- Connect thousands of VPCs
- Route table isolation
- VPN and Direct Connect integration
- Inter-region peering

**Routing Patterns:**
- **Isolated:** VPCs can't communicate with each other
- **Shared Services:** All VPCs can reach Shared Services VPC
- **Segmented:** Production/Dev/Test have separate routing domains

**Use Case:** A global enterprise uses Transit Gateway to connect 200+ VPCs across 5 AWS regions, with routing policies that isolate development from production while allowing both to access shared services.

---

### 5. **Egress VPC**
Centralized egress point for internet-bound traffic from private VPCs.

**Purpose:**
- Centralized internet egress
- Consistent security controls
- Simplified IP whitelisting

**Key Components:**
- NAT Gateways
- AWS Network Firewall
- IDS/IPS appliances
- Proxy servers
- URL filtering

**Benefits:**
- Single source IP for whitelisting
- Centralized traffic inspection
- Reduced NAT Gateway costs
- Consistent security policies

**Use Case:** A financial institution routes all internet-bound traffic from 30+ application VPCs through an Egress VPC with Network Firewall, inspecting all traffic and blocking connections to known malicious domains.

---

### 6. **Application VPC**
Workload-specific VPCs for running applications and services.

**Purpose:**
- Isolate applications and environments
- Dedicated network spaces per team/application
- Environment-specific configurations (dev, staging, prod)

**Common Patterns:**
- One VPC per environment per application
- Microservices in dedicated VPCs
- Data tier in separate VPCs

**This Template:** This repository provides the foundation for building an Application VPC with proper subnet segmentation, VPC endpoints for AWS services, and security controls.

---

## 🔌 VPC Endpoint Types

VPC Endpoints enable private connectivity to AWS services without requiring internet gateways, NAT devices, or public IPs.

### Gateway Endpoints

**Characteristics:**
- Specified as route table targets
- No additional charges
- Horizontally scaled and highly available
- Supports S3 and DynamoDB only

**How They Work:**
1. Create Gateway Endpoint in VPC
2. Add route to route table: `pl-xxx (S3) → vpce-xxx`
3. Traffic to S3 uses VPC endpoint instead of IGW

**Example Route Table Entry:**
```
Destination              Target
10.0.0.0/16              local
0.0.0.0/0                igw-xxx
pl-63a5400a (S3)         vpce-12345
```

**Use Case:** An ML pipeline reads training data from S3 without traversing the internet, reducing latency and improving security.

**Configuration in This Template:**
```hcl
enable_s3_gateway       = true
enable_dynamodb_gateway = true
```

---

### Interface Endpoints (AWS PrivateLink)

**Characteristics:**
- Elastic Network Interfaces (ENIs) in your subnets
- Charged per endpoint per hour + data processed
- Supports 100+ AWS services
- Private DNS enabled by default

**How They Work:**
1. Create Interface Endpoint in VPC
2. ENI created in specified subnets
3. Private DNS resolves service name to private IP
4. Traffic stays within AWS network

**Example DNS Resolution:**
```
sagemaker-runtime.us-east-1.amazonaws.com
→ vpce-0abc123-xyz456.sagemaker-runtime.us-east-1.vpce.amazonaws.com
→ 10.0.1.50 (private IP)
```

**Supported Services (Partial List):**
- SageMaker Runtime
- ECR (Docker & API)
- ECS
- Systems Manager
- Secrets Manager
- KMS
- CloudWatch Logs
- And 100+ more...

**Use Case:** A containerized application pulls images from ECR without internet access, improving security and reducing data transfer costs.

**Configuration in This Template:**
```hcl
enable_sagemaker_runtime = true
enable_ecr_dkr          = true
enable_ecr_api          = true
enable_logs             = true
```

---

### Gateway vs Interface: Decision Matrix

| Factor | Gateway Endpoint | Interface Endpoint |
|--------|-----------------|-------------------|
| **Cost** | Free | ~$0.01/hour + data charges |
| **Services** | S3, DynamoDB only | 100+ services |
| **Implementation** | Route table entry | ENI in subnet |
| **DNS** | N/A | Private DNS supported |
| **Availability** | Regional | AZ-level (deploy in multiple AZs) |
| **Security Groups** | N/A | Yes |
| **Use When** | S3/DynamoDB access | Other AWS services, strict private connectivity |

---

## 🚀 Quick Start

### Prerequisites

- Terraform 1.5 or later
- AWS CLI configured with appropriate credentials
- An AWS account with VPC creation permissions

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/terraform-aws-vpc-template.git
   cd terraform-aws-vpc-template
   ```

2. **Copy example variables**
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   ```

3. **Edit terraform.tfvars**
   ```bash
   # Customize your VPC configuration
   vim terraform.tfvars
   ```

4. **Initialize Terraform**
   ```bash
   terraform init
   ```

5. **Review the plan**
   ```bash
   terraform plan
   ```

6. **Deploy the infrastructure**
   ```bash
   terraform apply
   ```

### Basic Configuration Example

```hcl
# terraform.tfvars
region               = "us-east-1"
vpc_cidr             = "10.0.0.0/16"
availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
environment          = "dev"
project_name         = "ml-platform"

# Gateway Endpoints (free)
enable_s3_gateway       = true
enable_dynamodb_gateway = true

# Interface Endpoints (consider cost)
enable_sagemaker_runtime = true
enable_ecr_dkr          = true
enable_logs             = true

# Networking
enable_nat_gateway = true
single_nat_gateway = true  # false for HA across AZs
```

---

## 📦 Module Structure

```
terraform-aws-vpc-template/
├── main.tf                      # Root module orchestration
├── variables.tf                 # Input variables
├── outputs.tf                   # Output values
├── terraform.tfvars.example     # Example configuration
├── README.md                    # This file
│
└── modules/
    ├── vpc/                     # VPC and IGW
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── subnets/                 # Public and private subnets
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── route_tables/            # Route tables and associations
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── vpc_endpoints/           # Gateway and Interface endpoints
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    └── security_groups/         # Security groups for endpoints
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## ⚙️ Configuration

### Core Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `region` | string | `"us-east-1"` | AWS region |
| `vpc_cidr` | string | `"10.0.0.0/16"` | VPC CIDR block |
| `availability_zones` | list(string) | `["us-east-1a", "us-east-1b"]` | AZs for subnets |
| `environment` | string | `"dev"` | Environment name |
| `project_name` | string | `"demo"` | Project identifier |

### Subnet Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `public_subnet_cidrs` | list(string) | `["10.0.1.0/24", "10.0.2.0/24"]` | Public subnet CIDRs |
| `private_subnet_cidrs` | list(string) | `["10.0.11.0/24", "10.0.12.0/24"]` | Private subnet CIDRs |

### NAT Gateway

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `enable_nat_gateway` | bool | `true` | Enable NAT Gateway |
| `single_nat_gateway` | bool | `false` | Single NAT (cost savings) vs HA |

### Gateway Endpoints (Free)

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `enable_s3_gateway` | bool | `true` | Enable S3 Gateway Endpoint |
| `enable_dynamodb_gateway` | bool | `false` | Enable DynamoDB Gateway Endpoint |

### Interface Endpoints (Charged)

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `enable_sagemaker_runtime` | bool | `false` | SageMaker Runtime endpoint |
| `enable_ecr_dkr` | bool | `false` | ECR Docker endpoint |
| `enable_ecr_api` | bool | `false` | ECR API endpoint |
| `enable_logs` | bool | `false` | CloudWatch Logs endpoint |
| `enable_ssm` | bool | `false` | Systems Manager endpoint |

---

## 📊 Deployment

### Terraform Commands

```bash
# Initialize and download providers
terraform init

# Validate configuration syntax
terraform validate

# Format code to canonical style
terraform fmt -recursive

# Plan infrastructure changes
terraform plan -out=tfplan

# Apply changes
terraform apply tfplan

# Show current state
terraform show

# Destroy infrastructure (use with caution)
terraform destroy
```

### Outputs

After deployment, Terraform outputs key resource identifiers:

```hcl
Outputs:

vpc_id = "vpc-0abc123456789"
vpc_cidr = "10.0.0.0/16"
public_subnet_ids = [
  "subnet-0x1",
  "subnet-0x2"
]
private_subnet_ids = [
  "subnet-0y1",
  "subnet-0y2"
]
s3_gateway_endpoint_id = "vpce-0s3gateway"
sagemaker_runtime_endpoint_id = "vpce-0sagemaker"
nat_gateway_ids = ["nat-0abc123"]
```

---

## 🎯 Use Cases

### 1. ML/AI Workloads on AWS

**Scenario:** Deploy SageMaker training jobs that pull data from S3 and push models to ECR, all without internet access.

**Configuration:**
```hcl
enable_s3_gateway        = true
enable_sagemaker_runtime = true
enable_ecr_dkr          = true
enable_ecr_api          = true
```

**Architecture:**
- Private subnets host SageMaker notebook instances
- S3 Gateway Endpoint for training data access
- SageMaker Runtime Interface Endpoint for API calls
- ECR endpoints for pulling custom container images

---

### 2. Secure Container Deployments

**Scenario:** ECS/EKS clusters pulling images from ECR with no internet connectivity.

**Configuration:**
```hcl
enable_ecr_dkr = true
enable_ecr_api = true
enable_logs    = true
```

**Architecture:**
- Private subnets host ECS tasks
- ECR endpoints enable image pulls
- CloudWatch Logs endpoint for container logging

---

### 3. Serverless Data Processing

**Scenario:** Lambda functions processing data from S3 and DynamoDB.

**Configuration:**
```hcl
enable_s3_gateway       = true
enable_dynamodb_gateway = true
```

**Architecture:**
- Lambda functions in private subnets (VPC-enabled)
- Gateway endpoints reduce NAT costs to zero for S3/DynamoDB traffic
- Improved latency and security

---

## 🏢 Production Considerations

### What's Missing from This Template

This template is intentionally simplified. Production implementations require:

#### 1. **Multi-Account Strategy**
- AWS Organizations with SCPs
- Account vending automation
- Cross-account VPC connectivity
- Centralized billing and cost allocation

#### 2. **Transit Gateway Architecture**
- Hub-and-spoke network design
- Route table segmentation
- Inter-region peering
- VPN and Direct Connect integration

#### 3. **Advanced Security**
- Network Firewall for egress filtering
- VPC Flow Logs to S3 with Athena analysis
- GuardDuty threat detection
- Security Hub compliance checks
- Network segmentation with Security Groups and NACLs

#### 4. **High Availability**
- Multi-AZ NAT Gateways
- Interface endpoints in all AZs
- Load balancer integration
- Automated failover

#### 5. **Observability**
- CloudWatch Log Groups per endpoint
- VPC Flow Logs aggregation
- Cost and usage monitoring
- Performance metrics and alarms

#### 6. **Compliance & Governance**
- Resource tagging strategies
- Config rules for drift detection
- CloudFormation StackSets for multi-region
- Automated compliance scanning

#### 7. **Cost Optimization**
- Interface endpoint consolidation
- S3 Gateway endpoints over NAT
- VPC endpoint policies for fine-grained access
- Right-sizing NAT Gateways

---

## 🤝 Contact & Consulting

This template demonstrates foundational VPC patterns. For production-grade architecture including:

- **Multi-VPC Hub-and-Spoke Design** with Transit Gateway
- **Centralized Egress VPC** with Network Firewall
- **Shared Services VPC** for common infrastructure
- **Audit/Logging VPC** for compliance
- **Multi-Account Organization Setup**
- **Cost-Optimized Endpoint Strategy**
- **Security Controls & Compliance** (HIPAA, PCI-DSS, SOC 2)

### Let's Connect

- **Email:** londonhouse.email@gmail.com
- **LinkedIn:** [linkedin.com/in/christopher-london-aa78aa2b](https://www.linkedin.com/in/christopher-london-aa78aa2b)
- **Medium:** [@cloudybusiness1](https://medium.com/@cloudybusiness1)
- **GitHub:** [Your GitHub Profile]

---

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## 🙏 Acknowledgments

Built with best practices from:
- AWS Well-Architected Framework
- HashiCorp Terraform Documentation
- Enterprise AWS Architecture Patterns

---

**⭐ If this template helped you, consider starring the repo and sharing it with others building on AWS!**
