# Linux & Bash Cheatsheet

Linux is the backbone of modern infrastructure. Mastering the command line — from navigating the filesystem and managing processes to writing robust Bash scripts — is an essential skill for every DevOps engineer, SRE, and developer. This cheatsheet covers everyday commands and scripting patterns for Debian/Ubuntu-based and RHEL/CentOS-based distributions.

---

## Navigation & File Operations

### `pwd` — Print Working Directory
Outputs the absolute path of the current directory.
```bash
pwd
# /home/alice/projects
```

### `ls` — List Directory Contents
| Flag | Description |
|------|-------------|
| `-l` | Long format (permissions, owner, size, date) |
| `-a` | Include hidden files (starting with `.`) |
| `-h` | Human-readable file sizes (use with `-l`) |
| `-R` | Recursive listing |
| `-t` | Sort by modification time, newest first |
| `--color=auto` | Colorise output |

```bash
ls -lah /var/log
ls -lt ~/downloads | head -10    # 10 most recently modified files
```

### `cd` — Change Directory
```bash
cd /etc/nginx          # absolute path
cd ..                  # one level up
cd -                   # previous directory
cd ~                   # home directory
```

### `cp` — Copy Files and Directories
| Flag | Description |
|------|-------------|
| `-r` | Recursive (required for directories) |
| `-p` | Preserve timestamps, ownership, and permissions |
| `-u` | Only copy if source is newer than destination |
| `-v` | Verbose |

```bash
cp -rp /var/www/html /backup/html-$(date +%F)
cp -u config.yml /etc/myapp/config.yml
```

### `mv` — Move or Rename
```bash
mv old-name.txt new-name.txt          # rename
mv *.log /var/log/archive/            # move multiple files
```

### `rm` — Remove Files and Directories
| Flag | Description |
|------|-------------|
| `-r` | Recursive (for directories) |
| `-f` | Force — suppress prompts and errors |
| `-i` | Interactive — prompt before each removal |
| `-v` | Verbose |

```bash
rm -rf ./build/                        # remove directory tree
rm -i *.tmp                            # prompt for each .tmp file
```

### `mkdir` / `rmdir`
```bash
mkdir -p /opt/myapp/{bin,conf,logs}   # create nested dirs in one shot
rmdir empty-dir                        # remove only if empty
```

### `find` — Search for Files
| Flag | Description |
|------|-------------|
| `-name` | Match filename pattern |
| `-type f/d/l` | File, directory, or symlink |
| `-mtime -N` | Modified in last N days |
| `-size +Nc` | Larger than N bytes (`k`, `M`, `G` suffixes) |
| `-exec` | Execute a command on each result |

```bash
find /var/log -name "*.log" -mtime -7              # logs from the last week
find /home -type f -size +100M                     # files larger than 100 MB
find . -name "*.py" -exec grep -l "import os" {} \;  # Python files using os
find /tmp -type f -mtime +30 -delete               # delete files older than 30 days
```

### `ln` — Create Links
```bash
ln -s /opt/myapp/bin/myapp /usr/local/bin/myapp    # symbolic link
ln file.txt hardlink.txt                            # hard link
```

### `tree` — Visual Directory Tree
```bash
tree -L 2 /etc/nginx       # show 2 levels deep
tree -a -I ".git"          # include hidden, exclude .git
```

---

## File Permissions & Ownership

### `chmod` — Change Permissions
Permissions use octal notation (`rwx` = 4+2+1) or symbolic notation.

| Octal | Symbolic | Meaning |
|-------|----------|---------|
| `755` | `rwxr-xr-x` | Owner: rwx, Group/Other: r-x |
| `644` | `rw-r--r--` | Owner: rw, Group/Other: r |
| `600` | `rw-------` | Owner: rw only (good for SSH keys) |
| `777` | `rwxrwxrwx` | Full access for everyone (avoid in production) |

```bash
chmod 755 deploy.sh                  # make script executable
chmod -R 644 /var/www/html/*.html    # set html files read-only
chmod u+x,g-w,o-rwx script.sh       # symbolic: add exec for user, remove write for group
```

### `chown` — Change Ownership
```bash
chown alice:developers /opt/myapp    # set user and group
chown -R www-data:www-data /var/www  # recursive
chown --reference=ref.txt target.txt # copy ownership from another file
```

