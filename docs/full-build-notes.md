<div align="center">

# 🛠️ Full Build Notes — Wazuh Detection Engineering Lab

### Infrastructure Buildout · Attack Simulation · Detection Engineering

The complete chronological record of the lab as it was actually built on **July 11–12, 2026** — the original plan, the failures encountered, the false leads investigated, the fixes applied, and the evidence used to validate the final results.

<br>

![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh_4.14.6-005C99?style=for-the-badge)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Kali](https://img.shields.io/badge/Kali_Linux-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![Atomic Red Team](https://img.shields.io/badge/Atomic_Red_Team-D62828?style=for-the-badge)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE_ATT%26CK-C41E3A?style=for-the-badge)

</div>

<br>

> **Purpose:** show the troubleshooting and detection-engineering process behind the polished [project summary](../README.md) — what broke, how it was diagnosed, and how each fix was validated.

---

## Contents

**Overview**
- [Project outcome](#project-outcome)
- [How to read this document](#how-to-read)
- [Original architecture plan](#architecture-plan)
- [Final lab layout](#final-layout)

**Phase 1 — Infrastructure Buildout** *(complete)*
- [Five documented failures, each diagnosed and fixed](#phase-1)
- [Kali, Ollama, and the measured inference bottleneck](#phase-1)

**Phase 2 — Detection Engineering** *(three techniques closed)*
- [Baseline tests and two false leads](#phase-2)
- [Fix 1 — real-time FIM for `/tmp`](#fix-1)
- [Fix 2 — auditd ingestion and correlation rule 100100](#fix-2)

**Reference**
- [Troubleshooting playbook](#troubleshooting)
- [Limitations and open work](#limitations)
- [Evidence index](#evidence)

---

<a id="project-outcome"></a>

## ✅ Project outcome

- Built a four-VM security lab on Proxmox VE.
- Deployed a Wazuh all-in-one stack and enrolled a Linux victim endpoint.
- Ran Atomic Red Team tests and measured Wazuh's default coverage.
- Found and investigated three detection gaps.
- Applied two detection patterns: real-time FIM and audit-based process monitoring.
- Re-ran the tests and validated the new alerts in Wazuh.
- Phase 2 result: **three techniques closed**.
- Custom milestone: an auditd-backed Wazuh **correlation rule** for a command with no file artifact.

<a id="how-to-read"></a>

## 🧭 How to read this document

The original checklist was a plan, not a claim that every item had been completed.

The status table below separates completed work from open work. Later sections provide the chronological build and investigation record.

| Planned work | Status | Evidence or remaining work |
| --- | --- | --- |
| Deploy Wazuh all-in-one | Complete | Manager, indexer, and dashboard upgraded to 4.14.6 |
| Enroll 2–3 mixed endpoints | Partial | One Ubuntu victim agent enrolled and active |
| Confirm telemetry flow | Complete | Agent, FIM, and audit events validated |
| Add pfSense rules | Open | Rules between lab and management networks still need review |
| Run attack simulations | Complete for selected tests | T1059.004-1, T1082-3, and T1082-8 executed |
| Close default detection gaps | Complete for selected tests | Real-time FIM and rule 100100 validated |
| Tune false positives | Open | Rootcheck rule 510 identified as noisy; tuning not yet applied |
| Feed downstream AI projects | Planned | Wazuh API integration remains future work |
| Publish documentation | Complete for this repository | README, full notes, evidence, and reusable configurations are included |
| Configure remote access | Nice to have | WireGuard on pfSense remains optional |

<a id="architecture-plan"></a>

## 📐 Original architecture plan

The initial design used Wazuh's native all-in-one stack. A separate Elastic or Kibana deployment was not required for this lab.

The Wazuh VM was placed in the lab network instead of the management network. Agents need access to ports 1514 and 1515, while administrative access to the dashboard uses HTTPS on port 443.

The lab was planned around a 32 GB RAM budget:

| System | Purpose | Planned memory |
| --- | --- | --- |
| Wazuh | Manager, indexer, and dashboard | 8–12 GB |
| Ollama | Local LLM inference and log analysis | 8–12 GB |
| Victim | Disposable Atomic Red Team target | 2–4 GB |
| Kali | Attacker and scanning system | 4 GB |
| Proxmox | Hypervisor overhead | Remaining memory |

The intended order was:

1. Confirm the hardware upgrade and Proxmox installation.
2. Deploy Wazuh.
3. Enroll endpoints and verify alerts.
4. Build the disposable victim.
5. Run Atomic Red Team.
6. Tune detection controls.
7. Add Ollama and optional AI-assisted analysis.

The original plan also included mixed Windows, Linux, and macOS agents. Only the Linux victim was enrolled during this build session.

### Security and telemetry requirements from the plan

- Replace generated or default administrative credentials after installation.
- Allow only the required agent enrollment, event, and dashboard traffic.
- Generate a benign event before attack testing to prove the basic pipeline.
- Review the agent's `ossec.log` and inter-VLAN firewall policy when events do not arrive.
- Keep the victim disposable and use Proxmox snapshots before aggressive tests.

### Detection-development ideas from the plan

- Start with a small number of safe Atomic Red Team tests instead of the complete suite.
- Use Caldera later if automated, multi-stage adversary emulation becomes useful.
- Draft portable Sigma detections before converting them to Wazuh XML where practical.
- Record the technique, default behavior, detection change, and re-test result for each rule.
- Keep a decision log for noisy-rule suppression and other tuning.

### Downstream ideas from the plan

- Use real Wazuh API output for future AI-assisted log analysis.
- Send future Falco runtime alerts to Wazuh as a SIEM sink.
- Reuse attack and detection evidence in blue-team or Hack The Box write-ups.
- Capture screenshots during the work instead of reconstructing evidence later.
- Summarize the project in the broader Home-Lab repository and link to this dedicated repo.

<a id="final-layout"></a>

## 🗂️ Final lab layout

| VM | Role | Address | Resources or state |
| --- | --- | --- | --- |
| `wazuh-aio` | Wazuh manager, indexer, dashboard | `192.168.2.101` | 4 vCPU, 8 GB RAM, 70 GB disk |
| `victim-01` | Disposable Ubuntu target | `192.168.2.102` | 2 vCPU, 4 GB RAM, 35 GB disk |
| `kali-01` | Attacker and scanning system | `192.168.2.103` | 4 vCPU, 4 GB RAM, 50 GB disk |
| `ollama-01` | CPU-only local LLM inference | `192.168.2.104` | Reduced to 4 vCPU after a start failure |

These are private lab addresses. They document the internal design but are not public Internet endpoints.

---

<a id="phase-1"></a>

<div align="center">

# 🏗️ Phase 1 — Infrastructure Buildout

*Four VMs built, networked, and validated — with five failures diagnosed and fixed along the way.*

</div>

---

## 🚦 Starting point

Proxmox VE 9.2.2 was already installed on the Lenovo M710q host, named `pve01`.

Later Proxmox UI captures show 9.2.4 after host updates. The build record keeps 9.2.2 as the starting version and treats 9.2.4 as the later observed state.

The `local` and `local-lvm` storage pools appeared healthy. The session goal was to build the Wazuh, victim, Kali, and Ollama VMs.

## 🧯 Failure 1: Proxmox enterprise repository errors

### Symptom

The Proxmox task log showed a failed package database update.

`apt-get update` returned `401 Unauthorized` for the `pve` and `ceph-squid` repositories at `enterprise.proxmox.com`.

### False start

The first attempted fix targeted:

```text
/etc/apt/sources.list.d/pve-enterprise.list
```

That file did not exist.

### Diagnosis

Proxmox VE 9 used the newer deb822 `.sources` format. The active files were:

```text
pve-enterprise.sources
ceph.sources
```

The enterprise repositories require a subscription. This lab used the no-subscription repository instead.

### Fix

The enterprise source files were renamed with a `.disabled` suffix. A `pve-no-subscription.sources` file was added for `download.proxmox.com`.

### Validation

`apt-get update` completed without the previous `401` responses.

The “No valid subscription” login message remains cosmetic and does not block package management.

> **Lesson:** Repository instructions can become stale when a distribution changes its source-file format. Confirm the files actually loaded by the installed version before editing them.

## 🧯 Failure 2: KVM virtualization unavailable

### Symptom

The first Wazuh VM would not start. Proxmox reported a KVM virtualization error.

### Diagnosis

The host returned `0` for:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

The CPU supported virtualization, but VT-x was not enabled in firmware.

### Fix

On the physical M710q, the following options were enabled in Lenovo BIOS:

- **Advanced → CPU Setup → Intel Virtualization Technology (VT-x)**
- **Advanced → CPU Setup → VT-d**

Device Guard was not the relevant setting.

### Validation

After reboot, the CPU check returned `8`, matching the host's logical threads. The VM then started successfully.

> **Lesson:** When the hypervisor cannot see `vmx` or `svm`, no guest-level or remote software change can replace the missing firmware setting.

## 📥 Deploying Wazuh all-in-one

A fresh Ubuntu Server 22.04.5 VM was created at `192.168.2.101` with 4 vCPU, 8 GB RAM, and a 70 GB disk.

The official all-in-one installer was used:

```bash
wazuh-install.sh -a
```

The first attempt used `-a~` because of a copy-and-paste error. The installer returned an unknown-option error and made the typo easy to isolate.

The corrected command completed successfully. The dashboard became reachable at `https://192.168.2.101` with the generated administrator credentials.

The installed Wazuh version was initially 4.9.2.

## 🧯 Failure 3: Agent and manager version mismatch

### Symptom

The Ubuntu victim's Wazuh agent service was active locally, but the endpoint never appeared as active in the dashboard.

### Diagnosis

`/var/ossec/logs/ossec.log` on the victim contained:

```text
Agent version must be lower or equal to manager version
```

The agent repository installed version 4.14.6, while the manager still ran 4.9.2.

### Decision

The stack was upgraded instead of downgrading the agent. This avoided pinning the lab to an older manager version.

A Proxmox snapshot of the Wazuh VM was taken before the upgrade.

### Fix

The manager, indexer, dashboard, and Filebeat packages were upgraded together:

```bash
apt install --only-upgrade wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

Upgrading the components together reduced the risk of internal version mismatches.

## 🧯 Failure 4: Dashboard certificate filename mismatch

### Symptom

After the Wazuh upgrade, `wazuh-dashboard` entered a restart loop. Port 443 was not listening even when a brief status check appeared active.

### Diagnosis

`journalctl -u wazuh-dashboard` showed:

```text
Error: ENOENT: no such file or directory,
open '/etc/wazuh-dashboard/certs/dashboard-key.pem'
```

The earlier installation created:

```text
wazuh-dashboard-key.pem
wazuh-dashboard.pem
```

The upgraded configuration expected:

```text
dashboard-key.pem
dashboard.pem
```

### Fix

Symlinks preserved the old filenames while satisfying the upgraded configuration:

```bash
ln -s wazuh-dashboard-key.pem dashboard-key.pem
ln -s wazuh-dashboard.pem dashboard.pem
```

### Validation

After restarting the service, the restart loop stopped, port 443 listened again, and the dashboard loaded normally.

## ✅ Agent enrollment succeeds

The Wazuh agent on `victim-01` was restarted after the server-side upgrade.

The dashboard then showed one active agent. This validated the complete path from endpoint agent to manager, indexer, and dashboard.

## 💾 Thin-pool overcommit warning

Proxmox warned that the combined virtual disks totaled about 191 GB while the thin pool reported roughly 16 GB free.

`lvs` showed actual physical utilization near 29% of the 141 GB pool. This was thin provisioning, not immediate exhaustion.

The warning remains operationally important because Wazuh indexes, VM snapshots, and LLM models can consume real storage quickly.

## 🐉 Kali VM

Kali 2026.1 was installed as VM 102 at `192.168.2.103`.

Although 2026.2 had just been released, the rolling installation was brought current after first boot:

```bash
apt update && apt full-upgrade -y
```

The update was much larger than the Ubuntu updates because Kali includes a broad tool catalog.

SSH was not enabled by default, so it was enabled manually:

```bash
systemctl enable ssh
systemctl start ssh
```

## 🧠 Ollama VM and hardware limits

### vCPU ceiling

The Ollama VM was initially assigned 6 vCPU. Proxmox refused to start it with:

```text
MAX 4 vcpus allowed
```

The VM used CPU type `host`, and the physical processor had four cores. Reducing the VM to 4 vCPU allowed it to start.

### Resource decision

Wazuh and the victim form the always-available detection baseline. Kali and CPU-heavy LLM work are used when needed to control contention on the four-core host.

Ollama was also configured with `qm set 103 --onboot 1` for scheduled overnight analysis. The long-term startup policy still needs to be reconciled with the resource-conservation plan.

### CPU-only installation

The official Ollama installer reported:

```text
No NVIDIA/AMD GPU detected, Ollama will run in CPU-only mode
```

This confirmed that model performance would be limited by CPU inference rather than GPU acceleration.

## 🧯 Failure 5: Ollama disk sizing and interrupted downloads

### Symptom

An unattended `llama3.1:8b` pull stalled overnight.

### Diagnosis

The Ubuntu installer allocated only 24 GB of the VM's 50 GB virtual disk to the root logical volume.

The original pull also ran directly in SSH with no `tmux` or `nohup`, so a disconnected session could terminate it.

### Fix

The root logical volume and filesystem were expanded:

```bash
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

The model pull was then run inside `tmux`.

### Validation

After an SSH disconnect, `tmux attach` restored the active download without losing progress.

`llama3.1:8b` was pulled and tested with an accurate MITRE ATT&CK response.

The security-focused model was installed as `jimscard/whiterabbit-neo:13b`, a community GGUF conversion. A plain `whiterabbitneo` tag was not available in Ollama's library.

## 📊 Open WebUI and measured inference bottleneck

Docker and Open WebUI were installed. The browser interface was exposed at `http://192.168.2.104:8080` and connected to Ollama at `http://127.0.0.1:11434`.

Both models appeared in the selector.

### Symptom

An interactive request to `jimscard/whiterabbit-neo:13b` returned a generic response error in Open WebUI.

### Diagnosis

Ollama logs showed that the model had not crashed. Prompt processing took 114 seconds at 4.49 tokens per second, and generation ran at about 2 tokens per second.

Open WebUI timed out before the model finished.

A separate UI capture reported that the model did not support tools. That compatibility error is not evidence of the timeout; the performance diagnosis came from the Ollama service measurements above.

### Decision

- Use `llama3.1:8b` for interactive testing.
- Reserve `jimscard/whiterabbit-neo:13b` for unattended overnight jobs.
- Treat desktop GPU acceleration as a separate follow-on project.

> **Lesson:** Measured throughput turned a general hardware concern into an actionable operating policy. The slow model remained useful, but not for live interaction on this host.

## 🏁 Phase 1 closeout

At the end of Phase 1:

- Wazuh manager, indexer, and dashboard ran version 4.14.6.
- The victim agent was active.
- Kali was current and reachable over SSH.
- Ollama served two local models.
- Open WebUI provided browser access.
- CPU and storage limits were measured and documented.

Phase 1 proved the infrastructure worked. It did not yet prove that Wazuh could detect the selected attack simulations.

---

<a id="phase-2"></a>

<div align="center">

# 🎯 Phase 2 — Detection Engineering

*Testing what the default Wazuh ruleset catches — and closing the gaps it misses.*

</div>

---

## 🧭 Objective

The Phase 2 method was simple:

1. Run one Atomic Red Team test on `victim-01`.
2. Record the exact execution window.
3. Search Wazuh for relevant alerts.
4. Separate true detections from coincidental background events.
5. Diagnose the telemetry or rule gap.
6. Apply the smallest useful control.
7. Re-run the same test and validate the result.

## 🧮 Final validation matrix

| Technique/test | Initial result | Control applied | Validated result |
| --- | --- | --- | --- |
| T1059.004-1: Create and Execute Bash Shell Script | No relevant alert | Real-time FIM on `/tmp` | Rule 550 detected `/tmp/art.sh` twice |
| T1082-3: List OS Information | No relevant alert | Existing real-time FIM on `/tmp` | Rule 550 detected `/tmp/T1082.txt` |
| T1082-8: Hostname Discovery | No alert and no file artifact | auditd, Wazuh audit ingestion, correlation rule 100100 | Four level-5 alerts mapped to T1082 |

## 🧪 Baseline test 1: T1082-3

Atomic test T1082-3 completed with exit code 0. The first Wazuh search returned no relevant alert.

The test performs:

```bash
uname -a >> /tmp/T1082.txt
cat /etc/lsb-release >> /tmp/T1082.txt
uptime >> /tmp/T1082.txt
cat /tmp/T1082.txt
rm /tmp/T1082.txt
```

The output file is created and removed quickly, which became important during root-cause analysis.

## 🧪 Baseline test 2: T1059.004-1

The parent T1059 technique returned no Linux-compatible Atomic tests. The investigation moved to the Linux-specific T1059.004 Unix Shell sub-technique.

Seventeen Linux-compatible tests were available. Test 1 writes a Bash script, executes it, and removes it during cleanup.

```powershell
Invoke-AtomicTest T1059.004 -TestNumbers 1
```

The test completed with exit code 0, but the initial Wazuh review found no relevant detection.

## 🕵️ False lead 1: MITRE dashboard category

A one-hour search window returned 217 events labeled in the MITRE visualization as “Stored Data Manipulation.” This initially appeared related to the Atomic execution.

The search was narrowed to the test window, 12:28–12:32, using the dashboard's absolute time picker.

The picker accepted this format:

```text
Jul 12, 2026 @ 12:28:00.000
```

It rejected the ISO-like value `2026-07-12 12:28:00.000`.

The 217 events were CIS Ubuntu 22.04 LTS Security Configuration Assessment checks under rules 19007, 19008, and 19009.

They were scheduled compliance activity, not Atomic Red Team telemetry.

> **Lesson:** Dashboard visualizations are leads, not proof. The event's `rule.description`, source, timestamp, and decoded fields determine whether it is relevant.

## 🕵️ False lead 2: Rootcheck rule 510

A search for `.sh` returned two level-7 rule 510 events at 12:30:52, inside the execution window.

The timing looked promising, but the full records showed that rootcheck had flagged `/usr/bin/diff` with a broad generic trojan-signature pattern.

```text
bash|^/bin/sh|file\.h|/proc\.h|/dev/[^n]|^/bin/.*sh
```

The binary and signature were unrelated to the Atomic script. The events were coincidental false positives.

> **Lesson:** Timestamp proximity is not causation. Validate the observed path and event content against the test's actual behavior.

## 🔍 Confirming the real FIM gap

A direct search for `/tmp` returned no results.

The victim's default syscheck directories were limited to paths such as `/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, and `/boot`.

Two conditions caused the miss:

1. `/tmp` was outside the monitored directory scope.
2. The default FIM scan interval was 43,200 seconds, or 12 hours.

The Atomic cleanup removed the temporary script in about a second. A file that exists between periodic scans is invisible to scheduled FIM.

<a id="fix-1"></a>

## 🔧 Fix 1: Real-time FIM for `/tmp`

A targeted entry was added to `/var/ossec/etc/ossec.conf` on `victim-01`:

```xml
<directories realtime="yes">/tmp</directories>
```

The entry was kept separate from the lower-change system directories to avoid applying unnecessary real-time overhead to all existing paths.

The Wazuh agent was restarted:

```bash
sudo systemctl restart wazuh-agent
```

`ossec.log` confirmed registration:

```text
Directory set for real time monitoring: '/tmp'.
```

## ✅ Validation 1: T1059.004-1

The same Atomic test was run twice after the change.

| Time | Path | Action | Rule | Level |
| --- | --- | --- | --- | --- |
| 15:37:05.584 | `/tmp/art.sh` | Modified | 550: Integrity checksum changed | 7 |
| 15:37:58.325 | `/tmp/art.sh` | Modified | 550: Integrity checksum changed | 7 |

Wazuh also recorded changes to `/tmp/Invoke-AtomicTest-ExecutionLog.csv` on both runs.

The repeated result showed that the detection was reproducible and that the directory watch was not limited to a single filename.

## 🧭 Validation detour: A working fix looked broken

The FIM control was already working, but two investigation mistakes briefly made it appear unsuccessful.

### Timezone mismatch

Raw `ossec.log` timestamps were read as local time even though the log used UTC.

The victim itself also used `Etc/UTC`, while the dashboard displayed local time. The clocks were aligned with:

```bash
sudo timedatectl set-timezone America/New_York
sudo timedatectl set-ntp true
```

### Wrong search surface

A loose `.sh` search in **Threat Hunting → Events** returned no useful result.

The detection was visible in **Endpoints → victim-01 → FIM: Recent events**, which directly exposes syscheck activity.

An explicit `rule.groups: syscheck` filter is another reliable approach.

> **Lesson:** Before declaring a failed detection, align system time, note the display timezone, and use the product view designed for the telemetry source being tested.

## ✅ Validation 2: T1082-3 closes with the same control

T1082-3 also writes rapidly to `/tmp` and removes its output. No additional configuration was needed after the FIM change.

The test was re-run:

```powershell
Invoke-AtomicTest T1082 -TestNumbers 3
```

Wazuh recorded:

```text
Jul 12, 2026 @ 15:55:06.851
/tmp/T1082.txt
Integrity checksum changed
Rule 550, level 7
```

One path-focused control covered two tests because both shared the same fast write-and-delete behavior in `/tmp`.

This showed the value of controlling a reusable behavior pattern instead of creating a filename-specific alert.

## 🧪 Baseline test 3: T1082-8

T1082-8, Hostname Discovery, runs a bare `hostname` command with no arguments and no prerequisite.

It does not create a file artifact, so the FIM control could not observe it.

The baseline re-run produced no relevant event in Threat Hunting. This required a process-execution telemetry path.

<a id="fix-2"></a>

## 🔧 Fix 2: auditd telemetry and correlation rule 100100

### Step 1: Install auditd

`auditd` was not installed on `victim-01`.

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

### Step 2: Add a kernel audit rule

The following rule was added to `/etc/audit/rules.d/audit.rules`:

```text
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/hostname -k t1082_recon
```

The rule was loaded and listed:

```bash
sudo augenrules --load
sudo auditctl -l
```

### Step 3: Validate auditd independently

After running T1082-8, the endpoint was queried directly:

```bash
sudo ausearch -k t1082_recon
```

The result contained successful EXECVE records with:

```text
exe="/usr/bin/hostname"
comm="hostname"
success=yes
```

This proved the kernel audit layer worked before Wazuh was changed.

### Step 4: Enable Wazuh audit-log ingestion

Wazuh was not reading `/var/log/audit/audit.log`. The following block was added to the victim's `ossec.conf`:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

After an agent restart, `ossec.log` confirmed:

```text
Analyzing file: '/var/log/audit/audit.log'
```

### Step 5: Diagnose the silent rule path

Events still did not appear as alerts. A raw EXECVE record was passed to `wazuh-logtest` on the manager.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Decoding succeeded, including extraction of:

```text
audit.execve.a0: 'hostname'
```

The record matched built-in rule 80700, “Audit: Messages grouped,” at level 0.

The pipeline was working, but the generic event was intentionally silent until a more specific rule promoted it.

### Step 6: Add correlation rule 100100

The following rule was added to `/var/ossec/etc/rules/local_rules.xml` on the manager:

```xml
<group name="local,audit,t1082,discovery,">
  <rule id="100100" level="5">
    <if_sid>80700</if_sid>
    <field name="audit.execve.a0">hostname</field>
    <description>T1082 - System Information Discovery: hostname command executed (audit)</description>
    <mitre>
      <id>T1082</id>
    </mitre>
  </rule>
</group>
```

The Wazuh manager was restarted to load the rule.

## ✅ Validation 3: T1082-8

The exact Atomic test was run again:

```powershell
Invoke-AtomicTest T1082 -TestNumbers 8
```

Threat Hunting showed four matching events with:

- Rule ID `100100`
- Level `5`
- The custom description from `local_rules.xml`
- MITRE ATT&CK mapping `T1082`

This validated the full chain:

```text
Atomic test
  → Linux execve event
  → auditd rule
  → /var/log/audit/audit.log
  → Wazuh agent ingestion
  → Wazuh audit decoder
  → built-in parent rule 80700
  → correlation rule 100100
  → dashboard alert
```

## 🏁 Phase 2 closeout

The selected work ended with **three techniques closed**:

1. **T1059.004-1** — closed through real-time FIM on `/tmp`.
2. **T1082-3** — closed through the same real-time FIM control.
3. **T1082-8** — closed through auditd ingestion and correlation rule 100100.

The lab demonstrated two detection-engineering patterns:

- Real-time file monitoring for short-lived filesystem artifacts.
- Audit-based command execution detection for activity with no filesystem artifact.

---

<a id="troubleshooting"></a>

<div align="center">

# 🧰 Troubleshooting Playbook

*Reusable method distilled from the build.*

</div>

---

## 🧪 Reproduce the controls

Copy-paste sequences to recreate each validated control on an agent (`victim-01`) and the manager. Adapt paths to your environment and validate in a lab first. Reusable fragments live in [`../configs/`](../configs/).

### Real-time FIM for `/tmp`

Add this line inside the `<syscheck>` block of `/var/ossec/etc/ossec.conf` on the agent (fragment: `../configs/wazuh/agent-fim.xml`):

```xml
<directories realtime="yes">/tmp</directories>
```

```bash
sudo systemctl restart wazuh-agent
sudo grep "real time monitoring" /var/ossec/logs/ossec.log
```

### auditd telemetry for `hostname`

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd

# fragment: ../configs/auditd/t1082-hostname.rules
echo '-a always,exit -F arch=b64 -S execve -F path=/usr/bin/hostname -k t1082_recon' \
  | sudo tee -a /etc/audit/rules.d/audit.rules
sudo augenrules --load
sudo auditctl -l

# prove the kernel layer before touching Wazuh
hostname
sudo ausearch -k t1082_recon
```

### Wazuh audit ingestion and correlation rule 100100

Add the `<localfile>` audit block (fragment: `../configs/wazuh/agent-audit-log.xml`) to the agent's `ossec.conf`, and rule 100100 (fragment: `../configs/wazuh/local_rules.xml`) to `/var/ossec/etc/rules/local_rules.xml` on the manager, then reload both:

```bash
sudo systemctl restart wazuh-agent     # on the agent
sudo systemctl restart wazuh-manager   # on the manager
sudo /var/ossec/bin/wazuh-logtest      # optional: trace decoding and rule selection
```

### Run the tests and confirm

```powershell
Invoke-AtomicTest T1059.004 -TestNumbers 1
Invoke-AtomicTest T1082 -TestNumbers 3
Invoke-AtomicTest T1082 -TestNumbers 8
```

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log
```

## 🚑 When an Atomic test produces no alert

1. Confirm the Atomic test exited successfully.
2. Record the exact start and end time.
3. Verify the endpoint clock, timezone, and NTP state.
4. Identify the test's actual behavior and cleanup steps.
5. Check the telemetry source before checking Wazuh rules.
6. Review the decoded event, not only dashboard labels.
7. Use the Wazuh view specific to the telemetry source.
8. Re-run the exact test after one controlled change.

## 🧰 Tools that proved most useful

| Tool | Purpose |
| --- | --- |
| `ossec.log` | Agent enrollment, FIM registration, and logcollector status |
| `journalctl` | Service restart-loop and certificate failure diagnosis |
| `ausearch -k <key>` | Verify kernel audit events independently of Wazuh |
| `auditctl -l` | Confirm the active audit rules |
| `wazuh-logtest` | Trace pre-decoding, decoding, and rule matching |
| FIM: Recent events | Review endpoint syscheck activity directly |
| `lvs` | Separate thin-pool allocation warnings from physical usage |
| `tmux` | Preserve long model downloads across SSH disconnects |

## 🎓 Detection-design lessons

- A successful attack simulation does not guarantee a corresponding telemetry source exists.
- FIM scope and scan timing both matter for short-lived files.
- One behavioral control can cover multiple tests that share the same artifact pattern.
- Generic dashboard categories can point to unrelated scheduled activity.
- Level-0 parent rules can prove that events are decoded even when no alert is generated.
- Kernel-level auditing can cover commands that leave no filesystem artifact.
- Common administrative commands need careful scoping to avoid noisy detections.
- Repeated validation is stronger than a single screenshot or coincidental timestamp.

---

<a id="limitations"></a>

<div align="center">

# ⚠️ Limitations & Open Work

</div>

---

## 🧩 Detection coverage

This phase tested three selected Atomic Red Team procedures. It did not test the complete Wazuh ruleset or every procedure associated with the mapped ATT&CK techniques.

Planned expansion includes T1003 credential-dumping simulations and additional T1059.004 tests.

## 🔊 Noise tuning

Rootcheck rule 510 produced an unrelated `/usr/bin/diff` false positive during the test window.

The event was investigated, but no suppression or tuning change was applied. Any future tuning should preserve evidence and document why the event is safe to reduce.

## 🌐 Network controls

The original plan called for explicit pfSense rules between the lab VLAN and management access.

That work was not completed in this session. The final documentation should only claim the rules after they are configured and tested.

## 💻 Endpoint diversity

Only one Ubuntu victim was enrolled. Windows and macOS enrollment remain open if broader endpoint coverage is needed.

## 💾 Storage management

Thin-pool overcommit is not currently an outage, but Wazuh indexes, VM snapshots, and local models can turn virtual allocation into physical usage quickly.

Capacity and snapshot retention should be reviewed as the lab grows.

## 🤖 Local AI integration

Ollama and Open WebUI are operational, but automated Wazuh-to-LLM analysis is not yet implemented.

The 13B security model is too slow for reliable live use on this CPU-only host. GPU testing is a separate project and does not block the detection lab.

## 🔐 Remote access

WireGuard on pfSense remains a non-blocking future improvement for remote lab access.

The Proxmox UI, dashboard, and VMs should not be exposed directly to the Internet.

## 📝 Documentation follow-up

The original plan placed the write-up in the broader Home-Lab repository. The implementation now has this dedicated detection-engineering repository.

Any shared Home-Lab summary should link here and avoid duplicating stale implementation claims.

---

<a id="evidence"></a>

<div align="center">

# 🖼️ Evidence Index

*The repository includes screenshots for the major build and validation stages.*

</div>

---

## 🏗️ Infrastructure evidence

- [Proxmox host starting point](images/phase1-infrastructure/01-proxmox-host-starting-point.png)
- [Wazuh VM in the Ubuntu installer](images/phase1-infrastructure/02-wazuh-aio-vm-ubuntu-installer-boot.png)
- [Wazuh dashboard after login](images/phase1-infrastructure/04-wazuh-dashboard-overview-after-login.png)
- [Agent enrollment before the fix](images/phase1-infrastructure/05-wazuh-agents-summary-no-active-agent.png)
- [Active agent after the certificate fix](images/phase1-infrastructure/06-wazuh-agent-active-after-cert-fix.png)
- [Kali VM installation](images/phase1-infrastructure/07-kali-vm-build.png)
- [Ollama vCPU limit](images/phase1-infrastructure/08-ollama-vm-vcpu-limit-error.png)
- [Open WebUI with WhiteRabbitNeo selected](images/phase1-infrastructure/09-open-webui-whiterabbit-neo-13b.png)
- [Open WebUI tools-support error](images/phase1-infrastructure/12-open-webui-model-selector-supplementary.png)

## 🛡️ Detection-engineering evidence

- [Twenty-four-hour Threat Hunting view with 1,100 events](images/phase1-infrastructure/10-phase1-closeout-bottleneck-numbers.png)
- [Narrow Threat Hunting window with no results](images/phase1-infrastructure/11-phase1-closeout-supplementary.png)
- [T1059 Linux drilldown](images/phase2-detection-engineering/01-t1059-no-linux-atomics-drilldown-to-t1059004.png)
- [T1059.004-1 execution](images/phase2-detection-engineering/02-t1059004-1-bash-script-execution.png)
- [One-hour Wazuh baseline](images/phase2-detection-engineering/03-wazuh-overview-dashboard-baseline.png)
- [Exact test window and misleading category](images/phase2-detection-engineering/04-threat-hunting-wide-window-red-herring.png)
- [CIS benchmark false lead](images/phase2-detection-engineering/05-cis-benchmark-sca-false-lead-table.png)
- [`/tmp` search with no results](images/phase2-detection-engineering/06-absolute-time-picker-format-gotcha.png)
- [Rule 510 detail for `/usr/bin/diff`](images/phase2-detection-engineering/07-rootcheck-rule510-2-hits.png)
- [Default syscheck scope and frequency](images/phase2-detection-engineering/08-tmp-search-empty-fim-gap-confirmed.png)
- [Wazuh agent restart after the FIM change](images/phase2-detection-engineering/09-rootcheck-false-positive-usr-bin-diff.png)
- [T1082-8 and auditd setup workflow](images/phase2-detection-engineering/10-ossec-conf-realtime-tmp-directive-added.png)
- [Running agent with EDT timestamp](images/phase2-detection-engineering/11-victim01-timezone-ntp-fix.png)
- [Post-change Wazuh view with rule 550 visible](images/phase2-detection-engineering/12-fim-realtime-fix-validated-rule550.png)
- [Rule 100100 in `local_rules.xml`](images/phase2-detection-engineering/13-auditd-rule-t1082-recon-added.png)
- [Successful Wazuh manager restart](images/phase2-detection-engineering/14-local-rules-xml-custom-rule-100100.png)
- [Final rule 100100 validation](images/phase2-detection-engineering/15-custom-rule-100100-validated-final-win.png)

## 🚩 Next milestones

1. Keep the reusable configuration artifacts synchronized with future lab changes.
2. Test a technique that does not reuse the current `/tmp` behavior.
3. Document false-positive tuning decisions.
4. Review and validate pfSense access rules.
5. Enroll another operating system if mixed-endpoint coverage is desired.
6. Add Wazuh API integration only after the detection pipeline remains stable.

---

<div align="center">

*Chronological engineering log · part of the [Wazuh Detection Engineering Lab](../README.md) · linked from [Home-Lab](https://github.com/BrandonRoos/Home-Lab)*

</div>
