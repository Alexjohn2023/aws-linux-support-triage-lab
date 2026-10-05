# Baseline Checks

## Current verified state

- Nginx configuration test passes.
- `nginx.service` is enabled and active.
- Nginx listens on IPv4 and IPv6 TCP port 80.
- A local `curl -I http://localhost` returned `HTTP/1.1 200 OK`.
- The default Nginx welcome page loaded from a workstation through the EC2 public endpoint.

## Commands used

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo ss -lntp | grep ':80'
curl -I http://localhost
```

## Meaning of the results

- `nginx -t` checks configuration syntax; it does not prove that the service is running or externally reachable.
- `systemctl status` confirms that systemd reports the service as active.
- `ss` shows the process listening on port 80.
- `curl` checks the local HTTP response.
- The browser test checks the path from the workstation through the cloud network controls to Nginx.

## Broader host baseline commands

```bash
hostnamectl
cat /etc/os-release
uptime
free -h
df -h
df -i
ip addr
ip route
sudo systemctl --failed
sudo journalctl -p err -b --no-pager
sudo aa-status
sudo ufw status verbose
```

Capture only sanitized output. Remove public addresses, account details, usernames if needed, and any credentials before publishing.
