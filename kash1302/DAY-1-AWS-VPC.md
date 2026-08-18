# 🚀 DAY 1 — AWS VPC & NETWORKING

## Senior DevOps / AWS Engineer — Complete Interview Q&A

**Target:** 30+ LPA Senior DevOps / AWS Engineer  
**Focus:** Understanding + Practical Troubleshooting + Interview-ready Answers

---

# 📚 Table of Contents

1. [Production VPC Architecture](#1-production-vpc-architecture)
2. [What is a VPC](#2-what-is-a-vpc)
3. [Public vs Private Subnet](#3-public-vs-private-subnet)
4. [Private EC2 Internet Access](#4-how-does-private-ec2-access-internet)
5. [Private EC2 Internet Troubleshooting](#5-private-ec2-cannot-access-internet--troubleshooting)
6. [Security Group vs NACL](#6-security-group-vs-nacl)
7. [Inbound SG Rule for Internet Access](#7-does-ec2-need-inbound-sg-rule-for-internet-access)
8. [Ephemeral Ports](#8-ephemeral-ports)
9. [Multi-AZ](#9-why-multi-az)
10. [Why ALB](#10-why-do-we-need-alb-if-ecs-are-in-multiple-azs)
11. [ALB Target Unhealthy](#11-alb-target-unhealthy)
12. [ALB 502 Bad Gateway](#12-alb-target-healthy-but-users-get-502)
13. [ALB 502 vs 503](#13-alb-502-vs-503)
14. [EC2 to RDS Connectivity](#14-ec2-in-az-2-cannot-reach-rds)
15. [AZ Failure Scenario](#15-az-failure-scenario)
16. [Sticky Sessions](#16-should-i-enable-sticky-sessions)
17. [RDS Too Many Connections](#17-rds-too-many-connections)
18. [Connection Pool](#18-connection-pool)
19. [PostgreSQL Connection Investigation](#19-postgresql-connection-investigation)
20. [Third-Party API Troubleshooting](#20-third-party-api-troubleshooting)
21. [Third-Party API IP Allowlisting](#21-third-party-api-ip-allowlisting)
22. [Curl Works but Application Doesn't](#22-curl-works-but-application-doesnt)
23. [Troubleshooting Framework](#23-troubleshooting-framework)
24. [Important Commands](#24-important-commands)
25. [Senior Interview Answer Pattern](#25-senior-interview-answer-pattern)
26. [Day 1 Final Assessment](#day-1--final-assessment)
27. [Golden Points](#day-1--golden-points-to-remember)

---

# 1. Production VPC Architecture

### ❓ Interviewer

**Design a secure and highly available VPC for a production application.**

### ✅ Answer

I would create a custom VPC across at least two Availability Zones.

I would use:

- Public subnets → Internet-facing ALB
- Private application subnets → EC2/ECS/EKS workloads
- Private database subnets → RDS
- NAT Gateway → outbound internet access for private workloads
- Internet Gateway → internet connectivity for public resources
- WAF → protection in front of the ALB

### Architecture

```text
                         Internet
                            |
                           WAF
                            |
                           ALB
                    /               \
                 AZ-1               AZ-2
                  |                   |
              Private App          Private App
                EC2                   EC2
                  \                   /
                   \                 /
                    ---- RDS --------
                       PostgreSQL
