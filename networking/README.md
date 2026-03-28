# Networking Cheatsheet

Networking is the plumbing of infrastructure. Whether you're troubleshooting connectivity, securing traffic with TLS, automating HTTP integrations, or building VPN tunnels, the command-line tools covered in this cheatsheet are the daily workbench of every DevOps engineer and SRE. This guide covers interface configuration, DNS, HTTP, TLS, SSH, port scanning, packet capture, firewalls, Nginx, WireGuard, and performance testing.

---

## Interface & IP Configuration

### `ip` — Modern Network Configuration (replaces `ifconfig`)

The `ip` command is part of the `iproute2` suite and is the standard tool for Linux network configuration.

| Subcommand | Description |
|------------|-------------|
| `ip addr` | Show / manage IP addresses |
| `ip link` | Show / manage network interfaces |
| `ip route` | Show / manage routing table |
| `ip neigh` | Show / manage ARP/NDP cache |
| `ip netns` | Show / manage network namespaces |

```bash
# Show all interfaces and addresses
ip addr show
ip -br addr show                    # brief one-line-per-interface summary
ip addr show eth0                   # single interface

# Assign / remove an IP address
ip addr add 192.168.1.100/24 dev eth0
ip addr del 192.168.1.100/24 dev eth0

# Bring interface up or down
ip link set eth0 up
ip link set eth0 down

# Set MTU
ip link set eth0 mtu 9000

# Show ARP (neighbour) table
ip neigh show
ip neigh flush dev eth0             # flush stale entries
```

### `ifconfig` — Legacy Interface Tool
```bash
ifconfig                            # list all active interfaces
ifconfig eth0                       # specific interface details
ifconfig eth0 192.168.1.10 netmask 255.255.255.0 up
```

### `nmcli` — NetworkManager CLI (RHEL/Ubuntu with NM)
```bash
nmcli device status                         # overview of all devices
nmcli connection show                        # list saved connections
nmcli connection up "Wired connection 1"     # activate a connection
nmcli device wifi list                       # scan WiFi networks

# Create a static connection
nmcli connection add type ethernet ifname eth0 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.5/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "8.8.8.8 1.1.1.1" \
  con-name "static-eth0"
```

### `ss` — Socket Statistics

`ss` is the modern replacement for `netstat`.

| Flag | Description |
|------|-------------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-p` | Show process (PID/name) |
| `-n` | Numeric addresses (no DNS resolution) |
| `-s` | Summary statistics |
| `-4` / `-6` | IPv4 / IPv6 only |

```bash
ss -tulpn                           # all listening TCP/UDP with process info
ss -tnp state established           # established TCP connections
ss -s                               # socket summary
ss -tnp dst 10.0.0.5                # connections to a specific host
ss sport = :443                     # connections on source port 443
```

---

## Routing

### View and Manage Routes

```bash
ip route show                            # full routing table
ip route show table all                  # all routing tables (including policy routes)
ip route get 8.8.8.8                     # which route would be used to reach an IP

# Add / delete routes
ip route add 10.10.0.0/16 via 10.0.0.1 dev eth0
ip route del 10.10.0.0/16

# Default gateway
ip route add default via 192.168.1.1
ip route replace default via 192.168.1.1   # update without delete

