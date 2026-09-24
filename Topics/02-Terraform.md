# Senior DevOps Interview Questions: Terraform

### Q: How do you import existing, manually created cloud infrastructure into Terraform state management?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

When I'm onboarding legacy infrastructure that someone clicked together in the console, I don't just run `terraform import` and call it done — the big catch is that import only updates the state file, it doesn't write any HCL for you. So my actual sequence is: first I write a resource block in code with the basic parameters, like AMI and instance type, then I run something like `terraform import aws_instance.web i-1234567890abcdef0`, which pulls the live configuration into the state file.

After that, I always run `terraform plan` right away to spot any drift between what I wrote and what's actually running, and I keep adjusting the HCL until the plan shows zero changes. For example, importing a manually launched instance `i-0a1b2c3d4e5f6g7h8` — I'd define `resource "aws_instance" "legacy_app" { ami = "ami-0c55b159cbfafe1f0"; instance_type = "t3.medium" }`, run the import against that address, then keep refining until `plan` is clean. Skipping that last step is how people end up with Terraform trying to destroy a resource it just imported.

</details>

---

### Q: How do you structure Terraform configurations to manage multiple environments (Dev, Staging, Prod) without code duplication?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For production-grade multi-environment setups, I lean toward reusable modules combined with separate directories, rather than Terraform Workspaces. Modules encapsulate the actual infrastructure pattern — like a VPC module — parameterized by variables, and each environment gets its own thin folder that calls that module with its own `terraform.tfvars` and its own backend.

So the layout looks like a `modules/vpc/` folder holding the reusable logic, and then `environments/dev/main.tf` calling that module with dev variables and pointing its backend at an S3 key like `dev/terraform.tfstate`, while `environments/prod/main.tf` does the same thing but with prod variables and a completely separate state key. The reason I prefer this over Workspaces is that it gives hard state isolation — there's no shared backend where a wrong workspace selection could accidentally apply against production. Every environment's blast radius is physically separated by its own state file.

</details>

---

### Q: How do you configure a secure Terraform remote backend, and how do you recover if a state file is accidentally deleted?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always configure an S3 bucket with encryption and versioning enabled for the backend, paired with a DynamoDB table for state locking so two engineers can't apply at the same time and corrupt each other's changes. The backend block looks something like:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-tf-state"
    key            = "prod/app.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

If someone accidentally deletes the state file, my first move is pulling the previous version straight from S3's version history — that's usually a two-minute fix if versioning was actually enabled, which is exactly why I treat it as mandatory, not optional. If for some reason there's no backup at all, then it's a much rougher day — I'd inspect the live cloud resources and rebuild the state manually by running `terraform import` on every single one, then keep iterating with `plan` until it shows zero drift.

</details>

---

### Q: How do Local Exec and Remote Exec provisioners function in Terraform, and when should you use them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I treat provisioners as a genuine last resort. `local-exec` runs a script on the machine that's actually running `terraform apply` — like firing off a Slack notification. `remote-exec` is different, it SSHes or WinRMs straight into the resource that was just created and runs commands on it directly, something like:

```hcl
provisioner "remote-exec" {
  inline = [
    "chmod +x /tmp/script.sh",
    "/tmp/script.sh"
  ]
}
```

The reason I avoid these unless there's no other option is that provisioners break Terraform's declarative model — they don't get tracked as real resource state, they depend on network connectivity and external tools at apply time, and they can fail silently on a re-run without Terraform really knowing what to do about it. In practice, I'd much rather push that bootstrap logic into user data scripts, a Packer golden image, or a proper Ansible run after the fact — anything that's actually idempotent and doesn't leave Terraform guessing about what state the instance ended up in.

</details>

---

### Q: How do you manage sensitive credentials and database passwords securely in Terraform configurations?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never hardcode secrets in HCL or in `.tfvars` files, and I never let them get committed to Git. Any variable that's sensitive gets declared with `sensitive = true`, so Terraform redacts it from the CLI output and from CI logs. For something like a database password, I pull it dynamically at runtime from AWS Secrets Manager using a data source, rather than passing it in as a plain variable at all — something like reading `data.aws_secretsmanager_secret_version.db_secret.secret_string` straight into the `password` argument on the RDS resource.

Now, the big catch that a lot of people miss — `sensitive = true` only hides the value from your terminal and console output, it does **not** encrypt it out of the actual state file. The state file still holds the real plaintext value. So marking a variable sensitive is necessary, but it's not sufficient on its own — the state bucket itself still has to be encrypted and locked down to only the roles that genuinely need to read it.

</details>

---
