# Mini-Homelab: App auf Ubuntu, Proxy auf Rocky
*Stand: Sept 2025 – kleiner Lern- und Bewerbungs-Build.*

Ich wollte eine kleine Web-App so betreiben, wie man es in der Praxis macht:
App separat, Proxy separat, feste IPs, SSH nur mit Schlüssel, Firewalls an. **SELinux bleibt auf „enforcing“.**

**Stand (kurz)**
- **Ubuntu (192.168.0.20)**: Flask-App in venv, als `systemd`-Service `myapp` auf Port **8000**  
- **Rocky (192.168.0.21)**: **NGINX** als Reverse-Proxy auf Port **80**  
- **Security**: SSH **Key-Only** (Passwörter aus), **UFW**/**firewalld** aktiv, `getenforce = Enforcing`

**Warum so?**  
Trennung der Rollen (App ↔ Proxy) hält Updates einfach und Fehler klein. Nebenbei lerne ich beide Welten (Ubuntu/apt/ufw ↔ Rocky/dnf/firewalld/SELinux).

## Test in 60 Sekunden
```bash
# auf Rocky
curl -I http://192.168.0.21/      # → 200 OK

# Ausfall simulieren:
ssh moe@192.168.0.20 'sudo systemctl stop myapp'
curl -I http://192.168.0.21/      # → 502 Bad Gateway

# Fix:
ssh moe@192.168.0.20 'sudo systemctl start myapp'
curl -I http://192.168.0.21/      # → 200 OK
```
## Skizze
```SCSS
Windows-Host ──(SSH)──> ubuntu-app (192.168.0.20:8000, myapp)
        ▲
        │ proxy_pass
        ▼
rocky-gw (192.168.0.21:80, NGINX)
```
## Was schiefging (und mein Fix)
- netplan revertet (ich bestätigte netplan try nicht): Fix = netplan generate && netplan apply; danach ip -br a.
- Port 8000 belegt: ss -ltnp | grep :8000 → PID beendet → App nur noch per systemd starten.
- SELinux blockt Proxy-Fetch: 502 in NGINX, Audit-Log Deny → Fix = setsebool -P httpd_can_network_connect on.

## Belege/Dateien in diesem Repo
- ubuntu/app.py – Mini-App
- ubuntu/myapp.service – systemd-Unit
- rocky/app.conf – NGINX-vHost (Proxy auf 192.168.0.20:8000)
- ubuntu/ufw-status.txt – sudo ufw status verbose
- rocky/firewall-cmd.txt – firewall-cmd --list-services
- rocky/selinux.txt – getenforce + getsebool httpd_can_network_connect

## Nächste Schritte
- App hinter NGINX mit gunicorn betreiben
- SSH rate-limit (ufw limit) / fail2ban
- Gleiche Topologie mit Ansible reproduzierbar machen

## Files
- ubuntu/app.py – einfache Flask-App
- ubuntu/myapp.service – systemd-Unit für Autostart
- rocky/app.conf – Nginx Proxy Config


