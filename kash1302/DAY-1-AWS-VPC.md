Haan bhai 😄 samajh gaya. Tumhe **ek hi Markdown code block** chahiye jise directly **Copy → GitHub `.md` file → Paste** kar sako.

Main next se exactly isi format mein dunga:

````markdown
# 🚀 DAY 1 — AWS VPC & NETWORKING

## Q1. Design a production-ready AWS VPC.

### Answer

I would create a custom VPC across multiple Availability Zones.

The architecture would contain:

- Public subnets
- Private application subnets
- Private database subnets

The ALB would be deployed in public subnets.
EC2 application servers would be deployed in private subnets.
RDS PostgreSQL would be deployed in private database subnets.

I would use AWS WAF in front of the ALB and follow least-privilege Security Group rules.

### Architecture

```text
Internet
   |
   v
  WAF
   |
   v
  ALB
  / \
 /   \
AZ-1 AZ-2
 |     |
EC2   EC2
 \     /
  \   /
   RDS
````

### Senior-Level Interview Answer

> "I would design the production VPC across multiple AZs for high availability. The ALB would be placed in public subnets, application servers in private subnets, and RDS in private database subnets. I would use WAF in front of the ALB and Security Group references between ALB, application and database tiers."

---

## Q2. How will a private EC2 instance access the internet?

### Answer

A private EC2 instance can use a NAT Gateway for outbound internet access.

The NAT Gateway should be deployed in a public subnet.

Private subnet route table:

```text
0.0.0.0/0 → NAT Gateway
```

Public subnet route table:

```text
0.0.0.0/0 → Internet Gateway
```

### Flow

```text
Private EC2
    |
    v
Private Route Table
    |
    v
NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

### Interview Answer

> "For IPv4 outbound internet access, I would route the private subnet's default route to a NAT Gateway deployed in a public subnet. The NAT Gateway then uses the Internet Gateway to reach the internet."

---

## Q3. Private EC2 cannot download packages. How will you troubleshoot?

### Answer

I would troubleshoot layer by layer.

### 1. Check DNS

```bash
nslookup google.com
```

or:

```bash
dig google.com
```

### 2. Check routing

```bash
ip route
```

Verify:

```text
0.0.0.0/0 → NAT Gateway
```

### 3. Check NAT Gateway

Verify:

* NAT Gateway exists
* State is `Available`
* NAT Gateway is in a public subnet
* Elastic IP is attached

### 4. Check Public Route Table

Verify:

```text
0.0.0.0/0 → Internet Gateway
```

### 5. Check Security Group

Verify outbound HTTPS:

```text
TCP 443 → 0.0.0.0/0
```

### 6. Check NACL

Verify both inbound and outbound traffic because NACL is stateless.

### 7. Test connectivity

```bash
curl -v https://google.com
```

### 8. Check VPC Flow Logs

Look for:

```text
ACCEPT
```

or:

```text
REJECT
```

### 9. Check AWS Health

If multiple instances or AZs are affected, check AWS Health for any AWS-side issue.

### Senior-Level Answer

> "I would start from the affected EC2 and validate DNS, routing and HTTPS connectivity. Then I would verify the private subnet route to NAT Gateway, NAT Gateway availability and its public-subnet route to the IGW. After that I would check Security Groups, NACLs and VPC Flow Logs. If the issue is broader, I would check AWS Health."

---

## Q4. What is the difference between Security Group and NACL?

| Feature         | Security Group        | NACL                       |
| --------------- | --------------------- | -------------------------- |
| Scope           | Resource/ENI          | Subnet                     |
| Stateful        | Yes                   | No                         |
| Rules           | Allow                 | Allow + Deny               |
| Return Traffic  | Automatically allowed | Must be explicitly allowed |
| Rule Evaluation | All applicable rules  | Rules evaluated by number  |

### Interview Answer

> "Security Groups are stateful, resource-level firewalls, while NACLs are stateless, subnet-level firewalls. Security Groups support allow rules, whereas NACLs support both allow and deny rules."

---

## Q5. Why do we deploy applications across multiple AZs?

### Answer

The primary reason is **High Availability and failure isolation**.

For example:

```text
          ALB
         /   \
        /     \
     AZ-1     AZ-2
      |         |
     EC2       EC2
      ✅        ✅
```

If AZ-1 becomes unavailable:

```text
AZ-1 ❌
AZ-2 ✅
```

Traffic can continue to the healthy instances in AZ-2.

### Interview Answer

> "Multi-AZ protects the application from an Availability Zone-level failure and provides high availability. The application must also have sufficient capacity in the remaining AZs."

---

## Q6. ALB target is unhealthy. What will you check?

### Answer

First I would check the target group's health-check configuration.

For example:

```text
Protocol: HTTP
Port: 8080
Path: /health
```

Then:

### 1. Test locally

```bash
curl -v http://localhost:8080/health
```

### 2. Check application port

```bash
ss -lntp
```

### 3. Check application binding

The application should not be listening only on:

```text
127.0.0.1:8080
```

It may need to listen on:

```text
0.0.0.0:8080
```

depending on the application.

### 4. Check Security Group

EC2 Security Group should allow traffic from ALB Security Group.

```text
ALB-SG
   |
   | TCP 8080
   v
EC2-SG
```

### 5. Check NACL

Check both directions.

### 6. Check application logs

Verify whether the ALB health-check request is reaching the application.

### 7. Compare with healthy EC2

Compare:

* Application version
* Port
* Configuration
* Environment variables
* Security Group
* NACL
* Subnet

### Senior-Level Answer

> "I would verify that the health-check request can reach the application and that the application returns the expected status code. Then I would validate the target port, application binding, Security Group, NACL and application logs. I would also compare the unhealthy instance with a known healthy target."

---

# 🔥 DAY 1 — QUICK REVISION

## Private EC2 → Internet

```text
EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Public Route Table
 ↓
Internet Gateway
 ↓
Internet
```

## ALB → EC2

```text
Internet
 ↓
WAF
 ↓
ALB
 ↓
ALB-SG
 ↓
EC2-SG
 ↓
Application
```

## EC2 → RDS

```text
EC2
 ↓
EC2-SG
 ↓
VPC Local Route
 ↓
RDS-SG
 ↓
RDS
```

## Troubleshooting Order

```text
DNS
 ↓
Routing
 ↓
Security Group
 ↓
NACL
 ↓
NAT/IGW
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
Application
 ↓
Dependency
```

# 🎯 DAY 1 COMPLETE

**AWS VPC + Networking + NAT + SG/NACL + ALB + Multi-AZ + RDS Troubleshooting**

---

# 🚀 NEXT TOPIC

## DAY 2 — EC2 + ALB + Auto Scaling

* EC2 Deep Dive
* AMI
* EBS
* ENI
* User Data
* IAM Role
* Instance Metadata
* ALB Listeners
* Target Groups
* Health Checks
* 502 vs 503
* TLS Termination
* Path-Based Routing
* Host-Based Routing
* Auto Scaling Groups
* Launch Templates
* Scaling Policies
* Zero-Downtime Deployment

```
