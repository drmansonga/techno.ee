# techno.ee
Tasks for the course Linux II



08 challenge project brief · MD
# Challenge Track — Project Brief
 
**IT-2xx Linux Server Administration and Infrastructure Automation**
**Task title:** Reproducible hardened environment, extended
**Issued:** Week 2, on passing the qualifying gate
**Due:** 23:59 the Sunday before Week 15
**Weight:** 45% of the course grade
**Work mode:** individual — pairs are not permitted on this track
 
This is the complete specification. Everything you are graded on is stated here. If a requirement below is ambiguous, ask before Week 10; ambiguity resolved after Week 13 will be resolved in the grader's favour.
 
---
 
## 1. The task in one paragraph
 
Build a four-node Linux environment on a cloud provider, entirely from code, and secure it. A grader must be able to clone your repository into an empty account of their own, supply their own credentials in a variables file, run two commands, and end up with a working, hardened, monitored environment identical to yours. You must then be able to destroy it and rebuild it live, explain any line of it without notes, and diagnose faults introduced into it while you watch.
 
---
 
## 2. Constraints
 
| | |
|---|---|
| Provider | Oracle Cloud Always Free, AWS Free Tier, Hetzner Cloud, or a department Proxmox instance. Choose one by Week 5 and record it in ADR-01. |
| Base images | Ubuntu Server 24.04 LTS and Rocky Linux 9. **At least one node must run each family.** |
| Provisioning | Terraform ≥ 1.7 |
| Configuration | cloud-init and Ansible. `remote-exec` and `local-exec` provisioners are prohibited. |
| Firewall | Hand-written nftables on every node. `ufw` and `firewalld` are prohibited. |
| Cost | Your responsibility. Set a budget alert. Run `terraform destroy` after every working session. |
| Scanning | Only against your own environment, per the Week 1 rules of engagement. |
| AI assistance | Permitted. You must be able to explain every line you submit. This is enforced at the defence, not at submission. |
 
---
 
## 3. Required architecture
 
Four nodes on one private network. Exact names are required — the acceptance tests reference them.
 
| Terraform key | Hostname | OS family | Public IP | Role |
|---|---|---|---|---|
| `bastion` | `bastion-01` | your choice | yes | Only node reachable from the internet. SSH entry point. |
| `app` | `app-01` | your choice | yes (HTTPS only) | Web service over TLS. |
| `db` | `db-01` | your choice | **no** | PostgreSQL or MariaDB. |
| `logs` | `logs-01` | your choice | **no** | Log and audit collector. |
 
**Network plan.** Use these CIDRs. If your provider forbids them, document the substitution in the README.
 
| Segment | CIDR | Members |
|---|---|---|
| Private network (IPv4) | `10.30.0.0/16` | all four nodes |
| Private network (IPv6 ULA) | `fd30:30::/48` | all four nodes |
 
Assign fixed private addresses. Do not rely on DHCP ordering.
 
**Management source.** A single variable, `mgmt_cidr`, defines the only source address range permitted to reach the bastion's SSH port. The grader will set this to their own address.
 
---
 
## 4. Functional requirements
 
Each requirement is testable and is tested. Numbering is used in the mark sheet.
 
### 4.1 Provisioning
 
- **FR-01** `terraform apply` from an empty state creates all four nodes, the private network, and all provider-level firewall rules, with no manual steps.
- **FR-02** Every value a grader must change is a variable with a description and a default where a default is safe. `terraform.tfvars.example` documents all of them.
- **FR-03** At least two variables have `validation` blocks that reject an invalid value at plan time.
- **FR-04** Nodes are created with `for_each`, not `count`. You must be able to explain why at the defence.
- **FR-05** The node definition is a module in `modules/node/`, called four times.
- **FR-06** State is stored in a remote backend with encryption at rest and locking enabled.
- **FR-07** `.terraform.lock.hcl` is committed. `.terraform/`, `*.tfstate*`, and `*.tfvars` (except `.example`) are not.
### 4.2 Bootstrap and configuration
 
