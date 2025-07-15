

## Mini Project: Hosting a Dynamic Web App on AWS with Terraform, Docker, ECR, and ECS

This project demonstrates how to containerize a simple dynamic web application and deploy it to Amazon Elastic Container Service (ECS) using Terraform modules, Amazon Elastic Container Registry (ECR), and Docker.

### **Project Structure:**

```
terraform-ecs-webapp/
├── webapp/                 # Contains the dynamic web application and its Dockerfile
│   ├── app.js
│   ├── package.json
│   └── Dockerfile
├── main.tf                 # Main Terraform configuration, orchestrates modules
├── variables.tf            # Root module variables
├── outputs.tf              # Root module outputs
├── README.md               # This file
└── modules/
    ├── ecr/                # Terraform module for ECR repository
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── ecs/                # Terraform module for ECS cluster and service (Fargate)
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

### **Prerequisites:**

1.  **AWS Account:** An active AWS account.
2.  **AWS CLI Configured:** Ensure you have the AWS CLI installed and configured with credentials that have sufficient permissions to create IAM roles, ECR repositories, VPCs, Subnets, Security Groups, EC2 Load Balancers, and ECS resources (Cluster, Task Definitions, Services).
    ```bash
    aws configure
    ```
3.  **Terraform Installed:** [Download and install Terraform](https://developer.hashicorp.com/terraform/downloads).
4.  **Docker Installed:** [Download and install Docker Desktop](https://www.docker.com/products/docker-desktop/).

### **Instructions:**

#### **Step 1: Project Setup and Web Application Creation**

1.  **Create Project Directory:**

    ```bash
    mkdir terraform-ecs-webapp
    cd terraform-ecs-webapp
    ```

2.  **Create Module Directories:**

    ```bash
    mkdir -p modules/ecr
    mkdir -p modules/ecs
    ```

3.  **Create Web Application Files:**
    Create a `webapp` directory and add the following files:

      * **`webapp/app.js`**

        ```javascript
        // webapp/app.js
        const express = require('express');
        const app = express();
        const port = process.env.PORT || 80; // Standard HTTP port

        app.get('/', (req, res) => {
          res.send('<h1>Hello from Dynamic Web App on ECS!</h1><p>Running on Node.js</p><p>Current Time: ' + new Date().toLocaleString() + '</p>');
        });

        app.listen(port, () => {
          console.log(`Web app listening on port ${port}`);
        });
        ```

      * **`webapp/package.json`**

        ```json
        {
          "name": "dynamic-webapp",
          "version": "1.0.0",
          "description": "A simple Node.js web app for ECS deployment demo",
          "main": "app.js",
          "scripts": {
            "start": "node app.js"
          },
          "dependencies": {
            "express": "^4.19.2"
          }
        }
        ```

      * **`webapp/Dockerfile`**

        ```dockerfile
        # webapp/Dockerfile
        FROM node:20-alpine
        WORKDIR /app
        COPY package*.json ./
        RUN npm ci --only=production
        COPY . .
        EXPOSE 80
        CMD ["npm", "start"]
        ```

4.  **Test Docker Image Locally (Optional but Recommended):**
    Navigate into the `webapp` directory and build/run the Docker image:

    ```bash
    cd webapp
    docker build -t dynamic-webapp-local:latest .
    docker run -p 8080:80 dynamic-webapp-local:latest
    ```

    Open your browser to `http://localhost:8080`. You should see "Hello from Dynamic Web App on ECS\!". Press `Ctrl+C` in the terminal to stop the container.
    Go back to the root directory: `cd ..`

#### **Step 2: Create Terraform Configuration Files**

Create each `.tf` file listed below in its respective directory with the provided content.

  * **`main.tf`** (in `terraform-ecs-webapp/`)
  * **`variables.tf`** (in `terraform-ecs-webapp/`)
  * **`outputs.tf`** (in `terraform-ecs-webapp/`)
  * **`modules/ecr/main.tf`**
  * **`modules/ecr/variables.tf`**
  * **`modules/ecr/outputs.tf`**
  * **`modules/ecs/main.tf`**
  * **`modules/ecs/variables.tf`**
  * **`modules/ecs/outputs.tf`**
  * **`README.md`** (in `terraform-ecs-webapp/`) - This very document.

#### **Step 3: Customize Variables**

