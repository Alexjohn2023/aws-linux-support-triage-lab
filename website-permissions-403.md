# Incident: Nginx HTTP 403 — Incorrect File Permissions

## Incident Summary

| Field | Detail |
| --- | --- |
| Jira ticket | IT-3 |
| Date | October 10, 2026 |
| Environment | Ubuntu AWS EC2 lab instance |
| Affected resource | Dedicated Nginx test page |
| Impact | Minor / Localized |
| Urgency | Low |
| Status | Completed |
| Resolution | Fixed |

> This was a controlled exercise on a non-production instance. No customer or production system was affected.

## Objective

Diagnose a website access failure where Nginx remains running but cannot read a requested file.

## Working Baseline

The test file was:

```text
/var/www/html/support-permissions-lab.txt
```

It contained:

```text
Linux support lab: permissions test page
```

At 19:52:02 UTC:

- Nginx was active.
- The file belonged to `root:root`.
- File permissions were `644`.
- A local request returned HTTP 200 OK.

## Controlled Fault

Changed only the dedicated test file's permissions:

```bash
sudo chmod 600 /var/www/html/support-permissions-lab.txt
```

Permissions `600` allow the owner to read and write, with no access for the group or others. Because the owner was root, the Nginx worker account lost read access.

## Symptoms

- Nginx remained active.
- The test page returned HTTP 403 Forbidden.
- The Nginx error log recorded a file-access error.

## Investigation

### Check Service State

```bash
sudo systemctl is-active nginx
```

Result:

```text
active
```

### Inspect File Ownership and Permissions

```bash
sudo stat -c '%A %a %U:%G %n' \
/var/www/html/support-permissions-lab.txt
```

Result:

```text
-rw------- 600 root:root /var/www/html/support-permissions-lab.txt
```

### Inspect Nginx Process Owners

```bash
ps -eo user,pid,comm | grep '[n]ginx'
```

This helps distinguish the master process from the worker processes that serve requests.

### Reproduce the HTTP Failure

```bash
curl -i --max-time 5 \
http://127.0.0.1/support-permissions-lab.txt
```

Result:

```text
HTTP/1.1 403 Forbidden
```

### Review the Error Log

```bash
sudo tail -n 20 /var/log/nginx/error.log
```

At 19:53:52 UTC, the log recorded:

```text
open() "/var/www/html/support-permissions-lab.txt" failed
(13: Permission denied)
```

### Inspect the Full File Path

```bash
namei -l /var/www/html/support-permissions-lab.txt
```

This command displays ownership and permissions for each component of the path.
Directory traversal permissions matter as well as the file's read permissions.

## Root Cause

The root-owned test file had permissions `600`, preventing the Nginx worker account from reading it.

The HTTP response and error log confirmed a file-permissions failure.

## Corrective Action

Restored the public test page's original permissions:

```bash
sudo chmod 644 /var/www/html/support-permissions-lab.txt
```

No Nginx configuration change or service restart was required.

> Permissions `644` are appropriate for this public test file. They should not be applied indiscriminately to credentials, private keys, or sensitive application files.

## Recovery Verification

```bash
sudo stat -c '%A %a %U:%G %n' \
/var/www/html/support-permissions-lab.txt

curl -i --max-time 5 \
http://127.0.0.1/support-permissions-lab.txt
```

Verified results:

```text
-rw-r--r-- 644 root:root /var/www/html/support-permissions-lab.txt
HTTP/1.1 200 OK
```

The response body contained the expected test-page text.

Recovery was verified locally. An external recovery check was not recorded for this incident.

## Evidence

The following files were saved on the EC2 server:

```text
~/aws-linux-support-triage-evidence/permissions/
├── 01-working-baseline.txt
├── 02-permission-failure.txt
└── 03-permission-restored.txt
```

These files are not automatically included in this GitHub repository. Sanitized copies can be uploaded separately.

## Jira Documentation

IT-3 includes:

- Incident scope and symptoms.
- Working baseline and controlled fault.
- Internal investigation note.
- Corrective action and recovery verification.
- Simulated customer-facing update.
- Assignee: Alexander Njoku.
- Final status: Completed.
- Resolution: Fixed.

## Customer-Facing Recovery Update

“Access to the test page has been restored. The web server was running, but the page’s file permissions prevented it from being read. I corrected the permissions and verified that the page returns HTTP 200 locally. This was a controlled lab exercise; no customer or production service was affected.”

## Prevention and Support Lessons

- Validate permissions after deploying or changing website files.
- Check service health and the requested resource separately.
- Use error logs to distinguish permissions failures from other causes of HTTP 403.
- Inspect directory permissions along the complete file path.
- Apply a targeted correction rather than broad recursive permission changes.
- Avoid `chmod 777` as a troubleshooting shortcut.
- Verify the HTTP response after correcting access.
- Escalate if ownership, access policy, or security controls are unclear.

## Skills Demonstrated

- Linux ownership and permissions.
- Nginx service and error-log investigation.
- HTTP troubleshooting.
- Evidence collection.
- Targeted remediation.
- Jira incident documentation.
- Clear customer communication.
