# AWS Linux Support Triage Lab

A hands-on technical support lab built on an Ubuntu AWS EC2 virtual machine. This project demonstrates how I investigate Linux service, website-access, and SSH authentication failures, restore service, collect evidence, and communicate through Jira Service Management.

**Author:** Alexander Njoku  
**Environment:** Ubuntu on AWS EC2  
**Access tools:** Windows Git Bash and SSH  
**Web server:** Nginx  
**Ticketing:** Jira Service Management  

> All incidents are controlled exercises on a non-production lab instance. No customer or production system was affected. This project uses AWS infrastructure; the troubleshooting skills are relevant to cloud support environments, including Vultr.

## Project Goals

- Establish a working server baseline before introducing faults.
- Diagnose issues using Linux commands and system logs.
- Distinguish service, application, network, and access-control failures.
- Apply targeted corrective actions and verify recovery.
- Preserve administrator access during SSH troubleshooting.
- Document investigations, customer updates, and resolutions in Jira.
- Develop reusable troubleshooting procedures.

## Exercise Roadmap

| Exercise | Status | Skills Demonstrated |
| --- | --- | --- |
| Baseline verification | Complete | Service state, listeners, configuration, HTTP checks |
| Nginx service outage | Complete — IT-2 | Local diagnosis and external recovery verification |
| High CPU | Planned | Identify a controlled workload and verify CPU recovery |
| Isolated filesystem full | Planned | Investigate capacity and inodes without filling the root disk |
| Application 502 error | Planned | Distinguish a working Nginx proxy from a failed upstream |
| SSH and permissions | Complete — IT-3 and IT-4 | Diagnose HTTP 403 and SSH authentication failures using file permissions and logs |
| Host firewall blocks HTTP | Planned | Compare host firewall rules and cloud security controls |
| DNS lookup failure | Planned | Compare successful and failed DNS queries |
| Security review | Ongoing | Review SSH, users, UFW, and AppArmor |

Three roadmap categories are complete, covering four individual exercises:
baseline verification, Nginx recovery, website permissions, and SSH authentication.

## Lab Environment

| Component | Purpose |
| --- | --- |
| AWS EC2 | Hosts the Linux lab |
| Ubuntu | Operating system |
| Nginx | Serves HTTP requests |
| Windows Git Bash | Runs the local SSH client and external HTTP checks |
| systemd and journalctl | Service management and log investigation |
| Dedicated supportlab user | Isolates SSH authentication exercises |
| Jira Service Management | Records investigation, communication, and resolution |

The completed website exercises use HTTP. HTTPS, upstream application deployment,
and monitoring integrations are not yet demonstrated in this project.

## Support Workflow

Each incident follows this process:

1. Verify and record the working baseline.
2. Introduce a controlled fault.
3. Assess the symptoms and scope.
4. Gather service, log, permission, or connectivity evidence.
5. Identify the cause.
6. Apply a targeted correction.
7. Verify recovery.
8. Record technical findings and a customer-facing update.
9. Resolve the Jira ticket.
10. Update the project documentation.

For these exercises, Jira records were created after technical recovery to
document the simulated support workflow.

## Completed Exercise: Server Baseline

### Objective

Confirm the server's health and establish reference evidence before troubleshooting.

### Checks Performed

