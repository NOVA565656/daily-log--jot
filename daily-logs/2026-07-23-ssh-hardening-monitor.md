```markdown
# 2026-07-23 - SSH Hardening & Brute-Force Monitoring

**The Vibe (Raw Notes):**
- Changed SSH port to 2222 and almost forgot to allow it in UFW. Panic moment.
- Realized I was looking at the wrong fail2ban jail. Spent 20 minutes wondering why it wasn't banning anyone.
- Finally saw the ban pop up in `iptables -L` at 4:15 PM. Huge relief.

---

**Project Goal:** Harden a Linux SSH server (disable root, change port, enforce key-auth) and deploy fail2ban to automatically block brute-force attackers, then monitor the logs to see it working.

**Environment:** Ubuntu 22.04 Linux VM (or VPS) with SSH exposed to the internet (or local LAN).

---

**Timeline:**
- **02:30 PM:** Started editing `/etc/ssh/sshd_config`.
- **03:15 PM:** Nearly locked myself out. Forgot to open the new port in the firewall.
- **04:00 PM:** Fail2ban wasn't catching anything. Realized I edited the wrong `.local` file.
- **04:15 PM:** Saw the first IP banned. Victory.

---

## Phase 1: SSH Hardening (The IT Admin Hat)
- Edited `/etc/ssh/sshd_config` and made these changes:
  - `Port 2222` (changed from default 22)
  - `PermitRootLogin no`
  - `PasswordAuthentication no` (forced key-based auth)
  - `MaxAuthTries 3`
- **The Near-Disaster:** Restarted the SSH service (`systemctl restart sshd`) before opening port 2222 in UFW. Lost connection immediately. Had to use the VPS console (out-of-band) to run `ufw allow 2222/tcp` and reconnect. Always keep a second terminal session open when doing this!

---

## Phase 2: Log Analysis (The Security Hat)
- Started monitoring `/var/log/auth.log` to see the brute-force attempts:
  ```bash
  tail -f /var/log/auth.log | grep "Failed password"
```

· Observed multiple IPs hammering the old port 22 before I changed it. Confirmed the attack surface reduction worked—attacks dropped to zero on port 2222.

---

Phase 3: Fail2ban Deployment

· Installed fail2ban: sudo apt install fail2ban -y
· The Mistake: Edited /etc/fail2ban/jail.local but forgot to enable the [sshd] section by setting enabled = true. Spent 3 minutes running fail2ban-client status sshd and seeing "0 total banned" before I realized my config was ignored.
· The Fix: Set enabled = true and restarted the service.
· Final working config snippet:
  ```ini
  [sshd]
  enabled = true
  port    = 2222
  maxretry = 3
  bantime = 3600
  ```

---

Phase 4: Verification & Remediation

· Checked the banned IPs:
  ```bash
  sudo fail2ban-client status sshd
  ```
· Manually tested a ban by failing login 3 times from my other machine. Saw the IP appear in sudo iptables -L -n.
· This proves the automated remediation (blocking) is working without human intervention.

---

Final Outcome

· SSH service moved to non-standard port (security by obscurity + actual config hardening).
· Root login disabled completely.
· Automated brute-force blocking active with 1-hour bans.
· Log analysis skills sharpened for identifying malicious traffic patterns.

---

Artifacts (Useful commands for later)

```bash
# Watch for SSH attacks
sudo tail -f /var/log/auth.log | grep "Failed password"

# Check fail2ban status
sudo fail2ban-client status sshd

# Unban an IP if you lock yourself out
sudo fail2ban-client set sshd unbanip <YOUR_IP>

# Check firewall rules for the ban
sudo iptables -L -n | grep f2b
```

Key Takeaway

Always configure your firewall before restarting SSH, and always keep a backup console session open. Double-check enabled = true in fail2ban—the config is useless if that line isn't set.

```