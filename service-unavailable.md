# Incident: Nginx Service Unavailable

**Status:** Planned; not yet performed.

## Objective

Practice diagnosing a stopped web service from the browser symptom through the Linux service and listener checks, then restore it and verify recovery.

## Controlled simulation

Stop only Nginx; this does not stop SSH:

```bash
sudo systemctl stop nginx
```

Refresh the test page from the workstation and record the exact error and time.

## Investigation

```bash
sudo systemctl status nginx --no-pager
sudo journalctl -u nginx -n 50 --no-pager
sudo ss -lntp | grep ':80'
curl -I --max-time 5 http://localhost
```

Expected while stopped: systemd reports the service inactive, no Nginx listener appears on port 80, and the local HTTP request fails.

## Recovery and verification

```bash
sudo systemctl start nginx
sudo systemctl status nginx --no-pager
sudo ss -lntp | grep ':80'
curl -I http://localhost
```

Refresh the page from the workstation and confirm that the welcome page returns. Record actual output; do not claim a completed recovery until it has been observed.

## Incident note template

- Start time and time zone:
- User-visible symptom:
- Scope / affected service:
- Checks and results:
- Confirmed cause:
- Action taken:
- Local verification:
- External verification:
- Customer update:
- Escalation needed (yes/no) and evidence:
