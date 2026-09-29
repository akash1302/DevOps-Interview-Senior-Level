# Senior DevOps Interview Questions: Networking

### Q: How do VPC Subnets, Route Tables, and CIDR blocks dictate network traffic flow in AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

What actually makes a subnet public or private is the route table attached to it, not the subnet itself.

A public subnet's route table sends internet traffic straight to an Internet Gateway. A private subnet's route table sends it to a NAT Gateway instead, so it can go out, but nothing can come in directly.

Every subnet also has a local route, so traffic between subnets in the same VPC stays inside and never needs either gateway.

</details>

---

### Q: What is the fundamental difference between Security Groups and Network ACLs (NACLs) in AWS VPC security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Security Group sits on the server itself. If you allow traffic in, the reply is automatically allowed back out, no extra setup needed.

A NACL sits at the subnet level and is stricter — you have to allow both directions yourself, or things just stop working. The other big difference is a Security Group can only allow traffic, it can't block it. A NACL can do both and checks rules in order, which is why I use a NACL when I need to block one specific bad address across a whole subnet.

</details>

---

### Q: How do NAT Gateways and Internet Gateways differ in facilitating internet access for AWS VPC workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An Internet Gateway allows traffic both ways, for public servers with a real public address.

A NAT Gateway only goes one way — it lets private servers reach out to the internet, but nothing from outside can ever start a connection back in through it. So a private app server can call an outside service fine through the NAT Gateway, but a direct connection attempt to it gets blocked.

One thing to know — the NAT Gateway itself always has to sit in a public subnet, even though it's serving private servers.

</details>

---

### Q: How do AWS Site-to-Site VPN and AWS Direct Connect differ for connecting on-premises data centers to AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Site-to-Site VPN is an encrypted connection that still travels over the regular internet — quick and cheap, but performance depends on the internet that day.

Direct Connect is a real physical link straight from your building to AWS, so it skips the public internet completely and gives steady, low delay and much higher speed.

For a real setup, I'd use Direct Connect as the main path and keep a VPN as a backup, so traffic switches over automatically if the physical line ever goes down.

</details>

---

### Q: What actually happens, step by step, when you type a website address into a browser and hit enter?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, the computer needs to turn that website name into a real address, so it asks a DNS server to look it up. Once it has the address, it opens a connection to the server, usually a secure one, which involves a quick back-and-forth to agree on encryption first.

Then the browser sends the actual request, the server sends back the response, and the browser starts rendering it.

If any one of these steps is slow, the whole page feels slow, so when I'm troubleshooting a slow website, I check each step separately instead of guessing which one it is.

</details>

---

### Q: What's the difference between an Application Load Balancer and a Network Load Balancer, and when do you use each?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An Application Load Balancer works at the web traffic level — it can look at the actual request, like the URL path, and route different paths to different backend services. That's a good fit for normal web apps and APIs.

A Network Load Balancer works at a lower level — it just forwards raw connections without looking inside them, which makes it extremely fast and able to handle a huge number of connections.

I'd use a Network Load Balancer for something like a database proxy or a service that needs a fixed IP, and an Application Load Balancer for basically every normal web app.

</details>

---

### Q: How do you troubleshoot when two servers can't talk to each other, and you're not sure if it's a network problem or an app problem?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I test the raw connection itself first, separately from the app. If I can open a basic connection to the right port, the network path is fine, and the problem is actually inside the app — maybe it's not listening correctly, or it's rejecting the request for its own reasons.

If I can't even open a basic connection, it's a real network issue, and I check firewall rules on both ends first, then the routing in between.

Testing the raw connection first, before blaming either side, saves a lot of time chasing the wrong thing.

</details>

---

### Q: Service A can't connect to Service B, both are ECS tasks in different private subnets, and both are confirmed running. How do you methodically find the problem?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In AWS, this almost always comes down to Security Groups, NACLs, route tables, or the app itself, so I check them in that order, starting with the most common culprit.

First, Service B's Security Group needs an inbound rule allowing traffic from Service A's Security Group on the right port — I always reference the security group directly, not an IP range, since IPs change but that reference doesn't. Then I check Service A's outbound rule, since people often only check inbound.

If both look fine, I check the NACLs on both subnets next. And if I'm still stuck after all that, I'll use the VPC Reachability Analyzer — you give it the source and destination, and it walks the actual path and tells you exactly which rule is blocking it.

</details>

---
