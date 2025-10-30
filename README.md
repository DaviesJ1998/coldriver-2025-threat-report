# COLDRIVER 2025 — Threat Report  
**Short:** Analysis of the COLDRIVER multi-stage ClickFix malware chain (COLDCOPY → NOROBOT → YESROBOT / MAYBEROBOT)  
**Source:** The Hacker News — `https://thehackernews.com/2025/10/google-identifies-three-new-russian.html`  
**Report date:** 27 October 2025

---

## Project overview

This repository contains a concise, portfolio-quality write-up of an observed Russian state-actor espionage campaign tracked as **COLDRIVER** (Oct 2025). The attack uses a social-engineered HTML lure to convince victims to run a PowerShell command via the Windows Run dialog, which drops a DLL (`NOROBOT`) that stages either a Python backdoor (`YESROBOT`) or a PowerShell implant (`MAYBEROBOT`). The campaign targets high-value victims (NGOs, journalists, dissidents, policymakers) and emphasizes living-off-the-land techniques to evade detection.

This repo is meant for documentation and analyst portfolio use — it contains the report summary, enrichment per malware family, key behaviors, and suggested mitigations. No active indicators (IPs/hashes) were published in the source.

---

## Quick facts

- **Attack chain:** COLDCOPY (HTML lure) → NOROBOT (DLL loader) → YESROBOT (Python backdoor, short-lived) / MAYBEROBOT (PowerShell implant, persistent)  
- **Techniques:** Social engineering (fake CAPTCHA), Windows Run dialog (Win+R), PowerShell, `rundll32.exe`, HTTPS C2, living-off-the-land  
- **Targets:** NGOs, journalists, dissidents, policymakers (espionage)  
- **Status (as of 27 Oct 2025):** Active and evolving. Source reports no public IPs/hashes.

---

## Extracted IOCs (2–3 per family — behavioral / TTP-focused)

> Note: original reporting stated **no IPs/hashes public**. The IOCs below are tactical/behavioral rather than ephemeral IOCs.

- **COLDCOPY (HTML lure)**
  - File: `verify.html` (fake CAPTCHA page)
  - Tactic: Instructions to press **Win+R** and paste a long PowerShell command
  - Behavioral IOC: Presence of an HTML file that contains encoded / long PowerShell payloads and obvious instructions to use Windows Run dialog

- **NOROBOT (DLL dropper)**
  - Execution: launched via `rundll32.exe`
  - Role: first-stage DLL that downloads a second-stage implant
  - Behavioral IOC: Execution of `rundll32.exe` with unusual DLL path from temp/download folders immediately after a PowerShell run

- **YESROBOT (Python backdoor)**
  - C2: HTTPS to hard-coded server
  - Requirement: Full Python 3.8 installation (no virtualenv)
  - Behavioral IOC: New Python runtime installations / recently modified Python scripts communicating to external HTTPS endpoints

- **MAYBEROBOT (PowerShell implant)**
  - In-memory PowerShell implant; executes arbitrary PS code and system commands
  - Capabilities: download-and-execute from URL, run `cmd.exe`, exfiltrate files
  - Behavioral IOC: PowerShell processes with base64/obfuscated payloads launched post-`rundll32` execution; in-memory-only modules that avoid disk artifacts

---

## Summary / attack flow

1. **COLDCOPY HTML lure** — fake CAPTCHA page instructs the victim to press **Win+R** and paste a PowerShell command.  
2. The PowerShell runs and **drops NOROBOT** (DLL).  
3. **NOROBOT** executes via `rundll32.exe`, acts as downloader/launcher.  
4. NOROBOT fetches either **YESROBOT** (Python backdoor; short-lived) or **MAYBEROBOT** (PowerShell implant; persistent).  
5. Attackers maintain covert, long-term access to collect intelligence.

---

## Per-family technical notes (enriched)

### COLDCOPY
- **What:** HTML file masquerading as CAPTCHA to trick users into running a PowerShell command via Run dialog (Win+R).  
- **Spread:** Phishing emails / malicious links delivering `verify.html`.  
- **Impact:** Entry vector for the entire chain — enables NOROBOT deployment.  
- **Detection tips:** Look for `*.html` files in user download folders that contain long `powershell.exe` command strings or explicit Win+R instructions. Email gateway/URL scanning for pages with embedded obfuscated PowerShell or explicit "paste into Run" instructions.

### NOROBOT
- **What:** DLL dropper executed via `rundll32.exe`. Downloads/executes next-stage implants.  
- **Spread:** Delivered by COLDCOPY via PowerShell.  
- **Impact:** Initial code execution and persistence; blends in by using built-in Windows launcher.  
- **Detection tips:** Monitor suspicious `rundll32.exe` invocations, especially those sourcing DLLs from user temp/download directories; enable Sysmon module load and process create logging to catch parent/child trees.

### YESROBOT
- **What:** Python-based remote backdoor using HTTPS to a hard-coded C2. Minimal feature set (download, execute, exfil).  
- **Spread:** Deployed by NOROBOT in early/simpler versions; noisy due to full Python install.  
- **Impact:** Direct remote access for data theft — abandoned quickly due to detection.  
- **Detection tips:** Alert on new or unexpected Python installations and long-running Python processes making outbound HTTPS connections to unknown endpoints.

### MAYBEROBOT
- **What:** PowerShell implant (in-memory) with flexible C2 and run-time capabilities. Replaced YESROBOT as primary.  
- **Spread:** Deployed by NOROBOT. Active Jun–Sep 2025 (per source).  
- **Impact:** Persistent covert access for reconnaissance and data theft.  
- **Detection tips:** Monitor for obfuscated/encoded PowerShell, parent/child processes triggered after `rundll32`/PowerShell run, and non-standard PowerShell endpoints. Use Constrained Language Mode, script block logging, AMSI, and memory analysis to detect in-memory implants.

---

## Mitigations & recommendations

- **User awareness:** Train users to never paste commands into Run/PowerShell upon instruction from web pages or email. Treat requests to paste/run scripts as high risk.  
- **Email/web filtering:** Block or sandbox suspicious HTML attachments and pages that contain encoded PowerShell instructions.  
- **Endpoint controls:** Enable script block logging, AMSI, PowerShell Constrained Language Mode, and block/monitor execution of `rundll32.exe` from non-standard locations.  
- **Egress controls:** Monitor and restrict outbound HTTPS destinations; profile expected domains and alert on anomalies.  
- **Process monitoring:** Use Sysmon/EDR to detect `rundll32.exe` launching DLLs from user folders and unusual PowerShell child processes.  
- **Threat hunting:** Hunt for new Python installs on atypical machines, long-lived Python processes, and suspicious in-memory PowerShell activity.

---

## Repository structure (suggested)

