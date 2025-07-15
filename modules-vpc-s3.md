
-----

## Mini Project: Terraform Modules - VPC and S3 Bucket with Backend Storage

This project demonstrates how to use Terraform to create modularized configurations for an Amazon Virtual Private Cloud (VPC) and an Amazon S3 bucket. It also configures Terraform to use an Amazon S3 bucket with DynamoDB for state locking as the backend storage for its state file.

### **Project Structure:**

```
terraform-modules-vpc-s3/
├── backend.tf
├── main.tf
├── outputs.tf
├── variables.tf
├── README.md
└── modules/
    ├── s3/
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── vpc/
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

### **Prerequisites:**

1.  **AWS Account:** An active AWS account.
2.  **AWS CLI Configured:** Ensure you have the AWS CLI installed and configured with credentials that have sufficient permissions to create VPCs, S3 buckets, and DynamoDB tables in your chosen region.
3.  **Terraform Installed:** [Download and install Terraform](https://developer.hashicorp.com/terraform/downloads).
4.  **Pre-existing S3 Bucket for State:** You **MUST** manually create an S3 bucket to store your Terraform state file *before* running `terraform init`. This bucket should be globally unique.
      * Example command (replace `your-unique-terraform-state-bucket` and `us-east-1`):
        ```bash
        aws s3 mb s3://your-unique-terraform-state-bucket --region us-east-1
        ```
5.  **Pre-existing DynamoDB Table for State Locking:** You **MUST** manually create a DynamoDB table to handle state locking *before* running `terraform init`. This prevents concurrent modifications to the state file.
      * The table **must** be named `your-lock-table` (or whatever you set `dynamodb_table_name` to in `variables.tf`) and have a primary key named `LockID` of type `String`.
      * Example command (replace `your-lock-table` and `us-east-1`):
        ```bash
        aws dynamodb create-table \
            --table-name your-lock-table \
            --attribute-definitions \
                AttributeName=LockID,AttributeType=S \
            --key-schema \
                AttributeName=LockID,KeyType=HASH \
            --provisioned-throughput \
                ReadCapacityUnits=5,WriteCapacityUnits=5 \
            --region us-east-1
        ```

### **Instructions:**

1.  **Create Project Directory:**

    ```bash
    mkdir terraform-modules-vpc-s3
    cd terraform-modules-vpc-s3
    ```

2.  **Create Module Directories:**

    ```bash
    mkdir -p modules/vpc
    mkdir -p modules/s3
    ```

3.  **Create Terraform Configuration Files:**
    Create each file listed below in its respective directory with the provided content.

      * **`backend.tf` (in `terraform-modules-vpc-s3/`)**
      * **`variables.tf` (in `terraform-modules-vpc-s3/`)**
      * **`main.tf` (in `terraform-modules-vpc-s3/`)**
      * **`outputs.tf` (in `terraform-modules-vpc-s3/`)**
      * **`modules/vpc/main.tf`**
      * **`modules/vpc/variables.tf`**
      * **`modules/vpc/outputs.tf`**
      * **`modules/s3/main.tf`**
      * **`modules/s3/variables.tf`**
      * **`modules/s3/outputs.tf`**
      * **`README.md` (in `terraform-modules-vpc-s3/`)**

4.  **Customize Placeholders:**

      * In `variables.tf` (root), update `default` values for `aws_region`, `state_s3_bucket_name`, and `dynamodb_table_name` to match your pre-created resources and desired AWS region.
      * Review `modules/s3/variables.tf` for `bucket_name` and adjust if you want a specific bucket name for your new S3 bucket (not the state bucket).

5.  **Initialize Terraform:**

    ```bash
    terraform init
    ```

      * **Important:** This step will configure the S3 backend. If you haven't created the S3 state bucket and DynamoDB lock table beforehand, `terraform init` will fail.

6.  **Plan and Apply Terraform Configuration:**

    ```bash
    terraform plan
    terraform apply
    ```

      * Review the plan carefully and type `yes` when prompted to confirm the creation of resources.

7.  **Verify Resources:**

      * After `terraform apply` completes, check your AWS Management Console for the created VPC, subnets, S3 bucket, and the Terraform state file in your designated S3 state bucket.

8.  **Clean Up (Optional):**
    To destroy the created resources (excluding the S3 state bucket and DynamoDB lock table which were manually created):

    ```bash
    terraform destroy
    ```

      * Confirm by typing `yes`.

-----

### **Terraform Configuration Files:**

#### **`backend.tf`** (in `terraform-modules-vpc-s3/`)

```terraform
# backend.tf
# Configures the S3 backend for storing Terraform state.
# NOTE: The S3 bucket and DynamoDB table for state locking MUST be created manually
# before running 'terraform init'. Their names are specified in variables.tf.

