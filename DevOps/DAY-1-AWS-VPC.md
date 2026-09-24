# 🚀 DAY 1 — AWS VPC & NETWORKING
## Senior DevOps / AWS Engineer — Interview Preparation

> **Target:** Senior DevOps / AWS Engineer
>
> **Focus:** Practical understanding + Production Troubleshooting + Interview Confidence
>
> **Day 1 Topics:** VPC, Subnets, Route Tables, IGW, NAT Gateway, Security Groups, NACL, DNS, ALB, Multi-AZ, RDS, Third-Party APIs, SSL/TLS, Troubleshooting

---

# 📚 TABLE OF CONTENTS

1. [Production VPC Architecture](#1-production-vpc-architecture)
2. [Public and Private Subnets](#2-public-and-private-subnets)
3. [Security Group Design](#3-security-group-design)
4. [Private EC2 Internet Access](#4-private-ec2-internet-access)
5. [Private EC2 Cannot Download Packages](#5-private-ec2-cannot-download-packages)
6. [NAT Gateway Troubleshooting](#6-nat-gateway-troubleshooting)
7. [DNS Troubleshooting](#7-dns-troubleshooting)
8. [Internet Connectivity Commands](#8-internet-connectivity-commands)
9. [Security Group vs NACL](#9-security-group-vs-nacl)
10. [NACL Deny Rules](#10-nacl-deny-rules)
11. [AWS Service/Region Issue](#11-aws-serviceregion-issue)
12. [Third-Party API Troubleshooting](#12-third-party-api-troubleshooting)
13. [NAT IP Allowlisting](#13-nat-ip-allowlisting)
14. [SSL/TLS Troubleshooting](#14-ssltls-troubleshooting)
15. [Multi-AZ](#15-multi-az)
16. [ALB and Multi-AZ](#16-alb-and-multi-az)
17. [ALB Target Unhealthy](#17-alb-target-unhealthy)
18. [Application Listening on Localhost](#18-application-listening-on-localhost)
19. [EC2 to RDS Connectivity](#19-ec2-to-rds-connectivity)
20. [AZ Failure](#20-az-failure)
21. [Round Robin and Sticky Sessions](#21-round-robin-and-sticky-sessions)
22. [RDS Too Many Connections](#22-rds-too-many-connections)
23. [Database Connection Pool](#23-database-connection-pool)
24. [Slow Database Queries](#24-slow-database-queries)
25. [Production Troubleshooting Framework](#25-production-troubleshooting-framework)
26. [Important Troubleshooting Commands](#26-important-troubleshooting-commands)
27. [Senior-Level Interview Answer Pattern](#27-senior-level-interview-answer-pattern)
28. [Rapid Fire Questions](#28-rapid-fire-questions)
29. [Day 1 Final Revision](#29-day-1-final-revision)

---

# 1. Production VPC Architecture

## Q1. Design a production-ready AWS VPC for a web application.

### Answer

I would create a custom VPC and distribute the infrastructure across multiple Availability Zones for high availability.

I would divide the VPC into:

- Public subnets
- Private application subnets
- Private database subnets

The internet-facing Application Load Balancer would be deployed in public subnets.

The application EC2 instances would be deployed in private subnets.

The PostgreSQL RDS database would be deployed in private/database subnets.

I would also place AWS WAF in front of the ALB.

### Architecture

```text
                         INTERNET
                             |
                             v
                           AWS WAF
                             |
                             v
                    +----------------+
                    |      ALB       |
                    +----------------+
                       /          \
                      /            \
                     v              v
                  AZ-1             AZ-2
                   |                 |
                   v                 v
                EC2/App           EC2/App
              Private Subnet    Private Subnet
                   |                 |
                   +--------+--------+
                            |
                            v
                     RDS PostgreSQL
                    Private Subnets
```

### Security Group Flow

```text
Internet
   |
   | HTTPS 443
   v
ALB-SG
   |
   | Application Port
   v
EC2-SG
   |
   | PostgreSQL 5432
   v
RDS-SG
```

### Senior-Level Interview Answer

> "I would design a custom VPC across multiple Availability Zones. The ALB would be deployed in public subnets, application servers in private subnets, and RDS in private database subnets. I would use WAF in front of the ALB and Security Group references between the ALB, application and database tiers. For private workloads requiring outbound internet access, I would use NAT Gateway."

### Important Point

Do not say:

> "I will allow 0.0.0.0/0 on EC2."

Instead use:

```text
ALB-SG → EC2-SG
EC2-SG → RDS-SG
```

This follows least privilege.

---

# 2. Public and Private Subnets

## Q2. What makes a subnet public or private?

### Answer

A subnet is considered public when its associated route table has a route to an Internet Gateway.

Example:

```text
0.0.0.0/0 → Internet Gateway
```

A private subnet does not have a direct route to the Internet Gateway.

A private subnet can use a NAT Gateway for outbound internet access.

### Architecture

```text
PUBLIC SUBNET

EC2/ALB
   |
Route Table
   |
0.0.0.0/0
   |
Internet Gateway
   |
Internet
```

```text
PRIVATE SUBNET

EC2
   |
Route Table
   |
0.0.0.0/0
   |
NAT Gateway
   |
Internet Gateway
   |
Internet
```

### Senior-Level Answer

> "A subnet is public when its routing provides a path to an Internet Gateway. A private subnet doesn't have a direct route to the IGW. Private resources can use a NAT Gateway for outbound internet connectivity."

---

# 3. Security Group Design

## Q3. How would you configure Security Groups for this architecture?

### Answer

I would create separate Security Groups for each tier.

### ALB Security Group

Allow:

```text
Inbound:
TCP 443
Source: 0.0.0.0/0
```

Optionally HTTP 80 can be allowed only for redirecting HTTP to HTTPS.

### EC2 Security Group

Allow application traffic only from the ALB Security Group.

Example:

```text
Inbound:
TCP 8080
Source: ALB-SG
```

### RDS Security Group

Allow PostgreSQL only from the EC2 Security Group.

```text
Inbound:
TCP 5432
Source: EC2-SG
```

### Architecture

```text
Internet
   |
   | 443
   v
ALB-SG
   |
   | 8080
   v
EC2-SG
   |
   | 5432
   v
RDS-SG
```

### Senior-Level Answer

> "I would avoid IP-based allowlisting between application tiers where possible and use Security Group references. That way the database trusts only the application tier rather than an entire subnet or CIDR."

---

# 4. Private EC2 Internet Access

## Q4. EC2 is running in a private subnet. How can it download packages from the internet?

### Answer

I would use a NAT Gateway.

The NAT Gateway should be deployed in a public subnet.

The private subnet route table would contain:

```text
0.0.0.0/0 → NAT Gateway
```

The public subnet route table would contain:

```text
0.0.0.0/0 → Internet Gateway
```

### Complete Flow

```text
Private EC2
     |
     v
Private Route Table
     |
     | 0.0.0.0/0
     v
NAT Gateway
     |
     v
Public Route Table
     |
     | 0.0.0.0/0
     v
Internet Gateway
     |
     v
Internet
```

### Senior-Level Answer

> "For IPv4 outbound internet access, the private subnet would route its default traffic to a NAT Gateway deployed in a public subnet. The NAT Gateway then uses the Internet Gateway to reach the internet. The private EC2 remains without direct inbound internet exposure."

---

# 5. Private EC2 Cannot Download Packages

## Q5. EC2 is in a private subnet and suddenly cannot download packages. How will you troubleshoot?

### Answer

I would troubleshoot layer by layer.

---

## Step 1 — Check Package Repository

First I would verify whether the package repository itself is having an issue.

For example:

```text
apt repository
yum repository
Amazon Linux repository
Internal package repository
```

If the repository is down, the AWS network may be completely healthy.

---

## Step 2 — Check EC2 Network Configuration

Check IP and routing:

```bash
ip addr
```

```bash
ip route
```

I would verify the EC2 has a valid IP and default route.

---

## Step 3 — Check DNS

Use:

```bash
nslookup <repository-domain>
```

or:

```bash
dig <repository-domain>
```

If DNS resolution fails, investigate DNS before NAT.

---

## Step 4 — Check Security Group

Verify outbound HTTPS:

```text
Outbound
TCP 443
Destination: 0.0.0.0/0
```

---

## Step 5 — Check Private Subnet Route Table

Verify:

```text
0.0.0.0/0 → NAT Gateway
```

---

## Step 6 — Check NAT Gateway

Verify:

* NAT Gateway exists
* State is `Available`
* NAT Gateway is in a public subnet
* Elastic IP is attached

---

## Step 7 — Check Public Subnet Route Table

Verify:

```text
0.0.0.0/0 → Internet Gateway
```

---

## Step 8 — Check NACL

Because NACL is stateless, check both:

* Outbound traffic
* Return traffic

---

## Step 9 — Check Connectivity

```bash
curl -v https://<repository-domain>
```

---

## Step 10 — Check AWS Health

If multiple instances/AZs are affected, check AWS Health Dashboard for AWS-side incidents.

---

## Step 11 — Check Metrics

I would check NAT Gateway and network-related metrics to determine whether there is an unusual traffic pattern or capacity issue.

### Senior-Level Answer

> "I would first determine whether the issue is repository-specific or a general internet connectivity issue. Then I would validate DNS, routing, Security Groups, NACLs and the complete NAT Gateway path. I would test HTTPS directly from the affected EC2 and correlate the timing with CloudWatch metrics and AWS Health if the issue appears broader."

---

# 6. NAT Gateway Troubleshooting

## Q6. NAT Gateway exists, but private EC2 still cannot access the internet. What will you check?

### Answer

I would verify the complete network path.

```text
Private EC2
    |
    v
Private Route Table
    |
    | 0.0.0.0/0
    v
NAT Gateway
    |
    v
Public Route Table
    |
    | 0.0.0.0/0
    v
Internet Gateway
    |
    v
Internet
```

I would check:

1. Private subnet route table
2. NAT Gateway state
3. NAT Gateway subnet
4. NAT Gateway Elastic IP
5. Public subnet route table
6. Internet Gateway
7. EC2 Security Group
8. NACL
9. VPC Flow Logs
10. NAT Gateway metrics

### Important

The NAT Gateway should be deployed in a **public subnet**, not a private subnet.

---

# 7. DNS Troubleshooting

## Q7. How will you troubleshoot a DNS issue?

### Answer

I would first verify whether the domain resolves from the affected EC2.

```bash
nslookup api.example.com
```

or:

```bash
dig api.example.com
```

Then I would check:

* `/etc/resolv.conf`
* VPC DNS settings
* Route 53 records
* Private Hosted Zones
* Route 53 Resolver rules
* DNS response
* Whether only one host is affected or all hosts are affected

### Important

First separate:

```text
DNS problem
```

from:

```text
Network connectivity problem
```

If DNS doesn't resolve, testing TCP/HTTPS is premature.

### Senior-Level Answer

> "I would validate DNS resolution independently using nslookup or dig. If resolution fails, I would investigate the VPC DNS configuration, resolver configuration, Route 53 records and private hosted zones before moving to TCP-level troubleshooting."

---

# 8. Internet Connectivity Commands

## Q8. What commands would you use to troubleshoot connectivity?

### DNS

```bash
nslookup google.com
```

```bash
dig google.com
```

### Routing

```bash
ip route
```

### IP configuration

```bash
ip addr
```

### TCP connectivity

```bash
nc -vz google.com 443
```

### HTTPS

```bash
curl -v https://google.com
```

### TLS

```bash
openssl s_client \
  -connect api.example.com:443 \
  -servername api.example.com
```

### Listening ports

```bash
ss -lntp
```

### Important Correction

```bash
curl -V
```

only shows the curl version.

For connectivity troubleshooting, use:

```bash
curl -v https://google.com
```

---

# 9. Security Group vs NACL

## Q9. What is the difference between Security Group and NACL?

### Answer

| Feature         | Security Group          | NACL                       |
| --------------- | ----------------------- | -------------------------- |
| Scope           | Resource/ENI            | Subnet                     |
| Stateful        | Yes                     | No                         |
| Rules           | Allow                   | Allow + Deny               |
| Return traffic  | Automatically handled   | Must be explicitly allowed |
| Rule evaluation | All applicable rules    | Rule number order          |
| Typical use     | Resource-level security | Subnet-level filtering     |

### Security Group

Security Group is stateful.

If outbound traffic is allowed and the destination responds, the return traffic is automatically allowed for the established connection.

### NACL

NACL is stateless.

If outbound traffic is allowed, return traffic must also be explicitly allowed.

### Senior-Level Answer

> "Security Groups are stateful, resource-level firewalls, while NACLs are stateless, subnet-level firewalls. Security Groups support allow rules, whereas NACLs support both allow and deny rules."

---

# 10. NACL Deny Rules

## Q10. What happens if a NACL has a DENY rule?

### Answer

If the traffic matches a deny rule, the traffic is denied.

For example:

```text
Outbound HTTPS → ALLOW
Return Traffic → DENY
```

The connection can still fail because NACL is stateless.

### Example

```text
EC2
 |
 | Outbound TCP 443
 v
Internet
 |
 | Return Traffic
 v
NACL
 |
 X DENY
```

The connection will fail.

### Senior-Level Answer

> "If a matching NACL deny rule exists, the traffic is dropped. Since NACLs are stateless, I also need to verify the return path and ephemeral ports."

---

# 11. AWS Service/Region Issue

## Q11. What if your configuration looks correct but AWS itself may be facing an issue?

### Answer

I would first validate our own infrastructure rather than immediately blaming AWS.

Then I would determine the scope:

```text
One instance
    ↓
One subnet
    ↓
One AZ
    ↓
Multiple AZs
    ↓
Entire Region
```

If multiple workloads or AZs are affected simultaneously, I would check:

* AWS Health Dashboard
* AWS service status
* Regional issues
* Availability Zone issues
* Relevant service events

### Senior-Level Answer

> "I would first establish the blast radius and validate our configuration. If the issue affects multiple workloads or AZs simultaneously, I would check AWS Health for a possible AWS-side incident."

---

# 12. Third-Party API Troubleshooting

## Q12. EC2 can access Google but cannot access a third-party API. How will you troubleshoot?

### Answer

Since general internet access is working, I would narrow the issue to the destination-specific path.

First:

```bash
nslookup api.example.com
```

Then:

```bash
curl -v https://api.example.com
```

Then check:

1. DNS
2. TCP 443
3. TLS handshake
4. HTTP response
5. Authentication
6. Authorization
7. Request method
8. Required headers
9. API endpoint
10. Third-party IP allowlisting
11. NAT Gateway public IP
12. Application logs

### Troubleshooting Layers

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
Authentication
 ↓
Application
```

### Senior-Level Answer

> "Since general internet access is working, I would isolate the issue to the destination-specific path. I would verify DNS, TCP 443, TLS, HTTP response and authentication, then check whether the third party is restricting access by IP, domain or other authentication controls."

---

# 13. NAT IP Allowlisting

## Q13. Why would you check the NAT Gateway public IP with a third-party API?

### Answer

The EC2 is in a private subnet and has a private IP.

For example:

```text
EC2 Private IP:
10.0.2.10
```

When it accesses the internet through the NAT Gateway, the destination generally sees the NAT Gateway's public Elastic IP.

```text
EC2
10.0.2.10
   |
   v
NAT Gateway
   |
   | Elastic IP
   v
203.x.x.x
   |
   v
Third-Party API
```

If the third-party API uses IP allowlisting, the NAT Gateway's public IP must be allowlisted.

### Senior-Level Answer

> "For outbound traffic from a private subnet, the external destination generally sees the NAT Gateway's public IP rather than the EC2 private IP. Therefore I would verify that the NAT Gateway Elastic IP is allowlisted at the third party."

---

# 14. SSL/TLS Troubleshooting

## Q14. Third-party API is reachable but SSL/TLS handshake is failing. What will you check?

### Answer

I would first determine whether the failure occurs at:

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP
```

I would check:

* Certificate validity
* Certificate chain
* CA bundle
* TLS version
* Cipher compatibility
* SNI
* Hostname
* Proxy
* Application TLS configuration

### Test

```bash
curl -v https://api.example.com
```

For deeper TLS troubleshooting:

```bash
openssl s_client \
  -connect api.example.com:443 \
  -servername api.example.com
```

### Application-Level Checks

I would verify that the application is correctly passing:

* Target API domain
* API key
* Authentication token
* Required headers
* Request method
* Environment variables

### Senior-Level Answer

> "I would first identify whether the failure is during TCP connection, TLS negotiation or HTTP processing. I would use curl -v and openssl s_client to inspect the TLS handshake, certificate chain and SNI, and then verify the application's authentication and request configuration."

---

# 15. Multi-AZ

## Q15. Why do we deploy resources across multiple Availability Zones?

### Answer

The main reasons are:

* High availability
* Fault isolation
* Resilience against AZ failure
* Better application availability

For example:

```text
AZ-1
EC2-1
  |
  X  AZ FAILURE
  |
AZ-2
EC2-2
  |
  ✓ Application continues
```

If AZ-1 becomes unavailable, the application can continue serving traffic from AZ-2, assuming the architecture is designed correctly and enough capacity remains.

### Senior-Level Answer

> "The primary reason for Multi-AZ deployment is high availability and failure isolation. If one Availability Zone becomes unavailable, workloads in another AZ can continue serving traffic."

---

# 16. ALB and Multi-AZ

## Q16. Why do we need an ALB if EC2 instances are already deployed in multiple AZs?

### Answer

Multi-AZ and ALB solve different problems.

### Multi-AZ

Provides:

```text
High Availability
Failure Isolation
```

### ALB

Provides:

```text
Traffic Distribution
Health Checks
Single Entry Point
Path-Based Routing
Host-Based Routing
TLS Termination
```

### Architecture

```text
                    Client
                      |
                      v
                     ALB
                   /     \
                  /       \
               AZ-1       AZ-2
                |           |
              EC2-1       EC2-2
               ✓           ✓
```

### Senior-Level Answer

> "Multi-AZ provides infrastructure resilience, while ALB provides traffic distribution and health-aware routing. The ALB ensures requests are sent only to healthy targets."

---

# 17. ALB Target Unhealthy

## Q17. EC2-2 is running but ALB shows it as unhealthy. What will you check?

### Answer

I would troubleshoot the health-check path.

### Step 1 — Check Health Check Configuration

Example:

```text
Protocol: HTTP
Port: 8080
Path: /health
```

### Step 2 — Test from EC2

```bash
curl -v http://localhost:8080/health
```

### Step 3 — Check Application Port

```bash
ss -lntp
```

### Step 4 — Check Application Binding

The application should not be listening only on:

```text
127.0.0.1:8080
```

It may need to listen on:

```text
0.0.0.0:8080
```

depending on the application.

### Step 5 — Check EC2 Security Group

The EC2 Security Group should allow traffic from ALB Security Group.

```text
ALB-SG
   |
   | TCP 8080
   v
EC2-SG
```

### Step 6 — Check NACL

Check both inbound and outbound rules.

### Step 7 — Check Application Logs

Verify whether the health-check request is reaching the application.

### Step 8 — Compare EC2-1 and EC2-2

Check:

* Same application version
* Same health-check endpoint
* Same port
* Same configuration
* Same environment variables
* Same Security Group
* Same NACL behavior

### Senior-Level Answer

> "I would first verify whether the ALB health-check request can reach EC2-2 and whether the application returns the expected status code. Then I would check the target port, application binding, Security Group, NACL and logs. Since EC2-1 is healthy, I would use it as the baseline and compare EC2-2 against it."

---

# 18. Application Listening on Localhost

## Q18. What if the application is running only on localhost?

### Answer

Suppose:

```text
Application
127.0.0.1:8080
```

Then the application can be accessed from the same EC2 instance but may not be reachable through the EC2 private IP.

Check:

```bash
ss -lntp
```

Test locally:

```bash
curl http://localhost:8080/health
```

Then test through private IP:

```bash
curl http://<private-ip>:8080/health
```

If localhost works but private IP doesn't, investigate the application binding.

Depending on the application, it may need to bind to:

```text
0.0.0.0:8080
```

### Interview Answer

> "A service listening only on 127.0.0.1 accepts connections from the local host. For an ALB or another host to reach it, the service generally needs to listen on the appropriate network interface, commonly 0.0.0.0."

---

# 19. EC2 to RDS Connectivity

## Q19. EC2 in AZ-2 cannot connect to PostgreSQL RDS. How will you troubleshoot?

### Answer

I would start from EC2-2.

### Step 1 — DNS

```bash
nslookup <rds-endpoint>
```

### Step 2 — Test Port

```bash
nc -vz <rds-endpoint> 5432
```

or:

```bash
telnet <rds-endpoint> 5432
```

### Step 3 — Check RDS Security Group

RDS should allow:

```text
TCP 5432
Source: EC2-SG
```

Not:

```text
0.0.0.0/0
```

### Step 4 — Check Route

For same-VPC communication, the VPC local route handles communication.

```text
VPC CIDR → local
```

NAT Gateway is not required for EC2-to-RDS traffic inside the VPC.

### Step 5 — Check NACL

Check both EC2 and RDS subnet NACLs.

### Step 6 — Check RDS Status

Verify RDS is:

```text
Available
```

### Step 7 — Check VPC Flow Logs

Look for:

```text
ACCEPT
```

or:

```text
REJECT
```

### Step 8 — Compare AZ-1 and AZ-2

Since AZ-1 is working, compare:

* EC2 Security Group
* Subnet
* Route Table
* NACL
* DNS
* Network configuration

### Senior-Level Answer

> "Because AZ-1 is working, I would use it as the baseline and compare AZ-2 configuration against it. I would validate DNS, TCP 5432 connectivity, Security Groups, local VPC routing, NACLs, RDS status and VPC Flow Logs."

---

# 20. AZ Failure

## Q20. AZ-1 completely goes down. How will your application continue serving traffic?

### Answer

The application should be distributed across multiple AZs.

```text
                    ALB
                  /     \
                 /       \
              AZ-1       AZ-2
               ❌          ✓
               |           |
              EC2         EC2
```

If AZ-1 goes down, the ALB should continue routing traffic to healthy targets in AZ-2.

The remaining AZ should have enough capacity.

Auto Scaling can help maintain desired capacity.

The database tier should also be designed for high availability.

### Senior-Level Answer

> "If AZ-1 fails, the ALB should continue routing requests to healthy targets in AZ-2. The application tier must have sufficient capacity in the remaining AZ, and critical dependencies such as the database must also be designed without a single-AZ dependency."

---

# 21. Round Robin and Sticky Sessions

## Q21. Do we need to enable Round Robin in ALB for Multi-AZ?

### Answer

ALB distributes traffic across healthy targets using its load-balancing behavior.

We don't need to manually configure "Round Robin" as a separate switch to make Multi-AZ work.

The important part is:

* Both EC2 instances are registered in the Target Group
* Both are healthy
* ALB is deployed across AZs
* Security Groups allow traffic
* Application is stateless where possible

### Example

```text
                  ALB
                /     \
               /       \
            EC2-1     EC2-2
            Healthy   Healthy
```

Traffic can be distributed across healthy targets.

---

## Q22. Should we use sticky sessions?

### Answer

Not by default for a highly available stateless application.

Sticky sessions can cause requests from the same client to remain associated with one target.

For example:

```text
Client
  |
  +----> EC2-1
  |
  +----> EC2-1
  |
  +----> EC2-1
```

If EC2-1 fails, the session may be affected depending on how the application handles session state.

### Better Design

Use stateless application servers and externalize session state where required.

For example:

```text
EC2-1
EC2-2
EC2-3
   |
   v
Shared Session Store
```

### Senior-Level Answer

> "For highly scalable applications, I prefer stateless application servers and externalized session state rather than depending on sticky sessions. This improves load distribution and resilience."

---

# 22. RDS Too Many Connections

## Q23. Application is showing "too many database connections". How will you troubleshoot?

### Answer

I would first check RDS metrics.

Important metrics include:

* DatabaseConnections
* CPUUtilization
* FreeableMemory
* I/O-related metrics
* Performance Insights, if enabled

### Step 1 — Check when it started

For example:

```text
12:00 → 50 connections
12:15 → 100 connections
12:30 → 150 connections
12:45 → 200 connections
```

I would identify the exact time the connections started increasing.

### Step 2 — Correlate Application Logs

For the same timestamp, check application logs.

Identify:

* Which EC2 instance generated connections
* Which application component is responsible
* Whether traffic increased
* Whether connection errors started

### Step 3 — Check Connection Limit

Determine the database's connection capacity.

### Step 4 — Check Connection Pool

Check:

```text
maxPoolSize
minimumIdle
connectionTimeout
idleTimeout
maxLifetime
```

### Step 5 — Check for Connection Leaks

Look for connections that are not being released.

### Step 6 — Check Long-Running Queries

A slow query can keep connections occupied.

### Step 7 — Check PostgreSQL Sessions

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

Also investigate:

```text
active
idle
idle in transaction
```

### Senior-Level Answer

> "I would correlate the DatabaseConnections metric with application logs and traffic at the same timestamp. Then I would determine whether the issue is due to a connection pool configuration, connection leak, sudden traffic increase, long-running queries or the database connection limit."

---

# 23. Database Connection Pool

## Q24. How can connection pool configuration cause too many DB connections?

### Answer

Suppose there are:

```text
10 EC2 instances
```

and every application instance has:

```text
maxPoolSize = 50
```

Potential maximum connections:

```text
10 × 50 = 500 connections
```

If the RDS instance can support only:

```text
150 connections
```

the application can exhaust the database connection limit.

### Architecture

```text
EC2-1 → 50 connections
EC2-2 → 50 connections
EC2-3 → 50 connections
...
```

Total connections must be considered across all application instances.

### Senior-Level Answer

> "I would calculate aggregate connection capacity across all application instances rather than looking at one instance in isolation. The pool configuration needs to be sized according to database capacity, application concurrency and number of instances."

---

# 24. Slow Database Queries

## Q25. How can slow queries result in too many DB connections?

### Answer

A database connection is occupied while a query is running.

Suppose:

```text
Request 1
   |
   v
Connection 1
   |
Slow Query
```

The connection remains occupied for a long time.

As more requests arrive:

```text
Request 1 → Connection 1 → Slow
Request 2 → Connection 2 → Slow
Request 3 → Connection 3 → Slow
Request 4 → Connection 4 → Slow
...
```

Eventually:

```text
Connection Pool Exhausted
        |
        v
New Request
        |
        v
Connection Timeout
```

### PostgreSQL

Check active sessions:

```sql
SELECT pid,
       usename,
       state,
       query_start,
       query
FROM pg_stat_activity
WHERE state = 'active';
```

Check session states:

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

Investigate:

* Long-running queries
* Locks
* Blocked sessions
* Idle transactions
* Missing indexes
* Query execution plans

### Senior-Level Answer

> "Slow queries can indirectly cause connection exhaustion because each slow query holds a connection for longer. I would investigate query duration, locks, blocked sessions and connection-pool behavior rather than simply increasing the database connection limit."

---

# 25. Production Troubleshooting Framework

## Q26. How do you troubleshoot a production connectivity issue?

### Answer

I follow a structured troubleshooting approach.

---

## Step 1 — Understand the Error

First determine exactly what is failing.

Examples:

```text
DNS resolution failure
Connection timeout
Connection refused
TLS handshake failure
HTTP 4xx
HTTP 5xx
Application timeout
Database connection error
```

---

## Step 2 — Determine Blast Radius

Check whether the issue affects:

```text
One instance
    ↓
One subnet
    ↓
One AZ
    ↓
Multiple AZs
    ↓
Entire Region
    ↓
External dependency
```

---

## Step 3 — Reproduce

Try reproducing from the affected host.

---

## Step 4 — Isolate the Layer

```text
DNS
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

---

## Step 5 — Compare Healthy vs Unhealthy

If EC2-1 works and EC2-2 doesn't:

```text
EC2-1 = Baseline
EC2-2 = Compare
```

Compare:

* SG
* NACL
* Route Table
* Subnet
* Application version
* Environment variables
* Configuration
* Network settings

---

## Step 6 — Logs and Metrics

Correlate:

* Application logs
* ALB logs
* CloudWatch metrics
* VPC Flow Logs
* RDS metrics
* NAT Gateway metrics

Use the same timestamp.

---

## Step 7 — Identify Root Cause

Do not stop at:

> "Restart fixed it."

Find out why it failed.

---

## Step 8 — Apply Least Disruptive Fix

Examples:

* Correct route
* Correct SG
* Fix NACL
* Restart unhealthy service
* Roll back deployment
* Scale application
* Fix connection pool

---

## Step 9 — Validate

Confirm:

```text
Infrastructure
+
Application
+
Dependency
```

are working.

---

## Step 10 — Prevent Recurrence

Add:

* Monitoring
* Alerting
* Health checks
* Capacity planning
* Automation
* Documentation
* Runbooks

### Senior-Level Interview Answer

> "I don't troubleshoot randomly. I first understand the error and determine the blast radius. Then I reproduce the issue, isolate the failure layer, compare the affected resource with a healthy resource, correlate logs and metrics, identify the root cause, apply the least disruptive fix, validate it and finally implement preventive measures."

---

# 26. Important Troubleshooting Commands

## DNS

```bash
nslookup google.com
```

```bash
dig google.com
```

---

## IP Address

```bash
ip addr
```

---

## Routing

```bash
ip route
```

---

## TCP Connectivity

```bash
nc -vz <hostname> <port>
```

Example:

```bash
nc -vz database.example.com 5432
```

---

## Telnet

```bash
telnet <hostname> <port>
```

Example:

```bash
telnet database.example.com 5432
```

> `nc` is generally preferred when available.

---

## HTTP/HTTPS

```bash
curl -v https://google.com
```

---

## Curl Version

```bash
curl -V
```

> `curl -V` only shows the installed curl version.

---

## Listening Ports

```bash
ss -lntp
```

---

## TLS Handshake

```bash
openssl s_client \
  -connect api.example.com:443 \
  -servername api.example.com
```

---

## PostgreSQL Sessions

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

---

# 27. Senior-Level Interview Answer Pattern

Whenever interviewer gives you a production scenario, follow this structure.

### 1. Understand

> "First, I would understand the exact error and affected component."

### 2. Scope

> "I would determine whether the issue is isolated to one instance, one AZ, multiple AZs or the entire environment."

### 3. Reproduce

> "I would reproduce the issue from the affected workload."

### 4. Isolate

> "Then I would isolate whether the issue is DNS, network, TLS, application or dependency related."

### 5. Compare

> "I would compare the affected resource with a known healthy resource."

### 6. Logs and Metrics

> "I would correlate application logs, infrastructure metrics and network logs using the incident timestamp."

### 7. Root Cause

> "I would identify the actual root cause rather than just applying a workaround."

### 8. Fix

> "I would apply the least disruptive fix possible."

### 9. Validate

> "I would validate the fix from infrastructure, application and dependency perspectives."

### 10. Prevent

> "Finally, I would add monitoring, alerting or configuration improvements to prevent recurrence."

---

# 28. Rapid Fire Questions

## Q27. Is NAT Gateway required for EC2 to RDS communication?

### Answer

No.

If EC2 and RDS are communicating within the same VPC, the VPC local route handles the communication.

```text
VPC CIDR → local
```

NAT Gateway is not required.

---

## Q28. Is NAT Gateway required for private EC2 to access the internet?

### Answer

For IPv4 private-subnet outbound internet access, an outbound NAT mechanism such as NAT Gateway is required if there is no direct internet path.

---

## Q29. Where should NAT Gateway be deployed?

### Answer

Normally in a public subnet.

The public subnet must have:

```text
0.0.0.0/0 → Internet Gateway
```

---

## Q30. Is Security Group stateful?

### Answer

Yes.

---

## Q31. Is NACL stateful?

### Answer

No.

NACL is stateless.

---

## Q32. Does NACL support DENY rules?

### Answer

Yes.

NACL supports:

```text
ALLOW
DENY
```

---

## Q33. Does Security Group support DENY rules?

### Answer

No.

Security Groups only support allow rules.

---

## Q34. Why can NACL cause connection timeout?

### Answer

Because NACL is stateless.

If outbound traffic is allowed but return traffic is denied, the connection can fail.

---

## Q35. What does ALB health check do?

### Answer

ALB periodically checks the configured protocol, port and path of targets.

Only healthy targets receive traffic.

---

## Q36. Does a healthy ALB health check mean the entire application is healthy?

### Answer

No.

The health check may validate only one endpoint.

The application may still have:

* Database issues
* Dependency issues
* Other API failures
* Business logic failures

---

## Q37. What is the benefit of Multi-AZ?

### Answer

High availability and failure isolation.

---

## Q38. What happens if one AZ fails?

### Answer

Traffic can continue through healthy resources in another AZ, provided the application is properly designed for Multi-AZ and sufficient capacity remains.

---

## Q39. Why is ALB used with Multi-AZ?

### Answer

Multi-AZ provides resilience.

ALB provides:

* Traffic distribution
* Health checks
* Single entry point
* Routing

---

## Q40. What IP does a third-party API generally see when private EC2 accesses it through NAT Gateway?

### Answer

The NAT Gateway's public Elastic IP.

---

## Q41. What command checks DNS?

### Answer

```bash
nslookup
```

or:

```bash
dig
```

---

## Q42. What command checks TCP connectivity?

### Answer

```bash
nc -vz <host> <port>
```

---

## Q43. What command is useful for HTTPS troubleshooting?

### Answer

```bash
curl -v https://example.com
```

---

## Q44. What command checks TLS handshake?

### Answer

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com
```

---

## Q45. What can cause RDS "too many connections"?

### Answer

Possible causes:

* Connection pool too large
* Too many application instances
* Connection leaks
* Long-running queries
* Idle connections
* Idle transactions
* Sudden traffic increase
* Database connection limit reached

---

# 29. DAY 1 FINAL REVISION

# 🔥 Production Architecture

```text
                           INTERNET
                               |
                               v
                             WAF
                               |
                               v
                              ALB
                           /       \
                          /         \
                       AZ-1         AZ-2
                        |             |
                     EC2/App       EC2/App
                        |             |
                        +------+------+
                               |
                               v
                         RDS PostgreSQL
                         Private Subnet
```

---

# 🔥 Security Group Chain

```text
Internet
    |
    | TCP 443
    v
ALB-SG
    |
    | Application Port
    v
EC2-SG
    |
    | TCP 5432
    v
RDS-SG
```

---

# 🔥 Private EC2 → Internet

```text
Private EC2
     |
     v
Private Route Table
     |
     | 0.0.0.0/0
     v
NAT Gateway
     |
     v
Public Route Table
     |
     | 0.0.0.0/0
     v
Internet Gateway
     |
     v
Internet
```

---

# 🔥 EC2 → RDS

```text
EC2
 |
 | EC2-SG
 |
 v
VPC Local Route
 |
 v
RDS-SG
 |
 v
RDS
```

NAT Gateway is NOT required for this internal VPC communication.

---

# 🔥 ALB → EC2

```text
Client
  |
  v
WAF
  |
  v
ALB
  |
  v
Target Group
  |
  +--------+
  |        |
  v        v
EC2-1    EC2-2
Healthy  Healthy
```

---

# 🔥 Troubleshooting Flow

```text
        PROBLEM
           |
           v
      Understand
           |
           v
       Determine
        Scope
           |
           v
       Reproduce
           |
           v
          DNS
           |
           v
          TCP
           |
           v
          TLS
           |
           v
          HTTP
           |
           v
      Application
           |
           v
       Dependency
           |
           v
      Logs/Metrics
           |
           v
      Root Cause
           |
           v
          Fix
           |
           v
       Validate
           |
           v
        Prevent
```

---

# 🏆 DAY 1 GOLDEN INTERVIEW POINTS

## 1. Private EC2 Internet

> Private EC2 → Private Route Table → NAT Gateway → IGW → Internet.

---

## 2. NAT Gateway

> NAT Gateway is deployed in a public subnet and normally uses an Elastic IP for IPv4 internet access.

---

## 3. Security Group

> Security Group is stateful and resource-level.

---

## 4. NACL

> NACL is stateless and subnet-level.

---

## 5. NACL Return Traffic

> Because NACL is stateless, return traffic must also be explicitly allowed.

---

## 6. Multi-AZ

> Multi-AZ provides high availability and failure isolation.

---

## 7. ALB

> ALB distributes traffic only to healthy registered targets.

---

## 8. ALB vs Multi-AZ

> Multi-AZ provides resilience; ALB provides traffic distribution and health-aware routing.

---

## 9. EC2 → RDS

> EC2 and RDS in the same VPC communicate through the VPC local route. NAT Gateway is not required.

---

## 10. Third-Party API

> A third-party API may see the NAT Gateway's public IP and may require that IP to be allowlisted.

---

## 11. DNS

> Always isolate DNS problems from network connectivity problems.

---

## 12. TLS

> Use `curl -v` and `openssl s_client` to troubleshoot TLS/SSL issues.

---

## 13. ALB Health Check

> A healthy health-check endpoint doesn't necessarily mean the entire application is healthy.

---

## 14. Database Connections

> Don't look only at the RDS connection count. Correlate it with application instances, connection pools, traffic and slow queries.

---

## 15. Senior Troubleshooting

> Always determine the blast radius first and compare the failing component with a known healthy component.

---

# 🎯 DAY 1 INTERVIEW CHECKLIST

| Topic                      | Status |
| -------------------------- | ------ |
| VPC Architecture           | ✅      |
| Public Subnet              | ✅      |
| Private Subnet             | ✅      |
| Route Tables               | ✅      |
| Internet Gateway           | ✅      |
| NAT Gateway                | ✅      |
| Security Groups            | ✅      |
| NACL                       | ✅      |
| NACL Stateless Behavior    | ✅      |
| DNS Troubleshooting        | ✅      |
| Connectivity Testing       | ✅      |
| ALB                        | ✅      |
| Target Groups              | ✅      |
| ALB Health Checks          | ✅      |
| Multi-AZ                   | ✅      |
| AZ Failure                 | ✅      |
| EC2 → RDS                  | ✅      |
| RDS Security Group         | ✅      |
| RDS Connections            | ✅      |
| Connection Pool            | ✅      |
| Slow Queries               | ✅      |
| Third-Party API            | ✅      |
| NAT IP Allowlisting        | ✅      |
| SSL/TLS                    | ✅      |
| Production Troubleshooting | ✅      |

---

# 🧠 DAY 1 — FINAL SENIOR DEVOPS ANSWER

If interviewer asks:

> "How do you troubleshoot an AWS production networking issue?"

Answer:

> "First, I understand the exact error and determine the blast radius — whether it is isolated to one instance, subnet, AZ, multiple AZs or the complete environment.
>
> Then I reproduce the issue and isolate the failure layer, starting with DNS, followed by routing and TCP connectivity, then TLS, HTTP and finally the application or dependency layer.
>
> On AWS, I validate Security Groups, NACLs, route tables, NAT Gateway, Internet Gateway and VPC Flow Logs as applicable.
>
> I also compare the affected resource with a known healthy resource because configuration drift is a common cause in production.
>
> I correlate application logs, CloudWatch metrics and network logs using the same timestamp.
>
> Once I identify the root cause, I apply the least disruptive fix, validate the recovery from both infrastructure and application perspectives, and finally implement monitoring or preventive controls so the issue doesn't repeat."

---

# 🚀 DAY 1 COMPLETE

## AWS VPC & NETWORKING

### Covered:

* VPC Architecture
* Public/Private Subnets
* Route Tables
* Internet Gateway
* NAT Gateway
* Security Groups
* NACL
* DNS
* ALB
* Target Groups
* Health Checks
* Multi-AZ
* AZ Failure
* EC2 → RDS
* RDS Security Groups
* RDS Connections
* Connection Pools
* Slow Queries
* Third-Party API
* NAT IP Allowlisting
* SSL/TLS
* Production Troubleshooting
* Practical Linux Commands
* Senior-Level Interview Answers

---

# 📌 NEXT TOPIC

# 🚀 DAY 2 — EC2 + ALB + AUTO SCALING

### Topics

* EC2 Deep Dive
* AMI
* Instance Types
* EBS
* ENI
* User Data
* Instance Metadata
* IAM Role
* EC2 Troubleshooting
* ALB Listeners
* Listener Rules
* Target Groups
* Health Checks
* 502 vs 503
* TLS Termination
* Path-Based Routing
* Host-Based Routing
* Auto Scaling Groups
* Launch Templates
* Scaling Policies
* Target Tracking
* Step Scaling
* Multi-AZ ASG
* Instance Replacement
* Zero-Downtime Deployment

---

# 🎯 PREPARATION GOAL

> Understand the concept.
>
> Explain it in simple language.
>
> Troubleshoot it practically.
>
> Design it for production.
>
> Defend your design in front of the interviewer.
>