### `chgrp` — Change Group Ownership
```bash
chgrp docker /var/run/docker.sock
```

### `umask` — Default Permission Mask
```bash
umask           # show current mask (e.g. 0022 → new files get 644)
umask 027       # more restrictive: files 640, dirs 750
```

### Special Bits
```bash
chmod u+s /usr/bin/sudo    # setuid – run as file owner
chmod g+s /shared/dir      # setgid – new files inherit group
chmod +t /tmp              # sticky bit – only owner can delete their files
```

---

## Users & Groups

### `useradd` / `adduser`
```bash
useradd -m -s /bin/bash -G sudo,docker alice    # create user with home dir
adduser bob                                       # interactive, friendlier
```

### `usermod` — Modify a User
```bash
usermod -aG docker alice          # add alice to docker group (append, don't replace)
usermod -s /bin/zsh alice         # change login shell
usermod -L alice                  # lock account
usermod -U alice                  # unlock account
```

### `userdel` — Delete a User
```bash
userdel -r alice     # -r also removes home directory and mail spool
```

### `passwd` — Change Password
```bash
passwd alice           # change alice's password (as root)
passwd -e alice        # expire immediately (force change on next login)
passwd -l alice        # lock password
```

### `groupadd` / `groupmod` / `groupdel`
```bash
groupadd developers
groupmod -n eng developers     # rename group
groupdel eng
```

### `id` / `whoami` / `groups`
```bash
id alice                       # uid, gid, and supplementary groups
whoami                         # current effective user
groups                         # groups of the current user
```

### `su` / `sudo`
```bash
su - alice                      # switch to alice (login shell)
sudo -u alice command           # run single command as alice
sudo -i                         # interactive root shell via sudo
sudo !!                         # re-run last command with sudo
```

### `who` / `w` / `last`
```bash
who                  # users currently logged in
w                    # logged-in users and what they're doing
last -n 20           # last 20 login records
lastb                # failed login attempts
```

---

## Process Management

### `ps` — Process Snapshot
```bash
ps aux                          # all processes (BSD style)
ps aux --sort=-%mem | head -10  # top 10 by memory usage
ps -ef                          # full-format listing (POSIX style)
ps -u alice                     # processes owned by alice
```

### `top` / `htop`
```bash
top -b -n 1 -o %CPU | head -20  # batch mode, one snapshot, sorted by CPU
htop                             # interactive (install: apt install htop)
```

### `kill` / `pkill` / `killall`
| Signal | Number | Meaning |
|--------|--------|---------|
| `SIGTERM` | 15 | Graceful shutdown (default) |
| `SIGKILL` | 9 | Forceful kill (unblockable) |
| `SIGHUP` | 1 | Reload config (many daemons) |
| `SIGUSR1` | 10 | User-defined |

```bash
kill 1234                   # SIGTERM to PID 1234
kill -9 1234                # force kill
kill -HUP $(pidof nginx)    # reload nginx config
pkill -f "python manage.py" # kill by full command match
```

### `jobs` / `bg` / `fg` / `nohup`
```bash
sleep 300 &         # start in background; shows job number and PID
jobs -l             # list background jobs with PIDs
fg %1               # bring job 1 to foreground
bg %2               # resume stopped job 2 in background
nohup ./long-job.sh > output.log 2>&1 &   # immune to hangup signals
```

### `nice` / `renice`
```bash
nice -n 10 ./cpu-intensive.sh     # start with lower priority (+10 niceness)
renice -n 5 -p 1234               # change priority of running process
renice -n -5 -u alice             # raise priority for all of alice's processes
```

### `strace` / `lsof`
```bash
strace -p 1234 -e trace=network     # trace network syscalls of PID 1234
lsof -p 1234                        # files opened by process
lsof -i :8080                       # process listening on port 8080
```

---

## System Information & Resources

### `uname`
```bash
uname -a          # all info: kernel, hostname, architecture
uname -r          # kernel release only
```

### `hostnamectl`
```bash
hostnamectl                           # show hostname and OS details
hostnamectl set-hostname webserver01  # set hostname persistently
```

### `uptime`
```bash
uptime               # uptime, load averages (1, 5, 15 min)
uptime -p            # pretty format: "up 3 days, 4 hours"
```