- **FR-08** cloud-init on first boot creates an `admin` user with your public key, sets the hostname, installs nftables, and applies a role-appropriate ruleset. The node is firewalled before it is reachable.
- **FR-09** The nftables ruleset is generated per role with `templatefile()`, using addresses Terraform knows — not hard-coded.
- **FR-10** An Ansible playbook applies the full hardening baseline (§4.4). It is run after `apply`, using an inventory derived from Terraform outputs.
- **FR-11** The playbook is idempotent: a second consecutive run reports `changed=0` on every host.
- **FR-12** Ansible reaches `app`, `db`, and `logs` through the bastion via `ProxyJump`. No node other than the bastion accepts SSH from outside the private network.
### 4.3 Network policy
 
- **FR-13** Every node has a default-deny `input` and `forward` policy in a single `inet` table covering IPv4 and IPv6.
- **FR-14** ICMP and ICMPv6 are handled correctly. ICMPv6 neighbour discovery is permitted. Path MTU discovery works.
- **FR-15** `bastion` accepts SSH only from `mgmt_cidr`. Nothing else inbound.
- **FR-16** `app` accepts SSH only from the bastion's private address, and TCP 80 and 443 from anywhere. Port 80 redirects to 443.
- **FR-17** `db` accepts SSH only from the bastion's private address, and the database port only from the app's private address. **`db` has no route to the internet** — outbound connections fail.
- **FR-18** `logs` accepts SSH only from the bastion's private address, and the log ingest port only from the other three nodes' private addresses.
- **FR-19** Dropped inbound packets are logged with rate limiting on every node.
- **FR-20** All rulesets survive a reboot.
- **FR-21** IPv6 works end to end between nodes on the private network — not merely firewalled. If your provider's free tier cannot do this, a WireGuard overlay with the ULA prefix is acceptable and must be documented in the README.
### 4.4 Hardening baseline
 
Applied automatically to every node. Manual configuration scores zero.
 
- **FR-22** SSH: `PermitRootLogin no`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, explicit `AllowUsers` or `AllowGroups`, `X11Forwarding no`. `AllowTcpForwarding` enabled on the bastion only.
- **FR-23** Bastion login requires a second factor, **or** you implement short-lived SSH certificates instead and defend that choice in ADR-05.
- **FR-24** A least-privilege sudo policy exists in `/etc/sudoers.d/` for a non-admin operator role. It must contain no wildcards and no shell-capable binaries.
- **FR-25** A sysctl hardening file is applied, with every line justified in the documentation.
- **FR-26** Automatic security updates are enabled and have demonstrably run.
- **FR-27** SELinux is `Enforcing` on RHEL-family nodes; AppArmor is enforcing on Debian-family nodes. Disabling either is an automatic deduction.
- **FR-28** The web service's systemd unit is sandboxed. `systemd-analyze security` must score it below 5.0.
- **FR-29** AIDE is initialised on every node and its database is stored off-node.
- **FR-30** TLS on `app` uses a certificate with a correct chain and a SAN. `curl` succeeds without `-k` once the CA is trusted. TLS 1.2 and 1.3 only.
### 4.5 Logging and detection
 
- **FR-31** `bastion`, `app`, and `db` ship journal or syslog data to `logs-01` over the private network. Choose the mechanism and defend it.
- **FR-32** `auditd` rules on every node cover at minimum: changes to `/etc/shadow`, changes to `/etc/sudoers` and `/etc/sudoers.d/`, and execution of `sudo`.
- **FR-33** Log data on `logs-01` is separated per source host and survives a reboot of the sending node.
### 4.6 Backup and recovery
 
- **FR-34** The database is backed up on a schedule by a systemd timer, using a dump rather than a copy of live files.
- **FR-35** Backups are stored off the database node.
- **FR-36** You have performed a full restore test and recorded the result.
### 4.7 Pipeline
 
- **FR-37** A CI pipeline runs on every pull request and executes, at minimum: `terraform fmt -check`, `terraform validate`, `tflint`, a security scanner (`tfsec` or `checkov`), and `terraform plan`.
- **FR-38** The plan output is posted or published where a reviewer can read it.
- **FR-39** Credentials come from the CI platform's secret store. No credential appears anywhere in the repository or its history.
---
 
