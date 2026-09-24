# Senior DevOps Interview Questions: Terraform

### Q: How do you import existing, manually created cloud infrastructure into Terraform state management?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is `terraform import`, but the big catch is it only updates the state file — it does not write any code for you. So my real steps are: first I write the matching resource block in code myself, then I run the import against that resource, and then I run `plan` right away to check for any difference. I keep tweaking the code until `plan` shows zero changes. Skipping that last check is how people end up with Terraform trying to delete something it just imported.

</details>

---

### Q: How do you structure Terraform configurations to manage multiple environments (Dev, Staging, Prod) without code duplication?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build one shared module for each piece of infrastructure, like a VPC module, and then each environment gets its own small folder that calls that module with its own settings. So dev has its own folder pointing to its own state file, and prod has a completely separate folder pointing to a completely separate state file. I prefer this over Terraform Workspaces because the isolation is real and physical — there's no shared backend where picking the wrong workspace by mistake could touch production.

</details>

---

### Q: How do you configure a secure Terraform remote backend, and how do you recover if a state file is accidentally deleted?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always use an S3 bucket with encryption and versioning turned on, plus a DynamoDB table for locking, so two people can't run `apply` at the same time and step on each other. If someone deletes the state file by accident, I just pull the last good version straight from S3's version history — usually a two-minute fix. That's exactly why I treat versioning as required, not optional. If there's genuinely no backup at all, it's a much harder job — I'd have to check every real resource in the account and rebuild the state manually with `terraform import`, one resource at a time.

</details>

---

### Q: How do Local Exec and Remote Exec provisioners function in Terraform, and when should you use them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I treat these as a last resort. `local-exec` runs a command on my own machine after a resource is created, like sending a notification. `remote-exec` connects into the new resource itself, over SSH, and runs commands there directly. The problem is both of these sit outside Terraform's normal tracking — they can fail quietly on a re-run, and Terraform has no real way to know if they actually worked. So instead, I usually push that setup work into things like a startup script or a pre-built image, since those are safer and easier to repeat reliably.

</details>

---

### Q: How do you manage sensitive credentials and database passwords securely in Terraform configurations?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never go directly into code or a `.tfvars` file that's tracked in Git. Anything sensitive gets marked with `sensitive = true`, so Terraform hides it from the terminal and from CI logs. For something like a database password, I pull it in at run time from AWS Secrets Manager instead of passing it as plain text. The big catch people miss — `sensitive = true` only hides the value from the screen, it does **not** remove it from the state file. The state file still has the real value in it, so the S3 bucket holding that state also needs to be locked down and encrypted.

</details>

---
