# Senior DevOps Interview Questions: Networking

### Q: How do VPC Subnets, Route Tables, and CIDR blocks dictate network traffic flow in AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

What actually decides if a subnet is public or private isn't the subnet itself, it's the route table attached to it. A public subnet's route table sends default traffic straight to an Internet Gateway. A private subnet's route table instead sends it to a NAT Gateway, so it can go out but nothing can come in directly. Every subnet also keeps a local route for talking to other subnets in the same VPC, so internal traffic never has to leave through either gateway at all.

</details>

---

### Q: What is the fundamental difference between Security Groups and Network ACLs (NACLs) in AWS VPC security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Security Groups sit on the instance itself and are stateful — if you allow traffic in, the response is automatically allowed back out, no extra work needed. NACLs sit at the subnet level and are stateless, meaning you have to allow both directions yourself, or things will just hang. The other big difference — Security Groups can only allow traffic, they can't block it. NACLs can do both, and they process rules in strict numbered order, which is why I use a NACL when I need to hard-block a specific bad IP across a whole subnet.

</details>

---

### Q: How do NAT Gateways and Internet Gateways differ in facilitating internet access for AWS VPC workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An Internet Gateway allows traffic both ways, for public subnets with a real public IP. A NAT Gateway is one-way only — it lets private instances reach out to the internet, but nothing from outside can ever start a connection back in through it. So a private app server can call an external API just fine through the NAT Gateway, but if someone tries to connect to it directly from outside, it just gets dropped. One thing worth knowing — the NAT Gateway itself always has to sit in a public subnet, even though it's serving private instances.

</details>

---

### Q: How do AWS Site-to-Site VPN and AWS Direct Connect differ for connecting on-premises data centers to AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Site-to-Site VPN is an encrypted connection that still runs over the regular public internet — quick to set up, cheap, but you're stuck with whatever performance the internet gives you that day. Direct Connect is a real physical cable straight from your building to AWS, so it skips the public internet completely and gives you steady, low latency and much higher bandwidth. For a real setup, I'd run Direct Connect as the main path and keep the VPN running alongside it as a backup, so if the physical line ever goes down, traffic just fails over automatically.

</details>

---