terraform {
  backend "s3" {
    # These values will be taken from variables.tf in the root module.
    # We reference them here, but the actual values are supplied at `terraform init` time
    # or via -backend-config flags.
    # For simplicity, we directly hardcode them here but instruct the user
    # to update variables.tf. In a real scenario, these could be passed via CI/CD.
    bucket         = "your-unique-terraform-state-bucket-replace-me" # Replace with your actual state bucket name
    key            = "ecommerce-project/terraform.tfstate" # Path to the state file within the bucket
    region         = "us-east-1" # Replace with your actual AWS region for the backend bucket
    encrypt        = true # Enable server-side encryption for the state file
    dynamodb_table = "your-lock-table-replace-me" # Replace with your actual DynamoDB table name for locking
  }
}

# Provider configuration is intentionally not in backend.tf.
# The backend block takes care of the region for the state bucket.
# The AWS provider for resource creation is in main.tf.
```

#### **`variables.tf`** (in `terraform-modules-vpc-s3/`)

```terraform
# variables.tf (Root Module)

variable "aws_region" {
  description = "The AWS region to deploy resources into."
  type        = string
  default     = "us-east-1" # Set your desired default region
}

variable "state_s3_bucket_name" {
  description = "The name of the S3 bucket used for Terraform state backend. MUST exist beforehand."
  type        = string
  default     = "your-unique-terraform-state-bucket-replace-me" # **CRITICAL: Replace with your actual pre-created S3 bucket name**
}

variable "dynamodb_table_name" {
  description = "The name of the DynamoDB table used for Terraform state locking. MUST exist beforehand."
  type        = string
  default     = "your-lock-table-replace-me" # **CRITICAL: Replace with your actual pre-created DynamoDB table name**
}

# --- VPC Module Variables ---
variable "vpc_cidr_block" {
  description = "CIDR block for the VPC."
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr_blocks" {
  description = "List of CIDR blocks for public subnets."
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidr_blocks" {
  description = "List of CIDR blocks for private subnets."
  type        = list(string)
  default     = ["10.0.101.0/24", "10.0.102.0/24"]
}

# --- S3 Bucket Module Variables ---
variable "s3_bucket_name_prefix" {
  description = "Prefix for the S3 bucket name. A random suffix will be added."
  type        = string
  default     = "my-ecommerce-app-" # Your bucket will be like my-ecommerce-app-randomstring
}

variable "s3_bucket_acl" {
  description = "ACL for the S3 bucket."
  type        = string
  default     = "private"
  validation {
    condition     = contains(["private", "public-read", "public-read-write", "aws-exec-read", "authenticated-read", "bucket-owner-read", "bucket-owner-full-control", "log-delivery-write"], var.s3_bucket_acl)
    error_message = "Invalid S3 bucket ACL provided."
  }
}
```

#### **`main.tf`** (in `terraform-modules-vpc-s3/`)

```terraform
# main.tf (Root Module)

# Configure the AWS Provider
provider "aws" {
  region = var.aws_region
}

# --- Module: VPC ---
# Creates a VPC with public and private subnets, internet gateway, and route tables.
module "vpc" {
  source = "./modules/vpc" # Path to the VPC module

  # Pass variables to the VPC module
  vpc_cidr_block         = var.vpc_cidr_block
  public_subnet_cidr_blocks  = var.public_subnet_cidr_blocks
  private_subnet_cidr_blocks = var.private_subnet_cidr_blocks
}

# --- Module: S3 Bucket ---
# Creates an S3 bucket for the application (e.g., for frontend static files or uploads).
# This is separate from the S3 backend bucket.
module "application_s3_bucket" {
  source = "./modules/s3" # Path to the S3 module

  # Pass variables to the S3 bucket module
  bucket_name_prefix = var.s3_bucket_name_prefix
  bucket_acl         = var.s3_bucket_acl
  # Pass the region explicitly as the S3 bucket module might need it,
  # or if it implies a global bucket, ensure consistency.
  aws_region = var.aws_region
}
```

#### **`outputs.tf`** (in `terraform-modules-vpc-s3/`)

```terraform
# outputs.tf (Root Module)

output "vpc_id" {
  description = "The ID of the created VPC."
  value       = module.vpc.vpc_id
}

output "public_subnet_ids" {
  description = "List of IDs of the created public subnets."
  value       = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  description = "List of IDs of the created private subnets."
  value       = module.vpc.private_subnet_ids
}

output "application_s3_bucket_id" {
  description = "The ID (name) of the created application S3 bucket."
  value       = module.application_s3_bucket.bucket_id
}