## 5. Repository structure
 
Use this layout. The acceptance tests and the grader's reading order assume it.
 
```
.
├── README.md
├── .gitignore
├── .github/workflows/ci.yml          (or .gitlab-ci.yml)
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── backend.tf
│   ├── versions.tf
│   ├── terraform.tfvars.example
│   ├── .terraform.lock.hcl
│   ├── modules/node/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── templates/
│       ├── cloud-init.yaml.tftpl
│       └── nftables-<role>.conf.tftpl
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.sh                  (or inventory.py / static generated file)
│   ├── site.yml
│   ├── group_vars/
│   └── roles/
│       ├── common/
│       ├── ssh/
│       ├── firewall/
│       ├── audit/
│       └── ...
└── docs/
    ├── threat-model.md
    ├── detection.md
    ├── disaster-recovery.md
    ├── hardening-report.md
    ├── network-diagram.png            (or .svg)
    └── adr/
        ├── 001-provider.md
        ├── 002-mac-system.md
        ├── 003-database-exposure.md
        ├── 004-secrets-handling.md
        └── 005-log-shipping.md
```
 
---
 
## 6. Deliverables
 
Twelve items. Each has a required location and format. A deliverable in the wrong place or format is marked as absent.
 
**D1 — Terraform configuration.** `terraform/`. Satisfies FR-01 to FR-07.
 
**D2 — Bootstrap templates.** `terraform/templates/`. cloud-init and per-role nftables templates. Satisfies FR-08, FR-09.
 
**D3 — Ansible playbook.** `ansible/`. Satisfies FR-10 to FR-12 and the whole of §4.4.
 
**D4 — nftables rulesets.** Generated, not hand-placed. Satisfies §4.3. Include the rendered ruleset from each of the four nodes in `docs/hardening-report.md` as an appendix.
 
**D5 — `README.md`.** Root of the repository. Must contain, in this order:
1. What this environment is and what it runs.
2. Prerequisites: tool versions, provider account, permissions needed.
3. Exact build instructions — every command, in order, that takes a grader from `git clone` to a working environment. Assume no prior knowledge of your work.
4. How to connect to each node.
5. How to verify it works — the commands a grader should run and what output proves success.
6. How to destroy it.
7. Known limitations and anything a grader should not be surprised by.
Length is whatever it takes. It is graded by having someone who has never seen your project follow it.
 
**D6 — Threat model.** `docs/threat-model.md`, 3–5 pages, structured on four headings:
- *What are we protecting?* Assets, in order of value.
- *From whom?* At least three distinct adversaries with different capabilities and motives.
- *What can go wrong?* Walk the data flow. For each point, what an attacker at that position could do.
- *What are we doing about it, and what are we not?* A table mapping each control to the threat it addresses, followed by an explicit list of threats you decided not to address, each with a reason.
The final section is worth more than the first three. A threat model claiming to address everything is not believed.
 
**D7 — Hardening report.** `docs/hardening-report.md`. Must contain:
- Lynis output before and after hardening, with the hardening index for each node.
- OpenSCAP results against a CIS profile for the RHEL-family node.
- A findings triage table with one row per high and medium finding: finding, severity, what it prevents, applies here yes/no, action taken, justification.
- Every finding left open, with a reason.
- The `systemd-analyze security` score for the web service unit, before and after.
- Appendix: the rendered nftables ruleset from each node.
**D8 — Detection write-up.** `docs/detection.md`. For each of four attacks, performed against your own environment:
 
| Attack | Required evidence |
|---|---|
| SSH brute force against the bastion | Which log, on which node, and the actual log lines |
| Unauthorised sudo escalation attempt by the operator account | Same |
| Modification of a system binary or a file in `/etc` | Same |
| Outbound connection attempt from `db-01` | Same |
 
For each: the command you ran to generate it, the log entry that captured it, which node the evidence ended up on, and **what you would not detect and why**. The last column is graded most heavily.
 
