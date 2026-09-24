# Senior DevOps Interview Questions: Networking

## Q1. How do VPC Subnets, Route Tables, and CIDR blocks dictate network traffic flow in AWS?

### Answer
An AWS Virtual Private Cloud (VPC) defines an isolated virtual network bound to an IPv4 CIDR block (e.g., `10.0.0.0/16`). Subnets segment this CIDR block into smaller IP ranges allocated to specific Availability Zones. Subnet behavior depends on Route Table attachments: a **Public Subnet** has a route table entry directing default outbound traffic (`0.0.0.0/0`) to an Internet Gateway (IGW). A **Private Subnet** lacks an IGW route, directing `0.0.0.0/0` traffic instead to a NAT Gateway or retaining local-only VPC traffic routing (`10.0.0.0/16` -> `local`).

### Interview Answer
"Traffic routing in an AWS VPC is governed by route table entries associated with subnets. When I create a VPC with a `/16` CIDR, I divide it into `/24` subnets across multiple AZs. For public subnets, the route table contains `0.0.0.0/0 -> igw-xxxx`, allowing instances with public IPs to communicate with the internet. Private subnet route tables direct `0.0.0.0/0 -> nat-xxxx`, ensuring outbound-only connectivity, while internal VPC traffic is routed locally across all subnets."

### Practical Example
VPC Route Table Configuration:
* **VPC CIDR**: `10.0.0.0/16`
* **Public Subnet Route Table (`10.0.1.0/24`)**:
  * `10.0.0.0/16` -> `local`
  * `0.0.0.0/0` -> `igw-0123456789`
* **Private Subnet Route Table (`10.0.10.0/24`)**:
  * `10.0.0.0/16` -> `local`
  * `0.0.0.0/0` -> `nat-0987654321`

### Follow-up Questions
* How many IP addresses does AWS reserve in every created subnet CIDR block?
* What happens if two VPCs with overlapping CIDR blocks attempt to establish a VPC Peering connection?
* What is the difference between a main route table and a custom route table in AWS VPC?

### Key Points
* Subnets divide VPC CIDR blocks and are explicitly bound to single Availability Zones.
* Public subnets route default traffic (`0.0.0.0/0`) directly to an Internet Gateway.
* Private subnets route outbound traffic through a NAT Gateway and block direct inbound connections.

---

## Q2. What is the fundamental difference between Security Groups and Network ACLs (NACLs) in AWS VPC security?

### Answer
Security Groups and Network ACLs provide two complementary layers of firewall defense in AWS. **Security Groups** operate at the individual instance/ENI level, are **stateful** (return traffic is automatically allowed regardless of inbound rules), evaluate all rules before deciding, and support allow rules only. **Network ACLs (NACLs)** operate at the subnet boundary level, are **stateless** (outbound return traffic must be explicitly allowed), process rules in strict numerical order, and support both explicit ALLOW and DENY rules.

### Interview Answer
"Security Groups act as stateful firewalls attached to EC2 instances or ENIs. Since they are stateful, if an inbound request is allowed on port 443, the outbound response is automatically permitted. NACLs act as a stateless secondary firewall at the subnet boundary. Because NACLs are stateless, you must configure both inbound rules and outbound ephemeral port ranges. NACLs are ideal when you need explicit DENY rules to block specific malicious IP ranges across an entire subnet."

