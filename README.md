# aws-week2-vpc-alb-ha
A highly available, secure web tier on AWS built with a custom VPC, Application Load Balancer, and Auto Scaling Group across two Availability Zones. This project implements a production-style web tier on AWS designed for high availability, fault tolerance, and secure network isolation.

The system distributes traffic across multiple Availability Zones using an Application Load Balancer and maintains service continuity through Auto Scaling and health checks.

This project focuses on how infrastructure behaves under failure — not just how it is deployed.
---

## Objective

The goal of this project is to design and deploy a production-style web architecture on AWS that demonstrates:

- **Network isolation** using public and private subnets
- **High availability** by spreading workloads across multiple Availability Zones
- **Auto-healing** infrastructure using Auto Scaling Groups
- **Least-privilege security** using Security Group referencing

This was completed as part of Week 2 of a structured AWS learning program.

---

## Architecture Diagram

![Architecture Diagram](diagrams/week2-architecture.png)
---
## Request Flow

1. A user sends a request to the Application Load Balancer (ALB)
2. The ALB receives traffic via public subnets across two Availability Zones
3. The ALB forwards traffic to a target group of EC2 instances
4. EC2 instances run in private subnets and respond to the request
5. Health checks continuously monitor instance availability
6. If an instance fails, Auto Scaling replaces it automatically
   
## Components

| Resource | Name | Purpose |
|---|---|---|
| VPC | `week2-vpc-project` | Isolated network (10.0.0.0/16) |
| Public Subnet A | `10.0.1.0/24` | ALB node in AZ-A |
| Public Subnet B | `10.0.2.0/24` | ALB node in AZ-B |
| Private Subnet A | `10.0.11.0/24` | EC2 instances in AZ-A |
| Private Subnet B | `10.0.12.0/24` | EC2 instances in AZ-B |
| Internet Gateway | `week2-igw` | Public internet entry point |
| NAT Gateway | `week2-nat` | Outbound internet for private instances |
| Security Group (ALB) | `week2-sg-alb` | Allows HTTP 80 from internet |
| Security Group (EC2) | `week2-sg-web` | Allows HTTP 80 from ALB SG only |
| Launch Template | `week2-launch-template` | EC2 config + nginx bootstrap |
| Target Group | `week2-tg` | Pool of healthy EC2 instances |
| Application Load Balancer | `week2-alb` | Distributes traffic across AZs |
| Auto Scaling Group | `week2-asg` | Maintains 2 instances, replaces failures |

---

## Security Model
- EC2 instances are deployed in private subnets and are not directly accessible from the internet
- Only the ALB security group is allowed to communicate with EC2 instances
- Security group referencing is used instead of IP-based rules for dynamic scaling
- Outbound internet access for instances is provided via NAT Gateway
- No direct SSH access is enabled — access would be handled via SSM or bastion host in production
### Security Group Rules

**ALB Security Group (`week2-sg-alb`)**
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 (internet) |
| Outbound | All | All | 0.0.0.0/0 |

**EC2 Security Group (`week2-sg-web`)**
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | `week2-sg-alb` only |
| Outbound | All | All | 0.0.0.0/0 |

### Design Principles

- EC2 instances live in **private subnets** — not reachable from the internet directly
- Only the ALB can send traffic to instances (SG references SG, not IP ranges)
- Instances reach the internet for updates via **NAT Gateway** (outbound only)
- No SSH open to the internet — bastion host or SSM would be used for access in production

---
## Design Decisions and Tradeoffs

### Single NAT Gateway
A single NAT Gateway was used to reduce cost during development.

Tradeoff:
- Lower cost
- Introduces a single point of failure for outbound traffic

Production alternative:
- One NAT Gateway per Availability Zone for full resilience

---

### EC2-Based Web Tier
The application runs directly on EC2 instances using a launch template.

Tradeoff:
- Simpler to configure and understand
- Less portable and harder to scale compared to container-based deployments

Next step:
- Replace with Docker-based deployment and CI/CD pipeline

---

### No SSH Access
SSH was not exposed to the internet.

Tradeoff:
- Improves security posture
- Makes debugging more difficult

Production alternative:
- Use AWS Systems Manager (SSM) or a bastion host
  
## HA Test Evidence

### Test: Instance Termination + Auto Recovery

**Step 1 — Terminated instance `i-0abc123def456` in AZ-A**

ASG Activity log showed:
```
Terminating EC2 instance: i-0abc123def456  [AZ: ca-central-1a]
Launching a new EC2 instance to maintain desired capacity
Successfully launched: i-0xyz789ghi012   [AZ: ca-central-1a]
```

**Step 2 — ALB continued serving traffic throughout**

The browser was refreshed continuously during the replacement. The page remained available with no errors. Once the new instance passed health checks, it began receiving traffic.

**Result:** Zero downtime. Service remained available during instance failure and replacement.

## High Availability Validation
Result: Zero downtime observed. Traffic continued to be served throughout instance termination and replacement.

This demonstrates that load balancing, health checks, and Auto Scaling are correctly integrated.

### Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/vpc-subnets.png` | VPC subnet list with public/private labels |
| `screenshots/route-tables.png` | Public route table (IGW) vs private (NAT) |
| `screenshots/alb-listeners.png` | ALB listener forwarding HTTP:80 to target group |
| `screenshots/healthy-targets.png` | Both targets showing "healthy" status |
| `screenshots/asg-activity.png` | ASG activity log during termination + replacement |
| `screenshots/browser-az-a.png` | Browser showing Instance ID from AZ-A |
| `screenshots/browser-az-b.png` | Browser showing Instance ID from AZ-B |

---
## Production Improvements

- Add HTTPS using AWS Certificate Manager (ACM)
- Introduce Web Application Firewall (WAF) for security
- Implement centralized logging (CloudWatch Logs / ELK)
- Use container orchestration (ECS or EKS) instead of EC2-based deployment
- Deploy NAT Gateway per AZ for full fault tolerance
- Add scaling policies based on CPU or request count
  
## Costs & Teardown

### Estimated Costs (if left running 24 hrs)

| Resource | Approx. Cost |
|---|---|
| NAT Gateway | ~$0.045/hr + data transfer |
| ALB | ~$0.008/hr + LCU charges |
| 2x t3.micro EC2 | ~$0.021/hr total |
| **Total (24 hrs)** | **~$2–4** |

> ⚠️ NAT Gateway is the biggest cost driver. Always delete it when not actively using the environment.

---

## Key Concepts Learned

- **Public vs Private subnets** — Public subnets route to an IGW; private subnets use NAT for outbound only
- **IGW vs NAT Gateway** — IGW is bidirectional; NAT is outbound-only for private resources
- **ALB vs Target Group vs Listener** — ALB is the entry point; listener is the rule; target group is the destination
- **ELB health checks vs EC2 status checks** — ELB checks application-level health; ASG uses this to replace broken instances
- **SG referencing SG** — More secure and dynamic than IP allowlists; works even as instances are replaced

---
## Key Takeaways

- High availability requires coordination between load balancing and scaling
- Private subnets significantly reduce attack surface
- Auto Scaling Groups enable self-healing infrastructure
- Testing failure scenarios is essential to validate system design
  
## Region

`ca-central-1` (Canada)
