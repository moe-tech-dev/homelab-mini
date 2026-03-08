# Mini Homelab

A small two-server homelab project that separates the application server from
the reverse proxy layer.

The goal was to practice a more realistic setup than "run everything on one
machine": separate roles, SSH key-only access, active firewalls, and SELinux
left enabled.

## Overview

- Ubuntu app server
  Runs a small Flask app inside a virtual environment as a `systemd` service on
  port `8000`.
- Rocky Linux gateway
  Runs NGINX as a reverse proxy on port `80`.
- Security baseline
  SSH key-only login, firewall enabled, SELinux in `Enforcing` mode.

## Architecture

```text
Windows host --SSH--> ubuntu-app (192.168.0.20:8000, myapp)
                         ^
                         | proxy_pass
                         v
                   rocky-gw (192.168.0.21:80, NGINX)
```

## Why I Built It

I wanted a compact project that demonstrates:

- Linux administration on two distributions
- reverse proxy basics with NGINX
- `systemd` service management
- basic network separation
- practical hardening choices instead of disabling them

## Quick Test

```bash
# on Rocky
curl -I http://192.168.0.21/

# simulate app failure
ssh moe@192.168.0.20 'sudo systemctl stop myapp'
curl -I http://192.168.0.21/

# recover
ssh moe@192.168.0.20 'sudo systemctl start myapp'
curl -I http://192.168.0.21/
```

Expected behavior:

- healthy app: `200 OK`
- stopped app service: `502 Bad Gateway`
- restarted app service: `200 OK`

## Problems I Hit

- `netplan try` reverted because it was not confirmed in time
  Fix: regenerate and apply the config, then verify addresses with `ip -br a`
- port `8000` was already in use
  Fix: identify the process with `ss -ltnp | grep :8000`, stop it, and run the
  app only through `systemd`
- SELinux blocked the proxy connection
  Fix: enable outbound proxy access with
  `setsebool -P httpd_can_network_connect on`

## Files

- `homelab-mini/ubuntu/app.py`
  Small Flask application
- `homelab-mini/ubuntu/myapp.service`
  `systemd` unit for the Ubuntu app service
- `homelab-mini/rocky/app.conf`
  NGINX reverse proxy configuration for Rocky Linux

## What I Learned

- splitting app and proxy roles makes failures easier to isolate
- SELinux should be understood and configured, not just turned off
- a tiny project can still show useful ops skills if the setup is intentional

## Next Steps

- run the Flask app behind Gunicorn
- add rate limiting or fail2ban for SSH
- reproduce the setup with Ansible