**D9 — Disaster recovery test.** `docs/disaster-recovery.md`. Destroy the database node completely (`terraform destroy -target` or equivalent), rebuild it, and restore the data. Record:
- The exact recovery procedure, as a runbook someone else could follow at 03:00.
- Measured RTO (wall-clock time from destruction to service restored).
- Measured RPO (how much data was lost, given your backup schedule).
- Verification evidence: row counts or checksums before and after.
- What went wrong during the test and what you changed as a result.
A backup design with no restore evidence scores zero for this deliverable.
 
**D10 — Architecture decision records.** `docs/adr/`, five files, one page each, same structure in every file:
```
# ADR-00N: <title>
## Status
Accepted | Superseded by ADR-00X
## Context
What forced a decision.
## Options considered
At least two real alternatives, with what each would have cost.
## Decision
What you chose.
## Consequences
What this makes easy, what it makes hard, and what would make you revisit it.
```
Required subjects: 001 provider, 002 MAC system, 003 database exposure model, 004 secrets handling, 005 log shipping. Graded on the quality of reasoning about the **rejected** options.
 
**D11 — CI pipeline.** `.github/workflows/ci.yml` or equivalent. Satisfies FR-37 to FR-39. Your repository must show at least three pull requests where the pipeline ran, including one where it failed and one where the failure was fixed. A pipeline with no failing run in its history suggests it was added at the end.
 
**D12 — Network diagram.** `docs/network-diagram.png` or `.svg`. All four nodes, every interface, every network with both CIDRs, every permitted flow with its port and direction, and the management entry point. It must match the running environment; the grader will spot-check three claims against the live system.
 
---
 
## 7. Acceptance tests
 
These are the commands the grader will run. Run them yourself before submitting.
 
**A — Cold build.** From a clean clone, in the grader's own account:
```bash
git clone <your-repo> && cd <repo>/terraform
cp terraform.tfvars.example terraform.tfvars   # grader edits credentials and mgmt_cidr
terraform init && terraform apply
cd ../ansible && ansible-playbook -i inventory.sh site.yml
```
**Pass:** four nodes exist, the service responds, no manual step was needed beyond editing `terraform.tfvars`.
 
**B — Idempotence.**
```bash
ansible-playbook -i inventory.sh site.yml     # second consecutive run
```
**Pass:** `changed=0` for every host.
 
**C — Reachability matrix.** Every cell is tested.
 
| From ↓ To → | bastion:22 | app:443 | app:22 | db:5432 | db:22 | logs:22 |
|---|---|---|---|---|---|---|
| Grader's laptop (in `mgmt_cidr`) | **allow** | **allow** | deny | deny | deny | deny |
| Grader's laptop (outside `mgmt_cidr`) | deny | **allow** | deny | deny | deny | deny |
| bastion | — | **allow** | **allow** | deny | **allow** | **allow** |
| app | deny | — | — | **allow** | deny | deny |
| db | deny | deny | deny | — | — | deny |
 
Plus: `curl -I https://<app>` from `db-01` **must fail** (FR-17).
 
**D — Persistence.** Every node is rebooted. All rulesets, services, and mounts return.
 
**E — TLS.** `curl https://<app-name>` succeeds without `-k` after installing your CA. `testssl.sh` shows TLS 1.2/1.3 only.
 
**F — MAC.** `getenforce` returns `Enforcing` on the RHEL node. `aa-status` shows enforcing profiles on the Debian node.
 
**G — Sandbox.** `systemd-analyze security <web-unit>` returns a score below 5.0.
 
**H — Secrets.** The grader runs, on the full history:
```bash
git log --all -p | grep -inE "BEGIN (RSA|OPENSSH|EC|PRIVATE) KEY|api[_-]?token|password\s*=|secret_key"
```
Any hit is the automatic deduction in §9.
 
**I — Destroy.** `terraform destroy` leaves nothing behind in the provider console.
 
---
 
## 8. Milestones
 
Bring a laptop or a Cloud Shell session. Each is 20 minutes, 1:1, pass/fail. Failing one moves you to the standard track.
 
