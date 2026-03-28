# Networking Cheatsheet

> A comprehensive reference for network diagnostics, DNS, TLS/SSL, HTTP, firewalls, and packet analysis — with real-world DevOps scenarios.

---

## Table of Contents

- [Network Interfaces & Routing](#network-interfaces--routing)
- [DNS](#dns)
- [HTTP & curl](#http--curl)
- [TLS / SSL](#tls--ssl)
- [SSH](#ssh)
- [Port Scanning & Connectivity](#port-scanning--connectivity)
- [Packet Analysis](#packet-analysis)
- [Firewalls](#firewalls)
- [Load Balancing & Proxies](#load-balancing--proxies)
- [Nginx](#nginx)
- [VPN & Tunneling](#vpn--tunneling)
- [Network Performance](#network-performance)
- [Advanced Scenarios](#advanced-scenarios)

---

## Network Interfaces & Routing

```bash
# Show all interfaces and IPs
ip addr show
ip -4 addr show                    # IPv4 only
ip -6 addr show                    # IPv6 only
ip addr show eth0                  # specific interface

# Bring interface up/down
ip link set eth0 up
ip link set eth0 down

# Add/remove IP address
ip addr add 192.168.1.10/24 dev eth0
ip addr del 192.168.1.10/24 dev eth0

# Show routing table
ip route show
ip route show table all            # all routing tables

# Add / remove routes
ip route add 10.0.0.0/8 via 192.168.1.1
ip route add default via 192.168.1.1
ip route del 10.0.0.0/8

# Show ARP cache (IP → MAC mapping)
ip neigh show
arp -a

# Show network stats
ip -s link show eth0               # TX/RX packets and errors
ss -s                              # socket stats summary
cat /proc/net/dev                  # per-interface counters

# Persistent config (Netplan — Ubuntu 20+)
# /etc/netplan/00-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.10/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
# Apply: netplan apply
```

---

## DNS

```bash
# Basic lookup
nslookup example.com               # simple DNS query
nslookup example.com 8.8.8.8      # query specific DNS server

# dig — detailed DNS queries
dig example.com                    # A record
dig example.com A                  # explicit type
dig example.com MX                 # mail records
dig example.com NS                 # nameservers
dig example.com TXT                # TXT records (SPF, DKIM etc.)
dig example.com AAAA               # IPv6 address
dig example.com CNAME              # canonical name
dig example.com ANY                # all records

# dig with specific DNS server
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# Reverse DNS lookup (IP → hostname)
dig -x 142.250.80.46
nslookup 142.250.80.46

# Trace full DNS resolution path
dig +trace example.com

# Short output
dig +short example.com
dig +short example.com MX

# Check DNS propagation
dig @8.8.8.8 example.com           # Google DNS
dig @1.1.1.1 example.com           # Cloudflare
dig @9.9.9.9 example.com           # Quad9

# host — simple lookups
host example.com
host -t MX example.com
host 142.250.80.46                 # reverse lookup

# resolvectl (systemd-resolved)
resolvectl status
resolvectl query example.com
resolvectl flush-caches            # clear DNS cache

# Local DNS config
cat /etc/resolv.conf               # DNS servers used
cat /etc/hosts                     # static hostname mapping

# Add local hostname override
echo "127.0.0.1 myapp.local" >> /etc/hosts

# DNS record types reference
```

### DNS Record Types

| Type    | Purpose                                   | Example                              |
|---------|-------------------------------------------|--------------------------------------|
| `A`     | IPv4 address                              | `example.com → 93.184.216.34`       |
| `AAAA`  | IPv6 address                              | `example.com → 2606:2800::1`        |
| `CNAME` | Alias to another hostname                 | `www → example.com`                 |
| `MX`    | Mail exchange server                      | `priority 10 mail.example.com`      |
| `TXT`   | Text — SPF, DKIM, domain verification    | `"v=spf1 include:..."`              |
| `NS`    | Nameservers for the domain                | `ns1.example.com`                   |
| `PTR`   | Reverse DNS (IP → hostname)               | `34.216.184.93.in-addr.arpa`        |
| `SOA`   | Start of authority (zone info)            | Primary NS, admin, serial           |
| `SRV`  | Service location (port + priority)        | `_http._tcp.example.com`            |
| `CAA`   | CA allowed to issue certs for domain      | `0 issue "letsencrypt.org"`         |

---

## HTTP & curl

```bash
# Basic GET
curl https://example.com

# Follow redirects
curl -L https://example.com

# Show response headers only
curl -I https://example.com

# Show response + headers
curl -i https://example.com

# Verbose (full request/response)
curl -v https://example.com

# Save to file
curl -o output.html https://example.com
curl -O https://example.com/file.zip    # save with original name

# POST JSON
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane","email":"jane@example.com"}'

# POST form data
curl -X POST https://example.com/login \
  -d "username=jane&password=secret"

# PUT request
curl -X PUT https://api.example.com/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane Doe"}'

# DELETE request
curl -X DELETE https://api.example.com/users/1

# Bearer token auth
curl -H "Authorization: Bearer <TOKEN>" https://api.example.com/me

# Basic auth
curl -u username:password https://api.example.com/data

# Custom headers
curl -H "X-API-Key: abc123" \
     -H "Accept: application/json" \
     https://api.example.com/data

# Upload a file
curl -F "file=@/path/to/file.png" https://api.example.com/upload

# Set timeout
curl --connect-timeout 5 --max-time 30 https://api.example.com

# Skip TLS verification (dev only)
curl -k https://self-signed.example.com

# Use specific TLS cert
curl --cacert /path/to/ca.crt https://example.com

# Measure response time
curl -o /dev/null -s -w \
  "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  https://example.com

# Follow with cookies
curl -c cookies.txt -b cookies.txt https://example.com/login

# HTTP/2
curl --http2 https://example.com

# Download with progress bar
curl -# -O https://example.com/large-file.zip

# Retry on failure
curl --retry 5 --retry-delay 3 https://api.example.com/health

# Check HTTP status code only
curl -o /dev/null -s -w "%{http_code}" https://api.example.com/health
```

### HTTP Status Codes

| Range | Category     | Common Codes                                         |
|-------|--------------|------------------------------------------------------|
| 2xx   | Success      | `200 OK`, `201 Created`, `204 No Content`            |
| 3xx   | Redirect     | `301 Moved`, `302 Found`, `304 Not Modified`         |
| 4xx   | Client Error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests` |
| 5xx   | Server Error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

---

## TLS / SSL

```bash
# Check certificate of a remote server
openssl s_client -connect example.com:443 </dev/null

# Show certificate details
openssl s_client -connect example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -text

# Check expiry date
openssl s_client -connect example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -dates

# Check expiry (one-liner)
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -enddate

# Check certificate for a specific SNI (virtual hosting)
openssl s_client -connect example.com:443 -servername example.com </dev/null

# Check certificate chain
openssl s_client -connect example.com:443 -showcerts </dev/null

# View a local certificate file
openssl x509 -in cert.pem -noout -text
openssl x509 -in cert.pem -noout -subject -issuer -dates

# Verify a certificate against a CA
openssl verify -CAfile ca.crt cert.pem

# Generate a self-signed certificate
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -subj "/CN=localhost/O=Dev/C=US"

# Generate CSR (Certificate Signing Request)
openssl req -new -newkey rsa:2048 -nodes \
  -keyout domain.key \
  -out domain.csr \
  -subj "/CN=example.com/O=MyOrg/C=US"

# View CSR
openssl req -text -noout -verify -in domain.csr

# Generate with SAN (Subject Alternative Names)
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -extensions v3_req \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com,IP:127.0.0.1"

# Check private key matches certificate
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa  -noout -modulus -in key.pem  | md5sum
# → hashes must match

# Convert formats
openssl pkcs12 -export -in cert.pem -inkey key.pem -out cert.p12
openssl pkcs12 -in cert.pem -out output.pem --nodes   # p12 → pem
openssl x509 -in cert.der -inform DER -out cert.pem   # DER → PEM

# Let's Encrypt (certbot)
certbot certonly --standalone -d example.com -d www.example.com
certbot renew --dry-run
certbot renew
certbot certificates                   # list managed certs
```

---

## SSH

```bash
# Connect
ssh user@host
ssh -p 2222 user@host              # custom port
ssh -i ~/.ssh/my-key user@host     # specific key

# Run remote command
ssh user@host "sudo systemctl restart nginx"

# Run multiple commands
ssh user@host << 'EOF'
  sudo apt update
  sudo apt install -y nginx
  sudo systemctl enable --now nginx
EOF

# Copy files
scp file.txt user@host:/tmp/
scp user@host:/var/log/app.log ./  # download
scp -r ./dir user@host:/opt/       # recursive

# rsync (efficient sync)
rsync -avz ./dist/ user@host:/opt/app/dist/
rsync -avz --delete ./dist/ user@host:/opt/app/dist/  # mirror
rsync -avz -e "ssh -p 2222" ./dist/ user@host:/opt/   # custom port

# Tunnel — local port forward (access remote service locally)
ssh -L 5432:db-host:5432 user@bastion-host
# Now: psql -h localhost -p 5432   connects to remote DB

# Tunnel — remote port forward (expose local service remotely)
ssh -R 8080:localhost:3000 user@public-server
# Now: requests to public-server:8080 hit your local :3000

# Dynamic SOCKS proxy
ssh -D 1080 user@host
# Configure browser to use SOCKS5 proxy localhost:1080

# Jump host / bastion
ssh -J user@bastion user@private-host

# SSH config (~/.ssh/config)
Host bastion
  HostName 1.2.3.4
  User ubuntu
  IdentityFile ~/.ssh/bastion-key

Host private-server
  HostName 10.0.1.20
  User ubuntu
  ProxyJump bastion
  IdentityFile ~/.ssh/private-key

# Then simply:
ssh private-server

# Key management
ssh-keygen -t ed25519 -C "jane@example.com"    # generate key pair
ssh-keygen -t rsa -b 4096 -C "jane@example.com"

ssh-copy-id user@host                           # copy public key to server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys   # manual

# Fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub            # show fingerprint
ssh-keyscan github.com >> ~/.ssh/known_hosts    # pre-approve host

# SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l                                       # list loaded keys
```

### `sshd_config` Hardening

```bash
# /etc/ssh/sshd_config
Port 2222                          # non-default port
PermitRootLogin no                 # never allow root login
PasswordAuthentication no          # keys only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers ubuntu deploy           # whitelist users
X11Forwarding no
AllowTcpForwarding no              # unless tunnels are needed

# Restart after changes
systemctl restart sshd
```

---

## Port Scanning & Connectivity

```bash
# Check if a port is open (no nmap needed)
nc -zv example.com 443             # TCP check
nc -zv -u example.com 53           # UDP check
nc -w 3 -zv 10.0.0.1 22           # with 3s timeout

# Listen on a port (for testing)
nc -l 8080                         # listen, print received data
nc -l 8080 | cat                   # same

# Simple chat / file transfer with nc
nc -l 8080 > received.txt          # server: save to file
nc host 8080 < file.txt            # client: send file

# nmap — port scanning
nmap example.com                   # scan common ports
nmap -p 80,443,8080 example.com    # specific ports
nmap -p 1-1000 example.com         # port range
nmap -p- example.com               # all 65535 ports (slow)
nmap -sV example.com               # detect service versions
nmap -O example.com                # detect OS (root needed)
nmap -A example.com                # aggressive (OS + version + scripts)
nmap -sn 192.168.1.0/24           # ping scan (host discovery)
nmap -Pn example.com               # skip ping, assume host up
nmap --script ssl-cert example.com # check SSL certificate

# Check open ports on local machine
ss -tulnp                          # all listening sockets
ss -tulnp | grep :80               # filter by port
lsof -i :8080                      # what process owns port 8080
fuser 8080/tcp                     # PID using the port

# telnet (basic TCP check)
telnet example.com 80
# Type: GET / HTTP/1.0 then Enter twice

# ping
ping -c 4 example.com              # 4 packets
ping -i 0.2 example.com            # faster interval
ping -s 1400 example.com           # custom packet size (MTU test)
ping6 ipv6.example.com             # IPv6

# traceroute / tracepath
traceroute example.com
traceroute -T -p 443 example.com   # TCP traceroute on port 443
tracepath example.com              # no root needed
mtr example.com                    # live traceroute + stats
mtr --report --report-cycles 20 example.com   # report mode
```

---

## Packet Analysis

```bash
# tcpdump — capture packets
tcpdump                             # all traffic (verbose)
tcpdump -i eth0                     # specific interface
tcpdump -i any                      # all interfaces
tcpdump -n                          # don't resolve hostnames
tcpdump -nn                         # don't resolve hostnames or ports

# Filter by host
tcpdump host 10.0.0.1
tcpdump src 10.0.0.1                # source only
tcpdump dst 10.0.0.1                # destination only

# Filter by port
tcpdump port 443
tcpdump port 80 or port 443
tcpdump dst port 8080

# Filter by protocol
tcpdump tcp
tcpdump udp
tcpdump icmp

# Combine filters
tcpdump host 10.0.0.1 and port 443
tcpdump host 10.0.0.1 and not port 22

# Capture to file (for Wireshark)
tcpdump -i eth0 -w capture.pcap
tcpdump -r capture.pcap             # read from file

# Show packet contents
tcpdump -A port 80                  # ASCII output
tcpdump -X port 80                  # hex + ASCII

# Limit packet count
tcpdump -c 100 port 443

# Capture HTTP requests
tcpdump -i any -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# ss — socket statistics
ss -tulnp                           # listening TCP+UDP with process
ss -tnp                             # established TCP connections
ss -tp state established            # only established
ss -o state fin-wait-1              # specific state
ss -tnp dst 10.0.0.1               # connections to specific host
ss -tnp dst :443                    # connections to port 443
```

---

## Firewalls

### ufw (Ubuntu)

```bash
ufw status verbose
ufw enable
ufw disable
ufw reset                           # remove all rules

# Allow / deny
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 23/tcp
ufw allow from 10.0.0.0/8

# App profiles
ufw app list
ufw allow 'Nginx Full'
ufw allow 'OpenSSH'

# Allow from specific IP to specific port
ufw allow from 192.168.1.100 to any port 5432

# Delete rules
ufw delete allow 80/tcp
ufw delete 3                        # by rule number (ufw status numbered)

# Logging
ufw logging on
ufw logging medium
```

### iptables

```bash
# List rules
iptables -L -v -n
iptables -L -v -n --line-numbers    # with line numbers

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Allow from specific subnet
iptables -A INPUT -s 10.0.0.0/8 -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP

# Rate limiting (SSH brute force protection)
iptables -A INPUT -p tcp --dport 22 \
  -m state --state NEW \
  -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 \
  -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 5 --name SSH \
  -j DROP

# NAT — masquerade (share internet from one interface)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
echo 1 > /proc/sys/net/ipv4/ip_forward

# Port forwarding
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3000

# Save / restore
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4

# Flush all rules (DANGEROUS — drops all)
iptables -F
```

### nftables (modern replacement for iptables)

```bash
# List rules
nft list ruleset

# Basic ruleset
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input iif lo accept
nft add rule inet filter input tcp dport { 22, 80, 443 } accept

# Save
nft list ruleset > /etc/nftables.conf
```

---

## Load Balancing & Proxies

### HAProxy

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 50000

defaults
    mode    http
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    option  httplog

frontend http-in
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/myapp.pem
    redirect scheme https if !{ ssl_fc }
    default_backend app-servers

backend app-servers
    balance roundrobin
    option httpchk GET /health
    server app1 10.0.0.1:3000 check
    server app2 10.0.0.2:3000 check
    server app3 10.0.0.3:3000 check

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg    # validate config
systemctl reload haproxy                   # reload without downtime
```

---

## Nginx

```nginx
# /etc/nginx/nginx.conf — main config
worker_processes auto;
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    gzip          on;
    gzip_types    text/plain application/json application/javascript text/css;

    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# /etc/nginx/conf.d/myapp.conf

# ── Redirect HTTP → HTTPS ─────────────────────────────────
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

# ── HTTPS Server ──────────────────────────────────────────
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy no-referrer-when-downgrade;

    # ── Static files ──────────────────────────────────────
    location /static/ {
        root /opt/app;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # ── API reverse proxy ─────────────────────────────────
    location /api/ {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 60s;
    }

    # ── WebSocket support ─────────────────────────────────
    location /ws/ {
        proxy_pass         http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "Upgrade";
        proxy_set_header   Host $host;
    }

    # ── SPA fallback ──────────────────────────────────────
    location / {
        root  /opt/app/dist;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}

# ── Upstream load balancing ───────────────────────────────
upstream app_servers {
    least_conn;
    server 10.0.0.1:3000 weight=3;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000 backup;   # only used when others fail
    keepalive 32;
}
```

```bash
nginx -t                            # test config
nginx -T                            # test + dump full config
nginx -s reload                     # reload without downtime
nginx -s stop                       # fast stop
systemctl reload nginx              # graceful reload
```

---

## VPN & Tunneling

### WireGuard

```ini
# /etc/wireguard/wg0.conf — Server

[Interface]
Address    = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>
PostUp     = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown   = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey  = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32
```

```ini
# /etc/wireguard/wg0.conf — Client

[Interface]
Address    = 10.8.0.2/32
PrivateKey = <CLIENT_PRIVATE_KEY>
DNS        = 1.1.1.1

[Peer]
PublicKey    = <SERVER_PUBLIC_KEY>
Endpoint     = server.example.com:51820
AllowedIPs   = 0.0.0.0/0          # route all traffic through VPN
PersistentKeepalive = 25
```

```bash
# Key generation
wg genkey | tee private.key | wg pubkey > public.key

# Manage
wg-quick up wg0
wg-quick down wg0
wg show                            # status
systemctl enable wg-quick@wg0     # auto-start
```

### SSH Tunneling

```bash
# Local forward: access remote DB locally
ssh -L 5432:db.internal:5432 user@bastion -N -f
psql -h localhost -p 5432 -U admin mydb

# Remote forward: expose local dev server
ssh -R 80:localhost:3000 user@public-server -N -f

# Dynamic SOCKS5 proxy
ssh -D 1080 -q -C -N user@server
curl --socks5 localhost:1080 https://internal.example.com
```

---

## Network Performance

```bash
# Bandwidth test with iperf3
# Server
iperf3 -s

# Client
iperf3 -c server-ip
iperf3 -c server-ip -t 30          # 30 second test
iperf3 -c server-ip -P 4          # 4 parallel streams
iperf3 -c server-ip -R            # reverse (server sends to client)
iperf3 -c server-ip -u -b 100M   # UDP at 100 Mbps

# MTU discovery
ping -M do -s 1400 example.com     # test specific size
tracepath example.com              # shows PMTU

# Network latency
ping -c 100 example.com | tail -1  # avg/min/max
hping3 -S --flood -V -p 80 example.com   # TCP ping

# Bandwidth monitoring
iftop -i eth0                      # live per-connection bandwidth
nethogs                            # per-process bandwidth
nload                              # interface bandwidth graph
vnstat                             # historical bandwidth stats
vnstat -l -i eth0                  # live mode

# Check NIC speed and duplex
ethtool eth0
ethtool eth0 | grep -E "Speed|Duplex"

# TCP performance tuning
# /etc/sysctl.conf
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq

sysctl -p                          # apply changes
```

---

## Advanced Scenarios

### Scenario 1 — Debug why a site returns 502

```bash
# 1. Check if backend is up
curl -v http://localhost:3000/health

# 2. Check nginx error log
tail -50 /var/log/nginx/error.log

# 3. Check if backend port is listening
ss -tlnp | grep :3000

# 4. Check nginx upstream config
nginx -T | grep -A5 "upstream"

# 5. Trace the actual connection
curl -v --resolve example.com:443:127.0.0.1 https://example.com/api/
```

### Scenario 2 — Diagnose SSL certificate issues

```bash
# Check expiry
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# Check chain is complete
openssl s_client -connect example.com:443 -showcerts </dev/null 2>/dev/null \
  | grep "s:/CN"

# Check cert matches domain
openssl s_client -connect example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -subj_hash

# Verify cert + key match
diff \
  <(openssl x509 -noout -modulus -in cert.pem | md5sum) \
  <(openssl rsa  -noout -modulus -in key.pem  | md5sum)
```

### Scenario 3 — Find what's consuming bandwidth

```bash
# Per-process bandwidth (real time)
nethogs eth0

# Top connections by volume
iftop -i eth0 -n

# Capture and summarize traffic
tcpdump -i eth0 -w /tmp/cap.pcap -c 10000
tcpdump -r /tmp/cap.pcap -nn | awk '{print $3}' | sort | uniq -c | sort -rn | head
```

### Scenario 4 — Block an IP brute-forcing SSH

```bash
# Manual block
iptables -A INPUT -s 1.2.3.4 -j DROP

# Install and configure fail2ban (automated)
apt install fail2ban
cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled  = true
port     = ssh
maxretry = 5
bantime  = 3600
findtime = 600
EOF
systemctl enable --now fail2ban
fail2ban-client status sshd        # view banned IPs
fail2ban-client set sshd unbanip 1.2.3.4
```

### Scenario 5 — Test if MTU mismatch is causing packet loss

```bash
# Find working MTU by testing decreasing sizes
for size in 1472 1400 1300 1200; do
  result=$(ping -M do -s $size -c 3 -q example.com 2>&1)
  if echo "$result" | grep -q "0% packet loss"; then
    echo "MTU $((size + 28)) works"
    break
  else
    echo "MTU $((size + 28)) fails"
  fi
done
```

### Scenario 6 — Capture credentials-free HTTP traffic for debugging

```bash
# Capture HTTP on port 8080 (internal/dev only!)
tcpdump -i any -A -s 0 'tcp port 8080' 2>/dev/null \
  | grep -E "GET|POST|Host:|Content-Type:|Authorization:"
```

### Scenario 7 — Test load balancer is distributing traffic

```bash
for i in $(seq 1 10); do
  curl -s https://api.example.com/whoami | jq -r '.hostname'
done
# Should see different hostnames round-robined
```

---

## Quick Reference Card

| Task                            | Command                                                      |
|---------------------------------|--------------------------------------------------------------|
| Show all IPs                    | `ip addr show`                                               |
| Show routing table              | `ip route show`                                              |
| DNS lookup                      | `dig +short example.com`                                     |
| Full DNS trace                  | `dig +trace example.com`                                     |
| Check cert expiry               | `echo \| openssl s_client -connect host:443 2>/dev/null \| openssl x509 -noout -dates` |
| Test port open                  | `nc -zv host 443`                                            |
| All listening ports             | `ss -tulnp`                                                  |
| Who's on port 80                | `lsof -i :80`                                                |
| Capture traffic on port         | `tcpdump -i any port 443`                                    |
| Measure HTTP response time      | `curl -o /dev/null -s -w "%{time_total}" https://example.com` |
| Test bandwidth                  | `iperf3 -c server-ip`                                        |
| Live traffic per process        | `nethogs eth0`                                               |
| SSH tunnel (local forward)      | `ssh -L local_port:remote_host:remote_port user@bastion`     |

---

> When debugging network issues: connectivity first (`ping`) → routing (`ip route`) → DNS (`dig`) → port (`nc`) → TLS (`openssl`) → application (`curl -v`).
