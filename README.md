## Lab 2.A — A service and a timer
 
**Duration:** 50 minutes
**Machine:** Ubuntu VM (`ubuntu-01`, `192.168.56.10`)
**Goal:** Write a systemd service that runs as a dedicated non-root user on a timer. Deliberately hit the permission failure, diagnose it from the journal, and fix it correctly.
 
---
 
### Setup (before students start — instructor)
 
Confirm students have:
- SSH key access to the Ubuntu VM
- `sudo` privileges
- Week 1 snapshot `clean-install` in place (if they do not, they must make one now)
---
 
### Task 1 — Write the script (8 min)
 
"Create the script the service will run."
 
```bash
sudo nano /usr/local/bin/disk-report.sh
```
 
Students write the following contents exactly:
```bash
#!/bin/bash
echo "=== $(date --iso-8601=seconds) ===" >> /var/log/disk-report.log
df -h / >> /var/log/disk-report.log
```
 
Then make it executable:
```bash
sudo chmod 755 /usr/local/bin/disk-report.sh
```
 
Test it by hand as root first:
```bash
sudo /usr/local/bin/disk-report.sh
cat /var/log/disk-report.log
```
 
"It works as root. Good. Now the question: why would we not just run it as root?"
 
**Wait for answers. Expect: "if the script is compromised", "if there's a bug", "least privilege".**
 
"All correct. If this script contained a bug that executed arbitrary code — or if an attacker could modify it — a script running as root owns the machine. A script running as `reports` owns only what `reports` owns, which is one log file. We contain the blast radius."
 
---
 
### Task 2 — Create the service account (5 min)
 
```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  reports
```
 
Verify it:
```bash
grep reports /etc/passwd
id reports
```
 
Expected output:
```
reports:x:999:999::/home/reports:/usr/sbin/nologin
```
 
Students record: what each flag to `useradd` does and why.
 
| Flag | Reason |
|---|---|
| `--system` | UID below 1000; marks it as a service account |
| `--no-create-home` | Services do not need a home directory |
| `--shell /usr/sbin/nologin` | Interactive login produces an immediate logout |
 
---
 
### Task 3 — Write the unit files (10 min)
 
**Service unit:**
 
```bash
sudo nano /etc/systemd/system/disk-report.service
```
 
Contents:
```ini
[Unit]
Description=Append disk usage to log
Documentation=man:df(1)
After=local-fs.target
 
[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
```
 
"No `[Install]` section — this service is triggered by the timer, not enabled directly. Services driven by a timer do not need to be enabled themselves."
 
**Timer unit:**
 
```bash
sudo nano /etc/systemd/system/disk-report.timer
```
 
Contents:
```ini
[Unit]
Description=Run disk-report every five minutes
Documentation=systemd.time(7)
 
[Timer]
OnCalendar=*:0/5
Persistent=true
 
[Install]
WantedBy=timers.target
```
 
"`OnCalendar=*:0/5` means every hour, at minutes 0, 5, 10, 15 — every five minutes. We use a short interval for testing; we will change it before the end of the lab."
 
"`Persistent=true` — if the machine is off when the timer would have fired, run it immediately at next boot. For a daily report, this is usually what you want."
 
---
 
### Task 4 — Load and enable (5 min)
 
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk-report.timer
sudo systemctl list-timers disk-report.timer
```
 
"You enable the **timer**, not the service. The service has no Install section — enabling it does nothing. Only the timer needs to be enabled."
 
Expected output from `list-timers`:
```
NEXT                         LEFT       LAST                         PASSED  UNIT                ACTIVATES
Mon 2024-01-15 10:25:00 UTC  3min 44s   Mon 2024-01-15 10:20:00 UTC  16s     disk-report.timer   disk-report.service
```
 
---
 
### Task 5 — Watch it fail, diagnose, fix (15 min)
 
"Do not fix anything yet. Just watch."
 
```bash
journalctl -u disk-report.service -f
```
 
Wait for the first run (within five minutes). Students will see something like:
```
Jan 15 10:25:01 ubuntu-01 disk-report.sh[1234]: /usr/local/bin/disk-report.sh: line 2:
  /var/log/disk-report.log: Permission denied