output "application_s3_bucket_arn" {
  description = "The ARN of the created application S3 bucket."
  value       = module.application_s3_bucket.bucket_arn
}
```

#### **`README.md`** (in `terraform-modules-vpc-s3/`)

```markdown
# Terraform Mini Project: VPC and S3 Bucket with Backend Storage

This project demonstrates modular infrastructure provisioning using Terraform to create an AWS VPC and an AWS S3 bucket. It also configures Terraform to use an S3 bucket with DynamoDB for state locking as the backend storage for its state file.

## Project Structure

```

terraform-modules-vpc-s3/
├── backend.tf            \# Configures Terraform to use S3 for state storage
├── main.tf               \# Main configuration file that orchestrates modules
├── outputs.tf            \# Defines output variables from the root module
├── variables.tf          \# Defines input variables for the root module
├── modules/
│   ├── s3/               \# S3 Bucket module
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   └── vpc/              \# VPC module
│       ├── main.tf
│       ├── outputs.tf
│       └── variables.tf
└── README.md             \# This file

````

## Prerequisites

Before you begin, ensure you have the following:

1.  **AWS Account:** An active AWS account.
2.  **AWS CLI Configured:**
    * Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
    * Configure it with credentials that have sufficient permissions to create VPCs, S3 buckets, and DynamoDB tables in your chosen region.
        ```bash
        aws configure
        ```
3.  **Terraform Installed:** [Download and install Terraform](https://developer.hashicorp.com/terraform/downloads).
4.  **Pre-existing S3 Bucket for Terraform State:**
    * You **MUST** manually create a globally unique S3 bucket to store your Terraform state file *before* running `terraform init`.
    * Example command (replace `your-unique-terraform-state-bucket` and `us-east-1` with your desired bucket name and region):
        ```bash
        aws s3 mb s3://your-unique-terraform-state-bucket --region us-east-1
        ```
5.  **Pre-existing DynamoDB Table for Terraform State Locking:**
    * You **MUST** manually create a DynamoDB table to handle state locking *before* running `terraform init`. This table prevents concurrent modifications to the state file.
    * The table **must** be named `your-lock-table` (or whatever you set `dynamodb_table_name` to in `variables.tf`) and have a primary key named `LockID` of type `String`.
    * Example command (replace `your-lock-table` and `us-east-1`):
        ```bash
        aws dynamodb create-table \
            --table-name your-lock-table \
            --attribute-definitions \
                AttributeName=LockID,AttributeType=S \
            --key-schema \
                AttributeName=LockID,KeyType=HASH \
            --provisioned-throughput \
                ReadCapacityUnits=5,WriteCapacityUnits=5 \
            --region us-east-1
        ```

## Setup and Deployment Instructions

1.  **Clone this repository** (or manually create the directory structure and files as outlined above).
    ```bash
    git clone <your-repo-url> terraform-modules-vpc-s3
    cd terraform-modules-vpc-s3
    ```

2.  **Customize `variables.tf`:**
    * Open `variables.tf` in the root directory.
    * **Crucially, update the `default` values for `state_s3_bucket_name` and `dynamodb_table_name`** to match the exact names of the S3 bucket and DynamoDB table you created in the prerequisites.
    * Optionally, adjust `aws_region`, `vpc_cidr_block`, `public_subnet_cidr_blocks`, `private_subnet_cidr_blocks`, `s3_bucket_name_prefix`, and `s3_bucket_acl` to fit your specific requirements.

3.  **Initialize Terraform:**
    This command downloads necessary providers and initializes the S3 backend configuration.
    ```bash
    terraform init
    ```
    * If `terraform init` fails, double-check that your S3 state bucket and DynamoDB lock table exist and their names match the values in your `backend.tf` (and `variables.tf`).

4.  **Review the Execution Plan:**
    This command shows you what Terraform plans to create, modify, or destroy without actually making changes.
    ```bash
    terraform plan
    ```

5.  **Apply the Configuration:**
    This command applies the changes defined in your Terraform configuration files, creating the AWS resources.
    ```bash
    terraform apply
    ```
    * Type `yes` when prompted to confirm the execution.

## Verification

After `terraform apply` completes successfully:

* **AWS Management Console:** Log in to your AWS console and verify the creation of:
    * A new VPC with the specified CIDR block.
    * Public and private subnets within the VPC.
    * An Internet Gateway (IGW) attached to the VPC.
    * Route tables configured for public and private subnets.
    * A new S3 bucket (named `your-ecommerce-app-` followed by a random suffix).
* **S3 State Bucket:** Check your pre-created S3 state bucket (`your-unique-terraform-state-bucket`). You should see a `terraform.tfstate` file (or `ecommerce-project/terraform.tfstate` if you used that key path) stored there.
* **DynamoDB Table:** Verify the `your-lock-table` (or your chosen name) DynamoDB table shows active items when a Terraform operation is in progress, indicating state locking.

## Cleanup (Optional)

To destroy the AWS resources created by this Terraform configuration (this will NOT delete the S3 state bucket or DynamoDB lock table, as they were pre-created):

```bash
terraform destroy
````

  * Type `yes` when prompted to confirm the destruction.