Open `variables.tf` in the root directory and set your desired `aws_region`.

#### **Step 4: Build and Push Docker Image to ECR**

Before running Terraform, you need to build your web app's Docker image and push it to ECR. Terraform will *create* the ECR repository, but pushing the image is a manual step or part of a CI/CD pipeline (which is outside the scope of this particular project's Terraform execution).

1.  **Initialize Terraform (to create ECR repo first):**

    ```bash
    terraform init
    terraform apply -target=module.ecr # Apply only the ECR module first
    ```

      * Type `yes` to confirm. This will create your ECR repository.

2.  **Get ECR Repository URL:**
    After the above step, you can get the ECR repository URL from Terraform outputs or AWS console.

    ```bash
    terraform output ecr_repository_url
    ```

    Copy this URL. It will look like `123456789012.dkr.ecr.us-east-1.amazonaws.com/dynamic-webapp-repo`.

3.  **Log Docker into ECR:**
    Replace `your-aws-account-id` and `your-aws-region` with your actual values.

    ```bash
    aws ecr get-login-password --region your-aws-region | docker login --username AWS --password-stdin your-aws-account-id.dkr.ecr.your-aws-region.amazonaws.com
    ```

4.  **Build and Tag Docker Image:**
    Navigate back to the root `terraform-ecs-webapp` directory.

    ```bash
    docker build -t dynamic-webapp . # Build from the root context, specify webapp directory
    docker tag dynamic-webapp:latest <your-ecr-repository-url>:latest # Tag with your ECR URL
    ```

      * Example: `docker tag dynamic-webapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/dynamic-webapp-repo:latest`

5.  **Push Docker Image to ECR:**

    ```bash
    docker push <your-ecr-repository-url>:latest
    ```

      * Example: `docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/dynamic-webapp-repo:latest`

#### **Step 5: Deploy ECS Cluster and Web App**

1.  **Initialize Terraform (if not already done for ECR, or re-run to ensure all modules are initialized):**

    ```bash
    terraform init
    ```

2.  **Plan and Apply Terraform Configuration:**
    This command will create the ECS cluster, services, load balancer, and associated networking.

    ```bash
    terraform plan
    terraform apply
    ```

      * Review the plan carefully and type `yes` when prompted to confirm the creation of resources.

#### **Step 6: Access the Web Application**

1.  **Get Load Balancer DNS Name:**
    After `terraform apply` completes, the ALB DNS name will be shown in the Terraform outputs.

    ```bash
    terraform output webapp_alb_dns_name
    ```

2.  **Open in Browser:**
    Paste the `webapp_alb_dns_name` into your web browser. You should see your "Hello from Dynamic Web App on ECS\!" message. It might take a minute or two for the ECS service and ALB to become healthy.

#### **Step 7: Clean Up (Optional)**

To destroy all AWS resources created by this Terraform configuration (including ECR repository, ECS cluster, services, ALB, VPC, etc.):

```bash
terraform destroy
```

  * Type `yes` when prompted to confirm the destruction.

-----

### **Terraform Configuration Files:**

#### **`main.tf`** (in `terraform-ecs-webapp/`)

```terraform
# main.tf (Root Module)

# Configure the AWS Provider
provider "aws" {
  region = var.aws_region
}

# --- Module: ECR Repository ---
# Creates an Amazon Elastic Container Registry (ECR) repository for Docker images.
module "ecr_repo" {
  source = "./modules/ecr" # Path to the ECR module

  repository_name = "dynamic-webapp-repo" # Name for your ECR repository
}

# --- Module: ECS Cluster and Fargate Service ---
# Provisions an ECS cluster, necessary networking (VPC, subnets, SG),
# IAM roles, and deploys the web app as an ECS Fargate service exposed via ALB.
module "ecs_app" {
  source = "./modules/ecs" # Path to the ECS module

  # Pass necessary variables to the ECS module
  aws_region          = var.aws_region
  app_name            = "dynamic-webapp"
  ecr_repository_url  = module.ecr_repo.repository_url # Get ECR URL from the ECR module output
  container_port      = 80 # Port your web app listens on inside the container
  container_memory    = 512 # MB
  container_cpu       = 256 # vCPU units (0.25 vCPU)
  desired_count       = 1 # Number of running tasks
}
```

#### **`variables.tf`** (in `terraform-ecs-webapp/`)

