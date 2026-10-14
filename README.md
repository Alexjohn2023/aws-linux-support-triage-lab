# AWS Linux Support Triage Lab

A hands-on cloud technical support project built on an Ubuntu AWS EC2 virtual machine.

This lab demonstrates Linux troubleshooting, service recovery, SSH authentication, file permissions, security configuration review, evidence collection, and ticket handling through Jira Service Management.

**Author:** Alexander Njoku  
**Infrastructure:** AWS EC2  
**Operating system:** Ubuntu  
**Web server:** Nginx  
**Client tools:** Windows Git Bash and SSH  
**Ticketing platform:** Jira Service Management  

> All fault simulations were performed on a non-production lab instance. No customer or production system was affected. This project uses AWS infrastructure and demonstrates skills transferable to other cloud support environments.

## Project Goals

- Establish a working baseline before introducing faults.
- Investigate issues using Linux commands and system logs.
- Distinguish service, application, network, and access-control failures.
- Apply targeted corrections and verify recovery.
- Preserve administrator access during authentication troubleshooting.
- Review security controls and document remaining findings.
- Communicate technical findings clearly through support tickets.
- Develop reusable incident reports and troubleshooting procedures.

## Exercise Roadmap

| Exercise | Status | Skills Demonstrated |
| --- | --- | --- |
| Baseline verification | Complete | Service state, listeners, configuration, HTTP checks |
| Nginx service outage | Complete - IT-2 | Local diagnosis and external recovery verification |
| SSH and permissions | Complete - IT-3 and IT-4 | Diagnose HTTP 403 and SSH authentication failures using permissions and logs |
| Security review | Complete - IT-5; findings documented | Review users, SSH, permissions, process owners, UFW, AWS security groups, and AppArmor coverage |
| High CPU | Planned | Identify a controlled workload and verify CPU recovery |
| Isolated filesystem full | Planned | Investigate capacity and inodes without filling the root disk |
| Application 502 error | Planned | Distinguish a working Nginx proxy from a failed upstream |
| Host firewall blocks HTTP | Planned | Diagnose host firewall rules while preserving SSH access |
| AWS security group and network ACL troubleshooting | Planned | Diagnose blocked ingress, distinguish stateful and stateless controls, and check return traffic |
| DNS lookup failure | Planned | Compare successful and failed DNS queries |

**Progress:** Four of ten roadmap categories are complete.

These cover five individual activities: baseline verification, Nginx recovery,
website permissions, SSH authentication, and security review.

Security review completion means the configuration was assessed and findings
documented. It does not mean every hardening opportunity was remediated.

## Lab Environment

| Component | Purpose |
| --- | --- |
| AWS EC2 | Hosts the Linux lab |
| Ubuntu | Operating system |
| Nginx | Serves HTTP requests |
| Windows Git Bash | Client-side SSH and external HTTP checks |
| systemd | Service management |
| journalctl | System and authentication log investigation |
| supportlab | Dedicated non-administrator account for SSH exercises |
| UFW | Host firewall |
| AppArmor | Application confinement coverage review |
| AWS security group | Cloud-level inbound and outbound traffic controls |
| Jira Service Management | Investigation notes, customer updates, and resolution |

The completed website exercises use HTTP. HTTPS, Node.js upstream deployment,
and monitoring integrations have not yet been demonstrated in this project.

## Repository Documentation

| File | Purpose |
| --- | --- |
| README.md | Project overview, progress, and verified results |
| baseline-checks.md | Server baseline procedures |
| service-unavailable.md | Nginx service outage documentation |
| website-permissions-403.md | HTTP 403 permissions incident |
| ssh-authentication-failure.md | SSH authentication incident |
| linux-security-hardening.md | Security procedures, verified controls, and follow-up findings |
| .gitignore | Excludes sensitive or unnecessary files according to its configured patterns |

## Support Workflow

The incident exercises follow this process:

1. Verify the working baseline.
2. Introduce one controlled fault.
3. Assess symptoms and affected scope.
4. Gather relevant logs and system evidence.
5. Identify the cause.
6. Apply a targeted correction.
7. Verify recovery.
8. Document the technical investigation.
9. Provide a simulated customer-facing update.
10. Complete the Jira workflow and update documentation.

Jira records for the completed incidents were created after technical recovery
to document the simulated support workflow.

The security review follows an assessment workflow: inspect controls, record
findings, verify access, and document follow-up work.

## Completed Exercise: Server Baseline

### Objective

Establish the server's working state before troubleshooting.

### Checks Performed

- Current user and operating system.
- Uptime and load average.
- Memory availability.
- Root filesystem capacity and inode usage.
- Nginx configuration validity.
- Service state.
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

Evidence was saved on the EC2 server:

```text
~/aws-linux-support-triage-evidence/baseline/server-baseline.txt
```

## IT-2: Nginx Service Outage

**Date:** October 10, 2026  
**Ticket status:** Completed  
**Resolution:** Fixed  

### Scenario and Symptoms

