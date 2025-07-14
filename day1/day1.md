

## Dockerfile: Create a Dockerfile for NGINX,  Run a single NGINX container, Deploy a simple Pod running NGINX, Write a playbook to install NGINX on Ubuntu, and Write YAML-style HCL to create an AWS EC2 instance

Creating Create a Dockerfile for NGINX

FROM nginx:latest

LABEL maintainer = "praveenkumarpentamalla@gmail.com

COPY ./html /usr/share/nginx/html

COPY ./nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off"] 



## Docker Compose: Run a single NGINX container.


version: 3.8
service:
  nginx:
    image: nginx
    container_name: nginx-container
    port:
      - 8080:80
    volumes:
      - ./nginx-config: /etc/nginx/conf.d
      - ./public-html:/usr/share/nginx/html
    restart: unless-stopped
    networks:
      - nginx-nw
networks:
  nginx-nw:
    drive: bridge




## Deploy a simple Pod running NGINX.

apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    app: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: 64Mi
          cpu: 250m
        limits:
          memory: 128Mi
          cpu: 500i

## Write a playbook to install NGINX on Ubuntu.

inventory.ini

[web-app]

32.45.0.12


playbook.yaml


- name: Installing Nginx on Ubuntu
  hosts: web-app
  become: true
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes


## create an AWS EC2 instance.

provider "aws" {
  region = "ap-south-1"
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

resource "aws_security_group" "dev_security_group" {
  name        = "dev-security-group"
  description = "Allow SSH and HTTP"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 5432
    to_port     = 5432
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

resource "aws_instance" "dev_server" {
  ami                    = "ami-0f58b397bc5c1f2e8" 
  instance_type          = "t3a.medium"
  key_name               = "your-key-name"          
  vpc_security_group_ids = [aws_security_group.dev_security_group.id]
  subnet_id              = tolist(data.aws_subnets.default.ids)[0]
  associate_public_ip_address = true
  user_data              = file("${path.module}/setup.sh")
  
  tags = {
    Name = "dev-server"
  }
}

output "dev_server_public_ip" {
  value = aws_instance.dev_server.public_ip
}
