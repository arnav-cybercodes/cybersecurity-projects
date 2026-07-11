# Operation Pit Wall – Solution

This is my solution to GDGC Cloud Recruitment Problem Statement 2. I set up everything on my Ubuntu VM (@Dante) and worked through it as a Linux system for the task.I restored the server, secured it against further attacks, and brought race telemetry bac online before the race begin. 


---

## 1. User & Permission Management

Created the two required users and two required groups: Users-race_engineer & strategist Groups-pitwall & telemetry

```bash
sudo groupadd pitwall
sudo groupadd telemetry
sudo useradd -m race_engineer
sudo useradd -m strategist
sudo usermod -aG pitwall,telemetry race_engineer
sudo usermod -aG pitwall strategist
```

Verified with:
```bash
groups race_engineer   # race_engineer pitwall telemetry
groups strategist      # strategist pitwall
```

This matches the requirement exactly — `race_engineer` is in both groups, `strategist` is only in `pitwall`.

---

## 2. Directory Structure & Permissions

```bash
sudo mkdir -p /pitwall/{telemetry,strategy,backups,logs,secrets}
```
![image](https://github.com/user-attachments/assets/1cfffa44-06ef-47fd-b700-d5856e5d8ce1)

Then permissions, directory by directory:

**`secrets` — root only**
```bash
sudo chown root:root /pitwall/secrets
sudo chmod 700 /pitwall/secrets
```
`700` means only root can read, write, or even list the folder. Nobody else gets in, not even via the group. We know that read r=4, write w=2 and execute x=1.

**`logs` — readable by everyone, writable by nobody except root**
```bash
sudo chown root:root /pitwall/logs
sudo chmod 755 /pitwall/logs
```

**`telemetry` — only the telemetry group can modify it**
```bash
sudo chown root:telemetry /pitwall/telemetry
sudo chmod 2770 /pitwall/telemetry
```
I went with `2770` instead of `775`. The extra `2` sets the **setgid bit**, so any new file created inside automatically inherits the `telemetry` group instead of the creator's default group — keeps group ownership consistent over time. I also dropped "others" permissions entirely (`770` not `775`) since the requirement said only *authorized* users should touch it — public read access didn't seem necessary or safe here.

---

## 3. Telemetry Recovery

The URLs given in the problem statement were GitHub "blob" links, which point to the HTML preview page, not the actual file — so `wget` on those just downloads a webpage. I converted them to `raw.githubusercontent.com` links to get the real files:

```bash
wget https://raw.githubusercontent.com/WizardBornov/gdgc-cloud-assets/main/challenge-assets/telemetry.tar.gz
wget https://raw.githubusercontent.com/WizardBornov/gdgc-cloud-assets/main/challenge-assets/telemetry.sha256
```

Verified integrity:
```bash
sha256sum -c telemetry.sha256
# telemetry.tar.gz: OK
```

Then moved the verified archive into place:
```bash
sudo mv telemetry.tar.gz /pitwall/telemetry/
```

---

## 4. Service Deployment

Created a simple `index.html` with the required message and served it with Python's built-in HTTP server (root-owned since port 80 is a privileged port):

```bash
mkdir ~/telemetry_web && cd ~/telemetry_web
nano index.html
<!DOCTYPE html>
<html>
<head>
    <title>Telemetry Status</title>
</head>
<body>
    <h1>Race Status: GREEN</h1>
    <p>Telemetry Server Operational</p>
</body>
</html>
```

Confirmed it in the browser at `http://localhost` — page loads and shows the correct message.

---

## 5. Secret Protection

```bash
wget https://raw.githubusercontent.com/WizardBornov/gdgc-cloud-assets/main/challenge-assets/team_strategy.txt
sudo mv team_strategy.txt /pitwall/secrets/
sudo chown root:root /pitwall/secrets/team_strategy.txt
sudo chmod 600 /pitwall/secrets/team_strategy.txt
```

`600` restricts the file to root read/write only — not even the `secrets` directory's other implicit access matters here since the file itself is also locked down individually.

---
## Bonus Task 1: Automated Backup

I wrote a script that archives the telemetry directory with a timestamped filename, so backups don't overwrite each other.

```bash
#!/bin/bash
TIMESTAMP=$(date +%F_%H-%M-%S)
BACKUP_DIR="/pitwall/backups"
tar -czf "$BACKUP_DIR/telemetry_backup_$TIMESTAMP.tar.gz" /pitwall/telemetry
```

```bash
sudo chmod +x /pitwall/backup_telemetry.sh
sudo /pitwall/backup_telemetry.sh   # manual test run first
```

Manual run produced a correctly timestamped archive in `/pitwall/backups`. Then scheduled it with cron to run every minute:

```bash
sudo crontab -e
```
Added:
```
* * * * * /pitwall/backup_telemetry.sh
```
to the last line.

Verified with `sudo crontab -l`, and confirmed new backups appearing automatically every minute in `/pitwall/backups` without any manual intervention.

# Bonus Task 2: Firewall Hardening

Goal: only allow SSH (22), HTTP (80), and HTTPS (443) in; block everything else by default.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Verified with:
```bash
sudo ufw status verbose
```
Full output saved in [`configs/ufw_rules.txt`](configs/ufw_rules.txt).

I allowed 22 before enabling the firewall specifically to avoid getting locked out of my own SSH session — default-deny-incoming would otherwise cut off SSH access immediately on enable.
