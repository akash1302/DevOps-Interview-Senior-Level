# Senior DevOps Interview Questions: Networking

### Q: How do VPC Subnets, Route Tables, and CIDR blocks dictate network traffic flow in AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Traffic flow in a VPC really comes down to what's sitting in each subnet's route table, not the subnet itself. When I create a VPC with a `/16` CIDR, like `10.0.0.0/16`, I carve it into smaller `/24` subnets spread across multiple AZs. What actually makes a subnet "public" or "private" is purely the route table attached to it — nothing else.

A public subnet's route table has `0.0.0.0/0 -> igw-xxxx`, sending default traffic straight to an Internet Gateway. A private subnet's route table instead sends `0.0.0.0/0 -> nat-xxxx`, so outbound traffic goes through a NAT Gateway and nothing can initiate a connection inbound. Both route tables also carry a local route for the VPC's own CIDR, `10.0.0.0/16 -> local`, so traffic between subnets inside the VPC never has to leave through either gateway at all. It's worth knowing AWS always reserves a handful of IPs in every subnet CIDR for its own networking use, so your actual usable address count is a little less than the raw block size suggests.

</details>

---

### Q: What is the fundamental difference between Security Groups and Network ACLs (NACLs) in AWS VPC security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Security Groups are stateful firewalls attached at the instance or ENI level — if you allow inbound on port 443, the response traffic is automatically allowed back out, you don't have to configure that separately. NACLs sit at the subnet boundary instead, and they're stateless, which means you have to explicitly allow both the inbound request and the outbound ephemeral port range for the response, or connections will silently hang.

The other big difference is that Security Groups only support allow rules, evaluated as a whole, while NACLs support explicit deny rules and are processed strictly in numbered order. That's exactly why I reach for a NACL when I need to hard-block a specific malicious IP across an entire subnet — say, adding attacker IP `192.0.2.45` as an explicit deny at rule number 100 — since a Security Group has no way to express a deny rule at all. For the normal web server case, though, a Security Group allowing inbound `443` from anywhere, with the stateful response handled automatically, is all that's needed.

</details>

---

### Q: How do NAT Gateways and Internet Gateways differ in facilitating internet access for AWS VPC workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An Internet Gateway enables two-way traffic for public subnets — instances need a public IP to use it, and the IGW does a straightforward one-to-one NAT mapping so inbound connections can actually reach them. A NAT Gateway is a completely different direction — it's outbound-only, sitting in a public subnet with its own Elastic IP, letting private instances reach out to the internet without ever being reachable from it.

So a public web server takes inbound connections on 443 straight through the IGW using its public IP. A private app server calling an external payment API instead goes out through the NAT Gateway, which translates its private IP, say `10.0.10.15`, to its own Elastic IP, relays the request, and routes the response back — but if someone outside tries to connect inbound to that same Elastic IP, it's dropped, because NAT Gateway never accepts unsolicited inbound traffic. One thing that trips people up — the NAT Gateway has to physically live in a *public* subnet with its own IGW route, even though the traffic it's serving is coming from private subnets.

</details>

---

### Q: How do AWS Site-to-Site VPN and AWS Direct Connect differ for connecting on-premises data centers to AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Site-to-Site VPN is an encrypted IPsec tunnel running over the regular public internet, connecting an on-prem router to a Virtual Private Gateway or Transit Gateway. It's fast to set up and cheap, but you're at the mercy of internet jitter and bandwidth limits, since it's still riding on the public internet underneath the encryption.

Direct Connect is a completely different thing — it's a dedicated physical fiber link from the on-prem data center straight to an AWS Direct Connect location, bypassing the public internet entirely. That gets you consistent low latency, much higher bandwidth, anywhere from 1Gbps up to 100Gbps, and lower data egress costs on top of it. For a real hybrid setup, I'd run Direct Connect as the primary path for things like database replication or VM migration, and keep a Site-to-Site VPN running alongside it as an automated BGP failover — so if the physical fiber link ever goes down, traffic fails over to the VPN path instead of just dropping.

</details>

---