# Persistent routing (write to /etc/netplan/*.yaml or /etc/network/interfaces)
```

### `route` — Legacy Tool
```bash
route -n                             # show routing table (numeric)
route add -net 10.10.0.0 netmask 255.255.0.0 gw 10.0.0.1
```

### `traceroute` / `tracepath` / `mtr`

```bash
traceroute -n 8.8.8.8               # hop-by-hop path, no DNS
traceroute -T -p 443 example.com    # TCP trace on port 443 (bypass ICMP filters)
tracepath 8.8.8.8                   # no root required, detects MTU
mtr --report --report-cycles 20 8.8.8.8   # combined ping+traceroute report
mtr -n --tcp --port 443 example.com # TCP MTR to port 443
```

---

## DNS

### `dig` — DNS Interrogation Tool

`dig` is the most powerful DNS debugging tool. Every query returns the answer, authority, and additional sections.

| Flag | Description |
|------|-------------|
| `@server` | Use specific DNS server |
| `-t TYPE` | Query type: A, AAAA, MX, TXT, CNAME, NS, SOA, PTR, SRV |
| `+short` | Compact output — only the answer |
| `+noall +answer` | Show only the answer section |
| `+trace` | Full recursive trace from root servers |
| `-x` | Reverse lookup (PTR) |
| `+tcp` | Force TCP instead of UDP |

```bash
dig example.com                             # default A record query
dig example.com A +short                    # just the IP
dig example.com MX                          # mail exchange records
dig example.com TXT                         # SPF, DKIM, verification records
dig @8.8.8.8 example.com A                  # query Google's DNS directly
dig +trace example.com                      # trace delegation from root
dig -x 93.184.216.34                        # reverse PTR lookup
dig example.com ANY +noall +answer          # all record types
dig _https._tcp.example.com SRV             # SRV record for HTTPS
```

### `nslookup` — Interactive DNS Lookup
```bash
nslookup example.com                        # forward lookup
nslookup 93.184.216.34                      # reverse lookup
nslookup -type=MX example.com              # MX records
nslookup example.com 1.1.1.1               # use Cloudflare DNS
```

### `host` — Simple DNS Lookup
```bash
host example.com                           # A + AAAA + MX
host -t TXT example.com                   # TXT records
host 93.184.216.34                         # reverse lookup
host -v example.com                        # verbose
```

### Local DNS Debugging
```bash
cat /etc/resolv.conf                       # configured DNS resolvers
cat /etc/hosts                             # static hostname overrides
resolvectl status                          # systemd-resolved status
resolvectl query example.com              # query via systemd-resolved
systemd-resolve --statistics              # cache stats
```

---

## HTTP with curl

`curl` is the Swiss Army knife for HTTP/HTTPS/FTP and dozens of other protocols.

| Flag | Description |
|------|-------------|
| `-s` | Silent (no progress meter) |
| `-S` | Show errors even with `-s` |
| `-o FILE` | Write output to file |
| `-O` | Write output to file named from URL |
| `-L` | Follow redirects |
| `-I` | Fetch headers only (HEAD request) |
| `-i` | Include response headers in output |
| `-v` | Verbose (show request + response headers) |
| `-X METHOD` | HTTP method: GET, POST, PUT, DELETE, PATCH |
| `-H "Header: val"` | Add request header |
| `-d DATA` | Request body (implies POST) |
| `-u user:pass` | Basic authentication |
| `--data-urlencode` | URL-encode the data |
| `-k` | Skip TLS certificate verification (insecure!) |
| `--cacert FILE` | Custom CA bundle |
| `--cert FILE` | Client certificate |
| `-w FORMAT` | Custom output format (useful for timing) |
| `--retry N` | Retry N times on transient error |
| `--max-time N` | Total timeout in seconds |
| `--connect-timeout N` | Connection timeout in seconds |

```bash
# Basic GET
curl -sS https://api.example.com/health

# POST JSON
curl -sS -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Alice","email":"alice@example.com"}'

# Download a file with progress bar
curl -L -o kubectl https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl

# Follow redirects and show final URL
curl -sS -L -o /dev/null -w "%{url_effective}\n" http://example.com

# Measure response times
curl -o /dev/null -s -w \
  "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  https://example.com

# Upload a file
curl -F "file=@/path/to/archive.zip" https://upload.example.com/api/files

# Test an endpoint with retry logic
curl --retry 3 --retry-delay 2 --retry-connrefused \
  -sS https://api.example.com/ready

# Use a specific DNS server for a request (without /etc/hosts)
curl --resolve "example.com:443:93.184.216.34" https://example.com
```

---

## TLS/SSL with openssl

### Certificate Inspection
```bash
# Inspect a remote certificate
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>/dev/null \
  | openssl x509 -noout -text

# Quick certificate info (subject, issuer, dates)
openssl s_client -connect example.com:443 < /dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# Check certificate expiry
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -enddate

# Inspect a local certificate file
openssl x509 -in cert.pem -noout -text
openssl x509 -in cert.pem -noout -subject -issuer -dates

# Verify a certificate chain
openssl verify -CAfile ca-bundle.crt cert.pem
```

### Key and Certificate Generation
```bash
# Generate a 4096-bit RSA private key
openssl genrsa -out private.key 4096

# Generate a CSR (Certificate Signing Request)
openssl req -new -key private.key -out request.csr \
  -subj "/C=US/ST=California/O=My Company/CN=example.com"

# Self-signed certificate (valid 365 days)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost"

# Self-signed cert with SANs
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com,IP:10.0.0.1"

# Extract public key from certificate
openssl x509 -pubkey -noout -in cert.pem > public.key

# Test TLS handshake with specific protocol/cipher
openssl s_client -connect example.com:443 -tls1_3
openssl s_client -connect example.com:443 -cipher ECDHE-RSA-AES256-GCM-SHA384
```

### Certificate Conversion
```bash
# PEM to DER
openssl x509 -in cert.pem -outform DER -out cert.der

# DER to PEM
openssl x509 -in cert.der -inform DER -out cert.pem

# PFX/PKCS12 to PEM (extract cert + key)
openssl pkcs12 -in bundle.pfx -nocerts -nodes -out key.pem
openssl pkcs12 -in bundle.pfx -clcerts -nokeys -out cert.pem

# Create PFX from cert and key
openssl pkcs12 -export -out bundle.pfx -inkey key.pem -in cert.pem
```

---

## SSH

### Basic Usage
```bash
ssh user@host                         # connect
ssh -p 2222 user@host                 # custom port
ssh -i ~/.ssh/id_ed25519 user@host    # specify key
ssh -v user@host                      # verbose (debug)
ssh -q user@host                      # quiet
```

### Key Management
```bash
ssh-keygen -t ed25519 -C "alice@work" -f ~/.ssh/id_ed25519
ssh-keygen -t rsa -b 4096 -C "alice@work"

ssh-copy-id user@host                            # install public key on remote
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

ssh-keygen -lf ~/.ssh/id_ed25519.pub             # fingerprint
ssh-keygen -R host                               # remove host from known_hosts
```

### SSH Tunnels

```bash
# Local port forward: access remote DB through SSH
# Forwards localhost:5432 → remote-db:5432 via jump.example.com
ssh -L 5432:remote-db:5432 user@jump.example.com -N

# Remote port forward: expose local service to remote server
# Connects to remote-server and tunnels its port 8080 → localhost:3000
ssh -R 8080:localhost:3000 user@remote-server -N

# Dynamic SOCKS5 proxy – use SSH as a VPN-lite
ssh -D 1080 user@gateway.example.com -N
# Then: curl --socks5-hostname localhost:1080 http://internal-service/

# Persistent tunnel with autossh
autossh -M 0 -N -L 5432:db:5432 user@bastion \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3
```

### Jump Hosts (ProxyJump)
```bash
# Single jump host
ssh -J user@bastion user@internal-server

# Multi-hop
ssh -J user@bastion1,user@bastion2 user@target

# In ~/.ssh/config (preferred)
Host target
  HostName 10.0.1.50
  User ec2-user
  ProxyJump bastion
  IdentityFile ~/.ssh/id_ed25519

Host bastion
  HostName bastion.example.com
  User ec2-user
  IdentityFile ~/.ssh/id_ed25519
```

### SSH Config File `~/.ssh/config`
```
# Global defaults
Host *
  ServerAliveInterval 60
  ServerAliveCountMax 3
  AddKeysToAgent yes
  IdentityFile ~/.ssh/id_ed25519

# Work servers
Host webserver
  HostName 10.0.1.10
  User ubuntu
  Port 22

# GH shortcut
Host github.com
  User git
  IdentityFile ~/.ssh/id_github
  IdentitiesOnly yes
```

### `scp` / `rsync` over SSH
```bash
scp -i ~/.ssh/id_ed25519 localfile.txt user@host:/remote/path/
scp -r user@host:/remote/dir/ ./local-copy/

rsync -avz -e "ssh -i ~/.ssh/id_ed25519" ./dist/ user@host:/var/www/html/
rsync -avz --delete ./dist/ user@host:/var/www/html/   # mirror (delete extras on remote)
```

---

## Port Scanning

### `nmap` — Network Mapper

| Flag | Description |
|------|-------------|
| `-sS` | SYN scan (stealth, default as root) |
| `-sT` | TCP connect scan |
| `-sU` | UDP scan |
| `-sV` | Version detection |
| `-O` | OS detection |
| `-A` | Aggressive (OS+version+scripts+traceroute) |
| `-p PORTS` | Specific ports: `22,80,443` or `1-1000` or `-` for all |
| `-F` | Fast scan (top 100 ports) |
| `--top-ports N` | Scan top N most common ports |
| `-T0–T5` | Timing template (T3 = default, T5 = insane) |
| `-oN/-oX/-oG` | Output: normal / XML / grepable |
| `--script NAME` | Run NSE script |
| `-Pn` | Skip host discovery (assume host is up) |

```bash
nmap -sS -sV -O 192.168.1.1                         # SYN scan with version + OS
nmap -p 22,80,443,8080 192.168.1.0/24               # specific ports on subnet
nmap -p- --min-rate 5000 10.0.0.5                   # all 65535 ports fast
nmap -sU -p 53,67,123 10.0.0.1                      # UDP scan
nmap --script=http-headers,http-title 10.0.0.5 -p 80,443
nmap -Pn -sV --top-ports 1000 scanme.nmap.org       # skip ping, top 1000 ports
nmap -A -T4 192.168.1.0/24 -oX scan-results.xml     # aggressive, save to XML
```

### `nc` (netcat) — Swiss Army Knife for TCP/UDP

```bash
# Test if a port is open
nc -zv 10.0.0.5 443                          # TCP connect test
nc -zvu 10.0.0.5 53                          # UDP test
nc -zvw3 10.0.0.5 22                         # with 3s timeout

# Simple TCP listener (receiver)
nc -l -p 9000                                # listen on port 9000
nc -l 9000 > received-file.bin               # receive a file

# Connect and send data
echo "Hello" | nc 10.0.0.5 9000
nc 10.0.0.5 9000 < file-to-send.bin          # send a file

# Port scanning (lightweight)
nc -zv 10.0.0.5 20-25 80 443 2>&1 | grep succeeded
```

### `lsof` — List Open Files (including network sockets)

```bash
lsof -i                             # all network connections
lsof -i :8080                       # what's using port 8080
lsof -i TCP:80                      # TCP connections on port 80
lsof -i @10.0.0.5                   # connections to a specific host
lsof -p 1234 -i                     # network files opened by PID 1234
lsof -i -n -P                       # numeric hosts and ports (fast)
```

---

## Packet Capture with tcpdump

`tcpdump` captures raw network packets. Run as root or with `CAP_NET_RAW`.

| Flag | Description |
|------|-------------|
| `-i IFACE` | Interface to capture on (`any` for all) |
| `-n` | Don't resolve hostnames |
| `-nn` | Don't resolve hostnames or port names |
| `-v / -vv / -vvv` | Verbosity levels |
| `-c N` | Capture N packets then exit |
| `-w FILE` | Write to pcap file |
| `-r FILE` | Read from pcap file |
| `-A` | Print payload as ASCII |
| `-X` | Print payload as hex + ASCII |
| `-s SNAPLEN` | Capture only first N bytes per packet (0 = full) |
| `-e` | Print link-layer headers (MAC addresses) |
| `port N` | Filter by port |
| `host IP` | Filter by host |
| `tcp / udp / icmp` | Filter by protocol |
| `src / dst` | Direction filter |

```bash
# Capture HTTP traffic on all interfaces
tcpdump -i any -nn port 80 -A

# Capture traffic between two hosts
tcpdump -i eth0 -nn host 10.0.0.5 and host 10.0.0.10

# Capture DNS queries
tcpdump -i any -nn port 53