### Practical Example
Comparison matrix in action:
* Blocking a DDoS attacker IP (`192.0.2.45`): Added as an explicit **DENY** rule (Rule #100) in the **NACL** at the subnet border, dropping traffic before it reaches instances.
* Web server firewall: **Security Group** allows inbound TCP `443` from `0.0.0.0/0`. Outbound responses pass automatically due to stateful tracking.

### Follow-up Questions
* Why do NACLs require allowing outbound ephemeral ports (ports 1024–65535) for traffic to function?
* What is the default rule configuration for a newly created custom NACL versus the default VPC NACL?
* How do Security Group rule references (referencing another SG ID) simplify multi-tier security?

### Key Points
* Security Groups are stateful firewalls operating at the instance/ENI level (ALLOW rules only).
* NACLs are stateless firewalls operating at the subnet border (supports ALLOW and DENY rules).
* Security Groups evaluate all rules; NACLs process rules sequentially by rule number.

---

## Q3. How do NAT Gateways and Internet Gateways differ in facilitating internet access for AWS VPC workloads?

### Answer
An **Internet Gateway (IGW)** is a horizontally scaled, highly available VPC component that enables direct two-way communication between instances in public subnets and the public internet. It performs 1-to-1 IPv4 NAT mapping for instances with public IP addresses. A **NAT Gateway** is an outbound-only managed network translation service deployed in a public subnet. It enables instances in private subnets (lacking public IPs) to connect outbound to the internet or AWS services while preventing the internet from initiating inbound connections.

### Interview Answer
"An Internet Gateway enables bi-directional traffic for public subnets; instances must have public IPs to use it. A NAT Gateway provides one-way outbound internet access for private workloads. The NAT Gateway lives in a public subnet, has an Elastic IP, and translates private instance traffic so app servers can fetch software patches or external API data without exposing private instances to inbound internet threats."

### Practical Example
* **Public Web Server**: Uses Internet Gateway. Internet users connect inbound to port 443 via Public IP.
* **Private App Server**: Uses NAT Gateway. App server in private subnet initiates outbound HTTP call to external payment API (`api.stripe.com`). NAT Gateway translates private IP (`10.0.10.15`) to Elastic IP (`52.1.2.3`), relays request, and routes response back. Inbound direct connection attempts to `52.1.2.3` are dropped.

### Follow-up Questions
* Why must a NAT Gateway be physically deployed inside a Public Subnet?
* How do you configure highly available NAT Gateways across multiple Availability Zones?
* What are the cost components associated with AWS NAT Gateways?

### Key Points
* Internet Gateways enable bi-directional inbound and outbound traffic for public subnets.
* NAT Gateways enable outbound-only internet connectivity for private subnets.
* NAT Gateways require Elastic IPs and must reside in a public subnet with an IGW route.

---

## Q4. How do AWS Site-to-Site VPN and AWS Direct Connect differ for connecting on-premises data centers to AWS?

### Answer
**AWS Site-to-Site VPN** establishes an encrypted IPsec tunnel over the public internet between an on-premises VPN router and an AWS Virtual Private Gateway or Transit Gateway. It is fast to provision, cost-effective, but susceptible to public internet latency fluctuations and bandwidth limits. **AWS Direct Connect** establishes a dedicated, private physical fiber link from an on-premises data center to an AWS Direct Connect location. Direct Connect bypasses the internet entirely, providing consistent network performance, lower latency, higher bandwidth (1Gbps–100Gbps), and reduced data egress costs.

### Interview Answer
"AWS Site-to-Site VPN is an encrypted IPsec connection established over the public internet—it's cheap and quick to deploy, but subject to internet jitter and bandwidth caps. Direct Connect provides a dedicated physical fiber connection from on-prem to AWS. It bypasses the public internet completely, offering ultra-low latency, stable throughput, and lower data egress costs. For enterprise hybrid setups, I use Direct Connect for primary traffic and overlay a VPN for IPsec encryption and failover redundancy."

### Practical Example
Hybrid Enterprise Architecture:
* **Primary Connection**: AWS Direct Connect 10Gbps dedicated link handling core application database replication and hybrid VM migration.
* **Backup Connection**: AWS Site-to-Site IPsec VPN configured as an automated failover path over the public internet using BGP routing if Direct Connect physical fiber suffers an outage.

### Follow-up Questions
* How does BGP (Border Gateway Protocol) manage dynamic routing and failover between Direct Connect and VPN?
* Can you encrypt traffic flowing over an AWS Direct Connect link?
* What is the purpose of a Direct Connect Gateway when connecting to multiple AWS regions?

### Key Points
* Site-to-Site VPN uses IPsec encryption over the public internet (variable performance).
* Direct Connect provides dedicated physical network links (consistent low latency, high bandwidth).
* Combining Direct Connect and VPN delivers both dedicated high performance and IPsec encryption.


---