Jan 15 10:25:01 ubuntu-01 systemd[1]: disk-report.service: Main process exited, code=exited, status=1/FAILURE
```
 
**Do not tell them the fix. Ask:**
 
"Read the error. What does it tell you? What does 'Permission denied' on `/var/log/disk-report.log` mean?" 
 
Let students reason through it: the `reports` user is trying to write to a file it does not own. `/var/log` is owned by `root:root`. The file `/var/log/disk-report.log` was created by root when we tested it.
 
"What are your options?"
 
Collect all their ideas. The options are:
 
**Option A — chown the file:**
```bash
sudo touch /var/log/disk-report.log
sudo chown reports:reports /var/log/disk-report.log
```
Simple, works, but fragile — if the file is deleted the service breaks again, and `/var/log` still lets `reports` write anything there.
 
**Option B — use `StandardOutput=append:` in the service unit:**
```ini
[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
StandardOutput=append:/var/log/disk-report.log
StandardError=append:/var/log/disk-report.log
```
Move the append logic out of the script entirely. systemd opens the file as root on behalf of the service, so the `reports` user never needs write permission on the log directory. Clean and correct. The service still owns the log conceptually.
 
**Option C — use `RuntimeDirectory=` or `LogsDirectory=`:**
```ini
[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
LogsDirectory=disk-report
LogsDirectoryMode=0750
```
This creates `/var/log/disk-report/` owned by `reports` and managed by systemd. Update the script to write to `/var/log/disk-report/report.log`.
 
**The recommended solution for this lab is Option B.** Explain why: it is the least privilege approach — the script itself never needs a file path, the unit controls where output goes, and you can change the destination without touching the script.
 
Students implement Option B:
```bash
sudo nano /etc/systemd/system/disk-report.service
# Add the two StandardOutput lines
sudo systemctl daemon-reload
sudo journalctl -u disk-report.service -f
# Manually trigger a run:
sudo systemctl start disk-report.service
```
 
Confirm success in the journal:
```
Jan 15 10:30:01 ubuntu-01 systemd[1]: disk-report.service: Succeeded.
```
 
Check the log:
```bash
cat /var/log/disk-report.log
```
 
---
 
### Task 6 — Change the schedule (5 min)
 
```bash
sudo nano /etc/systemd/system/disk-report.timer
```
 
Change `OnCalendar=*:0/5` to `OnCalendar=daily`.
 
Before saving, validate the calendar expression:
```bash
systemd-analyze calendar "daily"
```
Output shows the next trigger times. Do this **before** reloading — it is the `visudo` equivalent for timer schedules.
 
```bash
sudo systemctl daemon-reload
sudo systemctl restart disk-report.timer
sudo systemctl list-timers disk-report.timer
```
 
---
 
### Checkpoint — all three, verified by the instructor
 
**1.** Timer listed in `systemctl list-timers`:
```bash
systemctl list-timers disk-report.timer
```
 
**2.** At least two successful runs in the journal:
```bash
journalctl -u disk-report.service --no-pager | grep Succeeded
```
Must show two lines.
 
**3.** Service runs as the `reports` user:
```bash
systemctl show disk-report.service -p User
# Expected: User=reports
```
 
Bonus check — confirm the service was never enabled directly:
```bash
systemctl is-enabled disk-report.service   # should say: static
systemctl is-enabled disk-report.timer     # should say: enabled
```
 
---
 
### Lab deliverable — write up in the repo (before next session)
 
Students commit to their Git repo, in a file `week2/lab2a.md`:
 
1. Both unit files, final versions.
2. The initial journal error output, annotated: what it said, what it told you, which part confirmed the cause.
3. Why Option B (`StandardOutput=append:`) is better than Option A (`chown`). One paragraph.
4. Output of `systemctl list-timers disk-report.timer` showing a future trigger time.
5. Two lines from `journalctl -u disk-report.service` showing two successful runs.
6. Answer this question in two sentences: why was the `reports` user created with `--no-create-home --shell /usr/sbin/nologin`, and what would an attacker gain if those flags were omitted?
---
 
## Common questions and how to answer them
 
**"Why not just run everything as root? It is simpler."**
"It is simpler — right until a bug or a compromised library does something destructive. If `disk-report.sh` has a bug that deletes files, running as `reports` means it can delete files `reports` owns. Running as root means it can delete any file on the system. The principle is containment: when something goes wrong, limit how far it can spread. We will quantify this exactly in Week 10 when we look at what a compromised service can reach."
 
**"What if I need the service to write to multiple places?"**
"Use `ReadWritePaths=` in the unit's `[Service]` section to list specific paths the service is allowed to write to. That is part of the sandboxing we cover in Week 10. For now, `StandardOutput=append:` is the right answer for a simple log."
 
**"The journal said the service failed but I could not find the error in the output."**
"Run `journalctl -u disk-report.service -e` — the `-e` jumps to the end. Or `journalctl -u disk-report.service --since '5 minutes ago'`. If the service exits non-zero, the last few lines before the `status=1/FAILURE` line contain the error. Always read upward from the failure line, not from the beginning."
 
**"Do I need to reload the daemon every time I edit a unit file?"**
"Yes. systemd reads unit files into memory at load time. If you edit a file on disk without reloading, you are running the old version. `systemctl daemon-reload` after every edit; `systemctl restart` after that if the service is running. A common mistake is editing the file and seeing no effect because the reload was skipped."
 
---
 
## Things likely to go wrong in the lab
 
**Student cannot SSH in.** First question: is the VM running? `VBoxManage list runningvms`. Second: is the host-only interface up? Check VirtualBox → Devices → Network (no orange icon). Third: did they change the IP? Check from the VirtualBox console with `ip addr`.
 
**`systemctl daemon-reload` not run after editing.** Symptom: the change has no effect. Ask: "After editing the file, what did you run?" If they say `systemctl restart`, ask what they ran before that. 
 
**`disk-report.timer` enabled but service never fires.** Check `systemctl list-timers` — is the timer in the list? If not, was `daemon-reload` run after creating the timer file? Also check `systemctl status disk-report.timer` for a `Loaded:` status that says `not-found`.
 
**Two students on the same VM.** If students share a VM (Path B from Week 0), unit names must be unique. Have them use their initials as a prefix: `jk-disk-report.service`.
 
**Log file still shows Permission denied after adding `StandardOutput=append:`.**
Usually means `daemon-reload` was skipped. Also check: did the script itself still have the `>> /var/log/disk-report.log` redirect? If the script redirects and the unit also redirects, the script's redirect runs as `reports` and fails. Remove the redirect from the script when using `StandardOutput=append:`.
 
