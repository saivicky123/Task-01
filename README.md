# Task-01
Linux Commands
Level 1 — Basic scripts
1) setup_users_groups.sh — create dev group + users, set home & skeleton
#!/usr/bin/env bash
# Usage: sudo ./setup_users_groups.sh user1,user2 ... 
set -euo pipefail

if [ $# -lt 1 ]; then
  echo "Usage: $0 user1,user2,..."
  exit 1
fi

USERS_CSV="$1"
GROUP="devteam"
DEFAULT_SHELL="/bin/bash"
DEFAULT_SKELETON="/etc/skel"

# Create group if missing
if ! getent group "$GROUP" >/dev/null; then
  groupadd --system "$GROUP"
  echo "Created group: $GROUP"
fi

IFS=',' read -ra USERS <<< "$USERS_CSV"

for u in "${USERS[@]}"; do
  if id "$u" &>/dev/null; then
    echo "User $u exists — adding to $GROUP"
    usermod -aG "$GROUP" "$u"
  else
    useradd -m -s "$DEFAULT_SHELL" -k "$DEFAULT_SKELETON" -G "$GROUP" "$u"
    echo "Created user $u with primary group $GROUP"
    passwd -l "$u" || true   # lock password by default (force SSH key usage)
  fi
done

echo "All done."

2) set_project_perms.sh — set permissions for project directories
#!/usr/bin/env bash
# Usage: sudo ./set_project_perms.sh /path/to/project devteam
set -euo pipefail

if [ $# -lt 2 ]; then
  echo "Usage: $0 /path/to/project groupname"
  exit 1
fi

PROJECT_DIR="$1"
GROUP="$2"

# Ensure directory exists
mkdir -p "$PROJECT_DIR"

chown -R root:"$GROUP" "$PROJECT_DIR"
# Set group read/write/execute and default setgid so new files inherit the group
find "$PROJECT_DIR" -type d -exec chmod 2775 {} \;
find "$PROJECT_DIR" -type f -exec chmod 0664 {} \;

echo "Permissions set: dirs=2775, files=664. Group=$GROUP"

3) install_pkgs.sh — install common packages (works for yum/apt)
#!/usr/bin/env bash
# Usage: sudo ./install_pkgs.sh git nginx default-jdk
set -euo pipefail

if [ $# -lt 1 ]; then
  echo "Usage: $0 pkg1 pkg2 ..."
  exit 1
fi

PKGS=( "$@" )

if command -v apt-get &>/dev/null; then
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y "${PKGS[@]}"
elif command -v yum &>/dev/null; then
  yum install -y "${PKGS[@]}"
elif command -v dnf &>/dev/null; then
  dnf install -y "${PKGS[@]}"
else
  echo "No supported package manager found (apt/yum/dnf)."
  exit 1
fi

echo "Installed: ${PKGS[*]}"

4) system_info.sh — quick system info summary
#!/usr/bin/env bash
# Usage: ./system_info.sh
set -euo pipefail

echo "=== HOST ==="
hostnamectl

echo -e "\n=== UPTIME ==="
uptime -p

echo -e "\n=== CPU ==="
lscpu | sed -n '1,8p'

echo -e "\n=== MEMORY ==="
free -h

echo -e "\n=== DISKS ==="
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

echo -e "\n=== TOP 5 PROCESSES (by CPU) ==="
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 6

echo -e "\n=== Open ports (ss) ==="
ss -tuln

Level 2 — Intermediate scripts & cron examples
5) backup_home.sh — simple daily backup (to /backup)
#!/usr/bin/env bash
# Usage: sudo ./backup_home.sh
set -euo pipefail

BACKUP_ROOT="/backup"
TIMESTAMP="$(date +%F_%H%M%S)"
SRC="/home"
DEST="$BACKUP_ROOT/home_$TIMESTAMP.tar.gz"
RETENTION_DAYS=7

mkdir -p "$BACKUP_ROOT"
tar -czf "$DEST" -C /home .
# rotate old backups
find "$BACKUP_ROOT" -type f -name 'home_*.tar.gz' -mtime +"$RETENTION_DAYS" -print -delete

echo "Backup created: $DEST"


Example cron (daily at 2:15am):
15 2 * * * /usr/local/bin/backup_home.sh >> /var/log/backup_home.log 2>&1

6) log_cleanup.sh — rotate/truncate old app logs (ad-hoc)
#!/usr/bin/env bash
# Usage: sudo ./log_cleanup.sh /var/log/myapp 30
set -euo pipefail

LOG_DIR="${1:-/var/log/myapp}"
KEEP_DAYS="${2:-30}"

# delete rotated/archive older than KEEP_DAYS
find "$LOG_DIR" -type f -name '*.log.*' -mtime +"$KEEP_DAYS" -exec rm -f {} \;

# compress logs older than 7 days
find "$LOG_DIR" -type f -name '*.log' -mtime +7 -exec gzip -9 {} \;

echo "Cleaned logs in $LOG_DIR (keep $KEEP_DAYS days)."

