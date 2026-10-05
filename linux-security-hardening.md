# Linux Server Security and Hardening

This guide documents a cautious first-response security review for the Ubuntu EC2 host in this lab. It focuses on collecting evidence and applying least privilege without risking remote access. The checks below are documentation for the project; record a change as completed only after performing and verifying it on the instance.

## Hardening goals

- Keep administrative access limited to authorized users and SSH keys.
- Expose only required network services.
- Apply security updates through a controlled maintenance process.
- Use least privilege for users, files, and services.
- Use Ubuntu's AppArmor protections and review system logs.
- Keep a recovery path before making remote access or firewall changes.

## 1. Record the system baseline

Run read-only checks first and save relevant, redacted output with a timestamp:

```bash
cat /etc/os-release
uname -r
whoami
id
last -a | head
sudo systemctl --failed
sudo ss -lntup
```

Review the output to identify the OS release, kernel, current account privileges, recent login records, failed units, and listening services. A listening port is evidence of a service; it is not by itself proof that the service is exposed publicly. Check the AWS security group and Ubuntu host firewall separately.

## 2. Review SSH access safely

Inspect the effective SSH daemon settings and authorized keys without printing private keys:

```bash
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|allowgroups) '
getent passwd | awk -F: '$3 == 0 {print $1}'
sudo find /home -maxdepth 3 -name authorized_keys -type f -print
```

Confirm that direct root login is disabled where appropriate, key-based access works, and only expected accounts have administrative access. Never copy a private key into the repository. Do not change the SSH port or restart SSH from your only remote session. Before any access-policy change, confirm a second working session or EC2 console recovery path and keep the current session open until the new access is verified.

## 3. Review network exposure and firewall state

Check the host firewall without changing it:

```bash
sudo ufw status verbose
sudo iptables -S
sudo nft list ruleset
```

The active firewall tooling varies by host; empty output or an inactive firewall needs interpretation in context. In AWS, review the instance security group separately. Allow only required inbound traffic; restrict SSH (TCP 22) to trusted source addresses where practical. Keep the rule for the lab's web service only while it is needed. Do not flush firewall rules or enable UFW until you have verified an SSH allow rule and a recovery path.

## 4. Review accounts, privileges, and file permissions

```bash
getent passwd
getent group sudo
sudo find /etc/ssh -maxdepth 1 -type f -printf '%m %u:%g %p\n'
stat -c '%A %a %U:%G %n' ~/.ssh ~/.ssh/authorized_keys 2>/dev/null
```

Investigate unexpected accounts, sudo membership, writable configuration files, or loose SSH key permissions. Make account changes only after confirming ownership and business need. Avoid broad recursive permission changes such as `chmod -R 777`.

## 5. Check updates and AppArmor

```bash
apt list --upgradable
sudo systemctl status apparmor --no-pager
sudo aa-status
```

Schedule security updates using the host's normal maintenance process and verify important services afterward. Ubuntu commonly uses AppArmor. Review its profile status and denials in system logs before changing a policy. Test a policy adjustment in complain mode in a controlled window, then return it to enforce mode and verify the application. Do not migrate this Ubuntu host to SELinux as a troubleshooting shortcut.

## 6. Review security-relevant logs

```bash
sudo journalctl -p warning..alert --since '24 hours ago' --no-pager
sudo journalctl -u ssh --since '24 hours ago' --no-pager
sudo journalctl -u nginx --since '24 hours ago' --no-pager
sudo grep -Ei 'apparmor|denied|authentication failure|failed password' /var/log/syslog | tail -50
```

Log paths and service names can vary. Redact public IP addresses, usernames, hostnames, tokens, and customer data before publishing examples. Treat an alert as a lead to investigate; correlate timestamps and service behavior before concluding that an attack or outage occurred.

## 7. Verify service health after an approved change

For the Nginx service used by this lab:

```bash
sudo nginx -t
sudo systemctl is-active nginx
sudo ss -lntp | grep ':80'
curl -I http://localhost
```

Compare results with the recorded baseline and test from the intended client path. Record the change, outcome, and any rollback performed. The project's initial baseline showed Nginx running and returning HTTP 200 locally; additional security exercises are not marked complete until they are performed and verified.

## Change and incident record template

- **Date/time and timezone:**
- **System and scope:**
- **Reason for review/change:**
- **Evidence collected:**
- **Change made (or “read-only review”):**
- **Verification result:**
- **Rollback or recovery path:**
- **Customer/internal update:**
- **Follow-up or escalation:**

## Safety boundaries

Do not publish credentials, private keys, AWS account IDs, public IP addresses, or unredacted console screenshots. Avoid risky remote exercises such as flushing firewall rules, locking down SSH without a verified recovery path, filling the root filesystem, or restarting networking over the only SSH connection.