### `free` — Memory Usage
```bash
free -h              # human-readable (MB/GB)
free -s 5            # refresh every 5 seconds
```

### `vmstat` — System Statistics
```bash
vmstat 2 5           # 5 samples every 2 seconds: CPU, memory, IO, swap
```

### `iostat` — Disk I/O Statistics
```bash
iostat -xz 1 5       # extended stats, hide empty devices, 5 samples
```

### `lscpu` / `nproc`
```bash
lscpu                 # CPU architecture details
nproc                 # number of processing units available
```

### `dmesg`
```bash
dmesg -T | tail -30   # kernel ring buffer with human-readable timestamps
dmesg | grep -i error
```

---

## Disk & Storage

### `df` — Disk Free Space
```bash
df -h                      # human-readable, all mounted filesystems
df -h /var                 # specific mount point
df -iH                     # inode usage
```

### `du` — Disk Usage
```bash
du -sh /var/log/*          # size of each item in /var/log
du -ah . | sort -rh | head -20  # top 20 largest files/dirs in current dir
```

### `lsblk` — List Block Devices
```bash
lsblk                      # tree view of all block devices
lsblk -f                   # also show filesystem type and UUID
```

### `fdisk` / `parted` — Partition Management
```bash
fdisk -l                          # list all partitions (as root)
parted /dev/sdb print             # show partition table
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary ext4 0% 100%
```

### `mkfs` — Create Filesystem
```bash
mkfs.ext4 /dev/sdb1
mkfs.xfs /dev/sdc1
```

### `mount` / `umount`
```bash
mount /dev/sdb1 /mnt/data              # mount device
mount -t nfs 10.0.0.5:/exports /mnt    # mount NFS share
umount /mnt/data                       # unmount
mount | grep sdb                       # list mounts matching sdb
```

### `/etc/fstab` — Persistent Mounts
```
# <device>               <mountpoint>  <type>  <options>           <dump> <pass>
UUID=abc-123             /data         ext4    defaults,noatime    0      2
10.0.0.5:/exports        /mnt/nfs      nfs     defaults,_netdev    0      0
```

### `lvs` / `vgs` / `pvs` — LVM
```bash
pvs                          # physical volumes
vgs                          # volume groups
lvs                          # logical volumes
lvcreate -L 20G -n data vg0  # create 20 GB LV
lvextend -L +10G /dev/vg0/data && resize2fs /dev/vg0/data  # grow LV + filesystem
```

---

## Networking Commands

### `ip` — Modern Network Configuration
```bash
ip addr show                          # all interfaces and addresses
ip addr show eth0                     # specific interface
ip link set eth0 up/down              # bring interface up or down
ip route show                         # routing table
ip route add default via 10.0.0.1    # add default gateway
ip neigh show                         # ARP table
```

### `ss` — Socket Statistics (modern `netstat`)
```bash
ss -tulpn                    # TCP+UDP listening sockets with process info
ss -s                        # summary statistics
ss -tnp state established    # established TCP connections with process
```

### `ping` / `traceroute` / `mtr`
```bash
ping -c 4 8.8.8.8
traceroute -n 8.8.8.8        # -n skip DNS lookups
mtr --report 8.8.8.8         # combines ping + traceroute
```

### `hostname` / `hostnamectl`
```bash
hostname -I              # all local IP addresses
hostnamectl status
```

---

## systemd & Services

### `systemctl` — Service Management
```bash
systemctl status nginx                # service status
systemctl start nginx                 # start
systemctl stop nginx                  # stop
systemctl restart nginx               # stop then start
systemctl reload nginx                # reload config without stopping
systemctl enable nginx                # start on boot
systemctl disable nginx               # don't start on boot
systemctl is-active nginx             # active|inactive|failed
systemctl is-enabled nginx            # enabled|disabled
```

### `journalctl` — View Logs
| Flag | Description |
|------|-------------|
| `-u` | Filter by service unit |
| `-f` | Follow (like `tail -f`) |
| `-n N` | Show last N lines |
| `--since` | Start time (e.g. `"2024-01-01"`, `"1 hour ago"`) |
| `--until` | End time |
| `-p err` | Filter by priority: `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug` |
| `-b` | Logs from current boot only |
| `--no-pager` | Don't paginate (good for scripts) |

