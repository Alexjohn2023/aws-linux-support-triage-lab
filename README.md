# AWS Linux Support Triage Lab

A hands-on cloud technical support project built on an Ubuntu AWS EC2 virtual machine.

This project demonstrates Linux troubleshooting, service recovery, SSH authentication, file permissions, security review, CPU and storage investigation, reverse-proxy troubleshooting, firewall diagnosis, AWS security group troubleshooting, DNS investigation, and Jira support documentation.

**Author:** Alexander Njoku  
**Infrastructure:** AWS EC2  
**Operating system:** Ubuntu  
**Web server:** Nginx  
**Application:** Node.js managed by systemd  
**Client tools:** Windows Git Bash and SSH  
**Ticketing:** Jira Service Management  

> This project uses AWS infrastructure and demonstrates skills transferable to other cloud support environments.

## Project Goals

- Establish a working baseline before introducing faults.
- Identify the failing layer using commands, logs, and comparison tests.
- Distinguish service, application, network, resource, and access-control failures.
- Apply targeted corrections and verify recovery.
- Preserve administrator access during authentication and firewall testing.
- Run application workloads under a non-administrator account.
- Review security controls and document remaining findings.
- Communicate clearly through internal notes and simulated customer updates.
- Develop reusable incident reports and troubleshooting procedures.

## Exercise Roadmap

| Exercise | Status | Skills Demonstrated |
| --- | --- | --- |
| Baseline verification | Complete | Service state, listeners, configuration, HTTP checks |
| Nginx service outage | Complete - IT-1 and IT-2 | Service diagnosis and local/external recovery verification |
| SSH and permissions | Complete - IT-3 and IT-4 | Diagnose HTTP 403 and SSH failures using permissions and logs |
| Security review | Complete - IT-5; findings documented | Assess users, SSH, permissions, process owners, UFW, AWS rules, and AppArmor coverage |
| High CPU | Complete - IT-6 | Identify a bounded non-root workload and verify automatic recovery |
| Isolated filesystem full | Complete - IT-7 | Distinguish capacity exhaustion from inode exhaustion and restore writes |
| Application 502 error | Complete - IT-8 | Distinguish a healthy Nginx proxy from a stopped upstream application |
| Host firewall blocks HTTP | Complete - IT-9 | Compare local HTTP success with an external firewall-induced timeout |
| AWS security group and network ACL troubleshooting | In progress — security group complete, IT-10; network ACL pending | Diagnose missing cloud ingress rules and assess subnet-level change scope |
| DNS lookup failure | Complete- IT-11 | Distinguish NXDOMAIN from a general resolver or web-server failure |

**Progress: Nine of ten roadmap categories are complete.**

All eleven Jira records are marked **Completed / Fixed**. Ticket count differs from roadmap count because SSH and permissions share a category, Nginx was exercised twice, and the AWS networking category includes separate security group and network ACL exercises.

The security review and DNS investigation required no configuration repair. Their Jira resolution label records workflow completion rather than remediation of all findings.

## Lab Environment

| Component | Purpose |
| --- | --- |
| AWS EC2 | Hosts the Linux lab |
| Ubuntu | Operating system |
| Nginx | Serves static content and proxies application requests |
| Node.js | Small HTTP application |
| systemd | Manages Nginx and the application service |
| supportlab | Dedicated non-administrator account |
| support-lab | Dedicated application group |
| Windows Git Bash | Client-side SSH and external HTTP tests |
| UFW | Host firewall |
| AWS security group | Instance-level cloud traffic controls |
| AppArmor | Application confinement coverage review |
| 16 MiB tmpfs | Isolated temporary filesystem for storage testing |
| systemd-resolved | Local DNS resolver service |
| Jira Service Management | Investigation, communication, and closure |

The completed web exercises use HTTP. HTTPS, ALB deployment, and monitoring integrations have not been demonstrated in this project.

## Application Layout

| Endpoint | Destination |
| --- | --- |
| Nginx `/` | Static files under `/var/www/html` |
| Nginx `/support-app/` | Node.js at `127.0.0.1:3000` |
| Node.js direct endpoint | Loopback access only |

Application service settings include:

