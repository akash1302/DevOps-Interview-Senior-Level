# Senior DevOps Interview Questions: Production Troubleshooting

## Q1. How do you systematically troubleshoot a Docker container that cannot access the internet?

### Answer
Troubleshooting a container lacking internet connectivity follows a step-by-step OSI model layer isolation:
1. **Verify Host Connectivity**: Execute `ping -c 2 8.8.8.8` and `dig google.com` on the underlying host to ensure the host itself has working internet and DNS.
2. **Verify Container Connectivity**: Execute `docker exec` into a test container (`busybox`) and test IP ping (`ping 8.8.8.8`) versus domain ping (`ping google.com`).
3. **Inspect DNS & Bridge Networks**: If IP ping succeeds but domain ping fails, inspect `/etc/resolv.conf` inside the container. If IP ping fails, verify that the container is attached to the default bridge network (`docker network inspect bridge`).
4. **Inspect Host Packet Forwarding & Firewalls**: Verify that Linux kernel IP forwarding is enabled (`sysctl net.ipv4.ip_forward = 1`) and inspect IPTables NAT forwarding rules (`iptables -t nat -L -n -v`).
5. **Restart Docker Network Stack**: If IPTables rules corrupted after firewall/daemon reloads, restart the Docker daemon (`systemctl restart docker`).

### Interview Answer
"I isolate the issue top-down. First, I verify the host machine has outbound internet access using `ping 8.8.8.8`. Next, I jump into a container and ping `8.8.8.8`. If IP ping works but `ping google.com` fails, it's a container DNS issue in `/etc/resolv.conf`. If IP ping fails completely inside the container, I check if Linux kernel packet forwarding is enabled via `sysctl net.ipv4.ip_forward`. If that's `1`, I check host IPTables NAT rules or restart the Docker service to rebuild the bridge network interfaces."

### Practical Example
Root Cause Scenario: A host server restarted, and UFW firewall reset IPTables, wiping Docker's `POSTROUTING` MASQUERADE rules.
* **Diagnosis**: Host pings internet successfully. Container pinging `8.8.8.8` times out. `sysctl net.ipv4.ip_forward` returns `1`. IPTables NAT table missing Docker rules.
* **Resolution**: Execute `systemctl restart docker` to force Docker to recreate its bridge network interfaces and repopulate IPTables NAT forwarding rules.

### Follow-up Questions
* Why does restarting the Docker daemon resolve corrupted bridge network routing?
* How do custom Docker networks differ from the default bridge network regarding DNS resolution?
* What role does `--net=host` play when debugging container networking issues?

### Key Points
* Rule out host-level internet and DNS issues before debugging container networks.
* Test numerical IP connectivity (`8.8.8.8`) separately from domain DNS resolution (`google.com`).
* Verify Linux kernel packet forwarding (`net.ipv4.ip_forward=1`) and IPTables NAT MASQUERADE rules.

---

## Q2. How do you diagnose and resolve a container or pod trapped in a CrashLoopBackOff state?

### Answer
`CrashLoopBackOff` indicates that a Kubernetes pod or Docker container repeatedly starts, fails, and restarts in an escalating delay loop. Diagnosis follows a precise sequence:
1. **Fetch Application Logs**: Run `kubectl logs <pod-name> --previous` to view the stderr/stdout output from the failed container instance before it crashed.
2. **Inspect Pod Description**: Run `kubectl describe pod <pod-name>` to view exit codes, termination reasons (e.g., `OOMKilled`), readiness/liveness probe failures, and lifecycle events.
3. **Verify Configuration & Dependencies**: Check for missing environment variables, invalid secret keys, syntax errors in config maps, or failed database connection handshakes.
4. **Debug Interactively**: If logs are empty, temporarily override entrypoint syntax (`command: ["sh", "-c", "sleep 3600"]`) to keep the container alive and inspect filesystem permissions interactively.

### Interview Answer
"When a pod enters `CrashLoopBackOff`, I first run `kubectl logs <pod> --previous` to see the stack trace right before the crash. Next, I run `kubectl describe pod` to check the exit code and events. If Exit Code is `137`, it was OOMKilled, meaning it exceeded its RAM limit. If Exit Code is `1`, it's an app runtime error like a missing DB connection secret. If logs are unhelpful, I temporarily override the container `command` to `sleep 3600`, exec into the pod, and test environment variables and connectivity manually."

### Practical Example
Common Crash Exit Codes:
* **Exit Code 137**: Process killed by Linux OOM (Out Of Memory) Killer. Resolution: Increase memory limits in container spec.
* **Exit Code 1**: Application exception (e.g., Unhandled NullPointer / Database connection failure). Resolution: Inspect app logs and fix configuration secrets.
* **Exit Code 127**: Command or executable script not found. Resolution: Verify `CMD` path in Dockerfile.

### Follow-up Questions
* What is the difference between a Liveness Probe failure and a Readiness Probe failure?
* How does setting `imagePullPolicy: Always` affect container startup troubleshooting?
* How do you troubleshoot a pod stuck in `ContainerCreating` or `Pending` status?

### Key Points
* Check previous container instance logs using `kubectl logs --previous`.
* Inspect exit codes in `kubectl describe pod` (Exit Code 137 = OOMKilled; Exit Code 1 = Application Error).
* Override container commands to `sleep` for interactive shell debugging when logs are unavailable.

---

## Q3. How do you troubleshoot and recover from an AWS Terraform state corruption or accidental deletion?

### Answer
Accidental deletion or corruption of a Terraform state file (`terraform.tfstate`) halts infrastructure operations. If state management follows production best practices (S3 remote backend with bucket versioning enabled), recovery is straightforward:
1. Navigate to the AWS S3 console or AWS CLI.
2. List object versions for `terraform.tfstate`.
3. Restore or download the previous uncorrupted S3 object version and overwrite the current state key.

If versioning was not enabled and no backup file exists, state must be reconstructed manually:
1. Write/verify all resource HCL blocks matching existing live infrastructure.
2. Execute `terraform import` for every live cloud resource sequentially to reconstruct state entries.
3. Run `terraform plan` iteratively until zero drift is detected.

### Interview Answer
"If our S3 remote state file is corrupted or deleted, my first step is leveraging S3 Bucket Versioning. I pull the previous version of the state object from S3 history and restore it, which takes under two minutes. If versioning was mistakenly disabled and no state backup exists, I lock all infrastructure changes, inspect live cloud resources via AWS CLI, and manually run `terraform import` for each resource until `terraform plan` confirms zero configuration diff."

### Practical Example
Restoring state via AWS CLI S3 Versioning:
```bash
# List state versions
aws s3api list-object-versions --bucket my-tf-state-bucket --prefix prod/terraform.tfstate

# Copy previous version over current key
aws s3api copy-object   --copy-source my-tf-state-bucket/prod/terraform.tfstate?versionId=v1_PreviousVersionID   --bucket my-tf-state-bucket   --key prod/terraform.tfstate
```

### Follow-up Questions
* Why is enabling S3 Bucket Versioning mandatory for Terraform remote backend storage?
* How do you force-release an abandoned state lock in DynamoDB if a CI pipeline crashes mid-run?
* What is `terraform refresh` and how does it reconcile state with live cloud infrastructure?

### Key Points
* S3 Bucket Versioning provides fast, automated recovery for deleted remote state files.
* If no backups exist, state must be rebuilt manually using iterative `terraform import` calls.
* Use `terraform force-unlock <LOCK-ID>` if a crashed pipeline leaves DynamoDB state locked.


---