## Challenges and Observations

  * **Backend Pre-creation:** A common hurdle is understanding that the S3 backend bucket and DynamoDB lock table must exist *before* `terraform init` can succeed. This is a bootstrapping step.
  * **Globally Unique S3 Bucket Names:** S3 bucket names must be globally unique across all AWS accounts.
  * **DynamoDB Primary Key:** The DynamoDB table for state locking *must* have a primary key named `LockID` of type `String`.
  * **Modularization Benefits:** Observe how the use of modules (`modules/vpc`, `modules/s3`) makes `main.tf` much cleaner and more readable, abstracting away the details of resource creation into reusable components.
  * **State Management:** The `terraform.tfstate` file is now stored remotely in S3, enabling collaboration and secure state management for teams.

This project provides a solid foundation for building more complex infrastructure using Terraform's modularity and robust state management features.

````

---

### **`modules/vpc/main.tf`** (in `terraform-modules-vpc-s3/modules/vpc/`)

```terraform
# modules/vpc/main.tf

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "ECommerce-VPC"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "ECommerce-IGW"
  }
}

# --- Public Subnets ---
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidr_blocks)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidr_blocks[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true # Automatically assign public IPs

  tags = {
    Name = "ECommerce-Public-Subnet-${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "ECommerce-Public-RT"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# --- Private Subnets (No direct internet access) ---
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidr_blocks)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr_blocks[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = false # No public IPs for private subnets

  tags = {
    Name = "ECommerce-Private-Subnet-${count.index + 1}"
  }
}

# No route table association to IGW for private subnets (they might use NAT Gateway later)
# For this project, they are purely private.

# Data source to get available AZs in the region
data "aws_availability_zones" "available" {
  state = "available"
}
````

### **`modules/vpc/variables.tf`** (in `terraform-modules-vpc-s3/modules/vpc/`)

```terraform
# modules/vpc/variables.tf

variable "vpc_cidr_block" {
  description = "CIDR block for the VPC."
  type        = string
}

variable "public_subnet_cidr_blocks" {
  description = "List of CIDR blocks for public subnets."
  type        = list(string)
}

variable "private_subnet_cidr_blocks" {
  description = "List of CIDR blocks for private subnets."
  type        = list(string)
}
```

### **`modules/vpc/outputs.tf`** (in `terraform-modules-vpc-s3/modules/vpc/`)

```terraform
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "The ID of the created VPC."
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of IDs of the created public subnets."
  value       = aws_subnet.public.*.id
}

output "private_subnet_ids" {
  description = "List of IDs of the created private subnets."
  value       = aws_subnet.private.*.id
}

output "internet_gateway_id" {
  description = "The ID of the created Internet Gateway."
  value       = aws_internet_gateway.main.id
}
```

### **`modules/s3/main.tf`** (in `terraform-modules-vpc-s3/modules/s3/`)

```terraform
# modules/s3/main.tf

resource "random_string" "bucket_suffix" {
  length  = 8
  special = false
  upper   = false
  numeric = true
}

resource "aws_s3_bucket" "main" {
  bucket = "${var.bucket_name_prefix}${random_string.bucket_suffix.result}"
  acl    = var.bucket_acl

  tags = {
    Name        = "${var.bucket_name_prefix}bucket"
    Environment = "Dev"
  }
}

# Optional: Add bucket versioning for data protection
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### **`modules/s3/variables.tf`** (in `terraform-modules-vpc-s3/modules/s3/`)

```terraform
# modules/s3/variables.tf

variable "bucket_name_prefix" {
  description = "Prefix for the S3 bucket name. A random suffix will be added."
  type        = string
}

variable "bucket_acl" {
  description = "ACL for the S3 bucket."
  type        = string
}

variable "aws_region" {
  description = "The AWS region where the S3 bucket will be created. Used for provider implicitly."
  type        = string
}
```

### **`modules/s3/outputs.tf`** (in `terraform-modules-vpc-s3/modules/s3/`)

```terraform
# modules/s3/outputs.tf

output "bucket_id" {
  description = "The ID (name) of the S3 bucket."
  value       = aws_s3_bucket.main.id
}

output "bucket_arn" {
  description = "The ARN of the S3 bucket."
  value       = aws_s3_bucket.main.arn
}

output "bucket_domain_name" {
  description = "The S3 bucket regional domain name."
  value       = aws_s3_bucket.main.bucket_regional_domain_name
}
```
