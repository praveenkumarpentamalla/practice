Got it 👍 since your Laravel + Socket.IO setup also needs **Redis**, let’s add that into the Terraform infra.

We’ll use **Amazon ElastiCache (Redis)** so it’s fully managed (instead of running Redis inside ECS).

---

# 📂 Updated Project Structure

```
terraform/
├── main.tf
├── provider.tf
├── variables.tf
├── vpc.tf
├── security.tf
├── ecr.tf
├── ecs.tf
├── redis.tf   👈 NEW
├── outputs.tf
```

---

## 1️⃣ redis.tf (ElastiCache Redis)

```hcl
resource "aws_elasticache_subnet_group" "redis" {
  name       = "${var.app_name}-redis-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "${var.app_name}-redis"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  port                 = 6379

  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [aws_security_group.redis_sg.id]
}

resource "aws_security_group" "redis_sg" {
  name   = "${var.app_name}-redis-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_sg.id] # only ECS tasks can connect
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 2️⃣ variables.tf (add Redis config variables)

```hcl
variable "redis_node_type" {
  description = "Instance type for Redis"
  type        = string
  default     = "cache.t3.micro"
}

variable "redis_port" {
  description = "Redis port"
  type        = number
  default     = 6379
}
```

---

## 3️⃣ ecs.tf (inject Redis into Laravel + Socket containers)

Update your **Task Definitions** so Laravel & Socket.IO containers get the **Redis host + port** via environment variables.

```hcl
resource "aws_ecs_task_definition" "laravel" {
  family                   = "${var.app_name}-laravel"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([
    {
      name      = "laravel"
      image     = "${aws_ecr_repository.laravel.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 8000
          hostPort      = 8000
        }
      ]
      environment = [
        {
          name  = "REDIS_HOST"
          value = aws_elasticache_cluster.redis.cache_nodes[0].address
        },
        {
          name  = "REDIS_PORT"
          value = tostring(var.redis_port)
        }
      ]
    }
  ])
}

resource "aws_ecs_task_definition" "socket" {
  family                   = "${var.app_name}-socket"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([
    {
      name      = "socket"
      image     = "${aws_ecr_repository.socket.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 3001
          hostPort      = 3001
        }
      ]
      environment = [
        {
          name  = "REDIS_HOST"
          value = aws_elasticache_cluster.redis.cache_nodes[0].address
        },
        {
          name  = "REDIS_PORT"
          value = tostring(var.redis_port)
        }
      ]
    }
  ])
}
```

---

## 4️⃣ outputs.tf (show Redis endpoint)

```hcl
output "redis_endpoint" {
  value = aws_elasticache_cluster.redis.cache_nodes[0].address
}
```

---

## 5️⃣ Laravel `.env` changes

Terraform injects Redis host/port into ECS containers as environment variables, so in your **Laravel `.env`**:

```env
REDIS_HOST=${REDIS_HOST}
REDIS_PORT=${REDIS_PORT}
```

---

## 🚀 Workflow Summary

1. Run Terraform:

   ```bash
   terraform init
   terraform apply
   ```

   → Creates VPC, ECS cluster, ECR repos, Redis cluster, services.

2. Push Docker images to ECR (Laravel + Socket).

3. ECS tasks auto-inject Redis endpoint, so both **Laravel** & **Socket.IO** connect to the same Redis backend.

---

✅ Now your Laravel app + Socket.IO will both use Redis in AWS.

---

Do you want me to also **add Redis authentication (password) and TLS support** in Terraform for production security, or keep it simple (open within VPC only)?




-----------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------------------

Perfect 👌 you want to automate deploying your **Laravel app + Socket.IO server + Redis** on AWS using **Terraform** with variables, so that you don’t need to manually configure ECS/ECR via the console.

I’ll give you a **step-by-step setup** with **complete Terraform files**.

---

## 📂 Project structure

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── ecr.tf
├── ecs.tf
├── vpc.tf
├── security.tf
├── provider.tf
```

---

## 1️⃣ provider.tf

```hcl
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile
}
```

---

## 2️⃣ variables.tf

