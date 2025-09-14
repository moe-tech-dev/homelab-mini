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
## Was mich gebissen hat (und Fix)

- netplan „revertet“ → netplan try rollt zurück, wenn man nicht bestätigt
Fix: netplan generate && netplan apply, danach ip -br a.
- Port 8000 belegt → ss -ltnp | grep :8000 zeigt PID
Fix: Prozess beenden, Dienst sauber via systemd starten.
- SELinux blockt Proxy-Fetch (NGINX→App) → 502 + Audit-Hinweisspur
Fix: setsebool -P httpd_can_network_connect on.

## Kern-Dateien (Belege im Repo)
- ubuntu/app.py – Mini-App
- ubuntu/myapp.service – systemd-Unit
- rocky/app.conf – NGINX-vHost (Proxy auf 192.168.0.20:8000)
- ubuntu/ufw-status.txt – sudo ufw status verbose
- rocky/firewall-cmd.txt – firewall-cmd --list-services
- rocky/selinux.txt – getenforce + getsebool httpd_can_network_connect

## Nächste Schritte
- App produktionsnäher: gunicorn hinter NGINX
- Mini-Monitoring (systemd Watchdog / node_exporter)
- Gleiches Setup mit docker-compose oder Ansible reproduzierbar machen