- Operating system and current user.
- Uptime and load average.
- Available memory.
- Root filesystem capacity and inode usage.
- Nginx configuration validity.
- Nginx service state.
- Listening TCP ports.
- Local HTTP response.

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
curl -I --max-time 5 http://127.0.0.1
```

### Verified Result

Nginx configuration passed validation, the service was active, port 80 was
listening, and the local website returned HTTP 200.

Baseline evidence was saved on the lab server as:

```text
~/aws-linux-support-triage-evidence/baseline/server-baseline.txt
```

## IT-2: Nginx Service Outage

**Date:** October 10, 2026  
**Jira status:** Completed  
**Resolution:** Fixed  

### Scenario

Nginx was intentionally stopped to simulate an unavailable website on the test instance.

### Symptoms

- Nginx was inactive.
- A local HTTP request failed to connect to port 80.

### Investigation

Checked service state and validated the Nginx configuration before recovery.

```bash
sudo systemctl is-active nginx
curl -I --max-time 5 http://127.0.0.1
sudo nginx -t
```

### Cause

The Nginx service had been intentionally stopped for the exercise.

### Corrective Action

```bash
sudo nginx -t && sudo systemctl start nginx
```

### Recovery Verification

```bash
sudo systemctl is-active nginx
sudo ss -lntp '( sport = :80 )'
curl -I --max-time 5 http://127.0.0.1
```

A separate request from Windows Git Bash to the instance's public IP also
returned HTTP 200.

### Outcome

Local and external HTTP access were verified. The investigation and
customer-facing recovery update were documented in Jira.

## IT-3: Website Permissions — HTTP 403 Forbidden

**Date:** October 10, 2026  
**Jira status:** Completed  
**Resolution:** Fixed  

### Scenario

A dedicated test file was used to simulate a website access failure without
stopping the web server.

### Working Baseline

The test file was owned by `root:root`, had permissions `644`, and returned HTTP 200.

```text
/var/www/html/support-permissions-lab.txt
```

### Controlled Fault

```bash
sudo chmod 600 /var/www/html/support-permissions-lab.txt
```

This restricted read access to the root owner.

### Symptoms

- Nginx remained active.
- The test page returned HTTP 403 Forbidden.

### Investigation

```bash
sudo systemctl is-active nginx

sudo stat -c '%A %a %U:%G %n' \
/var/www/html/support-permissions-lab.txt

ps -eo user,pid,comm | grep '[n]ginx'

curl -i --max-time 5 \
http://127.0.0.1/support-permissions-lab.txt

sudo tail -n 20 /var/log/nginx/error.log

namei -l /var/www/html/support-permissions-lab.txt
```

At 19:53:52 UTC, the Nginx error log recorded:

```text
open() "/var/www/html/support-permissions-lab.txt" failed
(13: Permission denied)
```

### Cause

The Nginx worker account could not read the root-owned file with permissions `600`.

### Corrective Action

```bash
sudo chmod 644 /var/www/html/support-permissions-lab.txt
```

### Recovery Verification

The saved recovery evidence confirmed permissions `644` and HTTP 200 locally.

No Nginx restart was required.

### Evidence Saved on the Lab Server

```text
permissions/01-working-baseline.txt
permissions/02-permission-failure.txt
permissions/03-permission-restored.txt
```

### Support Lesson

An active web-server service does not prove that every page is accessible.
HTTP responses, error logs, ownership, and permissions must be checked together.

## IT-4: SSH Key Authentication Failure

**Date:** October 10, 2026  
**Jira status:** Completed  
**Resolution:** Fixed  

### Scenario

A dedicated non-administrator account, `supportlab`, was configured for
Ed25519 public-key authentication from Windows Git Bash.

The original `ubuntu` administrator session remained open throughout the exercise.

### Working Baseline

- A fresh key-based login succeeded as `supportlab`.
- The user's `.ssh` directory had permissions `700`.
- Its `authorized_keys` file had permissions `600`.
- Ownership was `supportlab:supportlab`.
- The account had no `sudo` group membership.

### Controlled Fault

```bash
sudo chmod 666 /home/supportlab/.ssh/authorized_keys
```

This temporarily made the lab user's authorization file writable by other users.

### Symptom

A fresh SSH connection failed:

```text
Permission denied (publickey).
```

### Investigation

```bash
sudo stat -c '%A %a %U:%G %n' \
/home/supportlab/.ssh/authorized_keys