```terraform
# variables.tf (Root Module)

variable "aws_region" {
  description = "The AWS region to deploy resources into."
  type        = string
  default     = "us-east-1" # Set your desired default region
}
```

#### **`outputs.tf`** (in `terraform-ecs-webapp/`)

```terraform
# outputs.tf (Root Module)

output "ecr_repository_url" {
  description = "The URL of the created ECR repository."
  value       = module.ecr_repo.repository_url
}

output "ecs_cluster_name" {
  description = "The name of the created ECS cluster."
  value       = module.ecs_app.ecs_cluster_name
}

output "ecs_service_name" {
  description = "The name of the deployed ECS service."
  value       = module.ecs_app.ecs_service_name
}

output "webapp_alb_dns_name" {
  description = "The DNS name of the Application Load Balancer for the web app."
  value       = module.ecs_app.alb_dns_name
}
```

#### **`modules/ecr/main.tf`** (in `terraform-ecs-webapp/modules/ecr/`)

```terraform
# modules/ecr/main.tf

resource "aws_ecr_repository" "main" {
  name                 = var.repository_name
  image_tag_mutability = "MUTABLE" # Or IMMUTABLE based on your tagging strategy
  image_scanning_configuration {
    scan_on_push = true
  }

  tags = {
    Name = var.repository_name
  }
}
```

#### **`modules/ecr/variables.tf`** (in `terraform-ecs-webapp/modules/ecr/`)

```terraform
# modules/ecr/variables.tf

variable "repository_name" {
  description = "The name for the ECR repository."
  type        = string
}
```

#### **`modules/ecr/outputs.tf`** (in `terraform-ecs-webapp/modules/ecr/`)

```terraform
# modules/ecr/outputs.tf

output "repository_url" {
  description = "The URL of the ECR repository."
  value       = aws_ecr_repository.main.repository_url
}

output "repository_arn" {
  description = "The ARN of the ECR repository."
  value       = aws_ecr_repository.main.arn
}
```

#### **`modules/ecs/main.tf`** (in `terraform-ecs-webapp/modules/ecs/`)