# Capture and save to file for Wireshark analysis
tcpdump -i eth0 -nn -s 0 -w /captures/traffic.pcap

# Read a saved capture and filter
tcpdump -r traffic.pcap -nn 'tcp port 443'

# Capture SYN packets only (detect port scans or new connections)
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0'

# Capture ICMP (ping)
tcpdump -i any icmp

# Show HTTP request URLs
tcpdump -i eth0 -nn -A port 80 | grep -E 'GET|POST|Host:'
```

---

## Firewalls

### `ufw` — Uncomplicated Firewall (Ubuntu/Debian)

```bash
ufw status verbose                    # show rules and status
ufw enable                            # enable firewall
ufw disable                           # disable firewall
ufw reset                             # reset all rules (careful!)

# Allow / deny rules
ufw allow ssh                         # allow SSH (port 22)
ufw allow 80/tcp                      # allow HTTP
ufw allow 443/tcp                     # allow HTTPS
ufw allow 8080:8090/tcp               # port range
ufw deny 23/tcp                       # deny telnet

# From specific source
ufw allow from 10.0.0.0/8 to any port 5432    # PostgreSQL from internal only
ufw allow from 192.168.1.50 to any port 22    # SSH from specific IP only

# Delete a rule
ufw delete allow 80/tcp
ufw delete deny 23/tcp

# Numbered rules (easier to delete)
ufw status numbered
ufw delete 3                          # delete rule number 3

# Logging
ufw logging on                        # enable logging
ufw logging high                      # verbose logging
```

### `iptables` — Low-Level Packet Filtering

```bash
# List all rules (with line numbers and counters)
iptables -L -v -n --line-numbers
iptables -t nat -L -v -n             # NAT table

# Allow established connections and loopback
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT

# Allow specific services
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop everything else
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Rate-limit SSH to prevent brute force
iptables -A INPUT -p tcp --dport 22 -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m recent --update --seconds 60 --hitcount 5 --name SSH -j DROP