7) restart_service_if_failed.sh — watch & restart systemd service
#!/usr/bin/env bash
# Usage: sudo ./restart_service_if_failed.sh myapp.service
set -euo pipefail

SERVICE="${1:-}"
if [ -z "$SERVICE" ]; then
  echo "Usage: $0 <service-name>"
  exit 1
fi

if ! systemctl is-active --quiet "$SERVICE"; then
  echo "$(date): $SERVICE is not active. Attempting restart..."
  systemctl restart "$SERVICE"
  sleep 2
  if systemctl is-active --quiet "$SERVICE"; then
    echo "$(date): $SERVICE restarted successfully."
  else
    echo "$(date): Failed to restart $SERVICE. Check journalctl -u $SERVICE"
  fi
else
  echo "$(date): $SERVICE is active."
fi


You can run this from cron every 5 minutes if you want (but prefer systemd Restart= in unit file — see level 3).

8) healthcheck_http.sh — check HTTP endpoint and alert to syslog
#!/usr/bin/env bash
# Usage: ./healthcheck_http.sh http://localhost:8080/health 200
URL="${1:-http://localhost:8080/health}"
EXPECT_CODE="${2:-200}"

HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' "$URL" || echo "000")
if [ "$HTTP_CODE" != "$EXPECT_CODE" ]; then
  logger -t myhealthcheck "ALERT: $URL responded $HTTP_CODE (expected $EXPECT_CODE)"
  exit 2
else
  logger -t myhealthcheck "OK: $URL responded $HTTP_CODE"
  exit 0
fi


Cron example: */2 * * * * /usr/local/bin/healthcheck_http.sh http://localhost:8080/health 200

Level 3 — Advanced / Production-ready
9) myapp.service — example custom systemd service unit

Place /etc/systemd/system/myapp.service

[Unit]
Description=My App Service
After=network.target

[Service]
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start.sh
ExecStop=/opt/myapp/bin/stop.sh
Restart=on-failure
RestartSec=5
TimeoutStartSec=20
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


Commands to enable:

sudo systemctl daemon-reload
sudo systemctl enable --now myapp.service
sudo journalctl -u myapp.service -f

10) SSH hardening snippet (/etc/ssh/sshd_config edits)

You should back up before editing:

sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)


Recommended settings to add/ensure in /etc/ssh/sshd_config:

PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
AllowTcpForwarding no
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30s
ClientAliveInterval 300
ClientAliveCountMax 2
# optionally restrict users
# AllowUsers alice bob


Reload:

sudo systemctl reload sshd


If you need an automated sed script (be cautious):

sudo sed -i.bak -e 's/^#\?PermitRootLogin.*/PermitRootLogin no/' \
              -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
              -e 's/^#\?MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config
sudo systemctl reload sshd


Important: keep an open session while testing; don’t lock yourself out.

11) lvm_setup.sh — create PV, VG, LV and mount (example)

WARNING: This will destroy data on the device. Replace /dev/xvdb with your block device.

#!/usr/bin/env bash
# Usage: sudo ./lvm_setup.sh /dev/xvdb vg_data lv_data 20G /data
set -euo pipefail

DEV="$1"
VG="$2"
LV="$3"
SIZE="$4"   # e.g., 20G or 100%FREE
MOUNTPOINT="$5"

pvcreate "$DEV"
vgcreate "$VG" "$DEV"
lvcreate -n "$LV" -L "$SIZE" "$VG"
mkfs.ext4 /dev/"$VG"/"$LV"
mkdir -p "$MOUNTPOINT"
echo "/dev/$VG/$LV $MOUNTPOINT ext4 defaults 0 2" >> /etc/fstab
mount -a
echo "LVM created and mounted at $MOUNTPOINT"

12) Firewall snippets
firewalld (recommended on RHEL/CentOS/Fedora)
# open http, https, ssh, and a custom port 8080 permanently
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# list zones and ports
sudo firewall-cmd --list-all

iptables (basic example)
# allow established, allow ssh/http/https, drop rest (not persistent)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT


For persistence, install iptables-persistent (Debian) or save rules with service iptables save (RHEL).

13) logrotate config for app logs — /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 appuser appuser
    sharedscripts
    postrotate
        systemctl reload myapp.service >/dev/null 2>&1 || true
    endscript
}


Test: logrotate -d /etc/logrotate.d/myapp (dry run). Actual run: logrotate -f /etc/logrotate.d/myapp.

Extra: monitoring quick commands / script
sysmon.sh — lightweight monitoring snapshot (for cron or collection)
#!/usr/bin/env bash
OUT="/var/log/sysmon_$(date +%F_%H%M%S).log"
{
  echo "=== $(date) ==="
  echo "--- load ---"
  cat /proc/loadavg
  echo "--- memory ---"
  free -h
  echo "--- disk usage ---"
  df -h
  echo "--- top cpu processes ---"
  ps -eo pid,cmd,%mem,%cpu --sort=-%cpu | head -n 10
  echo "--- journal errors (last 200) ---"
  journalctl -p 3 -n 200 --no-pager || true
} > "$OUT"
echo "Wrote $OUT"