Nginx was intentionally stopped on the lab instance.

The service became inactive, and a local HTTP request failed to connect to port 80.

### Investigation

```bash
sudo systemctl is-active nginx
curl -I --max-time 5 http://127.0.0.1
sudo nginx -t
```

### Cause

The service had been intentionally stopped for the exercise.

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

A separate HTTP request from Windows Git Bash to the instance's public IP
also returned HTTP 200.

### Outcome

Local and external HTTP access were verified. The investigation and simulated
customer update were documented in Jira.

## IT-3: Website Permissions — HTTP 403

**Date:** October 10, 2026  
**Ticket status:** Completed  
**Resolution:** Fixed  

### Working Baseline

A dedicated test file returned HTTP 200 with ownership `root:root`
and permissions `644`:

```text
/var/www/html/support-permissions-lab.txt
```

### Controlled Fault

```bash
sudo chmod 600 /var/www/html/support-permissions-lab.txt
```

### Symptoms

Nginx remained active, but the test page returned HTTP 403 Forbidden.

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

At 19:53:52 UTC, the error log recorded:

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

Saved evidence confirmed permissions `644` and HTTP 200 locally.

No Nginx restart was required. External recovery verification was not recorded
for this incident.

### Evidence Files

```text
permissions/01-working-baseline.txt
permissions/02-permission-failure.txt
permissions/03-permission-restored.txt
```

### Support Lesson

A running web server can still fail to serve a resource. Check the HTTP response,
error logs, ownership, file permissions, and directory traversal permissions.

Permissions `644` were appropriate for this public test file, not for private
keys or sensitive application data.

## IT-4: SSH Authentication Failure

**Date:** October 10, 2026  
**Ticket status:** Completed  
**Resolution:** Fixed  

### Working Baseline

An Ed25519 key pair was generated on Windows. Only its public key was installed
for the dedicated `supportlab` account.

A fresh key-based login succeeded.

Verified controls:

- `.ssh` directory permissions: `700`.
- `authorized_keys` permissions: `600`.
- Ownership: `supportlab:supportlab`.
- No `sudo` group membership.

The original `ubuntu` administrator session remained open.

### Controlled Fault

```bash
sudo chmod 666 /home/supportlab/.ssh/authorized_keys
```

This temporarily made the lab user's authorization file writable by other users.

### Symptom

A fresh login failed:

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

SSH rejected the authorization file because other users could modify it.

### Corrective Action

```bash
sudo chmod 600 /home/supportlab/.ssh/authorized_keys
```

### Recovery Verification

A fresh key-based login succeeded. The server recorded the recovered session
opening at 20:38:45 UTC.

```bash
whoami
id
```

The output confirmed the expected non-administrator account.

No SSH service restart was required.

### Evidence Files

```text
ssh/02-key-access-baseline.txt
ssh/03-authentication-failure.txt
ssh/04-authentication-restored.txt
ssh/05-failure-log-detail.txt
```

### Support Lesson

The client error alone did not identify the cause. Server-side logs and file
permissions established why authentication failed.

A fresh login was required to verify recovery; an existing session was insufficient.

## IT-5: Linux Security Configuration Review

**Date:** October 14, 2026  
**Review status:** Complete; findings documented  

### Objective

Assess the existing security configuration and verify that SSH and HTTP access
remain available.

### Verified Controls

| Area | Finding |
| --- | --- |
| User privileges | supportlab has no sudo group membership |
| SSH public-key authentication | Enabled |
| Password authentication | Disabled |
| Keyboard-interactive authentication | Disabled |
| StrictModes | Enabled |
| SSH directories | 700 with correct user ownership |
| authorized_keys files | 600 with correct user ownership |
| Nginx workers | Run as non-root www-data |
| Nginx master | Runs as root |
| UFW | Active; default deny incoming and allow outgoing |
| UFW allowances | SSH 22 and HTTP 80 allowed for IPv4 and IPv6 |
| AWS security groups | One group attached |
| AWS SSH ingress | Restricted to the administrator's public IPv4 /32 |
| AWS HTTP ingress | Port 80 allowed from 0.0.0.0/0 |
| AWS egress | All traffic allowed |
| Access verification | Fresh supportlab SSH login succeeded |
| Website verification | External HTTP 200 at 05:35:42 UTC |

### AppArmor Coverage

The reviewed output showed:

- Two `chronyd` processes in enforce mode.
- `rsyslogd` in enforce mode.
- Amazon SSM Agent in complain mode.
- No demonstrated Nginx confinement.

An enabled AppArmor system does not establish that every application is confined.

### Follow-Up Findings

| Finding | Proposed Follow-Up |
| --- | --- |
| UFW permits SSH from any source | Evaluate narrower host-level source rules while preserving recovery access |
| PermitRootLogin is prohibit-password | Evaluate disabling direct root login after verifying named administrator access |
| AWS outbound traffic is unrestricted | Assess required traffic before introducing egress restrictions |
| Nginx confinement was not demonstrated | Develop and validate an appropriate AppArmor profile separately |