```ini
User=supportlab
Group=support-lab
WorkingDirectory=/srv/support-lab
ExecStart=/usr/bin/node /srv/support-lab/app.js
```

Nginx proxy configuration:

```nginx
location /support-app/ {
    proxy_pass http://127.0.0.1:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

## Support Workflow

1. Verify the working baseline.
2. Introduce a controlled fault or workload.
3. Assess symptoms and affected scope.
4. Gather logs and system evidence.
5. Identify the cause.
6. Apply a targeted correction or observe bounded workload termination.
7. Verify recovery.
8. Record technical findings.
9. Provide a simulated customer-facing update.
10. Complete the Jira workflow and update documentation.

Jira records were created after the technical investigation to document the simulated support workflow.

Security review followed an assessment workflow: inspect controls, document findings, verify access, and identify follow-up work.

## Baseline Verification

### Objective

Establish what healthy operation looks like before troubleshooting.

### Commands

```bash
date -u
whoami
hostname
cat /etc/os-release
uptime
free -h
df -h /
df -i /
sudo nginx -t
sudo systemctl is-active nginx
sudo ss -lntp
curl -I --max-time 5 http://127.0.0.1/
```

### Verified Results

- Nginx configuration passed validation.
- Nginx was active.
- Port 80 was listening.
- Local HTTP returned 200.
- Memory, filesystem capacity, and inode information were recorded.

Evidence:

```text
baseline/server-baseline.txt
```

## IT-1 and IT-2: Nginx Service Outage

**Dates:** October 6 and October 10, 2026  
**Status:** Completed / Fixed  

### Scenario

Nginx was intentionally stopped to simulate an unavailable website.

### Symptoms

- Service inactive.
- Local HTTP connection failed.

### Investigation and Recovery

```bash
sudo systemctl is-active nginx
curl -I --max-time 5 http://127.0.0.1/
sudo nginx -t
sudo systemctl start nginx
```

### Verification

```bash
sudo systemctl is-active nginx
sudo ss -lntp '( sport = :80 )'
curl -I --max-time 5 http://127.0.0.1/
```

For IT-2, a separate request from Windows Git Bash to the instance's public IP also returned HTTP 200.

### Lesson

Check configuration, service state, listeners, and HTTP behavior together. Verify externally when the reported problem concerns external access.

## IT-3: Website Permissions — HTTP 403

**Date:** October 10, 2026  
**Status:** Completed / Fixed  

### Baseline and Fault

A root-owned public test file returned HTTP 200 with permissions `644`.

```text
/var/www/html/support-permissions-lab.txt
```

Changed its permissions:

```bash
sudo chmod 600 /var/www/html/support-permissions-lab.txt
```

Nginx remained active, but the page returned HTTP 403.

### Investigation

```bash
sudo stat -c '%A %a %U:%G %n' \
/var/www/html/support-permissions-lab.txt

ps -eo user,pid,comm | grep '[n]ginx'

curl -i --max-time 5 \
http://127.0.0.1/support-permissions-lab.txt

sudo tail -n 20 /var/log/nginx/error.log
namei -l /var/www/html/support-permissions-lab.txt
```

At 19:53:52 UTC, Nginx logged:

```text
open() "/var/www/html/support-permissions-lab.txt" failed
(13: Permission denied)
```

### Cause and Recovery

The worker account could not read the file.

```bash
sudo chmod 644 /var/www/html/support-permissions-lab.txt
```

Permissions `644` and local HTTP 200 were verified. No Nginx restart was required.

### Evidence

```text
permissions/01-working-baseline.txt
permissions/02-permission-failure.txt
permissions/03-permission-restored.txt
```

### Lesson

An active service can still fail to serve an individual resource. Avoid broad permission changes such as `chmod 777`.

## IT-4: SSH Authentication Failure

**Date:** October 10, 2026  
**Status:** Completed / Fixed  

### Baseline

Configured Ed25519 key authentication for `supportlab`.

Verified:

- `.ssh` directory: `700`.
- `authorized_keys`: `600`.
- Ownership: `supportlab:supportlab`.
- No sudo group membership.
- Successful fresh SSH login.

The original administrator session remained open.

### Fault

```bash
sudo chmod 666 /home/supportlab/.ssh/authorized_keys
```

A fresh login failed:

```text
Permission denied (publickey).
```

### Diagnosis

```bash
sudo stat -c '%A %a %U:%G %n' \
/home/supportlab/.ssh/authorized_keys

