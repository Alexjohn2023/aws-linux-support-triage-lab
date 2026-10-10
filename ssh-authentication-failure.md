# Incident: SSH Key Login Rejected — Unsafe File Permissions

## Incident Summary

| Field | Detail |
| --- | --- |
| Jira ticket | IT-4 |
| Date | October 10, 2026 |
| Environment | Ubuntu AWS EC2 lab instance |
| Client | Windows Git Bash |
| Affected account | supportlab |
| Authentication | Ed25519 SSH key |
| Impact | Minor / Localized |
| Urgency | Low |
| Status | Completed |
| Resolution | Fixed |

> This was a controlled exercise on a non-production instance. No customer or production system was affected.

## Objective

Investigate an SSH public-key authentication failure, identify its cause using server logs, and restore access while preserving the existing administrator session.

## Lab Setup

A dedicated non-administrator account, `supportlab`, was used to isolate the exercise.

An Ed25519 key pair was generated on the Windows client. Only the public key was installed on the server.

The private key remained on the client and must not be uploaded to GitHub.

## Working Baseline

A fresh SSH login succeeded as `supportlab`.

Verified account information:

```text
uid=1001(supportlab) gid=1001(supportlab)
groups=1001(supportlab),100(users)
```

The account had no `sudo` group membership.

Expected server-side ownership and permissions:

| Resource | Owner | Permissions |
| --- | --- | --- |
| /home/supportlab/.ssh | supportlab:supportlab | 700 |
| /home/supportlab/.ssh/authorized_keys | supportlab:supportlab | 600 |

The original `ubuntu` administrator session remained open throughout the exercise.

## Controlled Fault

In the administrator session, changed the lab user's public-key authorization file to permissions `666`:

```bash
sudo chmod 666 /home/supportlab/.ssh/authorized_keys
```

This temporarily allowed other users to modify the file.

> This permission setting was an intentional lab fault. It is not a recommended configuration.

## Symptoms

After exiting the existing `supportlab` session, a fresh login was attempted from Windows Git Bash:

```bash
ssh -o IdentitiesOnly=yes \
-o PreferredAuthentications=publickey \
-i ~/.ssh/alex_support_lab_ed25519 \
supportlab@YOUR_INSTANCE_PUBLIC_IP
```

The connection failed:

```text
Permission denied (publickey).
```

Replace `YOUR_INSTANCE_PUBLIC_IP` with the instance's current public IP when reproducing the lab.

## Investigation

### Inspect File Ownership and Permissions

From the preserved `ubuntu` session:

```bash
sudo stat -c '%A %a %U:%G %n' \
/home/supportlab/.ssh/authorized_keys
```

Saved evidence showed:

```text
-rw-rw-rw- 666 supportlab:supportlab /home/supportlab/.ssh/authorized_keys
```

### Review SSH Configuration

```bash
sudo sshd -T | grep '^strictmodes '
```

`StrictModes` checks ownership and permissions associated with user authentication files. Configuration should be reviewed alongside logs and the actual file state.

### Review Server Logs

The initial service-filtered journal output did not expose the decisive error in the results reviewed. Searching the full system journal around the failed attempt revealed it:

```bash
sudo journalctl --since "2026-10-10 20:35:00 UTC" \
--until "2026-10-10 20:37:30 UTC" --no-pager |
grep -Ei 'sshd|authentication refused|bad ownership|bad modes'
```

At 20:36:34 UTC, the server recorded:

```text
Authentication refused: bad ownership or modes for file
/home/supportlab/.ssh/authorized_keys
```

## Root Cause

The `authorized_keys` file was writable by other users.

SSH rejected the file because of its unsafe permissions. The server log directly identified the authorization-file permissions problem.

The client message alone did not establish the cause; server-side evidence was needed.

## Corrective Action

Restored permissions from `666` to `600`:

```bash
sudo chmod 600 /home/supportlab/.ssh/authorized_keys
```

Ownership remained `supportlab:supportlab`.

No SSH service restart, firewall change, or key replacement was required.

## Recovery Verification

A fresh connection was attempted from Windows Git Bash using the same private key:

```bash
ssh -o IdentitiesOnly=yes \
-o PreferredAuthentications=publickey \
-i ~/.ssh/alex_support_lab_ed25519 \
supportlab@YOUR_INSTANCE_PUBLIC_IP
```

After successful login:

```bash
whoami
id
```

Verified username:

```text
supportlab
```

The server recorded the recovered session opening at 20:38:45 UTC.

The account remained a non-administrator.

## Timeline

All times are UTC on October 10, 2026.

| Time | Event |
| --- | --- |
| 20:34:18 | Server recorded successful public-key authentication before the fault |
| 20:36:34 | Fresh login rejected; server logged unsafe ownership or modes |
| 20:36:57 | Saved file-state evidence confirmed permissions 666 |
| 20:38:45 | Server recorded a successful session after permissions were restored |

## Evidence

Evidence files were saved under:

```text
~/aws-linux-support-triage-evidence/ssh/
```

Relevant filenames:

```text
02-key-access-baseline.txt
03-authentication-failure.txt
04-authentication-restored.txt
05-failure-log-detail.txt
```

Evidence saved on EC2 is not automatically included in this repository.
Review and sanitize logs before uploading copies.

Do not upload private keys, passphrases, tokens, or credentials.

## Jira Documentation

IT-4 includes:

- Scope and working baseline.
- Controlled authentication failure.
- Server-side diagnosis.
- Corrective action.
- Fresh-login recovery verification.
- Internal technical note.
- Simulated customer-facing update.
- Assignee: Alexander Njoku.
- Final status: Completed.
- Resolution: Fixed.

## Customer-Facing Recovery Update

“SSH access to the lab account has been restored. The public-key authorization file had permissions that allowed other users to modify it, so SSH rejected it as a security precaution. I corrected the permissions and verified a successful new login using the existing key. This was a controlled lab exercise; no customer or production system was affected.”

## Security Practices

- Used a dedicated non-administrator account.
- Generated the private key on the client.
- Installed only the public key on the server.
- Preserved the original administrator session.
- Limited the fault to the dedicated lab account.
- Restored secure permissions immediately after collecting evidence.
- Tested recovery with a new connection.
- Kept credentials out of documentation and source control.

## Prevention and Support Lessons

- Check the username and selected client key.
- Inspect server-side ownership and permissions.
- Review SSH logs before replacing keys or changing configuration.
- Keep `.ssh` directories appropriately restricted.
- Prevent group or other users from writing to `authorized_keys`.
- Preserve a working administrator or recovery session during access changes.
- Verify the server's host fingerprint before accepting an unfamiliar host.
- Never disable host-key verification to bypass an identity warning.
- Do not disable SSH security checks to work around unsafe permissions.
- A working existing session does not prove that new logins will succeed.

## Skills Demonstrated

- SSH key generation and authentication.
- Linux users and group membership.
- Secure file ownership and permissions.
- System journal investigation.
- Root-cause identification.
- Recovery verification.
- Jira incident handling.
- Customer communication.