These are hardening opportunities requiring assessment, not evidence that the
server was compromised.

### Commands Used

```bash
id supportlab
getent group sudo
getent group support-lab

sudo sshd -T | grep -E \
'^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin|strictmodes) '

sudo stat -c '%A %a %U:%G %n' \
/home/ubuntu/.ssh \
/home/ubuntu/.ssh/authorized_keys \
/home/supportlab/.ssh \
/home/supportlab/.ssh/authorized_keys

ps -eo user,group,pid,comm | grep -E '(sshd|nginx)'

sudo ufw status verbose
sudo aa-status
```

### Outcome

The review, internal findings, and simulated customer update were documented
in IT-5. The ticket was completed.

The review did not change these settings or remediate the follow-up findings.
Its completion must not be interpreted as a claim that all hardening is complete.

## Jira Support Records

| Ticket | Activity | Outcome |
| --- | --- | --- |
| IT-2 | Nginx service outage | Completed / Fixed |
| IT-3 | Website permissions — HTTP 403 | Completed / Fixed |
| IT-4 | SSH authorization-file permissions | Completed / Fixed |
| IT-5 | Security configuration review | Completed; findings remain documented |

Records include scope, technical findings, verification, internal notes,
simulated customer updates, and ownership.

These records describe lab activities rather than customer production incidents.

## Troubleshooting Commands

| Command | Purpose |
| --- | --- |
| systemctl is-active | Check service state |
| nginx -t | Validate Nginx configuration |
| nginx -T | Inspect complete Nginx configuration |
| journalctl | Review system and authentication logs |
| ss -lntp | Inspect listening TCP ports |
| curl -I / curl -i | Inspect HTTP headers or headers and body |
| stat | Inspect ownership and permissions |
| namei -l | Inspect permissions along a path |
| ps | Inspect processes and owners |
| id / getent | Inspect users and groups |
| ssh-keygen | Generate keys and inspect fingerprints |
| sshd -T | Inspect SSH daemon settings |
| chmod / chown | Apply targeted permissions or ownership changes |
| ufw status verbose | Inspect host firewall policy |
| aa-status | Inspect AppArmor profile coverage |
| free -h | Check memory availability |
| df -h / df -i | Check capacity and inode usage |
| tee | Save output while displaying it |

## Evidence Collection

Evidence is collected on the EC2 server under:

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
security/
```

Evidence files stored on EC2 are not automatically published to this repository.

Before uploading copies:

- Remove credentials, private keys, passwords, and tokens.
- Review screenshots and logs for personal information.
- Redact unnecessary IP addresses and infrastructure identifiers.
- Preserve the details needed to substantiate the findings.
- Confirm uploaded files match the documented exercise.

## Security Practices Demonstrated

- Used a named administrator account for privileged operations.
- Used a dedicated non-administrator account for SSH testing.
- Kept private keys on the client.
- Verified SSH file ownership and permissions.
- Preserved administrator access during authentication testing.
- Limited faults to designated lab resources.
- Confirmed Nginx workers run as a non-root account.
- Compared host firewall and cloud security controls.
- Distinguished verified controls from pending hardening work.
- Verified recovery before closing incident tickets.

The temporary `666` authorization-file setting was a controlled fault and was
restored to `600`.

## Planned Extensions

### High CPU

Generate a bounded workload, inspect its process and resource usage, stop only
the identified lab workload, and verify recovery.

### Isolated Filesystem Full

Use a small isolated filesystem to investigate capacity and inode usage without
filling the server's root filesystem.

### Application 502

Deploy a Node.js application under a non-root service account behind Nginx.
Simulate an upstream outage and distinguish proxy health from application health.

### Host Firewall Blocks HTTP

Introduce a scoped HTTP rule fault, compare local and external access, and
restore connectivity while preserving SSH.

### AWS Security Group and Network ACL Troubleshooting

Simulate faults separately on designated lab resources.

Check security group ingress, network ACL rule ordering, and return traffic.
Preserve recovery access and avoid changing shared subnet controls.

### DNS Lookup Failure

Compare successful and failed queries and distinguish name-resolution failures
from web-server and network failures.

### Monitoring and Automation

Validate manual troubleshooting before introducing automation.

Recovery scripts will act only on explicitly identified lab resources rather
than broadly killing processes or deleting files.

## Relevance to Cloud Technical Support

This project demonstrates:

- Linux command-line investigation.
- Evidence-based diagnosis.
- Service and authentication recovery.
- Security configuration assessment.
- Customer-facing communication.
- Technical documentation.
- Ticket ownership through completion.

The lab supports discussion of troubleshooting methods in cloud technical
support interviews. It does not claim direct Vultr platform experience.

## Author

**Alexander Njoku**

- [GitHub](https://github.com/Alexjohn2023)
- [Medium technical portfolio](https://medium.com/@alex2020global)
- [LinkedIn](https://www.linkedin.com/in/alexander-njoku-62040a194/)