```bash
journalctl -u nginx -f                         # follow nginx logs
journalctl -u nginx --since "2024-08-01" --until "2024-08-02"
journalctl -p err -b                           # errors since last boot
journalctl --disk-usage                        # how much space journal uses
journalctl --vacuum-size=500M                  # trim journal to 500 MB
```

### Writing a Unit File
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --port 8080
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment="ENV=production"
EnvironmentFile=/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload          # reload unit files after changes
systemctl enable --now myapp     # enable and start in one command
```

---

## Cron & Scheduling

### Crontab Syntax
```
# ┌──────── minute (0-59)
# │ ┌────── hour (0-23)
# │ │ ┌──── day of month (1-31)
# │ │ │ ┌── month (1-12)
# │ │ │ │ ┌ day of week (0-7, 0=Sun)
# │ │ │ │ │
# * * * * *  command
```

```bash
crontab -e                              # edit current user's crontab
crontab -l                              # list current user's crontab
crontab -u alice -l                     # list alice's crontab (as root)
crontab -r                              # remove (DELETE) crontab — be careful!
```

### Common Cron Examples
```cron
# Every day at 02:30
30 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1

# Every 15 minutes
*/15 * * * * /usr/local/bin/health-check.sh

# Every weekday at 08:00
0 8 * * 1-5 /opt/report.sh

# First day of each month at midnight
0 0 1 * * /opt/monthly-cleanup.sh

# Every Sunday at 04:00
0 4 * * 0 /opt/db-vacuum.sh
```

### `at` — One-Off Scheduling
```bash
echo "/opt/deploy.sh" | at 23:00              # run once at 11 PM tonight
echo "/opt/reboot.sh" | at now + 5 minutes    # in 5 minutes
atq                                            # list pending jobs
atrm 3                                         # remove job 3
```

### `systemd Timers` (modern cron)
```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true              # run missed executions on startup

