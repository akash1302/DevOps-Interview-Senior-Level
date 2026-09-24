# Senior DevOps Interview Questions: Networking

### Q: How do VPC Subnets, Route Tables, and CIDR blocks dictate network traffic flow in AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

What actually makes a subnet public or private is the route table attached to it, not the subnet itself. A public subnet's route table sends internet traffic straight to an Internet Gateway. A private subnet's route table sends it to a NAT Gateway instead, so it can go out, but nothing can come in directly. Every subnet also has a local route, so traffic between subnets in the same VPC stays inside and never needs to go through either gateway.

</details>

---

### Q: What is the fundamental difference between Security Groups and Network ACLs (NACLs) in AWS VPC security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Security Group sits on the server itself. If you allow traffic in, the reply is automatically allowed back out, no extra setup needed. A NACL sits at the subnet level, and it's stricter — you have to allow both directions yourself, or things just stop working. The other big difference is a Security Group can only allow traffic, it can't block it. A NACL can do both, and it checks rules in a strict order, which is why I use a NACL when I need to block one specific bad address across a whole subnet.

</details>

---

### Q: How do NAT Gateways and Internet Gateways differ in facilitating internet access for AWS VPC workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An Internet Gateway allows traffic both ways, for public servers with a real public address. A NAT Gateway only goes one way — it lets private servers reach out to the internet, but nothing from outside can ever start a connection back in through it. So a private app server can call an outside service just fine through the NAT Gateway, but if someone tries to connect to that server directly, it gets blocked. One thing to know — the NAT Gateway itself always has to sit in a public subnet, even though it's serving private servers.

</details>

---

### Q: How do AWS Site-to-Site VPN and AWS Direct Connect differ for connecting on-premises data centers to AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Site-to-Site VPN is an encrypted connection that still travels over the regular internet. It's quick to set up and cheap, but performance depends on the internet that day. Direct Connect is a real physical cable straight from your building to AWS. It skips the public internet completely, so you get steady, low delay and much higher speed. For a real setup, I'd use Direct Connect as the main path, and keep a VPN running as a backup, so if the physical line goes down, traffic switches over automatically.

</details>

---