sudo journalctl --since "2026-10-10 20:35:00 UTC" \
--until "2026-10-10 20:37:30 UTC" --no-pager
```

At 20:36:34 UTC:

```text
Authentication refused: bad ownership or modes for file
/home/supportlab/.ssh/authorized_keys
```

### Recovery

```bash
sudo chmod 600 /home/supportlab/.ssh/authorized_keys
```

A fresh key-based login succeeded at 20:38:45 UTC.

No SSH restart was required.

### Evidence

```text
ssh/02-key-access-baseline.txt
ssh/03-authentication-failure.txt
ssh/04-authentication-restored.txt
ssh/05-failure-log-detail.txt
```

### Lesson

Correlate client errors with server logs. Test a fresh connection after correcting authentication problems.

## IT-5: Security Configuration Review

**Date:** October 14, 2026  
**Status:** Completed; findings documented  

### Verified Controls

| Area | Result |
| --- | --- |
| Lab account | No sudo membership |
| SSH public-key authentication | Enabled |
| Password authentication | Disabled |
| Keyboard-interactive authentication | Disabled |
| StrictModes | Enabled |
| SSH directories | 700 with correct ownership |
| authorized_keys files | 600 with correct ownership |
| Nginx workers | Non-root www-data |
| Nginx master | root |
| UFW | Active; default deny incoming, allow outgoing |
| UFW allowances | SSH 22 and HTTP 80 for IPv4 and IPv6 |
| AWS security group | One attached group |
| AWS SSH ingress | Administrator's public IPv4 /32 |
| AWS HTTP ingress | TCP 80 from 0.0.0.0/0 |
| AWS egress | All traffic allowed |
| Access checks | Fresh SSH login and external HTTP 200 |

### AppArmor Findings

- Two chronyd processes were in enforce mode.
- rsyslogd was in enforce mode.
- Amazon SSM Agent was in complain mode.
- Nginx confinement was not demonstrated.

AppArmor being enabled does not prove that every process is confined.

### Follow-Up Work

- Assess narrower UFW SSH source rules.
- Evaluate disabling direct root login after verifying administrator access.
- Assess required outbound traffic before restricting egress.
- Develop and test Nginx AppArmor confinement separately.

`PermitRootLogin prohibit-password` still permits root key authentication when other requirements are satisfied.

### Outcome

Existing settings were reviewed, not remediated. Findings remain documented follow-up work.

## IT-6: CPU Workload Investigation

**Date:** October 14, 2026  
**Status:** Completed / Fixed  

### Workload

Ran without sudo as supportlab:

```bash
timeout -k 2s 60s nice -n 10 yes > /dev/null &
```

### Findings

At 06:06:13 UTC:

| Check | Result |
| --- | --- |
| Logical CPUs | 2 |
| Process | yes |
| PID / parent PID | 2165 / 2160 |
| Owner | supportlab |
| Nice value | 10 |
| top process CPU | 90.9% |
| ps process CPU | 99.9% |
| Local HTTP | 200 |

The workload consumed approximately one logical CPU, not the whole two-CPU server.

`ps` reports lifetime-average CPU usage, which can differ from interval-based measurements.

### Recovery

The configured timeout ended the workload automatically.

At 06:08:33–06:08:35 UTC:

- No yes workload remained.
- CPU idle measured 100.0% and 99.1%.
- Nginx remained active.
- Local HTTP returned 200.

No manual process termination or service restart was performed.

### Evidence

```text
resources/02-cpu-workload.txt
resources/03-cpu-recovery.txt
```

### Lesson

Identify process ownership and purpose before acting. High CPU alone does not justify terminating an unknown or critical process.

## IT-7: Isolated Filesystem Full

**Date:** October 16, 2026  
**Status:** Completed / Fixed  

### Setup

Mounted a dedicated temporary filesystem:

| Setting | Value |
| --- | --- |
| Mount point | /mnt/support-disk-lab |
| Type | tmpfs |
| Capacity | 16 MiB |
| Inodes | 1,024 |
| Owner | supportlab:supportlab |
| Permissions | 700 |
| Mount restrictions | nosuid, nodev, noexec |

The root filesystem was 33% used before the exercise.

### Fault

After verifying the mount, attempted a 17 MiB write as supportlab:

```bash
sudo -u supportlab dd if=/dev/zero \
of=/mnt/support-disk-lab/lab-fill.bin \
bs=1M count=17 status=none
```

### Findings at 05:26:55 UTC

- Write failed with “No space left on device.”
- Exit code: 1.
- Isolated filesystem: 100% full.
- Known test file: 16 MiB.
- Free inodes: 1,022.
- Root filesystem: unchanged at 33%.

### Recovery

Removed only the known disposable file:

```bash
sudo -u supportlab rm -- /mnt/support-disk-lab/lab-fill.bin
```

Created and read back a new file as supportlab.

At 05:27:53 UTC:

- Write exit code was 0.
- Expected contents were read back.
- Isolated filesystem usage returned to 1%.
- Root filesystem remained 33% used.
- Nginx was active and local HTTP returned 200.

The temporary filesystem was unmounted after verification.

### Evidence

```text
resources/04-filesystem-baseline.txt
resources/05-filesystem-full.txt
resources/06-filesystem-recovery.txt
```

### Limitations

This tested tmpfs capacity exhaustion, not EBS failure, root-disk exhaustion, inode exhaustion, or a storage-induced website outage.

### Lesson

Check both capacity and inodes. Verify recovery with a successful write and read-back test.

## IT-8: Nginx 502 — Stopped Upstream

**Date:** October 16, 2026  
**Status:** Completed / Fixed  

### Baseline

The Node.js application ran as supportlab:support-lab on loopback port 3000.

Both direct and proxied requests returned:

```text
HTTP/1.1 200 OK
Support lab application is healthy
```

### Fault

```bash
sudo systemctl stop support-lab-app
```

The application stopped at 05:52:17 UTC.

### Findings

| Check | Result |
| --- | --- |
| Nginx | Active |
| Application | Inactive |
| Port 3000 | No listener |
| Direct application request | Connection failed |
| /support-app/ | HTTP 502 |
| Static website / | HTTP 200 |

At 05:52:33 UTC, Nginx logged:

```text
connect() failed (111: Connection refused) while connecting to upstream
```

### Recovery

```bash
sudo systemctl start support-lab-app
```

Verified:

- Both services active.
- Application listening on 127.0.0.1:3000.
- Process owned by supportlab:support-lab.
- Direct application, proxy route, and static website returned HTTP 200 locally.

No Nginx restart or firewall change was required.

### Evidence

```text
application/01-proxy-baseline.txt
application/02-upstream-failure.txt
application/03-upstream-recovery.txt
```

### Lesson

A healthy proxy does not establish upstream health. Compare direct and proxied requests, listeners, service state, and logs.

## IT-9: UFW Blocks External HTTP

**Date:** October 16, 2026  
**Status:** Completed / Fixed  

### Fault

Inserted a source-specific TCP 80 deny rule before the existing HTTP allowance.

The rule was labelled:

```text
LAB-HTTP-BLOCK
```

SSH port 22 was not targeted.

### Findings

At 06:03:53 UTC:

- UFW showed the deny rule before the allow rule.
- Nginx remained active.
- Local HTTP returned 200.
- An external Windows request timed out after approximately five seconds.

### Recovery

Deleted the exact source-specific deny rule.

Verified:

- The test rule was absent.
- Original SSH and HTTP allowances remained.
- External HTTP returned 200 at 06:04:57 UTC.
- The administrator SSH session remained available.

No Nginx restart or AWS security group change was required.

### Evidence

```text
network/01-firewall-baseline.txt
network/02-firewall-block.txt
```

External timeout and recovery output were also captured in Windows Git Bash.

### Lesson

Compare local and external behavior before restarting services. Host firewall rule order can determine whether traffic reaches a healthy application.

## IT-10: Missing AWS HTTP Ingress Rule

**Date:** October 16, 2026  
**Status:** Completed / Fixed  

### Scope Check

Reviewed related resources before editing the security group and identified the lab instance and its network interface.

### Fault

Temporarily removed the inbound HTTP TCP 80 allowance.

The SSH rule remained unchanged.

### Findings

- Local EC2 HTTP returned 200 at 06:15:08 UTC.
- UFW retained its HTTP allowance.
- A new external connection timed out after approximately five seconds.

An initial diagnostic command was mistakenly run in Windows Git Bash. It was rerun on EC2; the Windows localhost result was excluded from server evidence.

### Recovery

Restored HTTP TCP 80 ingress from 0.0.0.0/0.

External HTTP returned 200 at 06:16:58 UTC.

No Nginx restart or UFW change was required.

### Evidence

```text
network/04-security-group-failure.txt
```

External timeout and recovery output were captured in Windows Git Bash.

### Lesson

A host firewall allowance does not override a missing cloud security group allowance.

Confirm the machine, destination IP, and network layer before interpreting results.

## IT-11: DNS NXDOMAIN Investigation

**Date:** October 16, 2026  
**Status:** Completed; no configuration repair required  

### Baseline

At 06:43:12 UTC:

- example.com returned NOERROR with IPv4 answers.
- getent also resolved the name.
- The local DNS stub was 127.0.0.53.
- Local Nginx returned HTTP 200.

The local stub address is not necessarily the upstream DNS server.

### Controlled Test

```bash
dig support-lab.invalid. A +time=3 +tries=1