# NAT: masquerade outbound traffic (share internet via eth0)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# Port forward: redirect port 80 → app on 8080
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# Save rules persistently
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4
```

---

## Nginx

### Reverse Proxy Configuration

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name app.example.com;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    # Modern TLS settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 90s;
        proxy_connect_timeout 10s;
    }

    # Static assets with caching
    location /static/ {
        alias /var/www/myapp/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### Load Balancing

```nginx
upstream backend {
    least_conn;                        # least-connections algorithm
    server 10.0.0.10:8080 weight=3;
    server 10.0.0.11:8080 weight=2;
    server 10.0.0.12:8080 backup;     # only used when others are down

    keepalive 32;                      # persistent connections to upstream
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";   # required for keepalive
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Nginx CLI Commands

```bash
nginx -t                          # test configuration syntax
nginx -T                          # test + print full merged config
nginx -s reload                   # graceful reload (no downtime)
nginx -s quit                     # graceful stop (drain connections)
nginx -s stop                     # fast stop

systemctl reload nginx            # reload via systemd
systemctl status nginx            # check status

# Check access and error logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

---

## WireGuard VPN

WireGuard is a modern, fast, cryptographically sound VPN implemented as a Linux kernel module.

### Server Setup

```bash
# Install WireGuard
apt install wireguard

# Generate key pair (do this on server and each peer)
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key
```

```ini
# /etc/wireguard/wg0.conf (server)
[Interface]
Address    = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server-private-key>

# Enable IP forwarding and masquerade
PostUp     = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown   = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Peer (client)
[Peer]
PublicKey  = <client-public-key>
AllowedIPs = 10.8.0.2/32
```

```ini
# /etc/wireguard/wg0.conf (client)
[Interface]
Address    = 10.8.0.2/24
PrivateKey = <client-private-key>
DNS        = 1.1.1.1

[Peer]
PublicKey  = <server-public-key>
Endpoint   = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0   # route all traffic through VPN
PersistentKeepalive = 25        # keep NAT mapping alive
```

```bash
# Enable IP forwarding persistently
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Start WireGuard
wg-quick up wg0
systemctl enable --now wg-quick@wg0

# Status and statistics
wg show
wg show wg0 peers
wg showconf wg0               # show current running config

# Add a peer at runtime (no restart)
wg set wg0 peer <client-public-key> allowed-ips 10.8.0.3/32
```

---

## Performance Testing

### `iperf3` — Network Bandwidth Testing

```bash
# Start server (on the remote end)
iperf3 -s                            # listen on default port 5201
iperf3 -s -p 9000                    # custom port
iperf3 -s -D                         # run as daemon

# Run client tests
iperf3 -c 10.0.0.5                   # basic TCP throughput test (10 seconds)
iperf3 -c 10.0.0.5 -t 30            # run for 30 seconds
iperf3 -c 10.0.0.5 -P 4             # 4 parallel streams
iperf3 -c 10.0.0.5 -u -b 100M       # UDP test at 100 Mbps
iperf3 -c 10.0.0.5 -R                # reverse: server sends to client
iperf3 -c 10.0.0.5 -J                # JSON output
iperf3 -c 10.0.0.5 --bidir          # bidirectional test
```

### `nethogs` — Per-Process Bandwidth Monitor

```bash
nethogs                               # monitor all interfaces
nethogs eth0                          # specific interface
nethogs -d 2 eth0                     # refresh every 2 seconds
nethogs -b eth0                       # batch mode (non-interactive)
```

### `bmon` / `iftop` / `nload`

```bash
bmon                                  # bandwidth monitor with graphs
iftop -i eth0 -n                      # per-connection bandwidth (like top)
nload eth0                            # simple send/receive graphs
```

### `ping` — ICMP Echo (latency baseline)

```bash
ping -c 10 -i 0.2 8.8.8.8           # 10 pings, 0.2s interval
ping -c 100 -q 8.8.8.8              # 100 pings, quiet (summary only)
ping -s 1400 -c 5 10.0.0.5          # large packet (MTU testing)
ping6 ::1                            # IPv6 loopback
```

---

## Common Patterns & Tips

1. **Always use `-nn` with tcpdump** to suppress both hostname and port name resolution. This avoids DNS lookups that slow down captures and can confuse output during live troubleshooting.

2. **Test port connectivity before debugging the application.** Confirm the port is open at the network level first:
   ```bash
   nc -zvw3 db.internal 5432 && echo "Port open" || echo "Port closed"
   ```

3. **Use `curl -w` to measure HTTP performance per phase** (DNS, TCP connect, TLS handshake, TTFB). This is far quicker than firing up a browser or APM tool:
   ```bash
   curl -o /dev/null -s -w "dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n" https://example.com
   ```

4. **Use `ip route get <dst>` to instantly see which interface and gateway will be used** before debugging a routing issue deeper:
   ```bash
   ip route get 8.8.8.8
   ip route get 10.10.0.50
   ```

5. **WireGuard peer issues? Check `wg show` first.** It tells you when each peer last performed a handshake and how many bytes were transferred. A stale handshake (>3 min) means the tunnel is dead.

6. **Create an SSH `~/.ssh/config` with `ProxyJump` for bastion hosts** instead of chaining `ssh` commands manually. It lets `scp`, `rsync`, `git`, and VSCode Remote all benefit automatically.

7. **Run iperf3 with multiple parallel streams (`-P`) for more realistic throughput numbers.** A single TCP stream is often limited by round-trip latency and doesn't saturate high-bandwidth links; 4–8 streams typically do.

8. **Use `openssl s_client` to test TLS without a browser:**
   ```bash
   # Check if a server presents the correct certificate and chain
   echo QUIT | openssl s_client -connect example.com:443 -servername example.com
   ```

9. **Combine `nmap` with NSE scripts for quick vulnerability checks:**
   ```bash
   nmap --script=ssl-enum-ciphers -p 443 example.com   # list TLS ciphers
   nmap --script=http-security-headers -p 443 example.com
   ```

10. **Preserve firewall rules across reboots on Debian/Ubuntu:**
    ```bash
    apt install iptables-persistent
    netfilter-persistent save    # saves to /etc/iptables/rules.v4 and rules.v6
    ```
