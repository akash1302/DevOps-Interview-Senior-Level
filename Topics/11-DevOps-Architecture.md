# Senior DevOps Interview Questions: DevOps Architecture

### Q: How would you design the end-to-end DevOps architecture for a company moving from one big monolith app to microservices?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't split everything at once. I'd start by picking one or two parts of the monolith that change often and cause the most pain, and pull those out first.

Each new service gets its own repo, its own pipeline, and its own database, so teams can deploy it without waiting on anyone else. All the services sit behind an API gateway, so the outside world still sees one clean entry point.

I'd also add proper tracing early, since once you have ten services instead of one, finding out where a request actually failed becomes the hard part.

</details>

---

### Q: How do you design a CI/CD platform that works for many different teams, without every team building their own pipeline from scratch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build one shared, reusable pipeline template that covers the common steps — build, test, scan, deploy — and teams just plug their app into it with a small config file.

That way, a security fix or a new best practice gets added once, in one place, and every team gets it automatically on their next run, instead of me updating forty pipelines by hand.

I still let teams customize the parts that are genuinely different for them, like test commands, but the core flow and guardrails stay the same for everyone.

</details>

---

### Q: How would you design a multi-region setup so the application keeps working even if one whole AWS region goes down?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd run the app fully in two regions, not just one with a cold backup, since a cold backup takes too long to wake up during a real outage.

Data replicates between the two regions continuously, so both sides stay up to date, and traffic gets routed using DNS health checks — if one region starts failing, traffic shifts to the other automatically within a minute or two.

The hard part is usually the database. I'd pick one region as the write leader and make sure the app can handle a short delay before data reaches the second region.

</details>

---

### Q: How do you decide between a monolith and microservices for a new product, from an infrastructure point of view?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For a brand new product, I usually start with a well-organized monolith, not microservices. Microservices add real cost — more pipelines, more monitoring, more network calls that can fail — and for a small team, that cost is bigger than the benefit early on.

I'd only move to microservices once specific parts of the app clearly need to scale differently, or once teams are genuinely stepping on each other deploying the same codebase.

Starting simple and splitting later, once the pain is real, is usually cheaper than guessing the right service boundaries too early.

</details>

---

### Q: How do you balance cost against reliability when designing infrastructure for a growing startup?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't build for maximum reliability on day one, that's expensive and mostly wasted early. I match the setup to what the business actually needs right now, and make it easy to add more reliability later without a full rebuild.

That usually means starting with two availability zones instead of one, using managed services instead of running everything ourselves, and adding real redundancy only around the parts that would actually hurt the business, like payments.

I revisit this every few months, since what's "good enough" changes fast as the company grows.

</details>

---

### Q: Walk me through how you would design a zero-downtime deployment strategy for a customer-facing application.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The key idea is making sure the new version is running and healthy before it gets any real traffic.

For a normal change, I use a rolling deployment — start new containers, check their health, then gradually remove the old version. For a higher-risk change, I use blue-green — both versions run side by side, I test the new one, then switch traffic over in one move so I can switch back instantly if needed.

For something really critical, like a payment change, I'll use canary instead — send a small slice of traffic, like 5%, watch errors and latency, and only increase it once I trust it. Either way, I never send everyone to the new version until I know it's healthy.

</details>

---

### Q: How would you design the infrastructure for a SaaS product that needs to support many customers, where one customer's problem should never affect another customer?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This comes down to where I draw the isolation line, and that depends on customer size.

For most regular customers, I'd use a shared setup — same database, same app servers — but with strict limits per customer, so one customer's traffic spike can't slow things down for everyone else. For big enterprise customers with real compliance needs, I'd give them their own dedicated environment.

I always design the app so it doesn't care which model it's running in — data is separated logically from day one, so moving a customer from shared to dedicated later isn't a full rewrite.

</details>

---

### Q: How do you design an observability strategy for a system with many microservices, so an incident doesn't take hours to find the root cause?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build this around three things working together, not just one dashboard. Logs tell you what happened in detail. Metrics tell you the overall health. Traces show you the full path of one request as it moves through every service.

The part people miss is tying all three together with one shared ID, so an alert on a metric lets me jump straight to the trace and the logs for that exact failure, instead of guessing which service is the problem.

I also make sure alerts are based on what the user actually feels, like slow page loads, not just raw server stats.

</details>

---

### Q: How would you design a platform so that developers can deploy their own apps without needing a DevOps engineer to do it for them every time?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is really about building a self-serve platform. Every team gets a standard way to describe their app — how much memory it needs, what it depends on — and the platform turns that into real infrastructure automatically, using the same safe patterns every time.

Developers get a simple way to deploy and see logs, without needing to understand the cloud setup underneath.

My job shifts from doing every deployment myself to building and maintaining the guardrails — security rules, cost limits, naming standards — that get applied automatically no matter which team is using it.

</details>

---

### Q: How do you design a disaster recovery plan, and how do you decide how much downtime and data loss the business can actually accept?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Before designing anything, I get two real numbers from the business — how long can we be down, and how much recent data can we afford to lose.

Those numbers decide the whole design. If an hour of downtime is fine, a simple backup-and-restore plan is enough, and it's cheap. If the business truly can't go down at all, like a payments system, I need a live standby ready to take over immediately, which costs a lot more.

In my experience, most companies don't need the expensive option for everything — only for the one or two systems that would really hurt the business if they failed.

</details>

---

### Q: The business gives you a hard target of 15 minutes maximum downtime and 5 minutes maximum data loss for the main e-commerce app. How would you actually architect that?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Those numbers are tight enough that a cold standby won't work — just waking up a cold environment can eat most of that 15-minute budget.

So I'd run the app live in two regions at once. Under normal conditions, one region handles the main traffic, but the second is already running and ready. The database is the hard part — I'd run a continuously-replicating copy in the second region, kept close enough behind to stay under that 5-minute data loss target.

For failover, I wouldn't rely on someone noticing and reacting by hand — automatic health checks detect the outage and switch traffic over, since a manual process almost never hits a 15-minute target once you count the time to notice and respond.

</details>

---

### Q: How do you approach designing infrastructure as code so that a growing platform team doesn't end up with messy, duplicated code everywhere?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I set up a small set of shared, well-tested modules — a standard way to create a database, a standard way to create a service — and every team builds on top of those instead of writing their own from scratch.

That keeps things consistent, and when I need to fix a security issue or a bad default, I fix it once in the shared module, and every team gets it automatically next time they update.

I'm also strict about review on those shared modules, since a small mistake there can affect every team at once.

</details>

---

### Q: How would you design a system so it can handle a sudden 10x jump in traffic, like from a big marketing campaign, without falling over?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I check whether the app can scale out just by adding more copies of itself. If it can, this becomes mostly about making sure autoscaling is actually tuned and tested ahead of time, not just configured and hoped for.

The real bottleneck is usually the database, since you can't just add more copies of it the same way. So I look at caching heavily-read data, and making sure the database isn't doing more work than it needs to.

For a known event, like a planned sale, I'll also scale things up ahead of time manually, instead of trusting autoscaling alone to react fast enough.

</details>

---

### Q: How do you decide when to use Kubernetes versus a simpler option like serverless functions or plain virtual machines, for a new platform?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't default to Kubernetes just because it's popular.

If the workload or the team is small, serverless functions are often simpler and cheaper, since there's no cluster to manage. Kubernetes makes sense once there are many services, or I need real control over deployment behavior across environments.

Plain virtual machines still have a place too, usually for something older that wasn't built to run in a container. The real question I ask is what the team can actually operate well, not what looks the most advanced on paper.

</details>

---