| | Week | You must demonstrate |
|---|---|---|
| **M1** | 5 | Repository with `.gitignore` as the first commit; Terraform provisions two or more nodes on a private network; `git log --all -p` shows no secrets. |
| **M2** | 10 | All four nodes provisioning from empty; hand-written nftables per role applied by cloud-init; IPv6 covered; live proof that `db` is unreachable from outside and has no outbound route; threat model draft submitted. |
| **M3** | 13 | Remote state with a lock demonstrated live; Ansible reporting `changed=0` on a second run; CI running on a pull request; logs arriving at `logs-01`; a timed `destroy` → `apply` → `ansible-playbook` cycle. |
 
---
 
## 9. Marking
 
| Criterion | Weight |
|---|---|
| Reproducibility — cold build from an empty account with no manual steps | 20% |
| Architecture — segmentation genuinely enforced, verified not asserted | 15% |
| Firewall policy — default-deny, IPv6 covered, generated, every rule explicable | 10% |
| Hardening baseline — applied automatically, verified by audit | 10% |
| Threat model and ADRs — reasoning quality, honesty of the non-goals | 15% |
| Detection and disaster recovery — evidence, not description | 10% |
| Documentation — a stranger can build and operate it from the README | 10% |
| Defence and live fault diagnosis | 10% |
 
**Automatic deductions, in percentage points off the project mark:**
 
| | |
|---|---|
| Any secret in Git history | −20 |
| `terraform.tfstate` committed | −10 |
| SELinux or AppArmor disabled rather than configured | −10 |
| SSH password authentication enabled on any node | −10 |
| `curl -k` / `--insecure` used during the demo to make TLS work | −5 |
| `remote-exec` or `local-exec` provisioner present | −5 |
| Manual step required during the cold build that is not in the README | −5 per step |
 
---
 
## 10. Defence — Week 15, 45 minutes, unaided
 
**Part A, 15 min — build.** Destroy and rebuild live from empty. Run the playbook. Demonstrate TLS, the reachability matrix, logs arriving at `logs-01`, and your most recent CI run.
 
**Part B, 10 min — fault injection.** Two faults are introduced while you look away: one at the infrastructure layer, one inside a host. You diagnose aloud. **Marking is on method** — form a hypothesis, choose a command that distinguishes between causes, read the output, change one thing at a time. Working methodically and not finishing scores above guessing correctly.
 
**Part C, 20 min — questions.** On any part of your repository, threat model, or ADRs. Sample:
- Show me a commit where you got something wrong. What did you learn?
- Your scanner raised a finding you did not fix. Defend that.
- Someone has your bastion's private key. Walk me through what they reach, in order, and where the first alert fires.
- Which Ansible task was hardest to make idempotent, and why?
- ADR-003 chose X over Y. What would have to change for Y to be correct?
- Delete this line from your ruleset. What breaks? Now do it and show me.
- Who can read your state bucket, and how do you know?
---
 
## 11. Submission
 
By the deadline, submit:
1. The repository URL, with the grader's account granted read access.
2. The commit hash you are submitting.
3. A single page listing which requirement number (FR-01 … FR-39) is satisfied where — file and line or directory. This is your map for the grader; without it, anything they cannot find is marked absent.
Leave the environment **destroyed** at submission. You rebuild it live at the defence.
 
---
 
## 12. Definition of done
 
Tick every line before you submit.
 
- [ ] Cloned the repository into a fresh directory and built it from nothing, following only my own README
- [ ] Someone else has read my README and told me where it was unclear
- [ ] `ansible-playbook` reports `changed=0` on a second run
- [ ] Every cell of the reachability matrix tested and recorded
- [ ] `curl -I https://...` from `db-01` fails
- [ ] All four nodes rebooted; everything returned
- [ ] `git log --all -p` grepped for secrets; clean
- [ ] `terraform.tfvars` is not in the repository; `terraform.tfvars.example` is
- [ ] `.terraform.lock.hcl` is committed
- [ ] Lynis run after hardening on all four nodes, output in the report
- [ ] Restore test performed, with timings recorded
- [ ] All four detection scenarios triggered and their log evidence captured
- [ ] Five ADRs written, each naming at least two real alternatives
- [ ] Network diagram matches the running environment
- [ ] Requirement map (§11.3) complete
- [ ] Environment destroyed
 