```terraform
# modules/ecs/main.tf

# --- Networking for Fargate ---
# Using a new VPC for isolated deployment for this project.
# In a real scenario, you might import an existing VPC.
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.app_name}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.app_name}-igw"
  }
}

# --- Public Subnets (for ALB and Fargate ENIs) ---
# Fetch available AZs dynamically
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count             = 2 # Deploy into 2 AZs for high availability
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 1) # e.g., 10.0.1.0/24, 10.0.2.0/24
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true # Fargate ENIs need public IPs for outbound if no NAT Gateway

  tags = {
    Name = "${var.app_name}-public-subnet-${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.app_name}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# --- Security Group for Web App (allowing HTTP traffic from ALB) ---
resource "aws_security_group" "webapp" {
  vpc_id = aws_vpc.main.id
  name   = "${var.app_name}-sg"
  description = "Allow HTTP traffic to web app"

  ingress {
    from_port   = var.container_port
    to_port     = var.container_port
    protocol    = "tcp"
    # Allow traffic from the ALB's security group (itself).
    # This will be updated by the ALB later if it's in the same VPC.
    # For now, allowing all private IPs for simplicity.
    cidr_blocks = ["10.0.0.0/8"] # Allow from within VPC
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] # Allow all outbound traffic
  }

  tags = {
    Name = "${var.app_name}-sg"
  }
}


# --- ECS Cluster ---
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-cluster"

  tags = {
    Name = "${var.app_name}-cluster"
  }
}

# --- IAM Roles for ECS ---
# ECS Task Execution Role (Allows ECS agent to pull images and publish logs)
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "${var.app_name}-ecs-task-execution-role"

  assume_role_policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [{
      Action    = "sts:AssumeRole",
      Effect    = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:${var.aws_region}:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Role (Allows application containers to interact with other AWS services)
resource "aws_iam_role" "ecs_task_role" {
  name = "${var.app_name}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [{
      Action    = "sts:AssumeRole",
      Effect    = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# You can attach additional policies to aws_iam_role.ecs_task_role
# if your application needs to access other AWS services (e.g., S3, DynamoDB)

# --- ECS Task Definition (Fargate) ---
resource "aws_ecs_task_definition" "webapp" {
  family                   = "${var.app_name}-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.container_cpu
  memory                   = var.container_memory
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name        = var.app_name
      image       = "${var.ecr_repository_url}:latest" # Ensure 'latest' tag is pushed
      cpu         = var.container_cpu
      memory      = var.container_memory
      essential   = true
      portMappings = [
        {
          containerPort = var.container_port
          hostPort      = var.container_port
          protocol      = "tcp"
        }
      ],
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/${var.app_name}-task"
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}

# --- CloudWatch Log Group for ECS Task Logs ---
resource "aws_cloudwatch_log_group" "ecs_task_logs" {
  name              = "/ecs/${var.app_name}-task"
  retention_in_days = 7 # Adjust retention as needed

  tags = {
    Name = "${var.app_name}-task-logs"
  }
}

# --- Application Load Balancer (ALB) ---
resource "aws_lb" "main" {
  name               = "${var.app_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public.*.id

  tags = {
    Name = "${var.app_name}-alb"
  }
}

resource "aws_security_group" "alb" {
  vpc_id = aws_vpc.main.id
  name   = "${var.app_name}-alb-sg"
  description = "Allow HTTP/HTTPS access to ALB"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Allow HTTP from anywhere
  }

  # For HTTPS if needed later:
  # ingress {
  #   from_port   = 443
  #   to_port     = 443
  #   protocol    = "tcp"
  #   cidr_blocks = ["0.0.0.0/0"]
  # }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.app_name}-alb-sg"
  }
}

resource "aws_lb_target_group" "webapp" {
  name        = "${var.app_name}-tg"
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip" # Required for Fargate

  health_check {
    path = "/" # Your app's health check endpoint
    protocol = "HTTP"
    matcher = "200"
  }

  tags = {
    Name = "${var.app_name}-tg"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.webapp.arn
  }
}

# --- ECS Service (Fargate) ---
resource "aws_ecs_service" "webapp" {
  name            = "${var.app_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.webapp.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.public.*.id
    security_groups = [aws_security_group.webapp.id]
    assign_public_ip = true # Fargate tasks need public IP if no NAT gateway for outbound
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.webapp.arn
    container_name   = var.app_name
    container_port   = var.container_port
  }

  lifecycle {
    ignore_changes = [desired_count] # Prevents Terraform from changing desired_count on manual scaling
  }

  depends_on = [
    aws_lb_listener.http # Ensure listener is ready before service tries to register
  ]

  tags = {
    Name = "${var.app_name}-service"
  }
}
```

#### **`modules/ecs/variables.tf`** (in `terraform-ecs-webapp/modules/ecs/`)

```terraform
# modules/ecs/variables.tf

variable "aws_region" {
  description = "The AWS region where ECS resources will be deployed."
  type        = string
}

variable "app_name" {
  description = "The base name for the application and associated resources."
  type        = string
}

variable "ecr_repository_url" {
  description = "The URL of the ECR repository containing the Docker image."
  type        = string
}

variable "container_port" {
  description = "The port on which the containerized application listens."
  type        = number
  default     = 80
}

variable "container_memory" {
  description = "The amount of memory (in MiB) to allocate to the container."
  type        = number
  default     = 512 # 0.5 GB
}

variable "container_cpu" {
  description = "The number of CPU units to allocate to the container (1024 units = 1 vCPU)."
  type        = number
  default     = 256 # 0.25 vCPU
}

variable "desired_count" {
  description = "The number of desired tasks for the ECS service."
  type        = number
  default     = 1
}
```

#### **`modules/ecs/outputs.tf`** (in `terraform-ecs-webapp/modules/ecs/`)

```terraform
# modules/ecs/outputs.tf

output "ecs_cluster_name" {
  description = "The name of the ECS cluster created."
  value       = aws_ecs_cluster.main.name
}

output "ecs_service_name" {
  description = "The name of the ECS service created."
  value       = aws_ecs_service.webapp.name
}

output "alb_dns_name" {
  description = "The DNS name of the Application Load Balancer."
  value       = aws_lb.main.dns_name
}

output "webapp_security_group_id" {
  description = "The ID of the security group attached to the web app ECS tasks."
  value       = aws_security_group.webapp.id
}

output "public_subnet_ids" {
  description = "IDs of the public subnets created for ECS Fargate."
  value       = aws_subnet.public.*.id
}
```
