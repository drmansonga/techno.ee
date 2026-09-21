Task 1 — Write the script (8 min)

"Create the script the service will run."

bash
sudo nano /usr/local/bin/disk-report.sh

Students write the following contents exactly:

bash
#!/bin/bash
echo "=== $(date --iso-8601=seconds) ===" >> /var/log/disk-report.log
df -h / >> /var/log/disk-report.log

Then make it executable:

bash
sudo chmod 755 /usr/local/bin/disk-report.sh

Test it by hand as root first:

bash
sudo /usr/local/bin/disk-report.sh
cat /var/log/disk-report.log

"It works as root. Good. Now the question: why would we not just run it as root?"

Wait for answers. Expect: "if the script is compromised", "if there's a bug", "least privilege".

"All correct. If this script contained a bug that executed arbitrary code — or if an attacker could modify it — a script running as root owns the machine. A script running as reports owns only what reports owns, which is one log file. We contain the blast radius."

Task 2 — Create the service account (5 min)
bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  reports

Verify it:

bash
grep reports /etc/passwd
id reports

Expected output:

reports:x:999:999::/home/reports:/usr/sbin/nologin

Students record: what each flag to useradd does and why.

Flag	Reason
--system	UID below 1000; marks it as a service account
--no-create-home	Services do not need a home directory
--shell /usr/sbin/nologin	Interactive login produces an immediate logout
Task 3 — Write the unit files (10 min)

Service unit:

bash
sudo nano /etc/systemd/system/disk-report.service

Contents:

ini
[Unit]
Description=Append disk usage to log
Documentation=man:df(1)
After=local-fs.target

[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh

"No [Install] section — this service is triggered by the timer, not enabled directly. Services driven by a timer do not need to be enabled themselves."

Timer unit:

bash
sudo nano /etc/systemd/system/disk-report.timer

Contents:

ini
[Unit]
Description=Run disk-report every five minutes
Documentation=systemd.time(7)

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target

"OnCalendar=*:0/5 means every hour, at minutes 0, 5, 10, 15 — every five minutes. We use a short interval for testing; we will change it before the end of the lab."

"Persistent=true — if the machine is off when the timer would have fired, run it immediately at next boot. For a daily report, this is usually what you want."

Task 4 — Load and enable (5 min)
bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk-report.timer
sudo systemctl list-timers disk-report.timer

"You enable the timer, not the service. The service has no Install section — enabling it does nothing. Only the timer needs to be enabled."

Expected output from list-timers:

NEXT                         LEFT       LAST                         PASSED  UNIT                ACTIVATES
Mon 2024-01-15 10:25:00 UTC  3min 44s   Mon 2024-01-15 10:20:00 UTC  16s     disk-report.timer   disk-report.service
Task 5 — Watch it fail, diagnose, fix (15 min)

"Do not fix anything yet. Just watch."

bash
journalctl -u disk-report.service -f

Wait for the first run (within five minutes). Students will see something like:

Jan 15 10:25:01 ubuntu-01 disk-report.sh[1234]: /usr/local/bin/disk-report.sh: line 2:
  /var/log/disk-report.log: Permission denied
Jan 15 10:25:01 ubuntu-01 systemd[1]: disk-report.service: Main process exited, code=exited, status=1/FAILURE

Do not tell them the fix. Ask:

"Read the error. What does it tell you? What does 'Permission denied' on /var/log/disk-report.log mean?"

Let students reason through it: the reports user is trying to write to a file it does not own. /var/log is owned by root:root. The file /var/log/disk-report.log was created by root when we tested it.

"What are your options?"

Collect all their ideas. The options are:

Option A — chown the file:

bash
sudo touch /var/log/disk-report.log
sudo chown reports:reports /var/log/disk-report.log

Simple, works, but fragile — if the file is deleted the service breaks again, and /var/log still lets reports write anything there.

Option B — use StandardOutput=append: in the service unit:

ini
[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
StandardOutput=append:/var/log/disk-report.log
StandardError=append:/var/log/disk-report.log

Move the append logic out of the script entirely. systemd opens the file as root on behalf of the service, so the reports user never needs write permission on the log directory. Clean and correct. The service still owns the log conceptually.

Option C — use RuntimeDirectory= or LogsDirectory=:

ini
[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
LogsDirectory=disk-report
LogsDirectoryMode=0750

This creates /var/log/disk-report/ owned by reports and managed by systemd. Update the script to write to /var/log/disk-report/report.log.

The recommended solution for this lab is Option B. Explain why: it is the least privilege approach — the script itself never needs a file path, the unit controls where output goes, and you can change the destination without touching the script.

Students implement Option B:

bash
sudo nano /etc/systemd/system/disk-report.service
# Add the two StandardOutput lines
sudo systemctl daemon-reload
sudo journalctl -u disk-report.service -f
# Manually trigger a run:
sudo systemctl start disk-report.service

Confirm success in the journal:

Jan 15 10:30:01 ubuntu-01 systemd[1]: disk-report.service: Succeeded.

Check the log:

bash
cat /var/log/disk-report.log
Task 6 — Change the schedule (5 min)
bash
sudo nano /etc/systemd/system/disk-report.timer

Change OnCalendar=*:0/5 to OnCalendar=daily.

Before saving, validate the calendar expression:

bash
systemd-analyze calendar "daily"

Output shows the next trigger times. Do this before reloading — it is the visudo equivalent for timer schedules.

bash
sudo systemctl daemon-reload
sudo systemctl restart disk-report.timer
sudo systemctl list-timers disk-report.timer
Checkpoint — all three, verified by the instructor

1. Timer listed in systemctl list-timers:

bash
systemctl list-timers disk-report.timer

2. At least two successful runs in the journal:

bash
journalctl -u disk-report.service --no-pager | grep Succeeded

Must show two lines.

3. Service runs as the reports user:

bash
systemctl show disk-report.service -p User
# Expected: User=reports

Bonus check — confirm the service was never enabled directly:

bash
systemctl is-enabled disk-report.service   # should say: static
systemctl is-enabled disk-report.timer     # should say: enabled
Lab deliverable — write up in the repo (before next session)

Students commit to the Git repo:

Both unit files, final versions.
The initial journal error output, annotated: what it said, what it told you, which part confirmed the cause.
Why Option B (StandardOutput=append:) is better than Option A (chown). One paragraph.
Output of systemctl list-timers disk-report.timer showing a future trigger time.
Two lines from journalctl -u disk-report.service showing two successful runs.
Answer this question in two sentences: why was the reports user created with --no-create-home --shell /usr/sbin/nologin, and what would an attacker gain if those flags were omitted?