[Install]
WantedBy=timers.target
```
```bash
systemctl enable --now backup.timer
systemctl list-timers --all
```

---

## Archives & Compression

### `tar` — Tape Archive (most common)
| Flag | Description |
|------|-------------|
| `-c` | Create archive |
| `-x` | Extract archive |
| `-t` | List contents |
| `-f` | Specify archive filename |
| `-z` | Compress with gzip (`.tar.gz`) |
| `-j` | Compress with bzip2 (`.tar.bz2`) |
| `-J` | Compress with xz (`.tar.xz`) |
| `-v` | Verbose |
| `-C` | Change to directory before extract |
| `--exclude` | Exclude pattern |

```bash
tar -czf backup.tar.gz /etc/nginx        # create gzip archive
tar -cJf backup.tar.xz /var/www          # create xz archive (better compression)
tar -tzf backup.tar.gz                   # list contents without extracting
tar -xzf backup.tar.gz -C /restore/     # extract to specific directory
tar -czf app.tar.gz --exclude="*.log" --exclude=".git" .
```

### `gzip` / `gunzip` / `zcat`
```bash
gzip largefile.log              # compress in place → largefile.log.gz
gunzip largefile.log.gz         # decompress
zcat largefile.log.gz | grep ERROR   # read compressed file without extracting
```

### `zip` / `unzip`
```bash
zip -r archive.zip ./mydir      # recursive zip
zip -e secret.zip sensitive.txt # password-protected
unzip archive.zip -d /dest/     # extract to destination
unzip -l archive.zip            # list contents
```

---

## Text Processing

### `grep` — Search Text
| Flag | Description |
|------|-------------|
| `-i` | Case-insensitive |
| `-r` / `-R` | Recursive |
| `-l` | Print only filenames |
| `-n` | Show line numbers |
| `-v` | Invert match |
| `-c` | Count matching lines |
| `-E` | Extended regex (same as `egrep`) |
| `-A N` / `-B N` / `-C N` | N lines after / before / both |
| `--color` | Highlight matches |

```bash
grep -rn "TODO" ./src/                         # find TODOs with line numbers
grep -E "ERROR|WARN" app.log                   # multiple patterns
grep -v "DEBUG" app.log | grep "2024-08"       # pipe filters
grep -c "FAIL" test-results.log                # count failures
grep -A 5 "NullPointerException" error.log     # 5 lines of context after match
```

### `awk` — Field Processing
`awk` processes files line by line, splitting each into fields (`$1`, `$2`, …, `$NF`).

```bash
awk '{print $1, $3}' access.log                    # print columns 1 and 3
awk -F: '{print $1, $3}' /etc/passwd               # custom delimiter (:)
awk '$9 == 500 {print $0}' access.log              # lines where field 9 = 500
awk '{sum += $5} END {print "Total:", sum}' data   # sum column 5
awk 'NR % 2 == 0' file                             # print even-numbered lines
awk '/start/,/end/ {print}' file                   # print between patterns
awk '{print NR": "$0}' file                        # prefix each line with number
```

### `sed` — Stream Editor
```bash
sed 's/foo/bar/'         file        # replace first occurrence per line
sed 's/foo/bar/g'        file        # replace all occurrences
sed -i 's/old/new/g'     file.txt    # in-place (edit the file)
sed -i.bak 's/old/new/g' file.txt    # in-place with .bak backup
sed -n '5,10p'           file        # print lines 5–10
sed '1d'                 file        # delete first line
sed '/^#/d'              file        # delete comment lines
sed 's/[[:space:]]*$//'  file        # strip trailing whitespace
```

### `sort` — Sort Lines
```bash
sort file.txt                        # alphabetical
sort -r file.txt                     # reverse
sort -n numbers.txt                  # numeric sort
sort -k 2 -t ',' data.csv            # sort by 2nd column, comma-delimited
sort -u file.txt                     # sort and remove duplicates
sort -rh sizes.txt                   # human-readable numeric reverse (e.g. for `du -h` output)
```

### `uniq` — Filter Duplicate Lines
`uniq` only removes **adjacent** duplicates — always sort first.

```bash
sort access.log | uniq               # remove duplicates
sort access.log | uniq -c            # prefix with count
sort access.log | uniq -cd           # show only duplicates with count
sort access.log | uniq -u            # show only unique lines
```

### `cut` / `paste` / `tr`
```bash
cut -d: -f1,3 /etc/passwd                  # extract fields 1 and 3
cut -c1-80 long-lines.txt                  # first 80 characters
paste file1.txt file2.txt                  # merge files side-by-side
tr 'a-z' 'A-Z'              < file.txt     # lowercase to uppercase
tr -d '\r'                  < file.txt     # remove carriage returns (Windows line endings)
tr -s ' '                   < file.txt     # squeeze consecutive spaces into one
```

### `wc` — Word/Line/Character Count
```bash
wc -l file.txt          # line count
wc -w file.txt          # word count
wc -c file.txt          # byte count
cat *.log | wc -l       # total lines across all log files
```

---

## Bash Scripting

### Variables
```bash
#!/usr/bin/env bash
name="Alice"
count=42
greeting="Hello, ${name}!"          # braces prevent ambiguity

readonly MAX_RETRIES=5               # constant
export PATH="/opt/myapp/bin:$PATH"   # export to child processes

# Command substitution
today=$(date +%F)
file_count=$(find . -type f | wc -l)

# Arithmetic
result=$(( count * 2 + 1 ))
(( count++ ))
```

### Conditionals
```bash
if [[ -f "/etc/nginx/nginx.conf" ]]; then
  echo "nginx is installed"
elif [[ -d "/etc/apache2" ]]; then
  echo "apache is installed"
else
  echo "no web server found"
fi

# Common test operators
# -f  file exists and is regular file
# -d  directory exists
# -z  string is empty
# -n  string is non-empty
# -e  path exists
# ==  string equality
# !=  string inequality
# -eq / -ne / -lt / -le / -gt / -ge  numeric comparisons
# &&  logical AND
# ||  logical OR

[[ "$USER" == "root" ]] && echo "running as root" || echo "not root"
```

### Loops
```bash
# for loop over a list
for server in web01 web02 web03; do
  echo "Deploying to $server"
  ssh "$server" 'sudo systemctl restart myapp'
done

# for loop with range
for i in {1..5}; do
  echo "Attempt $i"
done

# C-style for loop
for (( i=0; i<10; i++ )); do
  echo "$i"
done

# while loop
while read -r line; do
  echo "Processing: $line"
done < servers.txt

