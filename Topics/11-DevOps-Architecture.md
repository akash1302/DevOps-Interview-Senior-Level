# Senior DevOps Interview Questions: DevOps Architecture

### Q: How would you design the end-to-end DevOps architecture for a company moving from one big monolith app to microservices?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't split everything at once. I'd start by picking one or two parts of the monolith that change often and are causing the most pain, and pull those out first. Each new service gets its own repo, its own pipeline, and its own database, so teams can deploy it without waiting on anyone else. I'd put all the services behind an API gateway, so the outside world still sees one clean entry point, even though it's many services underneath. I'd also add proper tracing early, since once you have ten services instead of one, finding out where a request actually failed becomes the hard part.

</details>

---

### Q: How do you design a CI/CD platform that works for many different teams, without every team building their own pipeline from scratch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build one shared, reusable pipeline template that covers the common steps — build, test, scan, deploy — and teams just plug their app into it with a small config file. This way, a security fix or a new best practice gets added once, in one place, and every team gets it automatically on their next run, instead of me having to go update forty separate pipelines by hand. I still let teams customize the parts that are actually different for them, like test commands, but the core flow and the guardrails, like requiring a security scan, stay the same for everyone.

</details>

---

### Q: How would you design a multi-region setup so the application keeps working even if one whole AWS region goes down?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd run the app fully in two regions, not just one with a cold backup, since a cold backup takes too long to wake up during a real outage. Data would need to replicate between the two regions continuously, so both sides stay up to date. Traffic would be routed using DNS-based health checks, so if one region starts failing its health check, traffic automatically shifts to the other region within a minute or two, without anyone needing to press a button. The hard part is usually the database — I'd pick one region as the write leader, and make sure the app can handle a short delay in data reaching the second region.

</details>

---

### Q: How do you decide between a monolith and microservices for a new product, from an infrastructure point of view?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For a brand new product, I usually start with a well-organized monolith, not microservices. Microservices add real cost — more pipelines, more monitoring, more network calls that can fail — and for a small team, that cost is bigger than the benefit at the start. I'd only move to microservices once specific parts of the app clearly need to scale differently, or once different teams are stepping on each other trying to deploy the same codebase. Starting simple and splitting later, once the pain is real, is usually cheaper than guessing the right service boundaries too early.

</details>

---

### Q: How do you balance cost against reliability when designing infrastructure for a growing startup?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't build for five nines of reliability on day one, that's expensive and mostly wasted early on. Instead, I match the setup to what the business actually needs right now, and I make sure it's easy to add more reliability later without a full rebuild. That usually means starting with two zones instead of one, using managed services instead of running everything ourselves, and adding real redundancy only around the parts that would actually hurt the business if they went down, like payments. I revisit this every few months, since what's "good enough" changes fast as the company grows.

</details>

---

### Q: Walk me through how you would design a zero-downtime deployment strategy for a customer-facing application.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For zero-downtime deployment, I make sure the new version is running before I send traffic to it.

For example, if I have an application running on **ECS behind an ALB**, and I need to deploy version 2:

* For a normal change, I use a **rolling deployment**. I start new containers with version 2, check their health, and then gradually remove the old version.
* For a high-risk change, I use **blue-green deployment**. Version 1 and version 2 run separately. I test version 2 first, and once everything looks good, I switch the ALB traffic from version 1 to version 2.
* If it's a very critical change, like a payment-related change, I can use **canary deployment**. I send a small amount of traffic, like 5%, to version 2 and monitor errors, latency, and logs. If everything is good, I gradually increase the traffic.

The main idea is: **never send all users to the new version until I know it's healthy, and always keep a quick rollback option.**


</details>

---

### Q: How would you design the infrastructure for a SaaS product that needs to support many customers, where one customer's problem should never affect another customer?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is really about where I draw the isolation line, and that depends on the customer size. For most regular customers, I'd use a shared setup — same database, same app servers — but with strict limits per customer, so one customer sending a flood of traffic can't slow things down for everyone else. For big enterprise customers, especially ones with strict compliance needs, I'd give them their own separate environment entirely, sometimes their own database, sometimes their own whole account. I always design the app so it doesn't care which model it's running in — the customer's data is separated logically from day one, so moving a customer from shared to dedicated later isn't a full rewrite.

</details>

---

