# 2026-07-24 - Project PartyVPN (WireGuard for Friends)

**The Vibe (Raw Notes):**
- Spun up a $6 VPS just so we can all play old LAN games without using Hamachi.
- Spent 30 minutes pulling my hair out because clients connected but had no internet. Forgot to enable IPv4 forwarding. Classic.
- Once it worked, we all hopped on and played a round of CS 1.6. Totally worth it.

---

**Project Goal:** Deploy a WireGuard VPN server on a tiny VPS so that up to 5 friends can connect, see each other, and route their traffic through the VPS (or just talk to each other securely).

**Environment:** Ubuntu 22.04 VPS (1 CPU, 1GB RAM) + WireGuard.

---

**Timeline:**
- **08:00 PM:** Bought the VPS and installed WireGuard.
- **08:45 PM:** Clients connected but had zero connectivity. Realized I never turned on NAT.
- **09:15 PM:** Fixed the firewall and got ping responses.
- **09:30 PM:** Sent the config files to the squad. We were online.

---

## Phase 1: The Setup (The "IT" Part)
- Installed WireGuard: `sudo apt install wireguard -y`
- Generated the server private/public keys:
  ```bash
  cd /etc/wireguard/
  umask 077
  wg genkey | tee server_private.key | wg pubkey > server_public.key
  ```

- Created the server config (wg0.conf) with a /24 subnet (10.0.0.1/24) so we have plenty of IPs for the crew.

---

## Phase 2: The Critical Blunder (IP Forwarding)

- The Mistake: I configured the [Peer] sections perfectly, generated client configs, and sent them out. Everyone connected (wg show showed handshakes!), but no one could ping Google or each other.
- The Debugging: I spent 15 minutes checking the firewall, thinking UDP port 51820 was blocked. Turns out, Linux doesn't route traffic between interfaces by default.
- The Fix: I uncommented net.ipv4.ip_forward=1 in /etc/sysctl.conf and applied it with sysctl -p. Instant magic—the packets started flowing.

---

## Phase 3: The Firewall Dance (NAT)

- To let my friends use the VPS as an exit node (so they can hide their IP or bypass café Wi-Fi blocks), I had to set up a MASQUERADE rule in iptables:
  ```bash
  sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  ```
- Another hiccup: Forgot to persist the iptables rules. After a reboot, it broke again. Installed iptables-persistent to save the rules permanently.

---

## Phase 4: Client Config (The Fun Part)

- I used a simple script to generate each friend's config.
- The Second Mistake: I initially set AllowedIPs = 0.0.0.0/0 for my friends, which sent all their internet traffic through my tiny VPS. The bandwidth tanked.
- The Fix: I changed it to AllowedIPs = 10.0.0.0/24 instead. Now they only route traffic destined for the VPN subnet through the VPS, keeping their Netflix streaming on their own home internet. (If they want full tunneling later, I can flip the switch).
- Shared the configs via QR codes (qrencode -t ansiutf8 < client.conf) so they just scanned it with their WireGuard apps. Super slick.

---

## Final Outcome

- A fully functional, private gaming/browsing network.
- 3 friends connected simultaneously, pinging each other at <5ms latency on the VPN tunnel.
- We played a nostalgic 4-player game without needing port forwarding on our home routers.

---

## Artifacts (For the next time I spin one up)

```bash
# Check if IP forwarding is on (should return 1)
cat /proc/sys/net/ipv4/ip_forward

# Check WireGuard peer status
sudo wg show

# Restart the interface if I change configs
sudo wg-quick down wg0 && sudo wg-quick up wg0

# See who is connected (handshake timestamps)
sudo wg show | grep -A 10 "peers"
```

## Key Takeaway

ALWAYS turn on IP forwarding first. WireGuard connects in 2 seconds, but routing is what makes it work. Also, think carefully about whether your friends actually need their entire internet routed through you—save your bandwidth, use split tunneling (AllowedIPs = 10.0.0.0/24) unless they ask for the full privacy suite.