# loop over files
for file in /var/log/*.log; do
  gzip "$file"
done
```

### Functions
```bash
log() {
  local level="$1"
  local message="$2"
  echo "[$(date +%F\ %T)] [$level] $message"
}

retry() {
  local n=0
  local max=3
  local cmd=("$@")
  until "${cmd[@]}"; do
    (( n++ ))
    if (( n >= max )); then
      log "ERROR" "Command failed after $max attempts: ${cmd[*]}"
      return 1
    fi
    log "WARN" "Attempt $n failed. Retrying in 5s..."
    sleep 5
  done
}

# Call functions
log "INFO" "Starting deployment"
retry curl -sf https://api.example.com/health
```

### Argument Parsing
```bash
#!/usr/bin/env bash
# Usage: ./deploy.sh -e production -v 1.2.3 [-d]

usage() {
  cat <<EOF
Usage: $0 -e <environment> -v <version> [-d]
  -e  Target environment (staging|production)
  -v  Version to deploy
  -d  Dry-run mode
  -h  Show this help
EOF
  exit 1
}

DRY_RUN=false
while getopts ":e:v:dh" opt; do
  case $opt in
    e) ENV="$OPTARG" ;;
    v) VERSION="$OPTARG" ;;
    d) DRY_RUN=true ;;
    h) usage ;;
    :) echo "Option -$OPTARG requires an argument." >&2; exit 1 ;;
    \?) echo "Invalid option: -$OPTARG" >&2; exit 1 ;;
  esac
done

[[ -z "$ENV" || -z "$VERSION" ]] && usage
```

### Error Handling
```bash
#!/usr/bin/env bash
set -euo pipefail
# -e  exit immediately on error
# -u  treat unset variables as errors
# -o pipefail  pipeline fails if any command fails

trap 'echo "ERROR: Script failed at line $LINENO" >&2' ERR
trap 'cleanup' EXIT

cleanup() {
  echo "Cleaning up temporary files..."
  rm -f /run/myapp/deploy.lock
}

# Safely check before acting
if ! command -v docker &>/dev/null; then
  echo "docker is not installed" >&2
  exit 1
fi

# Capture stderr separately
output=$(some-command 2>error.log) || {
  echo "Command failed. See error.log" >&2
  exit 1
}
```

### Here Documents & Here Strings
```bash
# Heredoc – pass multi-line input to a command
cat <<EOF > /etc/myapp/config.yml
host: localhost
port: 8080
debug: false
EOF

# Here string – single-line stdin
grep "error" <<< "$log_output"
base64 <<< "Hello, World!"
```

---

## Common Patterns & Tips

1. **Always quote variables** to prevent word splitting and glob expansion: use `"$var"` not `$var`, and `"${array[@]}"` not `${array[@]}`.

2. **Use `set -euo pipefail`** at the top of every non-trivial script. It transforms silent failures into loud, traceable ones.

3. **Prefer `[[` over `[`** in Bash — the double-bracket version is safer, supports regex matching (`=~`), and doesn't require quoting variables as rigorously.

4. **Use `rsync` for large or incremental file copies** instead of `cp`:
   ```bash
   rsync -avz --progress --delete /source/ /dest/
   rsync -avz -e "ssh -p 2222" /local/ user@host:/remote/
   ```

5. **Use `tee` to log and display simultaneously:**
   ```bash
   ./deploy.sh 2>&1 | tee deploy-$(date +%F).log
   ```

6. **Inspect what a command does before running it** with `echo` or `--dry-run`:
   ```bash
   rsync -avz --dry-run /src/ /dst/
   echo rm -rf ./old-build   # preview the command first
   ```

7. **Use process substitution for diff without temp files:**
   ```bash
   diff <(ssh server1 cat /etc/nginx/nginx.conf) <(ssh server2 cat /etc/nginx/nginx.conf)
   ```

8. **Chain commands safely using `&&` and `||`:**
   ```bash
   make build && make test && make deploy || echo "Pipeline failed"
   ```

9. **Profile slow scripts** with `bash -x` or `PS4='+ $LINENO: '` to trace execution line by line.

10. **Use `shellcheck`** (static analysis for shell scripts) in your CI pipeline to catch common bugs before they reach production:
    ```bash
    shellcheck deploy.sh
    ```
