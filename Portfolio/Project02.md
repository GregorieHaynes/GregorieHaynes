
# Lab 2: Wazuh SIEM Tuning, Detection and Validation

## Overview

Lab 1 showed I could build a network. This lab covers the work a SOC analyst does day to day: reading alerts, deciding what is noise and what is not, and being able to explain why.

I deployed a Wazuh SIEM and ran it against my own Windows 11 machine. I investigated alerts, tuned out the noisy ones without turning off the detections underneath, found and fixed two critical vulnerabilities, and ran controlled activity to confirm the tuning had not blinded the system.

I used AI tools while building this lab. The troubleshooting, the exception logic, and the decisions about what was safe to suppress are mine, and the reasoning for each is documented below.

![Wazuh dashboard overview](./images/01-dashboard-overview.png)
*Wazuh up and running.*

---

## Contents

1. [Environment](#environment)
2. [Vulnerabilities found and fixed](#vulnerabilities-found-and-fixed)
3. [Tuning alert noise](#tuning-alert-noise)
4. [Validating with attack activity](#validating-with-attack-activity)
5. [Custom detection rule](#custom-detection-rule)
6. [MITRE ATT&CK mapping](#mitre-attck-mapping)
7. [What I learned and what I would do next](#what-i-learned-and-what-i-would-do-next)

---

## Environment

| Component | Detail |
|---|---|
| Host | Windows 11 |
| WSL | 2.4.12.0 |
| Ubuntu | 24.04.2 LTS |
| Wazuh server | Wazuh 4.14.7, single node, Docker on Ubuntu (WSL2) |
| Monitored endpoint | Windows 11 agent |
| Telemetry | Windows event logs, Sysmon, PowerShell logging, vulnerability detection |

**Why single node:** This is a lab, and the priority was spending time on detection logic and tuning rather than cluster administration and the resource overhead of running multiple VMs.

---

## Vulnerabilities found and fixed

Wazuh's vulnerability detection flagged two critical findings on my own machine. For each one I documented what it is, how I found it, why it matters, how I fixed it, and how I confirmed the fix.

### Finding 1: Hugging Face Transformers, CVE-2026-5241 (CVSS 9.6)

**What it is:** A flaw in the LightGlue model-loading path. The `trust_remote_code` setting is meant to stop arbitrary code from running when a model is loaded. In this path, the config loader reads that same value from the untrusted model's own `config.json` and overrides it. A malicious model repository can therefore execute code during initialization even when `trust_remote_code=False` was set explicitly.

| Field | Detail |
|---|---|
| CVE | CVE-2026-5241 |
| CVSS | 9.6 |
| How found | Wazuh vulnerability detection |
| Impact | The `trust_remote_code` safeguard can be silently overridden by a malicious model's config file, allowing arbitrary code execution during model loading |
| Remediation | Removed the affected `transformers` package |
| Verification | Re-scanned; finding no longer reported |

![CVE-2026-5241 detected by Wazuh](./images/02a-cve-transformers-wazuh.png)
*Detected on my system by Wazuh vulnerability detection.*

![CVE-2026-5241 research](./images/02b-cve-transformers-research.png)
*Evidence of the vulnerability from independent research.*

### Finding 2: Visual Studio Code, CVE-2026-47281 (CVSS 9.6)

**What it is:** [One or two plain-language sentences on what this elevation-of-privilege flaw allows.]

| Field | Detail |
|---|---|
| CVE | CVE-2026-47281 |
| CVSS | 9.6 |
| How found | Wazuh vulnerability detection |
| Impact | [One sentence] |
| Remediation | Updated VS Code to a patched version ([version]) |
| Verification | Re-scanned; finding no longer reported |

![CVE-2026-47281 detected by Wazuh](./images/03a-cve-vscode-wazuh.png)
*Detected on my system by Wazuh vulnerability detection.*

![CVE-2026-47281 research](./images/03b-cve-vscode-research.png)
*Evidence of the vulnerability from independent research.*

![VS Code updated version](./images/03c-vscode-updated-version.png)
*VS Code after the version update that remediated the finding.*

**Why this matters:** Scanning only helps if findings turn into fixes and the fixes get verified. Both of these were closed and re-checked.

---

## Tuning alert noise

Wazuh was noisy out of the box, and the noise increased significantly once Sysmon was added for deeper process-level visibility. After the Sysmon install, alert volume reached roughly 5,600 in a 24-hour period. The extra telemetry exposed a lot of legitimate background activity to Wazuh's rule set, and it needed deliberate tuning to separate normal system behavior from activity worth alerting on.

For each noisy rule I asked the same questions:

1. What exactly fired, and why?
2. Is the behavior legitimate on this machine?
3. Can I suppress it narrowly enough that the underlying detection still works elsewhere?
4. Did it actually work?

To find where the volume was coming from, I used a Painless script to rank the highest-volume alerts by process. That is what pointed me to the first exception below.

![Highest-volume alerts by process](./images/04a-noise-weighted-scale.png)
*Output of the Painless script showing which process activity was driving alert volume.*

### Worked example: Chrome browser process (rule 67027)

**What fired:** Rule 67027 triggers on process creation events (Windows event 4688). Chrome launching its own processes was generating a large share of the volume.

**The exception:**

```xml
<!-- Suppress Chrome -> Chrome browser process with no --type flag (4688) (AI Generated) -->
<rule id="100101" level="0">
  <if_sid>67027</if_sid>
  <field name="win.eventdata.parentProcessName" type="pcre2">^C:\\\\Program Files\\\\Google\\\\Chrome\\\\Application\\\\chrome\.exe$</field>
  <field name="win.eventdata.newProcessName" type="pcre2">^C:\\\\Program Files\\\\Google\\\\Chrome\\\\Application\\\\chrome\.exe$</field>
  <field name="win.eventdata.commandLine" type="pcre2" negate="yes">--type=</field>
  <description>Suppress Chrome-to-Chrome browser process (no --type) telemetry (4688)</description>
</rule>
```

![Chrome exception rule](./images/04b-chrome-exception.png)

**Why I scoped it this way:** The exception only matches when the parent and the new process are both the real Chrome executable in its install path, and the command line does not contain `--type=`. Chrome's child processes (renderer, GPU, utility) all carry a `--type=` flag, so they do not match and still alert. Only the main browser process is suppressed. It is not a blanket suppression of rule 67027.

**Overall result:** I applied several more exceptions using the same investigate-then-scope approach, covering other 67027 variants and browser child-process noise. Alert volume dropped from roughly 5,600 to roughly 2,400 per 24 hours, and I am continuing to tune it down.

---

## Validating with attack activity

Suppressing alerts is only safe if real attacks still get through, and a SOC analyst also has to recognize activity that is not an attack. I tested one of each.

### True Positive: Registry Persistence

I used PowerShell to add a Run-key registry entry, simulating a common startup-persistence technique:

```powershell
New-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "WazuhPersistTest" -Value "C:\WazuhSoc-Lab\NOTAREALPATH.exe" -Force
```

- **Result:** Rule 91844 fired at level 12, "Possible addition of new item to Windows startup registry"
- **Maps to:** T1547.001 (Registry Run Keys / Startup Folder), T1059.001 (PowerShell), T1112 (Modify Registry), across the Persistence, Privilege Escalation, Execution, and Defense Evasion tactics

![Registry modification captured by script block logging](./images/05a-runkey-command.png)
*PowerShell Script Block Logging (event 4104) capturing the registry change.*

![Wazuh alert detail](./images/05b-runkey-alert.png)
*Rule 91844 firing at level 12 with its MITRE ATT&CK mapping.*

**Why this matters:** Persistence detection still fires at full severity after the surrounding noise was tuned down.

### False Positive: Authentication Failure

I generated benign bad-password logon failures to test log ingestion and confirm Wazuh distinguishes routine authentication noise from an actual attack pattern.

- **Result:** Rule 60122 fired at level 5, "Logon Failure - Unknown user or bad password"
- **Assessment:** Isolated failed logons from my own testing. Level 5 is the expected severity for a single failure, with no escalation. A real brute-force pattern would show repeated failures in a short window, which a higher-severity rule is meant to catch.

![Failed logon alerts](./images/06-failed-logons.png)

**Why this matters:** Telling benign activity apart from an attack is as much a part of the job as catching the attack.

---

## Custom detection rule

In progress. The goal is a custom rule that makes the system more sensitive to specific activity I care about, instead of relying only on default rule behavior.

---

## MITRE ATT&CK mapping

| Activity | Rule | Technique | Tactic |
|---|---|---|---|
| Run-key persistence | 91844 | T1547.001, T1059.001, T1112 | Persistence, Privilege Escalation, Execution, Defense Evasion |

---

## What I learned and what I would do next

Setting Wazuh up on Docker gave me a system I could run continuously against my own Windows 11 desktop, with the Windows 11 agent installed directly on it. That served three purposes: a lab environment to learn in, hands-on practice, and real security coverage for a machine where I only run Windows Defender. Knowing what's actually happening on my own system, rather than assuming it's fine, is exactly the mindset I want to bring into a security role.

The hands-on log analysis taught me more than I expected. I learned to read parent and child process relationships, trace command-line activity, and understand what a registry change actually looks like at the log level, partly by making one myself and watching Wazuh catch it.

The two CVEs I found were the clearest lesson in the value of visibility. I didn't know either vulnerability existed until Wazuh's scanning flagged them. I updated VS Code to resolve the first, and removed the affected Transformers package to resolve the second. Having that visibility is what let me fix problems I wouldn't have otherwise known to look for.

The most frustrating part of the lab was noise. Adding Sysmon gave me far more visibility into process activity, but it also multiplied the alert volume significantly. Working through that taught me how to investigate a noisy rule properly instead of just suppressing it: find what's actually driving the volume, confirm the behavior is legitimate, and scope the exception narrowly enough that it doesn't blind the system to something that actually matters. I used a Painless script to identify which processes were generating the most alerts, which is what led directly to the Chrome exception documented above. I've brought the noise down from roughly 5,600 to roughly 2,400 alerts a day, and I expect to keep tuning it further.

Right now I'm building a SOAR project, using AI-assisted coding, to automate and speed up how I create and manage exception rules for this lab. The next step beyond that is writing custom detection logic to make the system more sensitive to the specific activity I actually care about, rather than relying only on default rule behavior. That work is still in progress.

Overall, this lab is where I actually learned what the job looks like day to day: investigating, tuning, validating, and documenting why each decision was safe. I'm looking forward to applying that in a real SOC environment.

---

**Other labs:** [Lab 1: Network and Active Directory](../lab-01) | [Lab 3: Wazuh API app](../lab-03)
