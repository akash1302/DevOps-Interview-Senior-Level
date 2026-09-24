# Senior DevOps Interview Questions: Terraform

### Q: How do you import existing, manually created cloud infrastructure into Terraform state management?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is `terraform import`. But it only updates the state file. It does not write the code for you. So first, I write the matching resource block in code myself. Then I run the import. Then I run `plan` right away to check if anything looks different. I keep fixing the code until `plan` shows no changes at all. If I skip that last step, Terraform might try to delete the thing I just imported.

</details>

---

### Q: How do you structure Terraform configurations to manage multiple environments (Dev, Staging, Prod) without code duplication?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I write one shared piece of code, called a module, for each thing I need, like a VPC. Then each environment — dev, staging, prod — has its own small folder that calls that same module with its own settings. Each environment also has its own separate state file. I like this better than Terraform Workspaces, because the separation is real. There is no shared file where picking the wrong option by mistake could touch production.

</details>

---

### Q: How do you configure a secure Terraform remote backend, and how do you recover if a state file is accidentally deleted?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I store the state file in an S3 bucket, with encryption and versioning turned on. I also use a lock, so two people can't run `apply` at the same time and break each other's work. If someone deletes the state file by mistake, I just restore the last good version from S3's history. That usually takes two minutes. This is exactly why versioning matters — without it, if the file is lost, I'd have to check every real resource by hand and rebuild the state one piece at a time.

</details>

---

### Q: How do Local Exec and Remote Exec provisioners function in Terraform, and when should you use them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I only use these as a last option. `local-exec` runs a command on my own machine after a resource is made. `remote-exec` connects into the new server and runs a command there. The problem with both is Terraform doesn't really track what they do. They can fail quietly, and running them again might not fix it. So instead, I usually set up new servers using a startup script or a ready-made image, since those are safer and easier to repeat.

</details>

---

### Q: How do you manage sensitive credentials and database passwords securely in Terraform configurations?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never go into the code, and never get saved in Git. If a value is sensitive, I mark it with `sensitive = true`, so Terraform hides it on the screen and in the logs. For something like a database password, I pull it in from a secrets manager at run time, instead of typing it in directly. One thing to know — `sensitive = true` only hides the value from the screen. It does not remove it from the state file. The real value is still sitting in there, so the state file itself also needs to be locked down.

</details>

---

### Q: What's the difference between `count` and `for_each` in Terraform, and why does it matter when resources change?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`count` tracks resources by their position in a list, like the first one, second one, and so on. The problem is, if that list ever changes order, Terraform thinks a completely different resource is now sitting at that position, and it tries to delete and recreate things that didn't actually need to change. `for_each` tracks resources by a real key, like a name, instead of a position. So if the list order changes but the actual names stay the same, Terraform correctly leaves those resources alone. I default to `for_each` now for anything where the list of items might change over time.

</details>

---

### Q: How do you safely upgrade a Terraform provider version without breaking existing infrastructure?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never let the provider version float freely — I pin it to a specific version, and I commit the lock file to Git, so everyone and every pipeline uses the exact same version. When I do want to upgrade, I bump the version in a branch, run it against a non-production environment first, and carefully read the plan output before applying anything. Some provider upgrades quietly change how a resource behaves, not just the version number, so I never assume an upgrade is safe just because it installed without an error.

</details>

---

### Q: How do you handle a Terraform state file that's grown huge and makes every plan slow?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A giant state file is usually a sign that too much infrastructure is being managed in one place. Every `plan` has to check every single resource in that file, so the bigger it gets, the slower everything gets, even for a tiny change. My fix is splitting it up — separate state files for separate layers, like networking, databases, and applications, each managed on its own. If one layer needs information from another, I read it through a safe, read-only reference, instead of putting everything in one giant file just for convenience.

</details>

---
