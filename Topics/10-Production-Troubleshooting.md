# Senior DevOps Interview Questions: Production Troubleshooting

### Q: How do you systematically troubleshoot a Docker container that cannot access the internet?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I isolate this top-down, host first, then container, so I'm never chasing a container-level symptom that's actually a host problem. First I check the host itself with `ping -c 2 8.8.8.8` — if that fails, nothing below it matters yet, it's an upstream issue. Once the host's confirmed fine, I exec into a throwaway `busybox` container and ping `8.8.8.8` directly.

If that IP ping works but `ping google.com` fails, it's a DNS problem specific to the container, usually traced to `/etc/resolv.conf`. If the IP ping fails outright, I check whether the host has kernel packet forwarding enabled with `sysctl net.ipv4.ip_forward` — it needs to read `1` — and then inspect `iptables -t nat -L -n -v` for Docker's NAT rules. One real incident I'd walk through — a host restarted, UFW reset and wiped out Docker's `POSTROUTING` MASQUERADE rules, so the host could reach the internet fine but every container timed out. `sysctl` confirmed forwarding was still enabled, so the actual fix was `systemctl restart docker`, which forces Docker to rebuild its bridge network and repopulate those NAT rules from scratch.

</details>

---

### Q: How do you diagnose and resolve a container or pod trapped in a CrashLoopBackOff state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First thing I run is `kubectl logs <pod> --previous`, since the current container instance has already restarted and its logs are fresh — I need the output from right before it actually crashed. Then `kubectl describe pod` to see the exit code and the surrounding events, because the exit code tells you what kind of problem you're actually dealing with.

Exit code `137` means it was killed by the Linux OOM killer — that's a memory limit problem, fix is raising `limits.memory` based on real usage. Exit code `1` is usually an application-level failure, something like a missing database secret or an unhandled exception — that's a code or config problem, not a platform one. Exit code `127` means the command or script Docker's trying to run doesn't even exist at that path, which points straight back to the `CMD` in the Dockerfile. If the logs genuinely don't explain anything, I'll temporarily override the container's command to `sleep 3600`, exec into it while it's held open, and manually check environment variables, file permissions, and connectivity by hand.

</details>

---

### Q: How do you troubleshoot and recover from an AWS Terraform state corruption or accidental deletion?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the state file gets corrupted or deleted and we've actually followed best practice — S3 backend with bucket versioning turned on — recovery is fast. I list the object versions for `terraform.tfstate` and copy the last good version back over the current key:

```bash
aws s3api list-object-versions --bucket my-tf-state-bucket --prefix prod/terraform.tfstate

aws s3api copy-object \
  --copy-source my-tf-state-bucket/prod/terraform.tfstate?versionId=v1_PreviousVersionID \
  --bucket my-tf-state-bucket \
  --key prod/terraform.tfstate
```

That's usually a two-minute fix, which is exactly why I treat versioning as mandatory on every state bucket, not optional. If versioning was somehow disabled and there's genuinely no backup anywhere, it's a much longer day — I freeze all infrastructure changes first, then go resource by resource through the live account, writing matching HCL and running `terraform import` for each one, checking `plan` after every import until it finally shows zero drift. One other thing worth knowing for a crashed pipeline specifically — if a CI job dies mid-apply and leaves the DynamoDB lock held, `terraform force-unlock <LOCK-ID>` is how you release it without corrupting anything, rather than manually deleting the lock row in DynamoDB.

</details>

---
