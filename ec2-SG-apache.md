
-----

## Mini Project: EC2 Module and Security Group Module with Apache2 UserData

This project demonstrates how to use Terraform to create modularized configurations for deploying an EC2 instance, attaching a specific Security Group, and installing Apache2 using a UserData script upon instance launch.

### **Project Structure:**

```
terraform-ec2-apache/
├── apache_userdata.sh       # Script to be executed on EC2 instance launch
├── main.tf                  # Main Terraform configuration, orchestrates modules
├── variables.tf             # Root module variables
├── outputs.tf               # Root module outputs
├── README.md                # This file
└── modules/
    ├── ec2/                 # Terraform module for EC2 instance
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── security_group/      # Terraform module for Security Group
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

### **Prerequisites:**

1.  **AWS Account:** An active AWS account.
2.  **AWS CLI Configured:** Ensure you have the AWS CLI installed and configured with credentials that have sufficient permissions to create EC2 instances, Security Groups, and manage key pairs.
    ```bash
    aws configure
    ```
3.  **Terraform Installed:** [Download and install Terraform](https://developer.hashicorp.com/terraform/downloads).
4.  **AWS Key Pair:** You should have an existing EC2 Key Pair in your chosen AWS region for SSH access to the instance. If not, create one via the EC2 console or AWS CLI:
    ```bash
    aws ec2 create-key-pair --key-name my-ec2-key --query 'KeyMaterial' --output text > my-ec2-key.pem
    chmod 400 my-ec2-key.pem
    ```
    (Replace `my-ec2-key` with your desired key pair name).

### **Instructions:**

#### **Step 1: Project Setup and UserData Script Creation**

1.  **Create Project Directory:**

    ```bash
    mkdir terraform-ec2-apache
    cd terraform-ec2-apache
    ```

2.  **Create Module Directories:**

    ```bash
    mkdir -p modules/ec2
    mkdir -p modules/security_group
    ```

3.  **Create UserData Script:**
    Create the file `apache_userdata.sh` in the root of your project (`terraform-ec2-apache/`):

      * **`apache_userdata.sh`**

        ```bash
        #!/bin/bash
        sudo yum update -y
        sudo yum install -y httpd
        sudo systemctl start httpd
        sudo systemctl enable httpd
        echo "<h1>Hello World from $(hostname -f)</h1>" | sudo tee /var/www/html/index.html
        ```

      * **Make UserData script executable (optional for Terraform, good for local testing):**

        ```bash
        chmod +x apache_userdata.sh
        ```

#### **Step 2: Create Terraform Configuration Files**

Create each `.tf` file listed below in its respective directory with the provided content.

  * **`main.tf`** (in `terraform-ec2-apache/`)
  * **`variables.tf`** (in `terraform-ec2-apache/`)
  * **`outputs.tf`** (in `terraform-ec2-apache/`)
  * **`modules/ec2/main.tf`**
  * **`modules/ec2/variables.tf`**
  * **`modules/ec2/outputs.tf`**
  * **`modules/security_group/main.tf`**
  * **`modules/security_group/variables.tf`**
  * **`modules/security_group/outputs.tf`**
  * **`README.md`** (in `terraform-ec2-apache/`) - This very document.

#### **Step 3: Customize Variables**

Open `variables.tf` in the root directory and update the `default` values for:

  * `aws_region`
  * `instance_key_pair_name` (must match an existing AWS EC2 Key Pair in your account)
  * `ami_id` (a suitable Amazon Linux 2 AMI ID for your chosen region).

#### **Step 4: Deploy Infrastructure**

1.  **Initialize Terraform:**

    ```bash
    terraform init
    ```

2.  **Plan and Apply Terraform Configuration:**

    ```bash
    terraform plan
    terraform apply
    ```

      * Review the plan carefully and type `yes` when prompted to confirm the creation of resources.

#### **Step 5: Access the Web Server**

1.  **Get EC2 Public IP:**
    After `terraform apply` completes, the public IP address of the EC2 instance will be shown in the Terraform outputs.

    ```bash
    terraform output ec2_public_ip
    ```

2.  **Open in Browser:**
    Paste the `ec2_public_ip` into your web browser. You should see the "Hello World" message from Apache2.

3.  **SSH into the instance (Optional):**
    You can SSH into the instance to verify Apache2 status:

    ```bash
    ssh -i /path/to/your/my-ec2-key.pem ec2-user@<EC2_PUBLIC_IP>
    ```

    (Replace `/path/to/your/my-ec2-key.pem` with your key's path and `<EC2_PUBLIC_IP>` with the actual IP).
    Once logged in, you can check Apache's status: `sudo systemctl status httpd`

#### **Step 6: Clean Up (Optional)**

To destroy all AWS resources created by this Terraform configuration:

```bash
terraform destroy
```

  * Type `yes` when prompted to confirm the destruction.

-----

### **Terraform Configuration Files:**

#### **`main.tf`** (in `terraform-ec2-apache/`)

```terraform
# main.tf (Root Module)

# Configure the AWS Provider
provider "aws" {
  region = var.aws_region
}

# Data source to retrieve default VPC and public subnets
# This is for simplicity in a mini-project.
# In a real-world scenario, you might define your VPC explicitly or use a VPC module.
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }

  filter {
    name   = "map-public-ip-on-launch"
    values = ["true"]
  }
}


# --- Module: Security Group ---
# Creates a Security Group with rules for SSH and HTTP access.
module "web_security_group" {
  source = "./modules/security_group" # Path to the Security Group module

