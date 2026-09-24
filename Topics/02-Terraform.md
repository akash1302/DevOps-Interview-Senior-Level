# Senior DevOps Interview Questions: Terraform

## Q1. How do you import existing, manually created cloud infrastructure into Terraform state management?

### Answer
To bring unmanaged infrastructure under Terraform control without recreating resources, you use the `terraform import` command. First, you write a matching resource configuration block in your Terraform `.tf` file representing the resource attributes. Next, you execute `terraform import <RESOURCE_TYPE>.<RESOURCE_NAME> <RESOURCE_ID>`. Terraform calls the cloud API, retrieves the resource state, and writes it directly to the state file (`terraform.tfstate`). Finally, you run `terraform plan` to verify that your code matches the live infrastructure attributes.

### Interview Answer
"When onboarding manually created legacy infrastructure, I first write a resource block in code specifying basic parameters like AMI and instance type. Then I run `terraform import aws_instance.web i-1234567890abcdef0`. This pulls the existing live configuration into the Terraform state file. Afterwards, I run `terraform plan` to spot drift between my code and the state, adjusting the HCL until the plan shows zero changes needed."

### Practical Example
Importing an manually launched EC2 instance (`i-0a1b2c3d4e5f6g7h8`):
1. Define in code:
```hcl
resource "aws_instance" "legacy_app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}
```
2. Execute: `terraform import aws_instance.legacy_app i-0a1b2c3d4e5f6g7h8`
3. Execute `terraform plan` and refine code attributes until no diff exists.

### Follow-up Questions
* Does `terraform import` generate HCL code automatically in older versus newer Terraform versions?
* How do you handle resource dependencies when importing complex multi-resource stacks?
* What happens if you import a resource without defining its corresponding resource block in code?

### Key Points
* `terraform import` updates the state file but does not write HCL code automatically.
* You must write matching resource configuration blocks before or alongside importing.
* `terraform plan` is mandatory post-import to verify zero configuration drift.

---

## Q2. How do you structure Terraform configurations to manage multiple environments (Dev, Staging, Prod) without code duplication?

### Answer
Managing multiple environments without duplicating code is achieved using Terraform Modules paired with either separate directory structures or Terraform Workspaces. Terraform Modules encapsulate infrastructure logic into reusable blocks parameterized by variables. With directory separation, distinct folders (e.g., `environments/dev`, `environments/prod`) call the shared modules using environment-specific `terraform.tfvars` files and separate backend state files. Workspaces allow using a single configuration file while maintaining isolated state files per workspace.

### Interview Answer
"For production-grade multi-environment setups, I prefer using reusable Terraform Modules combined with directory separation over Workspaces. Modules encapsulate resource patterns, and each environment (dev, prod) gets its own folder with its own `main.tf` calling the module, a distinct `terraform.tfvars` file, and an isolated remote S3 backend. This ensures hard state isolation, preventing accidental production modifications when working in dev."

### Practical Example
Module structure:
```
modules/
  vpc/
environments/
  dev/
    main.tf (calls modules/vpc with dev vars)
    backend.tf (S3 key: dev/terraform.tfstate)
  prod/
    main.tf (calls modules/vpc with prod vars)
    backend.tf (S3 key: prod/terraform.tfstate)
```

### Follow-up Questions
* Why is directory separation generally preferred over Terraform Workspaces for separate production environments?
* How do you pass output variables from a VPC module to an EKS module?
* How do version constraints on custom modules prevent breaking changes across environments?

### Key Points
* Modules abstract infrastructure logic into reusable, parameterized units.
* Separate environment directories ensure state file isolation and blast radius reduction.
* Environment-specific variable files (`.tfvars`) provide tailored parameters (e.g., instance sizing).

---

## Q3. How do you configure a secure Terraform remote backend, and how do you recover if a state file is accidentally deleted?

### Answer
A secure remote backend stores `terraform.tfstate` outside local disk drives—typically in an AWS S3 bucket with server-side encryption, versioning enabled, and restricted IAM access policies. State locking is enforced using a DynamoDB table to prevent concurrent execution conflicts. If a state file is deleted or corrupted, recovery involves restoring the previous state version from S3 bucket versioning. If no backup exists, you must manually rebuild state using `terraform import` for each live resource.

