# Operation Pit Wall – Solution

This is my solution to GDGC Cloud Recruitment Problem Statement 2. I set everything up on an Ubuntu VM (VirtualBox) and worked through it as a Linux sysadmin/security task — restoring a "downed" telemetry server, locking down access, and getting a basic web service back online.


---

## 1. User & Permission Management

Created the two required users and two required groups:

```bash
sudo groupadd pitwall
sudo groupadd telemetry
sudo useradd -m race_engineer
sudo useradd -m strategist
sudo usermod -aG pitwall,telemetry race_engineer
sudo usermod -aG pitwall strategist
```

I used `-aG` (append) instead of `-G` on purpose — `-G` alone overwrites a user's existing group list, which isn't what you want when you're adding someone to multiple groups.

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

Then permissions, directory by directory:

**`secrets` — root only**
```bash
sudo chown root:root /pitwall/secrets
sudo chmod 700 /pitwall/secrets
```
`700` means only root can read, write, or even list the folder. Nobody else gets in, not even via the group.

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

**Proof it actually works:** when I tried `cd /pitwall/telemetry` and `cd /pitwall/secrets` as my regular login user (`dante.os`, not in the `telemetry` group and not root), I got `Permission denied` both times. That's the permission model doing exactly what it's supposed to.

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
nano index.html   # Race Status: GREEN / Telemetry Server Operational
sudo python3 -m http.server 80
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

## Assumptions & Challenges

- The GitHub blob links needed to be swapped for raw links before `wget` would pull actual file content — this wasn't stated explicitly in the problem statement but was necessary to get real files instead of HTML.
- `sudo cd` doesn't work because `cd` is a shell built-in, not a standalone binary — you can't elevate into a directory that way. Used `sudo cat <path>` instead to verify file contents inside restricted folders without needing a root shell.
- I read "authorized modifications" for the telemetry directory as meaning group members only, with no public access at all, so I used `770` rather than `775`.

---

## Bonus Tasks

Status: **not completed yet** — attempting these next:
- [ ] Automated backup script + cron job
- [ ] Firewall hardening (ufw: allow 22/80/443, deny rest)
- [ ] SSH hardening (disable root login, disable password auth, restrict to `race_engineer`)

This README will be updated once those are done, with the relevant scripts added to `scripts/` and configs added to `configs/`.

---

## Screenshots

See the `screenshots/` folder for evidence of:
- User and group creation
- Directory structure and permissions (`ls -ld`)
- Telemetry download + SHA-256 verification
- Web server running (terminal + browser)
- Secret file permissions
- Permission-denied proof as a non-privileged user
-