```hcl
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "aws_profile" {
  description = "AWS CLI profile"
  type        = string
  default     = "default"
}

variable "app_name" {
  description = "Application name"
  type        = string
  default     = "civilbook"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "Public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  description = "Private subnets"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "container_port" {
  description = "Container port (Laravel = 8000, Socket.IO = 3001)"
  type        = number
  default     = 8000
}
```

---

## 3️⃣ vpc.tf

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = {
    Name = "${var.app_name}-vpc"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = element(var.public_subnets, count.index)
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private" {
  count      = length(var.private_subnets)
  vpc_id     = aws_vpc.main.id
  cidr_block = element(var.private_subnets, count.index)
}
```

---

## 4️⃣ security.tf

```hcl
resource "aws_security_group" "ecs_sg" {
  name        = "${var.app_name}-ecs-sg"
  description = "Allow HTTP/HTTPS/WebSocket"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow WebSocket"
    from_port   = 3001
    to_port     = 3001
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 5️⃣ ecr.tf

```hcl
resource "aws_ecr_repository" "laravel" {
  name = "${var.app_name}-laravel"
}

resource "aws_ecr_repository" "socket" {
  name = "${var.app_name}-socket"
}
```

👉 After `terraform apply`, build & push images:

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.us-east-1.amazonaws.com

# Build & push
docker build -t civilbook-laravel ./laravel
docker tag civilbook-laravel:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-laravel:latest
docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-laravel:latest

docker build -t civilbook-socket ./socket-server
docker tag civilbook-socket:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-socket:latest
docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-socket:latest
```

---

## 6️⃣ ecs.tf

```hcl
resource "aws_ecs_cluster" "this" {
  name = "${var.app_name}-cluster"
}

resource "aws_ecs_task_definition" "laravel" {
  family                   = "${var.app_name}-laravel"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([
    {
      name      = "laravel"
      image     = "${aws_ecr_repository.laravel.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 8000
          hostPort      = 8000
        }
      ]
    }
  ])
}

resource "aws_ecs_task_definition" "socket" {
  family                   = "${var.app_name}-socket"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([
    {
      name      = "socket"
      image     = "${aws_ecr_repository.socket.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 3001
          hostPort      = 3001
        }
      ]
    }
  ])
}

resource "aws_ecs_service" "laravel" {
  name            = "${var.app_name}-laravel-service"
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.laravel.arn
  launch_type     = "FARGATE"
  desired_count   = 1

  network_configuration {
    subnets         = aws_subnet.public[*].id
    security_groups = [aws_security_group.ecs_sg.id]
    assign_public_ip = true
  }
}

resource "aws_ecs_service" "socket" {
  name            = "${var.app_name}-socket-service"
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.socket.arn
  launch_type     = "FARGATE"
  desired_count   = 1

  network_configuration {
    subnets         = aws_subnet.public[*].id
    security_groups = [aws_security_group.ecs_sg.id]
    assign_public_ip = true
  }
}
```

---

## 7️⃣ outputs.tf

```hcl
output "ecr_laravel" {
  value = aws_ecr_repository.laravel.repository_url
}

output "ecr_socket" {
  value = aws_ecr_repository.socket.repository_url
}

output "ecs_cluster" {
  value = aws_ecs_cluster.this.name
}
```

---

## 🚀 Deployment Steps

1. **Init Terraform**

   ```bash
   terraform init
   ```

2. **Plan**

   ```bash
   terraform plan -out=tfplan
   ```

3. **Apply**

   ```bash
   terraform apply tfplan
   ```

4. **Push Images** (after infra created):

   ```bash
   docker build -t civilbook-laravel ./app
   docker tag civilbook-laravel:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-laravel:latest
   docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-laravel:latest

   docker build -t civilbook-socket ./socket-server
   docker tag civilbook-socket:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-socket:latest
   docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/civilbook-socket:latest
   ```

5. **Update ECS Services** so they pull the latest images.

---

👉 Now you have a **Terraform automated infra** with:

* VPC + subnets
* Security groups
* ECR repos
* ECS Fargate cluster
* Laravel service (port 8000)
* Socket.IO service (port 3001)

---

Would you like me to also **add an Application Load Balancer (ALB)** in Terraform so your users can access both Laravel (`/`) and WebSockets (`/socket.io`) behind the same HTTPS domain (`beta.civilbook.in`)?
