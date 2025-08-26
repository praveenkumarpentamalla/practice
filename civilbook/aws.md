pure AWS Console, click-by-click guide to move your 3 services (Laravel, Socket.IO, Redis) behind an ALB on ECR + ECS Fargate, with / to Laravel and /socket.io/* to the socket server.

⚠️ Quick Plan (simple & works):

Store images in ECR

Run 2 ECS services (Laravel port 8000, Socket port 3001)

Put an Application Load Balancer in front

Use path rule /socket.io/* → socket server

For Redis, you can keep your current DB for now; later move to ElastiCache (recommended).

0) Prep (one time)

Decide your AWS Region (e.g., us-east-1) — use it everywhere below.

Make sure your Docker images are built locally (civilbook-civil-web, civilbook-socket-server) and include:

Laravel route /health that returns 200 OK

Node server /health already exists in your server.js

Own the domain beta.civilbook.in (preferably in Route 53).

1) Create ECR repositories (console)

Open ECR → Repositories → Create repository.

Create two repos:

civil-app-web

socket-server

After each is created, click the repo → View push commands.
Copy those commands; you’ll run them on your server/laptop to push your local images.

You must still push via Docker CLI. The console gives you the exact commands:

Login to ECR

Tag your local image to the ECR URI

Push

Push both images (latest tag is fine for now).

2) Get an HTTPS certificate (ACM)

Open Certificate Manager (ACM) → Request → Request a public certificate.

Add domain: beta.civilbook.in.

DNS validation → Next → Confirm.

ACM shows a CNAME to add. If your domain is in Route 53, click Create record in Route 53.

Wait until the certificate status becomes Issued.

3) Security groups (EC2 console)

Create two:

3.1 ALB-SG

Inbound:

HTTP 80 from 0.0.0.0/0

HTTPS 443 from 0.0.0.0/0

Outbound: Allow all

3.2 ECS-Tasks-SG

Inbound:

TCP 8000 from ALB-SG

TCP 3001 from ALB-SG

Outbound: Allow all

This means only the ALB can reach your containers.

4) Target groups (EC2 → Target Groups)

Create two “Application” target groups (target type = IP):

4.1 TG-Laravel

Target type: IP

Protocol: HTTP

Port: 8000

VPC: (same as your ALB/ECS)

Health check:

Path: /health

Success codes: 200

(Leave intervals default for now)

4.2 TG-Socket

Target type: IP

Protocol: HTTP

Port: 3001

Health check Path: /health

You won’t register IPs manually; ECS will do that when services start.

5) Application Load Balancer (EC2 → Load Balancers)

Create load balancer → Application Load Balancer

Name: e.g., civilbook-alb

Scheme: Internet-facing

IP type: IPv4

Listeners:

Add HTTP : 80

Add HTTPS : 443 (you’ll pick the cert in a moment)

Availability Zones: pick 2 public subnets in your VPC

Security Group: select ALB-SG

Listener configuration:

For HTTPS:443, choose your ACM certificate

Set Default action → forward to TG-Laravel

Create ALB.

5.1 Listener rules

Go to Listeners tab → Edit rules on HTTPS:443

Rule: If Path is /socket.io/* → Forward to TG-Socket

Keep Default → TG-Laravel

On HTTP:80, Edit → Add rule: Redirect to HTTPS (443).

5.2 ALB attributes (optional but nice)

Edit attributes → set Idle timeout to 120 seconds (WebSockets).

6) Route 53 DNS

Hosted zones → your zone → Create record

Name: beta.civilbook.in

Type: A – IPv4 address

Alias: Yes → choose your ALB

Save. (Propagation can take a few minutes.)

7) ECS Cluster (ECS console)

Open ECS → Clusters → Create Cluster

Networking only (Fargate)

Name it, pick your VPC

Create

Quick path: We’ll run tasks in public subnets with Assign public IP = ENABLED to avoid NAT complexity. (Later move to private subnets + NAT.)

8) Task Definitions (ECS console)
8.1 Laravel task definition

Task definitions → Create new task definition

Launch type: Fargate

OS/Architecture: Linux/X86_64

Task size: CPU 0.5 vCPU, Memory 1 GB (adjust if needed)

Task role: (none for now)

Execution role: create/select ecsTaskExecutionRole

Container → Add container:

Name: civil-app-web

Image: (ECR image URI for civil-app-web:latest)

Port mappings: 8000 (tcp)

Env variables (minimum):

APP_ENV=production

APP_URL=https://beta.civilbook.in

LOG_CHANNEL=stack

LOG_LEVEL=debug

DB_HOST=54.147.175.9

DB_PORT=3306

DB_DATABASE=civilbook

DB_USERNAME=...

DB_PASSWORD=...

CACHE_DRIVER=redis (later swap to ElastiCache if you want)

SESSION_DRIVER=redis

QUEUE_CONNECTION=redis

REDIS_HOST= (fill later if moving to ElastiCache)

Logging:

Log driver: awslogs

Log group: /ecs/civil-app-web (create)

Region: your region

Stream prefix: ecs

(Optional) Health check (container-level): skip; ALB will check /health.

Create the task definition.

8.2 Socket task definition

Create new task definition again (Fargate)

CPU 0.25 vCPU, Memory 0.5 GB is fine

Same roles as above

Add container:

Name: socket-server

Image: (ECR URI for socket-server:latest)

Port mappings: 3001 (tcp)

Env variables:

PORT=3001

CORS_ORIGIN=https://beta.civilbook.in

LARAVEL_API_URL=https://beta.civilbook.in

Logging: awslogs → /ecs/socket-server

Create the task definition.

9) Services (run the tasks)
9.1 Laravel service

ECS → Clusters → your cluster → Create → Create service

Deployment configuration:

Compute: Fargate

Task definition: civil-app-web:latest

Service name: civil-app-web-svc

Desired tasks: 1

Networking:

VPC: same as ALB

Subnets: public subnets (Quick path)

Security group: ECS-Tasks-SG

Auto-assign public IP: ENABLED

Load balancing:

Enable Application Load Balancer

Select your ALB

Listener: HTTPS 443

Target group: choose TG-Laravel

Container to load balance: pick civil-app-web:8000

Create service.

9.2 Socket service

Create service again

Task definition: socket-server:latest

Name: socket-server-svc

Desired tasks: 1

Networking: same VPC/subnets, ECS-Tasks-SG, public IP ENABLED

Load balancing:

Enable ALB

ALB: same as above

Listener: HTTPS 443

Target group: TG-Socket

Container to LB: socket-server:3001

Create service.

After a minute or two, go to EC2 → Target Groups and check both target groups show healthy targets.

10) Point DNS & test

Confirm Route 53 record beta.civilbook.in → ALB (Alias) exists

Visit:

https://beta.civilbook.in/health → should return OK (Laravel)

https://beta.civilbook.in/socket.io/?EIO=4&transport=polling → should return a small JSON (Socket.IO handshake)

Your client/mobile should connect with:

URL: https://beta.civilbook.in

Path: /socket.io

No port in URL

Troubleshooting (console-first)

ALB 502/504:

EC2 → Target Groups → Targets → unhealthy?

Check security groups (ECS-Tasks-SG must allow 8000/3001 from ALB-SG)

Env vars correct?

Task logs (CloudWatch) show app is listening on the right port?

WebSocket not upgrading:

Check ALB Listener rules: /socket.io/* → TG-Socket

ALB Idle timeout ≥ 120s

Client uses https://beta.civilbook.in (no port), transports websocket or fallback to polling

Laravel can’t reach DB:

DB firewall must allow connections from your ECS task public IPs (Quick path) or from the VPC/NAT (Prod path)

CORS:

Socket container env: CORS_ORIGIN=https://beta.civilbook.in

Logs:

CloudWatch Logs → /ecs/civil-app-web and /ecs/socket-server

Optional (production hardening later)

Put tasks in private subnets (Disable public IP), add NAT Gateway

Use ElastiCache Redis and update REDIS_HOST

Store secrets in SSM/Secrets Manager and reference in task defs

Add Auto Scaling on services

Add AWS WAF in front of the ALB
