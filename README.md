# AWS Linux Support Triage Lab

A hands-on Linux troubleshooting and server-security lab running on an Ubuntu EC2 virtual machine.

This project demonstrates a technical support workflow: identify impact, investigate the relevant system layer, collect evidence, restore service, verify recovery, and document findings.

## Project Status

- Initial Nginx deployment and baseline verification: complete.
- First Nginx service-outage exercise: complete on October 10, 2026.
- GitHub account connection to Atlassian: complete.
- Jira-to-commit linking: awaiting verification.
- Additional incident exercises: planned.

## Objectives

- Practice Linux troubleshooting on an AWS EC2 instance.
- Distinguish service, network, DNS, permissions, storage, and resource issues.
- Collect evidence before and after corrective actions.
- Verify recovery from both the server and an external workstation.
- Document incidents and track related work in Jira.
- Maintain troubleshooting documentation in GitHub.

## Environment

| Component | Details |
| --- | --- |
| Cloud platform | AWS |
| Compute | EC2 virtual machine |
| Operating system | Ubuntu Server 24.04 LTS |
| Web server | Nginx 1.28.3 |
| Current web content | Default Nginx welcome page |
| Remote access | SSH |
| Support tracking | Jira |
| Version control | Git and GitHub |

Docker is not used in this project.

## Architecture

HTTP requests from the workstation pass through the EC2 security group and the Ubuntu host firewall, when enabled, before reaching Nginx on port 80.

Administrative access uses SSH on port 22. The EC2 security-group SSH rule should be restricted to the administrator's current public IP address.

The EC2 security group and Ubuntu host firewall are separate controls. Both must permit the required traffic.

## Repository Contents

| File | Purpose |
| --- | --- |
| README.md | Project overview, progress, and exercise roadmap |
| baseline-checks.md | Initial server and web-service verification |
| service-unavailable.md | Service troubleshooting notes and incident documentation |
| linux-security-hardening.md | Linux security checks and hardening notes |
| .gitignore | Exclusions for files that should not enter version control |

Individual incident documents are updated as evidence and recovery steps are recorded.

## Completed Baseline Verification

The following checks were completed on October 5, 2026:

| Check | Result |
| --- | --- |
| `sudo nginx -t` | Configuration syntax valid; test successful |
| `sudo systemctl status nginx --no-pager` | Nginx active and running |
| `sudo ss -lntp` | Nginx listening on port 80 over IPv4 and IPv6 |
| `curl -I http://localhost` | HTTP/1.1 200 OK |
| Browser test from workstation | Default Nginx welcome page loaded |

These results established a working baseline before incident exercises.

## Completed Exercise: Nginx Service Outage

**Date:** October 10, 2026  
**Environment:** Personal AWS lab  
**Result:** HTTP service recovery verified

### Symptoms and Evidence

During the outage, the following command returned `inactive`:

    sudo systemctl is-active nginx

A local HTTP check failed to connect to port 80:

    curl -I --max-time 5 http://127.0.0.1

This confirmed that HTTP requests failed locally while the Nginx service was inactive.

### Finding

Nginx was not running at the time of the failed local check.

The observed service state explains the unavailable local HTTP endpoint. These checks alone do not establish why the service stopped.

### Recovery Verification

After Nginx service recovery, HTTP checks against the instance's public endpoint returned:

    HTTP/1.1 200 OK

Successful checks were performed from:

- The EC2 instance.
- An external Windows workstation using Git Bash.

The external test confirmed that the HTTP endpoint was reachable through the network path at the time of verification.

### Lessons Learned

- Check service state and local connectivity early.
- Separate a local service failure from a network-access problem.
- Use an external workstation to verify end-to-end recovery.
- Distinguish the observed failure from its underlying cause.
- Record the exact corrective commands in the incident report.

## Incident Workflow

For each exercise:

1. **Identify impact:** Record the affected service, symptoms, scope, and time.
2. **Collect evidence:** Capture relevant command output before making changes.
3. **Investigate:** Check the service, host, network, DNS, permissions, or storage layer.
4. **Determine the finding:** Separate confirmed facts from hypotheses.
5. **Restore service:** Apply a targeted corrective action.
6. **Verify recovery:** Repeat the failed test and check externally where appropriate.
7. **Document:** Record evidence, actions, results, and follow-up work.
8. **Update Jira:** Link the relevant repository changes to the incident ticket.

## Exercise Roadmap

| Exercise | Status | Skills Demonstrated |
| --- | --- | --- |
| Baseline verification | Complete | Service state, listeners, configuration, HTTP checks |
| Nginx service outage | Complete | Local diagnosis and external recovery verification |
| High CPU | Planned | Identify a controlled workload and verify CPU recovery |
| Isolated filesystem full | Planned | Investigate capacity and inodes without filling the root disk |
| Application 502 error | Planned | Distinguish a working Nginx proxy from a failed upstream |
| SSH and permissions | Planned | Diagnose access failures using an isolated test setup |
| Host firewall blocks HTTP | Planned | Compare host firewall rules and cloud security controls |
| DNS lookup failure | Planned | Compare successful and failed DNS queries |
| Security review | Ongoing | Review SSH, users, UFW, and AppArmor |

The application 502 exercise will require a test upstream application. The current deployment serves the default Nginx page.

## Jira and GitHub Integration

The GitHub account `Alexjohn2023` is connected to Atlassian through GitHub for Atlassian.

To associate repository work with a Jira ticket, include the correct ticket key in the branch name, commit message, or pull request title.

Example commit message, if `IT-2` is the corresponding incident:

    IT-2 Document Nginx outage troubleshooting and recovery

After pushing the commit, verify that it appears on the corresponding Jira work item. The account connection is complete; commit linking still needs to be verified.

## Lab Safety

- Perform controlled failures only in the lab.
- Keep SSH restricted to the administrator's IP address.
- Ensure SSH is allowed before enabling or changing UFW.
- Maintain an EC2 recovery option before testing remote-access changes.
- Use bounded workloads for CPU exercises.
- Use a small, isolated filesystem for disk-full exercises.
- Do not fill the root filesystem.
- Do not flush firewall rules or restart networking without a recovery plan.
- Use test files and accounts for permission exercises.
- Review AppArmor changes carefully and verify Nginx afterward.

## Evidence and Privacy

Before publishing evidence:

- Remove credentials, tokens, private keys, and AWS account IDs.
- Redact public IP addresses and sensitive infrastructure details.
- Review screenshots and command output for personal information.
- Never commit `.pem` files or secret-bearing configuration files.

Use placeholders such as `<EC2_PUBLIC_IP>` in published examples.

## Cost Management

Stop the EC2 instance when it is not needed. Review remaining storage and other allocated resources, and monitor AWS Billing for ongoing costs.

## Completion Criteria

An incident exercise is complete when:

- The failure has been observed.
- Diagnostic evidence has been collected.
- The finding is supported by evidence.
- Corrective actions have been recorded.
- Recovery has been verified.
- The incident documentation and Jira ticket have been updated.

## Author

Alexander Njoku  
GitHub: https://github.com/Alexjohn2023