  sg_name = "${var.project_name}-web-sg"
  vpc_id  = data.aws_vpc.default.id # Use the default VPC ID
}

# --- Module: EC2 Instance ---
# Creates an EC2 instance and installs Apache2 using UserData.
module "web_server_ec2" {
  source          = "./modules/ec2" # Path to the EC2 module

  ami_id            = var.ami_id
  instance_type     = var.instance_type
  key_name          = var.instance_key_pair_name
  user_data         = file("${path.module}/apache_userdata.sh") # Reads the content of the script
  security_group_ids = [module.web_security_group.security_group_id] # Pass SG ID from module output
  subnet_id         = tolist(data.aws_subnets.public.ids)[0] # Pick the first public subnet
}
```

#### **`variables.tf`** (in `terraform-ec2-apache/`)

```terraform
# variables.tf (Root Module)

variable "aws_region" {
  description = "The AWS region to deploy resources into."
  type        = string
  default     = "us-east-1" # Set your desired default region
}

variable "project_name" {
  description = "A prefix for resource names to help with identification."
  type        = string
  default     = "apache-web-server"
}

variable "ami_id" {
  description = "The AMI ID for the EC2 instance (e.g., Amazon Linux 2 in us-east-1)."
  type        = string
  # Find latest Amazon Linux 2 AMI for your region:
  # aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text
  default     = "ami-053b01d0a1b02562d" # Example for us-east-1 (Amazon Linux 2 LTS) - VERIFY THIS FOR YOUR REGION
}

variable "instance_type" {
  description = "The EC2 instance type."
  type        = string
  default     = "t2.micro"
}

variable "instance_key_pair_name" {
  description = "The name of the EC2 Key Pair to use for SSH access."
  type        = string
  default     = "your-key-pair-name" # **CRITICAL: Replace with your actual AWS Key Pair Name**
}
```

#### **`outputs.tf`** (in `terraform-ec2-apache/`)

```terraform
# outputs.tf (Root Module)

output "ec2_public_ip" {
  description = "The public IP address of the EC2 instance."
  value       = module.web_server_ec2.public_ip
}

output "ec2_instance_id" {
  description = "The ID of the created EC2 instance."
  value       = module.web_server_ec2.instance_id
}

output "security_group_id" {
  description = "The ID of the security group created for the web server."
  value       = module.web_security_group.security_group_id
}
```

#### **`modules/ec2/main.tf`** (in `terraform-ec2-apache/modules/ec2/`)

```terraform
# modules/ec2/main.tf

resource "aws_instance" "web_server" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  user_data     = var.user_data # This will be the content of apache_userdata.sh
  # Associate with the security group and place in a public subnet
  vpc_security_group_ids = var.security_group_ids
  subnet_id              = var.subnet_id
  associate_public_ip_address = true # Required to get a public IP

  tags = {
    Name = "${var.instance_name}-web-server"
  }
}
```

#### **`modules/ec2/variables.tf`** (in `terraform-ec2-apache/modules/ec2/`)

```terraform
# modules/ec2/variables.tf

variable "ami_id" {
  description = "The AMI ID for the EC2 instance."
  type        = string
}

variable "instance_type" {
  description = "The EC2 instance type."
  type        = string
}

variable "key_name" {
  description = "The name of the EC2 Key Pair to use."
  type        = string
}

variable "user_data" {
  description = "The UserData script to execute on instance launch."
  type        = string
}

variable "security_group_ids" {
  description = "List of Security Group IDs to associate with the instance."
  type        = list(string)
}

variable "subnet_id" {
  description = "The ID of the subnet to launch the instance into."
  type        = string
}

variable "instance_name" {
  description = "Name tag for the EC2 instance."
  type        = string
  default     = "WebApp"
}
```

#### **`modules/ec2/outputs.tf`** (in `terraform-ec2-apache/modules/ec2/`)

```terraform
# modules/ec2/outputs.tf

output "instance_id" {
  description = "The ID of the created EC2 instance."
  value       = aws_instance.web_server.id
}

output "public_ip" {
  description = "The public IP address of the EC2 instance."
  value       = aws_instance.web_server.public_ip
}

output "private_ip" {
  description = "The private IP address of the EC2 instance."
  value       = aws_instance.web_server.private_ip
}
```

#### **`modules/security_group/main.tf`** (in `terraform-ec2-apache/modules/security_group/`)

```terraform
# modules/security_group/main.tf

resource "aws_security_group" "web_access" {
  name        = var.sg_name
  description = "Security group for web access to Apache server"
  vpc_id      = var.vpc_id

  # Ingress rule for HTTP (port 80)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Allow HTTP from anywhere
    description = "Allow HTTP access"
  }

  # Ingress rule for SSH (port 22) - for administration
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # WARNING: 0.0.0.0/0 is open to the world. Restrict this in production!
    description = "Allow SSH access"
  }

  # Egress rule (all outbound traffic allowed)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = var.sg_name
  }
}
```

#### **`modules/security_group/variables.tf`** (in `terraform-ec2-apache/modules/security_group/`)

```terraform
# modules/security_group/variables.tf

variable "sg_name" {
  description = "The name of the security group."
  type        = string
}

variable "vpc_id" {
  description = "The ID of the VPC to create the security group in."
  type        = string
}
```

#### **`modules/security_group/outputs.tf`** (in `terraform-ec2-apache/modules/security_group/`)

```terraform
# modules/security_group/outputs.tf

output "security_group_id" {
  description = "The ID of the created security group."
  value       = aws_security_group.web_access.id
}
```