sudo journalctl --since "2026-10-10 20:35:00 UTC" \
--until "2026-10-10 20:37:30 UTC" --no-pager
```

At 20:36:34 UTC, the system journal recorded:

```text
Authentication refused: bad ownership or modes for file
/home/supportlab/.ssh/authorized_keys
```

Saved evidence confirmed permissions `666`.

### Cause

SSH rejected the authorization file because its permissions allowed other users
to modify it.

### Corrective Action

```bash
sudo chmod 600 /home/supportlab/.ssh/authorized_keys
```

### Recovery Verification

A fresh SSH login succeeded after the correction. The server recorded the
`supportlab` session opening at 20:38:45 UTC.

```bash
whoami
id
```

The output confirmed the expected non-administrator account.

No SSH service restart was required.

### Evidence Saved on the Lab Server

```text
ssh/02-key-access-baseline.txt
ssh/03-authentication-failure.txt
ssh/04-authentication-restored.txt
ssh/05-failure-log-detail.txt
```

### Support Lesson

A public-key authentication failure can be caused by server-side ownership or
permissions. Client errors should be investigated alongside server logs.

A fresh connection is necessary to verify recovery; an existing SSH session
does not prove that new authentication attempts will succeed.

## Jira Incident Management

| Ticket | Exercise | Final Status | Resolution |
| --- | --- | --- | --- |
| IT-2 | Nginx service outage | Completed | Fixed |
| IT-3 | Website permissions — HTTP 403 | Completed | Fixed |
| IT-4 | SSH authorization-file permissions | Completed | Fixed |

Each ticket includes:

- Scope and symptoms.
- Investigation and identified cause.
- Corrective action.
- Recovery verification.
- Internal technical note.
- Customer-facing update for the simulated scenario.
- Assignment and final resolution.

These are lab tickets, not records of customer production incidents.

## Troubleshooting Commands Used

| Command | Purpose |
| --- | --- |
| `systemctl is-active` | Check service state |
| `nginx -t` | Validate Nginx configuration |
| `nginx -T` | Inspect the complete Nginx configuration |
| `journalctl` | Review system and service logs |
| `ss -lntp` | Inspect listening TCP ports |
| `curl -I` | Inspect HTTP response headers |
| `curl -i` | Inspect HTTP headers and response body |
| `stat` | Check file ownership and permissions |
| `namei -l` | Inspect permissions along a file path |
| `ps` | Inspect processes and their owners |
| `id` | Verify user and group membership |
| `ssh-keygen` | Generate keys and inspect fingerprints |
| `chmod` | Change file permissions |
| `chown` | Set file ownership |
| `free -h` | Check memory availability |
| `df -h` | Check filesystem capacity |
| `df -i` | Check inode usage |
| `tee` | Save command output while displaying it |

## Evidence Collection

Evidence is stored on the lab server under:

```text
~/aws-linux-support-triage-evidence/
```

Categories include:

```text
baseline/
nginx/
network/
permissions/
resources/
ssh/
```

Evidence files saved on EC2 are not automatically uploaded to this repository.
Sanitized copies will be added separately.

Before publishing evidence:

- Remove private keys, passwords, tokens, and other credentials.
- Review logs and screenshots for personal or customer information.
- Redact unnecessary IP addresses and infrastructure identifiers.
- Preserve the technical details needed to support the findings.

## Security Practices Demonstrated

- Used a named administrator account with `sudo` for privileged commands.
- Created a separate non-administrator user for SSH testing.
- Generated a dedicated Ed25519 key pair on the client computer.
- Kept the private key on the client.
- Applied appropriate ownership and permissions to SSH files.
- Preserved administrator access during authentication testing.
- Limited faults to designated lab resources.
- Verified recovery before resolving tickets.

The intentional `666` permission setting was a temporary fault simulation.
It was restored to `600` immediately after evidence collection.

## Planned Extensions

### High CPU

Generate a bounded workload, identify its process and resource usage, stop the
known lab workload, and verify recovery.

### Isolated Filesystem Full

Use a small, isolated test filesystem to investigate capacity and inode
conditions without filling the server's root filesystem.

### Application 502

Deploy a Node.js application behind Nginx, simulate an upstream outage, and
distinguish proxy health from application health.

### Host Firewall Blocks HTTP

Compare local and external connectivity, host firewall rules, and AWS security
controls while preserving SSH access.

### DNS Lookup Failure

Compare successful and failed DNS queries and distinguish name-resolution
problems from HTTP connectivity failures.

### Monitoring and Automation

Add monitoring only after validating the manual troubleshooting procedures.
Any recovery automation will be limited to explicitly identified lab resources.

## Relevance to Technical Support

This project demonstrates:

- Linux command-line investigation.
- Evidence-based diagnosis.
- Service and authentication recovery.
- Security-conscious access management.
- Clear technical documentation.
- Customer-facing communication.
- Ticket ownership through resolution.

The lab provides practical examples for cloud technical support interviews.
It does not claim direct Vultr platform experience.

## Author

**Alexander Njoku**

- [GitHub](https://github.com/Alexjohn2023)
- [Medium technical portfolio](https://medium.com/@alex2020global)
- [LinkedIn](https://www.linkedin.com/in/alexander-njoku-62040a194/)
