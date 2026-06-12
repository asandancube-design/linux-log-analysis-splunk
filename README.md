# Linux Log File Analysis, Automation & SIEM Visualization

**Project by:** Philisiwe Ncube  
**Date:** June 2026  
**Track:** SOC Analyst | Blue Team  

---

## Overview

This project involved extracting, analysing, and visualizing real system logs from a macOS endpoint using command-line tools and Splunk Enterprise. No simulated environments. Real logs, real findings.

The goal was to replicate what a SOC analyst does during log triage — identify anomalies, find the source of errors, automate the process, and visualize the results in a SIEM.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| macOS Terminal (zsh) | Log extraction and command-line analysis |
| `log`, `grep`, `awk`, `sort`, `uniq` | Log parsing and pattern detection |
| Bash scripting | Automation |
| Splunk Enterprise | SIEM ingestion, search, and visualization |

---

## Environment

- Device: MacBook Air (Apple Silicon, ARM64)
- OS: macOS Darwin Kernel 24.6.0
- No paid subscriptions or cloud labs used

---

## Step 1: Extract System Logs

Used the macOS `log` command to pull the last hour of system logs and save them to a file:

```bash
log show --last 1h --style syslog > ~/Desktop/system_logs.txt
```

**Result:** 1,582,170 log lines — roughly 266MB of real endpoint data.

---

## Step 2: Manual Log Triage

Ran targeted grep searches to count event severity levels:

```bash
grep -i "error" ~/Desktop/system_logs.txt | wc -l
grep -i "warning" ~/Desktop/system_logs.txt | wc -l
grep -i "fail" ~/Desktop/system_logs.txt | wc -l
```

### Findings

| Severity | Count |
|----------|-------|
| Errors | 80,099 |
| Failures | 45,703 |
| Warnings | 929 |

**Key observation:** The low warning count versus high failure count indicated that many failures were occurring silently — without prior warning events. This is a pattern that would be missed without log visibility.

---

## Step 3: Identify Top Error Sources

Used `awk`, `sort`, and `uniq` to find which processes were generating the most errors:

```bash
grep -i "error" ~/Desktop/system_logs.txt | awk '{print $4}' | sort | uniq -c | sort -rn | head -10
```

### Top Error Sources

| Process | Error Count | Description |
|---------|------------|-------------|
| searchpartyuseragent | 8,410 | Apple Find My network service |
| runningboardd | 4,539 | App lifecycle manager |
| imagent | 4,356 + 2,514 | iMessage agent |
| trustd | 2,737 | Certificate and trust validation |
| launchd | 2,363 | Core system process manager |
| cloudd | 1,765 | iCloud sync daemon |

<img width="1280" height="494" alt="WhatsApp Image 2026-06-12 at 14 03 38" src="https://github.com/user-attachments/assets/c902b36b-6fa5-471f-9772-221dd1c1ee13" />

**Root cause identified:** The majority of errors were network-dependent services failing after a VPN disconnection event. The OpenVPN logs showed `write UDPv4: Network is unreachable` at 11:09 AM — which directly preceded the spike in errors across Find My, iMessage, and iCloud services.

---

## Step 4: Automation Script

Wrote a bash script to automate the full analysis so it can be re-run at any time:

```bash
#!/bin/bash
echo "=== LOG ANALYSIS REPORT ==="
echo "Total lines: $(wc -l < ~/Desktop/system_logs.txt)"
echo "Errors: $(grep -ic 'error' ~/Desktop/system_logs.txt)"
echo "Warnings: $(grep -ic 'warning' ~/Desktop/system_logs.txt)"
echo "Failures: $(grep -ic 'fail' ~/Desktop/system_logs.txt)"
echo "Top error sources:"
grep -i "error" ~/Desktop/system_logs.txt | awk '{print $4}' | sort | uniq -c | sort -rn | head -5
```

Made it executable and ran it:

```bash
chmod +x ~/Desktop/log_analysis.sh
~/Desktop/log_analysis.sh
```

### Script Output

```
=== LOG ANALYSIS REPORT ===
Total lines:  1582170
Errors: 80099
Warnings: 929
Failures: 45703
Top error sources:
8410 searchpartyuseragent[705]:
4539 runningboardd[385]:
4356 imagent[666]:
2737 trustd[584]:
2363 launchd[1]:
```

---

## Step 5: SIEM Ingestion with Splunk

Installed Splunk Enterprise locally and uploaded the log file.

- **Source type created:** mac_system_logs
- **Events indexed:** 1,238,099
- **Host:** Mac.lan

### SPL Query Used

```
source="system_logs.txt" (error OR fail OR warning) | timechart count by host
```

### Dashboard

Created a saved dashboard titled **"Mac System Log Analysis"** showing error, failure, and warning events over time.

**Key visual finding:** A clear spike in error volume between 11:09 AM and 11:12 AM — directly correlated with the VPN disconnection event identified in manual analysis.

---

## Key Takeaways

1. **Log volume means nothing without context.** 1.5 million lines is overwhelming — the job is knowing what to filter for.
2. **Silent failures are dangerous.** High failure counts with low warnings means issues are happening without alerting. In a real SOC environment this would warrant alert tuning.
3. **Root cause correlation matters.** Connecting the VPN drop to the cascade of network service failures is exactly the kind of analysis that separates reactive log reading from real threat investigation.
4. **Automation saves time.** A reusable script means this analysis can run on any future log file in seconds.
___

<img width="1470" height="839" alt="Screenshot 2026-06-12 at 13 28 04" src="https://github.com/user-attachments/assets/54e44243-7e4d-446f-abe8-3d1ca87b0a86" />

---

## Connect

- LinkedIn: www.linkedin.com/in/philisiwe-ncube-258263360
- Certifications: ISC2 Certified in Cybersecurity (CC)
- Currently studying: AZ-900 Azure Fundamentals

> *Self-taught. Durban-based. Building in public.*
