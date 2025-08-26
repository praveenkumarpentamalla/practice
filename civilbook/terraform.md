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