### Q: How do you design an observability strategy for a system with many microservices, so an incident doesn't take hours to find the root cause?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build this around three things working together, not just one dashboard. Logs tell you what happened in detail. Metrics tell you the overall health, like error rate and response time. Traces show you the full path of one request as it moves through every service, so you can actually see which one slowed it down or failed. The key part people miss is tying all three together with one shared ID, so when I see an alert on a metric, I can jump straight to the exact logs and trace for that failure, instead of guessing which service is the problem. I also make sure alerts are based on what the user actually feels, like slow page loads, not just raw server stats that might not mean anything to the customer.

</details>

---

### Q: How would you design a platform so that developers can deploy their own apps without needing a DevOps engineer to do it for them every time?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is basically building an internal platform for developers to self-serve. I'd give every team a standard way to describe their app — things like how much memory it needs, what it depends on — and the platform turns that into real infrastructure automatically, using the same safe patterns every time. Developers get a simple way to deploy and see logs, without needing to understand the underlying cloud setup at all. My job shifts from doing every deployment myself to building and maintaining the guardrails — security rules, cost limits, naming standards — that get applied automatically, no matter which team is using the platform.

</details>

---

### Q: How do you design a disaster recovery plan, and how do you decide how much downtime and data loss the business can actually accept?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Before building anything, I get two real numbers from the business — how long can we be down, and how much recent data can we afford to lose if something breaks badly. Those two numbers decide the whole design. If the business can accept an hour of downtime, a simple backup-and-restore plan is enough, and it's cheap. If the business truly cannot go down at all, like a payments system, I need a live backup running at all times, ready to take over immediately, which costs a lot more to run. Most companies I've worked with actually don't need the expensive option for everything — only for the one or two systems that would really hurt the business if they failed.

</details>

---

### Q: The business gives you a hard target of 15 minutes maximum downtime and 5 minutes maximum data loss for the main e-commerce app. How would you actually architect that?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Those numbers are tight enough that a cold standby in a second region won't work — waking up a cold environment alone can eat most of that 15-minute budget. So I'd run the app live in two regions at the same time, not one region with a backup sitting idle. Under normal conditions, one region handles the main traffic, but the second region is already running and ready, not something we're starting from scratch during an incident.

The database is really the hard part here. I'd run a replica of the main database in the second region, continuously catching up, so it's never more than a few minutes behind — that's what actually gets us under the 5-minute data loss target. For things like uploaded files, I'd keep them synced between regions automatically. For fast-changing data, like user sessions, I'd use a database built to replicate across regions in near real time, since a slower replication method wouldn't be fast enough for that piece. For the actual failover, I wouldn't rely on someone noticing and reacting by hand — I'd set up automatic health checks that detect the primary region failing and switch traffic to the second region on their own, since manual failover alone almost never hits a 15-minute target once you include the time for someone to notice, get paged, and actually respond.

</details>

---

### Q: How do you approach designing infrastructure as code so that a growing platform team doesn't end up with messy, duplicated code everywhere?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I set up a small set of shared, well-tested building blocks — like a standard way to create a database, or a standard way to create a service — and every team builds on top of those, instead of writing their own from scratch. This keeps things consistent, and when I need to fix a security issue or a bad default, I fix it once in the shared building block, and every team gets the fix automatically the next time they update. I also make sure changes to those shared building blocks go through real review, since a small mistake there can affect every team at once, not just one.

</details>

---

### Q: How would you design a system so it can handle a sudden 10x jump in traffic, like from a big marketing campaign, without falling over?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The first thing I check is whether the app can even scale out by just adding more copies of itself — if it can, this becomes mostly about making sure auto scaling is actually tuned correctly, and tested before the real event, not just configured and hoped for. The usual real bottleneck isn't the app servers, it's the database, since you can't just add more copies of it the same way. So I look at caching heavily-read data, and making sure the database isn't doing more work than it needs to. For a known event, like a planned sale, I'll also scale things up ahead of time manually, instead of trusting auto scaling alone to react fast enough for a sudden spike.

</details>

---

### Q: How do you decide when to use Kubernetes versus a simpler option like serverless functions or plain virtual machines, for a new platform?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't default to Kubernetes just because it's popular. If the workload is small, or the team is small, serverless functions are often simpler and cheaper, since there's no cluster to manage at all. Kubernetes starts making sense once I have many services, need fine control over how they're deployed, or need to run the same setup consistently across different environments. Plain virtual machines still have a place too, usually for something old that just wasn't built to run in a container. The real question I ask is what the team can actually operate well, not what looks the most advanced on paper.

</details>

---
