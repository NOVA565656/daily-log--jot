```markdown
# 2026-07-22 - Reverse Shell Hunter (Detection + Remediation)

**The Vibe (Raw Notes):**
- Sysmon install errored out on first try because I forgot Admin rights. Annoying.
- Spent 15 minutes debugging why my detection script wasn't firing. Turns out I was looking at the wrong Sysmon property index.
- Finally got it working at 11:30 AM. Coffee needed.

---

**Project Goal:** Build a detection pipeline to catch outbound C2 beacons (reverse shells) on Windows using Sysmon + PowerShell, then auto-kill them.

**Environment:** Kali (Attacker) → Windows 10/Server (Target) with Sysmon installed.

---

**Timeline:**
- **09:15 AM:** Started setup.
- **10:45 AM:** Hit the Admin rights error on Sysmon install.
- **11:30 AM:** Detection script finally fired correctly.

---

## Phase 1: Environment Setup
- Installed Sysmon v14+ with SwiftOnSecurity default config.
- **The Mistake:** Ran the installer without Admin rights. Got `Access Denied`. Realized my mistake, right-clicked PowerShell → "Run as Administrator", and the install succeeded.
- Verified with: `Get-Service Sysmon` → Status: Running.
- Key logs leveraged: **Sysmon Event ID 3** (Network Connection) and **Event ID 1** (Process Creation).

---

## Phase 2: Attack Simulation (Generalized)
- Simulated a C2 beacon establishing an outbound TCP connection from the Windows host to a remote Kali listener.
- **The Mistake:** Started the outbound connection before starting the listener. Got `Connection refused`. Corrected the order (listener first, then beacon), and the handshake succeeded.
- Confirmed live connection via Windows Resource Monitor (TCP Established).

---

## Phase 3: Detection Logic (The Core)
- **Challenge:** Sysmon Event 3 stores Process ID at index 0, Process Name at index 4, and Destination IP at index 7.
- **Another Debugging Victory:** My initial script used `$_.Properties[5].Value` for the Destination IP, but it kept returning blank. I dumped `$Events[0].Properties` to inspect the object structure and discovered the IP was actually at index 7. Fixed the code, and it worked immediately.
- **Final detection query:**
  ```powershell
  $Events = Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; ID=3} -MaxEvents 50
  $ShellConnections = $Events | Where-Object { 
      $_.Properties[4].Value -match "cmd.exe|powershell.exe" -and 
      $_.Properties[7].Value -notin @("127.0.0.1", "::1", $env:COMPUTERNAME)
  }
  if ($ShellConnections) {
      Write-Host "🚨 REVERSE SHELL DETECTED!" -ForegroundColor Red
      $ShellConnections | Select-Object TimeCreated, 
          @{N='Process';E={$_.Properties[4].Value}}, 
          @{N='DestIP';E={$_.Properties[7].Value}}
  }
```

---

Phase 4: Remediation (Auto-Kill)

· Extended detection script to terminate the offending process immediately.
· Wrapped Stop-Process in a null-check to handle already-disconnected shells so it doesn't throw ugly red errors in the console.
· Remediation snippet:
  ```powershell
  $PIDToKill = $ShellConnections[0].Properties[0].Value
  if ($PIDToKill) {
      Stop-Process -Id $PIDToKill -Force -ErrorAction SilentlyContinue
      Write-Host "💀 Killed malicious process ID: $PIDToKill" -ForegroundColor Magenta
  }
  ```

---

Final Outcome

· Full detection-to-remediation cycle under 10 seconds.
· Script can be scheduled via Task Scheduler for persistent monitoring.
· This project demonstrates log analysis, threat hunting, and automated response—all without a commercial SIEM.

---

Artifacts (Command references)

```powershell
# Sysmon install (Admin required)
.\Sysmon64.exe -accepteula -i config.xml

# Detection filter (Event 3 indexes)
# Index 0 = PID, Index 4 = Image, Index 7 = Destination IP
```

Key Takeaway

Sysmon Event 3 indexes are 0-indexed; Destination IP lives at $_.Properties[7].Value, not 5. Double-check your property arrays when writing detection rules.

```
```