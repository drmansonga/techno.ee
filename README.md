# Linux II Lab 9 — Hardening and Maintenance

This lab introduces the practical skills required to harden a Linux system, reduce unnecessary exposure, and maintain a secure baseline through SSH hardening and automated patching.

The lab is split into two parts:

- Lab 9.A — Measure and reduce
- Lab 9.B — SSH and patching

---

## Objectives

By the end of this lab, you should be able to:

- Capture the “before” state of a system and compare it to the “after” state.
- Identify and remove unnecessary services, packages, and accounts safely.
- Justify each removal in terms of:
  - what the service or package exposed
  - what would break without it
- Harden SSH configuration correctly and validate it before restarting or reloading services.
- Configure ProxyJump through a router VM to reach a second host on the LAN.
- Enable unattended security updates on Ubuntu and Rocky.
- Prove that automatic updates are functioning.
- Verify the attack surface from outside the system using port scanning and SSH authentication testing.

---

## Environment

This lab is designed for a Linux environment containing at least:

- a VM that is intentionally bloated and insecure
- a second VM or host to test SSH access
- a router VM used for bastion-style access
- Ubuntu 24.04 or Rocky Linux 9 systems as appropriate

 You expected to build  yourself an intentionally bloated VM

---

## Deliverables

You must submit or retain the following evidence:

- `before.txt`
- `after.txt`
- diff between `before.txt` and `after.txt`
- justification table for each removal
- both SSH daemon configuration files
- working `ProxyJump` configuration
- evidence that automatic updates ran successfully

---

# Lab 9.A — Measure and reduce

## Purpose

This snapshot is required evidence for the Week 11 report.

You are expected to measure the “before” state, reduce unnecessary software and services, and then document the difference.

## Step 1: Capture the pre-hardening snapshot

Run the following command and save the output to `~/before.txt`:

```bash
{ ss -tlnp; systemctl list-units --type=service --state=running; dpkg -l | wc -l; find / -perm -4000 -type f 2>/dev/null; } > ~/before.txt
```

This captures:

- listening network services
- running system services
- installed package count
- SUID binaries

This is required evidence and will be used as the baseline before remediation.

## Step 2: Remove unnecessary services and packages

You create, a deliberately bloated VM. It may include:

- a desktop environment
- Samba
- an FTP server
- telnetd
- an unused database
- Avahi
- CUPS
- stale user accounts

Reduce it using:

```bash
sudo systemctl disable --now cups avahi-daemon
sudo apt purge telnetd vsftpd samba
sudo apt autoremove --purge
```

For every removal, record and justify the answer to two critical questions:

1. What did this expose?
2. What will break without it?

A removal that cannot be justified is considered incorrect, even if it appears like a sensible cleanup.

## Step 3: Audit accounts

Check for:

- accounts with UID ≥ 1000
- accounts with a valid shell that should not have one
- accounts with no password set
- expired accounts

Use:

```bash
sudo passwd -S -a | grep -v " P "
sudo lastlog
```

Also review users and shell assignments carefully. Remove or fix stale or unnecessary accounts where appropriate.

## Step 4: Measure the post-hardening state

After the cleanup, save the new state to `~/after.txt`:

```bash
{ ss -tlnp; systemctl list-units --type=service --state=running; dpkg -l | wc -l; find / -perm -4000 -type f 2>/dev/null; } > ~/after.txt
```

Then compare the before/after state:

```bash
diff -u ~/before.txt ~/after.txt
```

## Checkpoint

You should be able to show:

- a shorter `ss -tlnp` output
- a clear `diff`
- a justification table with a row per change

## Justification table

Use a table like this in your report or notes:

| Change | What did this expose? | What would break without it? |
|---|---|---|
| Removed CUPS | Unused print service reduced exposure to network-printing attack surface | Printing features would fail for local or remote users |
| Removed Avahi | Eliminated multicast service discovery exposure | Local service discovery may fail in some LAN-based workflows |
| Purged Telnet | Prevented cleartext remote login and credential leakage | Legacy telnet clients would no longer connect |
| Purged vsftpd | Removed insecure file-transfer service attack surface | FTP-based file transfers would no longer work |
| Purged Samba | Stopped SMB exposure on the system | Windows file sharing and SMB client integration would break |
| Disabled desktop environment | Reduced attack surface and unnecessary software | GUI login and local desktop workflows would not be available |

Add one row per service or package removed and explain both consequences clearly.

---

