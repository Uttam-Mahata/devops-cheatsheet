# Linux & Bash Cheatsheet

> A comprehensive reference for Linux administration and Bash scripting — from file management and process control to systemd, cron, and shell scripting patterns.

---

## Table of Contents

- [Navigation & File Management](#navigation--file-management)
- [File Permissions & Ownership](#file-permissions--ownership)
- [Users & Groups](#users--groups)
- [Process Management](#process-management)
- [System Information](#system-information)
- [Disk & Storage](#disk--storage)
- [Package Management](#package-management)
- [Networking](#networking)
- [systemd & Services](#systemd--services)
- [Cron & Scheduling](#cron--scheduling)
- [Archives & Compression](#archives--compression)
- [Text Processing](#text-processing)
- [Environment & Shell](#environment--shell)
- [Bash Scripting](#bash-scripting)
- [Advanced Scenarios](#advanced-scenarios)

---

## Navigation & File Management

```bash
# Navigation
pwd                        # print working directory
cd /etc/nginx              # absolute path
cd ..                      # go up one level
cd -                       # go back to previous directory
cd ~                       # go to home directory

# List files
ls -la                     # long format, show hidden files
ls -lh                     # human-readable sizes
ls -lt                     # sort by modification time
ls -lR                     # recursive

# Create
mkdir -p /opt/app/config   # create nested dirs
touch file.txt             # create empty file
touch -t 202401011200 f    # set timestamp

# Copy
cp file.txt backup.txt
cp -r ./src ./src-backup   # recursive copy
cp -p file.txt dest/       # preserve permissions + timestamps

# Move / Rename
mv old.txt new.txt
mv ./src /opt/app/src

# Delete
rm file.txt
rm -rf ./old-dir           # recursive force (CAREFUL)
rmdir empty-dir            # remove empty directory only

# Find files
find / -name "nginx.conf" 2>/dev/null
find /var/log -name "*.log" -mtime -7    # modified in last 7 days
find /tmp -size +100M                     # files larger than 100MB
find . -type f -empty                     # empty files
find . -name "*.sh" -exec chmod +x {} \; # find and execute

# Symbolic links
ln -s /opt/app/config /etc/app-config    # create symlink
ls -la /etc/app-config                    # shows -> target
unlink /etc/app-config                    # remove symlink

# File info
stat file.txt              # full metadata (size, inode, times)
file image.png             # detect file type
du -sh ./node_modules      # directory size (human readable)
du -sh * | sort -rh | head -10   # top 10 largest items
```

---

## File Permissions & Ownership

```bash
# View permissions
ls -la file.txt
# -rwxr-xr-- 1 user group 1024 Jan 01 12:00 file.txt
#  ^  ^  ^
#  |  |  └── others: r-- (4)
#  |  └───── group:  r-x (5)
#  └──────── owner:  rwx (7)

# chmod — symbolic mode
chmod u+x script.sh        # add execute for owner
chmod g-w file.txt         # remove write for group
chmod o=r file.txt         # set others to read-only
chmod a+x script.sh        # add execute for all

# chmod — numeric mode
chmod 755 script.sh        # rwxr-xr-x
chmod 644 file.txt         # rw-r--r--
chmod 600 .ssh/id_rsa      # rw------- (private key)
chmod 777 file.txt         # rwxrwxrwx (avoid in production)

# chmod recursive
chmod -R 755 /opt/app

# chown — change owner
chown user file.txt
chown user:group file.txt
chown -R appuser:appgroup /opt/app

# Special bits
chmod u+s /usr/bin/passwd  # setuid — runs as file owner
chmod g+s /shared/dir      # setgid — new files inherit group
chmod +t /tmp              # sticky bit — only owner can delete

# Default permissions (umask)
umask                      # show current umask (e.g. 0022)
umask 027                  # new files: 640, dirs: 750
```

### Permission Reference

| Numeric | Symbolic  | Meaning              |
|---------|-----------|----------------------|
| `7`     | `rwx`     | read + write + exec  |
| `6`     | `rw-`     | read + write         |
| `5`     | `r-x`     | read + exec          |
| `4`     | `r--`     | read only            |
| `0`     | `---`     | no permissions       |

---

## Users & Groups

```bash
# User info
whoami                     # current user
id                         # uid, gid, groups
id username                # info for another user
who                        # logged-in users
w                          # logged-in users + activity
last                       # login history

# Create user
useradd -m -s /bin/bash jane           # create with home dir + bash shell
useradd -m -G sudo,docker jane        # add to groups on creation
passwd jane                            # set password

# Modify user
usermod -aG docker jane               # add to group (append)
usermod -s /bin/zsh jane              # change shell
usermod -l newname oldname            # rename user
usermod -L jane                        # lock account
usermod -U jane                        # unlock account

# Delete user
userdel jane                           # delete user
userdel -r jane                        # delete user + home dir

# Groups
groupadd developers
groupdel developers
groups jane                            # list groups for user
gpasswd -d jane developers            # remove user from group

# Switch user
su - jane                             # switch to jane (login shell)
sudo -u jane command                  # run command as jane
sudo -i                               # root shell via sudo

# sudoers
visudo                                # safely edit /etc/sudoers

# /etc/sudoers example
jane ALL=(ALL:ALL) NOPASSWD: /usr/bin/docker   # specific command, no password
%developers ALL=(ALL) ALL                       # group can sudo everything

# User files
cat /etc/passwd                       # all users (no passwords)
cat /etc/group                        # all groups
cat /etc/shadow                       # hashed passwords (root only)
```

---

## Process Management

```bash
# View processes
ps aux                     # all processes (BSD style)
ps -ef                     # all processes (UNIX style)
ps aux | grep nginx        # filter processes
pgrep nginx                # get PIDs by name
pgrep -u jane              # processes by user

# Interactive process viewers
top                        # live process monitor
htop                       # better top (if installed)

# Kill processes
kill <PID>                 # SIGTERM (graceful)
kill -9 <PID>              # SIGKILL (force)
kill -HUP <PID>            # SIGHUP (reload config)
killall nginx              # kill all by name
pkill -f "node server.js"  # kill by pattern

# Background jobs
command &                  # run in background
jobs                       # list background jobs
fg %1                      # bring job 1 to foreground
bg %1                      # resume job 1 in background
nohup command &            # run immune to hangup
disown %1                  # detach job from shell

# Priority
nice -n 10 command         # start with lower priority (-20 high, 19 low)
renice -n 5 -p <PID>       # change priority of running process

# Process tree
pstree                     # tree view of processes
pstree -p <PID>            # tree from specific PID

# Signals reference
# SIGTERM (15) — graceful shutdown (default kill)
# SIGKILL  (9) — force kill, cannot be caught
# SIGHUP   (1) — reload config
# SIGINT   (2) — interrupt (Ctrl+C)
# SIGSTOP (19) — pause process (Ctrl+Z)
# SIGCONT (18) — resume paused process

# strace — trace system calls
strace -p <PID>            # attach to running process
strace -e trace=network command   # only network calls

# lsof — list open files/sockets
lsof -p <PID>              # files opened by process
lsof -i :80                # process using port 80
lsof -u jane               # files opened by user
```

---

## System Information

```bash
# OS info
uname -a                   # kernel version + arch
cat /etc/os-release        # distro info
hostnamectl                # hostname + OS details

# Hardware
lscpu                      # CPU info
lsmem                      # memory layout
lspci                      # PCI devices
lsblk                      # block devices (disks)
lsusb                      # USB devices

# Memory
free -h                    # RAM usage (human readable)
vmstat -s                  # virtual memory stats
cat /proc/meminfo          # detailed memory info

# CPU
mpstat 1 5                 # CPU stats every 1s, 5 times
top -bn1 | grep "Cpu(s)"   # one-shot CPU snapshot

# Uptime & load
uptime                     # uptime + load averages
cat /proc/loadavg          # 1m 5m 15m load averages

# Environment
env                        # all environment variables
printenv HOME              # specific variable
uname -r                   # kernel version only

# Limits
ulimit -a                  # all limits for current session
ulimit -n 65536            # set open file limit
cat /proc/<PID>/limits     # limits for a running process

# Date & time
date                       # current date/time
date -u                    # UTC time
date +"%Y-%m-%d %H:%M:%S"  # formatted
timedatectl                # timezone + NTP status
timedatectl set-timezone UTC
```

---

## Disk & Storage

```bash
# Disk usage
df -h                      # filesystem usage (human readable)
df -hT                     # include filesystem type
du -sh /var/log            # size of directory
du -sh * | sort -rh        # sorted by size

# Disk info
lsblk                      # block device tree
fdisk -l                   # partition table (root needed)
blkid                      # UUIDs and filesystem types

# Mount / unmount
mount /dev/sdb1 /mnt/data
umount /mnt/data
mount -t nfs server:/share /mnt/nfs      # NFS mount
mount | grep sdb                          # show current mounts

# /etc/fstab — persistent mounts
# UUID=xxx  /data  ext4  defaults  0  2

# Filesystem operations
mkfs.ext4 /dev/sdb1        # format as ext4
mkfs.xfs /dev/sdb1         # format as xfs
fsck /dev/sdb1             # check filesystem (unmounted)
tune2fs -l /dev/sda1       # ext4 metadata
resize2fs /dev/sda1        # resize ext4 after LVM expansion

# LVM
pvdisplay                  # physical volumes
vgdisplay                  # volume groups
lvdisplay                  # logical volumes
lvextend -L +10G /dev/vg0/lv0   # extend LV by 10GB
resize2fs /dev/vg0/lv0          # grow filesystem after extend

# Swap
swapon --show              # show swap usage
swapoff /swapfile          # disable swap
fallocate -l 2G /swapfile  # create 2GB swap file
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

---

## Package Management

### apt (Debian/Ubuntu)

```bash
apt update                         # refresh package index
apt upgrade                        # upgrade installed packages
apt full-upgrade                   # upgrade + remove obsolete
apt install nginx                  # install package
apt install -y nginx               # no confirmation prompt
apt remove nginx                   # remove (keep config)
apt purge nginx                    # remove + config files
apt autoremove                     # remove unused dependencies
apt search nginx                   # search packages
apt show nginx                     # package info
apt list --installed               # list installed packages
dpkg -l | grep nginx               # check if installed
dpkg -L nginx                      # list files from package
```

### yum / dnf (RHEL/CentOS/Fedora)

```bash
dnf update                         # update all
dnf install nginx                  # install
dnf remove nginx                   # remove
dnf search nginx                   # search
dnf info nginx                     # info
dnf list installed                 # list installed
rpm -qa | grep nginx               # check if installed
rpm -ql nginx                      # list files from package
```

---

## Networking

```bash
# Interface info
ip addr show                       # all interfaces + IPs
ip addr show eth0                  # specific interface
ip link show                       # link status
ifconfig                           # legacy (net-tools)

# Routing
ip route show                      # routing table
ip route add default via 192.168.1.1   # add default gateway
ip route del 10.0.0.0/8

# DNS
cat /etc/resolv.conf               # DNS servers
resolvectl status                  # systemd-resolved status
hostnamectl                        # hostname info

# Connectivity
ping -c 4 google.com
traceroute google.com
mtr google.com                     # live traceroute

# Ports
ss -tulnp                          # all listening ports + process
ss -tulnp | grep :80               # filter by port
netstat -tulnp                     # legacy (net-tools)
lsof -i :8080                      # what's on port 8080

# Firewall (ufw)
ufw status
ufw allow 80/tcp
ufw allow from 10.0.0.0/24 to any port 22
ufw deny 23
ufw enable
ufw disable

# Firewall (firewalld)
firewall-cmd --list-all
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --reload

# iptables
iptables -L -n -v                  # list all rules
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -j DROP          # drop everything else
iptables-save > /etc/iptables/rules.v4   # persist rules
```

---

## systemd & Services

```bash
# Service management
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx             # reload config without restart
systemctl status nginx
systemctl enable nginx             # start on boot
systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx

# List services
systemctl list-units --type=service
systemctl list-units --type=service --state=failed
systemctl list-unit-files --type=service

# View logs for a service
journalctl -u nginx
journalctl -u nginx -f             # follow
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -n 100         # last 100 lines
journalctl -u nginx -p err         # only errors

# System logs
journalctl -f                      # follow all logs
journalctl --since "2024-01-01"
journalctl -b                      # logs since last boot
journalctl -b -1                   # logs from previous boot
journalctl --disk-usage            # how much log space is used
journalctl --vacuum-time=7d        # delete logs older than 7 days

# systemd units
systemctl cat nginx                # view unit file
systemctl edit nginx               # override unit file (drop-in)
systemctl daemon-reload            # reload after unit file changes

# Targets (like runlevels)
systemctl get-default              # current default target
systemctl set-default multi-user.target
systemctl isolate rescue.target    # switch target now
```

### Custom Service Unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/app
ExecStart=/usr/bin/node /opt/app/server.js
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal
Environment=NODE_ENV=production
EnvironmentFile=/opt/app/.env

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now myapp
```

---

## Cron & Scheduling

```bash
# Edit crontab for current user
crontab -e

# List crontab
crontab -l

# Edit another user's crontab
crontab -u jane -e

# Remove crontab
crontab -r

# System-wide cron dirs
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
/etc/cron.d/               # drop-in cron files
```

### Cron Syntax

```
┌───────── minute       (0–59)
│ ┌─────── hour         (0–23)
│ │ ┌───── day of month (1–31)
│ │ │ ┌─── month        (1–12)
│ │ │ │ ┌─ day of week  (0–7, 0=Sun)
│ │ │ │ │
* * * * *  command

# Examples
0  2 * * *   /opt/scripts/backup.sh       # daily at 02:00
*/5 * * * *  /opt/scripts/health-check.sh # every 5 minutes
0  9 * * 1   /opt/scripts/report.sh       # every Monday at 09:00
0  0 1 * *   /opt/scripts/cleanup.sh      # 1st of every month
@reboot      /opt/scripts/startup.sh      # on every reboot
```

### at — one-time scheduling

```bash
echo "/opt/scripts/deploy.sh" | at 02:00 tomorrow
echo "systemctl restart nginx" | at now + 5 minutes
atq                        # list scheduled jobs
atrm 3                     # remove job 3
```

---

## Archives & Compression

```bash
# tar
tar -cvf archive.tar ./dir           # create
tar -xvf archive.tar                 # extract
tar -xvf archive.tar -C /opt/        # extract to specific dir
tar -tvf archive.tar                 # list contents

# tar + gzip
tar -czvf archive.tar.gz ./dir
tar -xzvf archive.tar.gz

# tar + bzip2 (smaller, slower)
tar -cjvf archive.tar.bz2 ./dir
tar -xjvf archive.tar.bz2

# tar + xz (smallest, slowest)
tar -cJvf archive.tar.xz ./dir
tar -xJvf archive.tar.xz

# zip / unzip
zip -r archive.zip ./dir
zip -r archive.zip ./dir -x "*.git*"  # exclude .git
unzip archive.zip
unzip archive.zip -d /opt/

# gzip / gunzip (single files)
gzip file.txt               # creates file.txt.gz, removes original
gunzip file.txt.gz
gzip -k file.txt            # keep original
gzip -d file.txt.gz         # decompress (same as gunzip)

# Quick size comparison
gzip  -c file | wc -c
bzip2 -c file | wc -c
xz    -c file | wc -c
```

---

## Text Processing

```bash
# View files
cat file.txt
less file.txt              # paginated (q to quit, / to search)
head -20 file.txt          # first 20 lines
tail -20 file.txt          # last 20 lines
tail -f /var/log/syslog    # follow file in real time

# Search
grep "error" app.log
grep -i "error" app.log    # case insensitive
grep -r "TODO" ./src       # recursive
grep -n "error" app.log    # show line numbers
grep -v "debug" app.log    # invert match (exclude)
grep -c "error" app.log    # count matches
grep -A 3 "error" app.log  # 3 lines after match
grep -B 2 "error" app.log  # 2 lines before match
grep -E "error|warn" app.log  # regex (extended)

# Stream editing
sed 's/foo/bar/' file.txt           # replace first occurrence per line
sed 's/foo/bar/g' file.txt          # replace all occurrences
sed -i 's/foo/bar/g' file.txt       # in-place edit
sed -n '10,20p' file.txt            # print lines 10–20
sed '/^#/d' file.txt                # delete comment lines
sed '/^$/d' file.txt                # delete empty lines

# awk — column processing
awk '{print $1}' file.txt           # print first column
awk -F: '{print $1}' /etc/passwd    # custom delimiter
awk '/error/ {print $0}' app.log    # filter lines
awk '{sum += $1} END {print sum}'   # sum a column
awk 'NR==5' file.txt                # print line 5
ps aux | awk '{print $2, $11}'      # PID + command

# sort, uniq, wc
sort file.txt
sort -r file.txt                    # reverse
sort -n file.txt                    # numeric sort
sort -k2 file.txt                   # sort by column 2
sort file.txt | uniq                # remove duplicates
sort file.txt | uniq -c             # count occurrences
wc -l file.txt                      # count lines
wc -w file.txt                      # count words

# cut — extract columns
cut -d: -f1 /etc/passwd             # first field, colon delimited
cut -d, -f2,4 data.csv              # fields 2 and 4

# tr — translate/delete characters
echo "HELLO" | tr 'A-Z' 'a-z'      # lowercase
echo "hello world" | tr -d ' '     # delete spaces
echo "a::b::c" | tr -s ':'         # squeeze repeated chars

# diff
diff file1.txt file2.txt
diff -u file1.txt file2.txt         # unified format (like git diff)
diff -r dir1/ dir2/                 # compare directories
```

---

## Environment & Shell

```bash
# Variables
MY_VAR="hello"             # set variable
export MY_VAR="hello"      # export to child processes
unset MY_VAR               # remove variable
echo $MY_VAR               # use variable
echo ${MY_VAR:-default}    # use default if unset
echo ${MY_VAR:=default}    # set and use default if unset
echo ${#MY_VAR}            # length of variable
echo ${MY_VAR^^}           # uppercase
echo ${MY_VAR,,}           # lowercase

# PATH
echo $PATH
export PATH=$PATH:/opt/myapp/bin
# Make permanent: add to ~/.bashrc or ~/.profile

# Shell config files (order of execution)
# Login shell:     /etc/profile → ~/.bash_profile → ~/.bashrc
# Interactive:     ~/.bashrc
# All users:       /etc/environment (key=value only)
source ~/.bashrc            # reload config
. ~/.bashrc                 # same, shorthand

# History
history                    # command history
history | grep docker      # search history
!!                          # repeat last command
!ssh                        # repeat last ssh command
Ctrl+R                      # reverse search history
HISTSIZE=10000             # history size
HISTFILESIZE=20000
HISTCONTROL=ignoredups     # no duplicate entries

# Aliases
alias ll='ls -la'
alias gs='git status'
alias k='kubectl'
# Put in ~/.bashrc for persistence

# Functions
mkcd() { mkdir -p "$1" && cd "$1"; }
```

---

## Bash Scripting

### Script Template

```bash
#!/usr/bin/env bash
set -euo pipefail          # exit on error, unbound vars, pipe fail
IFS=$'\n\t'                # safer word splitting

# ── Constants ──────────────────────────────────────────────
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/myapp/deploy.log"

# ── Logging ────────────────────────────────────────────────
log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] INFO  $*" | tee -a "$LOG_FILE"; }
warn() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] WARN  $*" | tee -a "$LOG_FILE" >&2; }
err()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR $*" | tee -a "$LOG_FILE" >&2; }

# ── Cleanup trap ───────────────────────────────────────────
cleanup() {
  log "Cleaning up..."
  rm -f /tmp/deploy-$$-*
}
trap cleanup EXIT

# ── Usage ──────────────────────────────────────────────────
usage() {
  echo "Usage: $0 [OPTIONS]"
  echo "  -e ENV    Environment (staging|production)"
  echo "  -t TAG    Image tag to deploy"
  echo "  -h        Show this help"
  exit 1
}

# ── Parse arguments ────────────────────────────────────────
ENV=""
TAG=""
while getopts "e:t:h" opt; do
  case $opt in
    e) ENV="$OPTARG" ;;
    t) TAG="$OPTARG" ;;
    h) usage ;;
    *) usage ;;
  esac
done

[[ -z "$ENV" ]] && { err "Environment is required"; usage; }
[[ -z "$TAG" ]] && { err "Tag is required"; usage; }

log "Deploying $TAG to $ENV"
```

### Control Flow

```bash
# if / elif / else
if [[ "$ENV" == "production" ]]; then
  echo "deploying to prod"
elif [[ "$ENV" == "staging" ]]; then
  echo "deploying to staging"
else
  echo "unknown environment"
fi

# Comparison operators
[[ "$a" == "$b" ]]     # string equal
[[ "$a" != "$b" ]]     # string not equal
[[ "$a" < "$b" ]]      # string less than
[[ -z "$a" ]]          # string is empty
[[ -n "$a" ]]          # string is not empty
[[ "$a" =~ ^[0-9]+$ ]] # regex match

# Numeric comparisons
[[ $n -eq 0 ]]         # equal
[[ $n -ne 0 ]]         # not equal
[[ $n -lt 10 ]]        # less than
[[ $n -gt 10 ]]        # greater than
[[ $n -le 10 ]]        # less or equal
[[ $n -ge 10 ]]        # greater or equal

# File tests
[[ -f file.txt ]]      # file exists and is a regular file
[[ -d /opt/app ]]      # directory exists
[[ -e path ]]          # path exists (any type)
[[ -r file.txt ]]      # readable
[[ -w file.txt ]]      # writable
[[ -x script.sh ]]     # executable
[[ -s file.txt ]]      # file is non-empty
[[ file1 -nt file2 ]]  # file1 newer than file2

# Loops
for i in 1 2 3; do echo "$i"; done

for file in /var/log/*.log; do
  echo "Processing $file"
done

for i in $(seq 1 10); do echo "$i"; done

while [[ "$count" -lt 5 ]]; do
  echo "$count"
  ((count++))
done

until systemctl is-active --quiet nginx; do
  echo "Waiting for nginx..."
  sleep 2
done

# break and continue
for i in $(seq 1 10); do
  [[ $i -eq 5 ]] && break
  [[ $((i % 2)) -eq 0 ]] && continue
  echo "$i"
done

# case statement
case "$ENV" in
  production)  REPLICAS=5 ;;
  staging)     REPLICAS=2 ;;
  development) REPLICAS=1 ;;
  *)           err "Unknown env: $ENV"; exit 1 ;;
esac
```

### Functions & Error Handling

```bash
# Function with return value
get_pod_count() {
  local namespace="$1"
  kubectl get pods -n "$namespace" --no-headers | wc -l
}

count=$(get_pod_count production)
echo "Running pods: $count"

# Check command success
if ! kubectl apply -f deployment.yaml; then
  err "Deployment failed"
  exit 1
fi

# Retry logic
retry() {
  local max="$1"; shift
  local delay="$1"; shift
  local attempt=1

  until "$@"; do
    if (( attempt >= max )); then
      err "Command failed after $max attempts: $*"
      return 1
    fi
    warn "Attempt $attempt failed. Retrying in ${delay}s..."
    sleep "$delay"
    ((attempt++))
  done
}

retry 5 10 curl -sf https://api.example.com/health

# Arrays
fruits=("apple" "banana" "cherry")
echo "${fruits[0]}"           # first element
echo "${fruits[@]}"           # all elements
echo "${#fruits[@]}"          # count
fruits+=("date")              # append

for fruit in "${fruits[@]}"; do
  echo "$fruit"
done

# Associative arrays (bash 4+)
declare -A config
config[host]="localhost"
config[port]="5432"
echo "${config[host]}"
for key in "${!config[@]}"; do
  echo "$key = ${config[$key]}"
done

# String manipulation
str="Hello, World!"
echo "${str:7:5}"             # substring: World
echo "${str/World/Bash}"      # replace first
echo "${str//l/L}"            # replace all
echo "${str#Hello, }"         # strip prefix
echo "${str%!}"               # strip suffix
```

### Here Documents

```bash
# Write multi-line content
cat > /etc/nginx/conf.d/myapp.conf << 'EOF'
server {
    listen 80;
    server_name app.example.com;
    location / {
        proxy_pass http://localhost:3000;
    }
}
EOF

# Pass to a command
ssh user@host << 'ENDSSH'
  sudo systemctl restart nginx
  sudo systemctl status nginx
ENDSSH
```

---

## Advanced Scenarios

### Scenario 1 — Watch a log for an error and alert

```bash
tail -F /var/log/app/error.log | while read -r line; do
  if echo "$line" | grep -q "CRITICAL"; then
    echo "CRITICAL ERROR: $line" | mail -s "Alert" admin@example.com
  fi
done
```

### Scenario 2 — Rotate log files manually

```bash
LOG=/var/log/myapp/app.log
mv "$LOG" "${LOG}.$(date +%Y%m%d)"
gzip "${LOG}.$(date +%Y%m%d)"
kill -USR1 $(pgrep myapp)    # signal app to reopen log file
find /var/log/myapp -name "*.gz" -mtime +30 -delete  # cleanup old logs
```

### Scenario 3 — Check if a service is up, restart if not

```bash
#!/usr/bin/env bash
SERVICE="nginx"

if ! systemctl is-active --quiet "$SERVICE"; then
  echo "$(date): $SERVICE is down. Restarting..." >> /var/log/watchdog.log
  systemctl restart "$SERVICE"
  if systemctl is-active --quiet "$SERVICE"; then
    echo "$(date): $SERVICE restarted successfully" >> /var/log/watchdog.log
  else
    echo "$(date): FAILED to restart $SERVICE" >> /var/log/watchdog.log
  fi
fi
```

### Scenario 4 — Parallel execution with wait

```bash
pids=()

for region in us-east-1 eu-west-1 ap-southeast-1; do
  aws ec2 describe-instances --region "$region" > "instances-$region.json" &
  pids+=($!)
done

for pid in "${pids[@]}"; do
  wait "$pid" || echo "Job $pid failed"
done

echo "All regions queried"
```

### Scenario 5 — Safely source environment file

```bash
# Load .env without executing arbitrary code
if [[ -f .env ]]; then
  set -a
  # shellcheck disable=SC1091
  source <(grep -E '^[A-Z_]+=.*' .env | sed 's/#.*//')
  set +a
fi
```

### Scenario 6 — Find and kill a process on a port

```bash
kill_port() {
  local port="$1"
  local pid
  pid=$(lsof -ti :"$port")
  if [[ -n "$pid" ]]; then
    echo "Killing process $pid on port $port"
    kill -9 "$pid"
  else
    echo "No process found on port $port"
  fi
}

kill_port 8080
```

### Scenario 7 — Disk usage alert script

```bash
#!/usr/bin/env bash
THRESHOLD=80

df -h | awk 'NR>1 {gsub(/%/,""); if ($5 > '"$THRESHOLD"') print $0}' | \
while read -r line; do
  mount=$(echo "$line" | awk '{print $6}')
  usage=$(echo "$line" | awk '{print $5}')
  echo "WARNING: $mount is at ${usage}% usage"
done
```

---

## Quick Reference Card

| Task                          | Command                                      |
|-------------------------------|----------------------------------------------|
| Find large files              | `find / -size +100M -type f 2>/dev/null`     |
| Who is logged in              | `w`                                          |
| Listening ports               | `ss -tulnp`                                  |
| Follow service logs           | `journalctl -u service -f`                   |
| Restart service on crash      | `Restart=on-failure` in systemd unit         |
| Check disk usage              | `df -h && du -sh /*`                         |
| Kill process on port          | `kill -9 $(lsof -ti :8080)`                  |
| Run script every 5 min        | `*/5 * * * * /path/script.sh` in crontab     |
| Safe bash script header       | `set -euo pipefail`                          |
| Recursive replace in files    | `grep -rl "old" . \| xargs sed -i 's/old/new/g'` |

---

> `set -euo pipefail` at the top of every script. It has saved more production systems than any other single line.