### Interview Answer
"I configure an S3 bucket with AES-256 encryption, access logging, and bucket versioning enabled, combined with a DynamoDB table for `LockID` state locking. If someone accidentally deletes the state file, I restore the last known good version directly from S3 bucket history. If backups are completely absent, I inspect live cloud resources and run `terraform import` iteratively to rebuild the state file from scratch."

### Practical Example
Backend configuration in `backend.tf`:
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

### Follow-up Questions
* What specific DynamoDB attribute is required to enable Terraform state locking?
* What happens when two engineers execute `terraform apply` simultaneously on a locked state?
* How do IAM policies prevent unauthorized developers from reading sensitive state data in S3?

### Key Points
* Remote backends provide state centralization, state locking, and team collaboration.
* S3 Bucket Versioning is the primary line of defense against state loss or corruption.
* DynamoDB handles state locking to block concurrent mutation state races.

---

## Q4. How do Local Exec and Remote Exec provisioners function in Terraform, and when should you use them?

### Answer
Provisioners run local or remote commands after a resource is created or destroyed. `local-exec` executes scripts locally on the machine running `terraform apply`. `remote-exec` connects to the newly created remote resource via SSH or WinRM to execute scripts directly on the target instance. Provisioners should be used as a last resort because they break Terraform's declarative model, depend on external runtime tools, and make state management harder to track.

### Interview Answer
"I treat provisioners as a last resort. `local-exec` runs scripts on my build server—like firing a notification—while `remote-exec` SSHs into a launched EC2 instance to execute bash commands. However, because provisioners don't model resource state declaratively and can fail silently on re-runs, I prefer cloud-init, user data scripts, Packer golden images, or Ansible for post-provisioning configuration."

### Practical Example
Using `remote-exec` to set executable permissions and launch a script:
```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/script.sh",
      "/tmp/script.sh"
    ]
  }
}
```

### Follow-up Questions
* What happens to a resource in Terraform state if a `remote-exec` provisioner script fails?
* What is the purpose of the `on_failure = continue` argument in provisioner blocks?
* Why is Packer or cloud-init preferred over provisioners for bootstrap configurations?

### Key Points
* `local-exec` runs locally on the machine executing Terraform commands.
* `remote-exec` connects via SSH/WinRM to execute scripts on the provisioned host.
* Provisioners break declarativity and should be replaced by User Data, Ansible, or golden images.

---

## Q5. How do you manage sensitive credentials and database passwords securely in Terraform configurations?

### Answer
Sensitive credentials must never be hardcoded in `.tf` configuration files or committed to Version Control Systems (VCS). Sensitive input variables should be marked with `sensitive = true` to redact their values from CLI output and execution logs. Credentials should be fetched dynamically at runtime from secure secrets managers (such as AWS Secrets Manager or HashiCorp Vault) using data sources, or supplied via environment variables (`TF_VAR_secret_name`).

### Interview Answer
"I never hardcode secrets in HCL or `.tfvars` files. I declare variables with `sensitive = true` so Terraform redacts them from terminal output and CI logs. For runtime secrets like database passwords, I store them in AWS Secrets Manager or HashiCorp Vault, and fetch them dynamically using data sources. Keep in mind that state files still contain plain-text values, so the remote backend S3 bucket must be strongly encrypted and access-restricted."

### Practical Example
Fetching a database secret from AWS Secrets Manager:
```hcl
data "aws_secretsmanager_secret_version" "db_secret" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "db" {
  allocated_storage = 20
  engine            = "postgres"
  instance_class    = "db.t3.micro"
  username          = "dbadmin"
  password          = data.aws_secretsmanager_secret_version.db_secret.secret_string
}
```

### Follow-up Questions
* Why are values marked with `sensitive = true` still visible in the raw `terraform.tfstate` file?
* How do environment variables formatted as `TF_VAR_<var_name>` simplify CI/CD pipeline secrets injection?
* How does HashiCorp Vault integrate with Terraform for dynamic short-lived credentials?

### Key Points
* Never hardcode secrets or commit secret `.tfvars` files to git repositories.
* Setting `sensitive = true` suppresses secret values from CLI logs and console output.
* Use AWS Secrets Manager or Vault data sources to inject credentials dynamically.


---