# Lab 9.B — SSH and patching

## Purpose

This part of the lab focuses on SSH hardening, secure remote access, and automated patching.

## Rule: never test a change with your only active SSH session

Before making changes to SSH configuration, remember:

- never close the only working SSH connection
- always open a second SSH session first
- validate the configuration before restarting or reloading
- test from a second session before disconnecting the original

This is a required habit and should be explicitly followed throughout the lab.

## Step 1: Harden `sshd_config`

Configure the SSH daemon securely on both VMs.

Validate the configuration before reloading:

```bash
sudo sshd -t
```

Reload the SSH daemon, not restart it, so existing sessions remain active:

```bash
sudo systemctl reload sshd
```

This is the professional method because reloading keeps current connections alive while applying the new configuration.

## Step 2: Disable password authentication

After confirming key-based login works, disable password authentication.

This is critical. On Ubuntu 24.04, be aware that the system may include drop-in files under `/etc/ssh/sshd_config.d/` that override the main configuration. A common trap is editing only `/etc/ssh/sshd_config`, while a drop-in file re-enables passwords unexpectedly.

Check relevant files:

```bash
ls /etc/ssh/sshd_config.d/
sudo grep -R "PasswordAuthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

If the active configuration re-enables password login, fix the drop-in file or remove the conflicting override.

Then verify:

```bash
ssh -o PreferredAuthentications=password user@host
```

This must be refused.

## Step 3: Configure ProxyJump through the router VM

Use the Windows `~/.ssh/config` file to create a bastion pattern, where access to the LAN host happens through the router VM.

Example:

```sshconfig
Host router
    HostName 10.10.10.1
    User admin
    IdentityFile ~/.ssh/id_ed25519

Host lan-b
    HostName 10.10.10.20
    User user
    ProxyJump router
    IdentityFile ~/.ssh/id_ed25519
```

Then test by connecting through the bastion:

```bash
ssh lan-b
```

This rehearses the exact bastion pattern required in the final project.

## Step 4: Configure unattended security upgrades

### Ubuntu

Enable unattended upgrades:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Verify the config:

```bash
sudo grep -R "Unattended-Upgrade" /etc/apt/apt.conf.d/
```

### Rocky Linux

Enable automatic security updates via `dnf-automatic`:

```bash
sudo dnf install -y dnf-automatic
sudo systemctl enable --now dnf-automatic.timer
```

Review configuration in:

```bash
/etc/dnf/automatic.conf
```

## Step 5: Force an update and prove it works

Do not assume the service is enabled correctly. Force a run and inspect the logs.

### Ubuntu

```bash
sudo unattended-upgrade --dry-run
sudo unattended-upgrade
sudo journalctl -u unattended-upgrades -n 100 --no-pager
```

### Rocky

```bash
sudo dnf-automatic
sudo journalctl -u dnf-automatic -n 100 --no-pager
```

You must provide evidence showing that the automatic update mechanism ran successfully.

## Step 6: Verify hardening from the outside

From the router VM, scan the target host:

```bash
nmap -sV 10.10.10.20
```

Compare the open ports with the baseline in `before.txt`.

Then test password authentication from outside:

```bash
ssh -o PreferredAuthentications=password user@host
```

It should be refused. This confirms that password-based SSH is disabled.

---

## Evidence to keep

You should retain evidence for all of the following:

- `before.txt`
- `after.txt`
- diff output
- service and package removal justification
- account audit results
- SSH configuration or relevant drop-in config
- `ProxyJump` configuration
- automatic update logs or journal output
- `nmap` scan results
- SSH password attempt refusal output

---

## Final notes

The key lesson in this lab is that security is not just about adding rules or toggling switches — it is about understanding what is exposed, what is unnecessary, and how to validate the system before and after hardening.

A good final submission should demonstrate:

- measured before/after comparison
- justified removals
- secure SSH configuration
- working remote access through a bastion
- automated patching with proof
- verification from the outside

---

## Submission checklist

Before submitting, confirm that you have:

- [ ] captured `before.txt`
- [ ] captured `after.txt`
- [ ] recorded the diff
- [ ] documented each removal with justification
- [ ] hardened SSH on both VMs
- [ ] validated `sshd -t`
- [ ] reloaded SSH without closing the active session
- [ ] disabled password authentication
- [ ] configured `ProxyJump`
- [ ] enabled unattended updates
- [ ] proved automatic updates ran
- [ ] verified external port exposure and password refusal