curl --noproxy '*' -I --connect-timeout 3 --max-time 5 \
http://support-lab.invalid/

dig example.com. A +time=3 +tries=1
```

### Findings

| Check | Result |
| --- | --- |
| support-lab.invalid | NXDOMAIN |
| Curl | Could not resolve host; exit code 6 |
| example.com comparison | NOERROR with IPv4 answers |
| Local website | HTTP 200 |

### Conclusion

The deliberately nonexistent hostname failed as expected.

Successful comparison queries showed that resolution worked for a known-good name. No DNS configuration, record, or service changes were required.

### Evidence

```text
dns/01-dns-baseline.txt
dns/02-nxdomain-comparison.txt
```

### Limitations

This did not test DNS timeouts, SERVFAIL, authoritative-record repair, or propagation.

The Jira Fixed label records workflow completion, not a repaired DNS configuration.

## Network ACL Exercise — Pending

Inspection established that:

- The default network ACL is associated with six subnets.
- The current lab subnet contains four network interfaces, including other instances.
- Modifying that shared ACL or changing the existing subnet association could affect unrelated lab resources.

The existing shared ACL was left unchanged during this inspection.

The proposed next step is a dedicated subnet and temporary test resource with a separately configured ACL. Creation and association have not been verified.

### Planned Learning Outcomes

- Understand subnet-level scope.
- Evaluate rules in ascending numerical order.
- Distinguish stateless network ACLs from stateful security groups.
- Check inbound traffic and required return traffic.
- Preserve administrator access.
- Verify recovery and restore the original configuration.

## Jira Support Records

| Ticket | Activity | Final Workflow State |
| --- | --- | --- |
| IT-1 | Initial Nginx outage | Completed / Fixed |
| IT-2 | Nginx outage with external verification | Completed / Fixed |
| IT-3 | Website permissions — HTTP 403 | Completed / Fixed |
| IT-4 | SSH authorization-file permissions | Completed / Fixed |
| IT-5 | Security configuration review | Completed / Fixed |
| IT-6 | Bounded CPU workload | Completed / Fixed |
| IT-7 | Isolated filesystem capacity exhaustion | Completed / Fixed |
| IT-8 | Nginx upstream failure — HTTP 502 | Completed / Fixed |
| IT-9 | Host firewall blocking HTTP | Completed / Fixed |
| IT-10 | Missing AWS HTTP ingress rule | Completed / Fixed |
| IT-11 | DNS NXDOMAIN investigation | Completed / Fixed |

All eleven records were confirmed in the Jira work-item list.

These are controlled lab records. IT-5 retains hardening findings, and IT-11 required no repair.

## Troubleshooting Commands

| Command | Purpose |
| --- | --- |
| systemctl | Inspect and manage services |
| nginx -t / nginx -T | Validate or inspect Nginx configuration |
| journalctl | Review system and application logs |
| ss -lntp | Inspect listening TCP ports |
| curl -I | Send a HEAD request and display response headers |
| curl -i | Display response headers and body |
| stat / namei -l | Inspect ownership, permissions, and path access |
| ps / top | Inspect processes and resource usage |
| nproc | Report available processing units |
| pgrep | Find matching processes |
| timeout / nice | Bound execution and adjust scheduling priority |
| id / getent | Inspect identity, groups, or name-service results |
| ssh-keygen | Generate keys and inspect fingerprints |
| sshd -T | Inspect SSH settings |
| chmod / chown | Apply targeted permission or ownership changes |
| ufw status numbered | Inspect ordered host firewall rules |
| aa-status | Inspect AppArmor coverage |
| free -h | Inspect memory availability |
| findmnt | Identify mount points and filesystem types |
| df -h / df -i | Inspect capacity and inode availability |
| du -h | Inspect file space usage |
| mount / umount | Attach or detach the temporary filesystem |
| dd | Generate bounded test data |
| dig | Inspect DNS responses and status codes |
| tee | Save output while displaying it |

## Evidence Collection

Evidence is stored on EC2 under:

```text
~/aws-linux-support-triage-evidence/
```

Categories used include:

```text
baseline/
nginx/
permissions/
ssh/
security/
resources/
application/
network/
dns/
```

Evidence stored on EC2 or Windows is not automatically uploaded to this repository.

Before publishing copies:

- Remove private keys, passwords, tokens, and credentials.
- Review logs and screenshots for personal information.
- Redact unnecessary IP addresses and infrastructure identifiers.
- Preserve details needed to support the findings.
- Exclude output collected on the wrong machine.
- Confirm published evidence matches the documented exercise.

## Security and Change-Control Practices

- Used named accounts rather than routine direct root sessions.
- Used sudo for administrator operations.
- Ran Node.js and CPU workloads as supportlab.
- Kept application port 3000 bound to loopback.
- Preserved administrator access during authentication and firewall tests.
- Kept private SSH keys on the client.
- Limited faults to designated resources.
- Used bounded CPU and storage tests.
- Removed only the identified disposable storage file.
- Checked resource scope before cloud rule changes.
- Avoided modifying a shared network ACL.
- Documented findings separately from completed remediation.

## Important Limitations

- These are controlled simulations, not production incident claims.
- CPU recovery occurred through the configured timeout.
- Storage testing used tmpfs, not an EBS failure.
- DNS testing used a deliberately nonexistent name.
- Security findings were documented rather than fully remediated.
- Successful HTTP checks do not establish complete application functionality, security, or performance.
- The network ACL fault exercise remains pending.

## Planned Extensions

### Network ACL Troubleshooting

Complete an isolated exercise covering rule order, subnet scope, and return traffic.

### Monitoring and Alerting

Add monitoring after validating manual procedures. Demonstrate detection and recovery separately from service-state checks.

### Guarded Automation

Limit scripts to explicitly identified lab resources. Avoid broad process termination or indiscriminate file deletion.

### Cloud Architecture Practice

Additional topics include:

- Public load balancers and private application tiers.
- Load-balancer target health.
- IAM roles and temporary credentials.
- HTTPS and certificate troubleshooting.
- Recovery from partial Terraform deployment failures.

These are study topics or proposed extensions unless separately demonstrated.

## Relevance to Technical Support

This project demonstrates:

- Systematic Linux investigation.
- Service and authentication recovery.
- HTTP error diagnosis.
- CPU and filesystem analysis.
- Host-versus-cloud firewall comparison.
- DNS response interpretation.
- Security configuration assessment.
- Change-scope awareness.
- Customer-facing communication.
- Evidence collection and ticket ownership.

My troubleshooting approach is:

**Investigate → Identify the cause → Apply the appropriate action → Verify the result → Document and communicate.**

This project uses AWS and does not claim direct Vultr platform experience.

## Author

**Alexander Njoku**

- [GitHub](https://github.com/Alexjohn2023)
- [Medium technical portfolio](https://medium.com/@alex2020global)
- [LinkedIn](https://www.linkedin.com/in/alexander-njoku-62040a194/)
