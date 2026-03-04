# SSH Ed25519 Key Generation Guide

## PuTTY + Linux Server

> Why Ed25519?
> - Not affected by NIST P-521 nonce bias vulnerability (PuTTY 0.80 and earlier)
> - Designed by Daniel Bernstein - no NSA/NIST curve involvement
> - Faster and smaller than RSA/ECDSA
> - Recommended for all new SSH keys

---

## Requirements

- Linux server, Ubuntu is common
- PuTTY or other app you use for ssh.
- PuTTYgen (have it already open)
- Lynis, AIDE.
- Diceware passwords (see EFF website)
- VeraCrypt to store server keys, with diceware passwords.
- Powershell
- Optional: Unbound DNS.

```
# If you dont have it yet:
sudo install aide lynis rkhunter
```

## Prepare

Generating new keys is a sensitive practice. But we don't change it very often, so we can take our sweet time.

First, make sure both server and your PC are not compromised. If it is, then it will be useless to generate new keys.

What you could do on your PC:

- Close all running apps (in taskmanager)
- In PC (windows) firewall: deny `outbound` port 80, 443. Or whitelist port 22 only.
- Optional run `Wireshark` in background for monitoring, make it capture all traffic. 
- Optional run `Unbound` to capture DNS traffic.

On server:

- Make sure you ran any `apt updates` with https, instead of http.
- Shutdown logwatch: `sudo systemctl stop cron`
- Make sure you ran a `Lynis` and `rkhunter` scan before.
- If you can afford it: close all ports, only allow port 22 temporarily.
- Be prepared to clean all your logs (see end of readme for a bash script to clean logs)
- Have AIDE installed.

## Overview

```
Step 1 - Generate Ed25519 key pair on server
Step 2 - Add public key to authorized_keys
Step 3 - Import private key into PuTTY
Step 4 - Test new key login
Step 5 - Revoke old key, shred it.
Step 6 - Store private key in VeraCrypt
```

---


## Step 1 - Generate Ed25519 Key Pair on Server

SSH into your server using your existing key first.

### Before key generation - establish baseline
```
sudo aide --check --config=/etc/aide/aide.conf
```
### Generate Ed25519 key pair
```
ssh-keygen -t ed25519 -C "your_label_here" -f ~/.ssh/id_ed25519

# You will be prompted for a passphrase
# Use a strong Diceware passphrase
# This encrypts the private key at rest
```

What this creates:
```
~/.ssh/id_ed25519        - private key (keep secret)
~/.ssh/id_ed25519.pub    - public key (safe to share)
```

Verify the key was created:
```
ls -la ~/.ssh/
cat ~/.ssh/id_ed25519.pub
```

Output should look like:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... your_label_here
```

---

## Step 2 - Add public key to authorized_keys

```
# Append public key to authorized_keys
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# Set correct permissions (critical)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Verify it was added
cat ~/.ssh/authorized_keys
```

---

## Step 3 - Copy private key to windows (securely)

Never use `cat` for private keys - it exposes key material in terminal scrollback, logwatch, bash history, syslogs, ISP snapshots, PuTTY session logs, and screen capture malware.

Secure method - variable or clipboard:

```
# Read into variable - nothing displayed yet
IFS= read -r -d '' PRIVKEY < ~/.ssh/id_ed25519

# Display once for copying (CTRL+SHIFT+C)
printf '%s\n' "$PRIVKEY"

# Immediately wipe from shell memory
unset PRIVKEY

# Clear terminal scrollback
clear && printf '\033[3J'

# Clear  history of this operation
> .bash_history
history -c
```

Even cleaner - pipe directly to clipboard (never displays, but requires xclip):

```
# Read, copy to clipboard, wipe - key never rendered on screen
IFS= read -r -d '' PRIVKEY < ~/.ssh/id_ed25519
printf '%s\n' "$PRIVKEY" | xclip -selection clipboard
unset PRIVKEY
clear && printf '\033[3J'
> .bash_history
history -c
```

Then paste directly into `PuTTYgen` - key never rendered in terminal at all.

Why variable is safer than cat:

```
cat:
  ├── Displays to stdout        - visible on screen
  ├── Captured by terminal logs - PuTTY logging
  ├── Lives in scrollback       - persists after session
  └── Readable by screen capture malware

Variable:
  ├── Stored in memory only     - not displayed
  ├── No stdout output          - nothing to capture
  ├── unset wipes it            - removed from shell memory
  └── Never exported            - invisible to child processes
```

After pasting into PuTTYgen - clear Windows clipboard:

```
# PowerShell - wipe clipboard immediately after paste
Set-Clipboard -Value " "
```

On Windows:

1. Paste clipboard contents into PuTTYgen import dialog
2. Immediately clear clipboard with above PowerShell command
3. Save as `id_ed25519` (no extension) inside a `VeraCrypt` container

---

## Step 4 - Convert private key for PuTTY (PuTTYgen)

PuTTY uses `.ppk` format - you need to convert:

1. Open PuTTYgen (comes with PuTTY)
2. Click Conversions - Import key
3. Navigate to a VeraCrypt container
4. Select `id_ed25519`
5. Enter your Diceware passphrase when prompted
6. Verify key type shows EdDSA / Ed25519
7. Add a Key comment (e.g. server name)
8. Click Save private key
9. Save as `id_ed25519.ppk` in your VeraCrypt container
10. Enter a strong passphrase for the `.ppk` file

> Tip: Use a different Diceware passphrase for the `.ppk` than the original key - compartmentalization.

---

## Step 5 - Configure PuTTY to use new key

1. Open PuTTY
2. Load your saved session or enter server IP
3. Navigate to: Connection → SSH → Auth → Credentials
4. Under Private key file for authentication → Browse
5. Select your `id_ed25519.ppk` from VeraCrypt container
6. Go back to Session
7. Save the session

---

## Step 6 - Test new key login

Critical: Do NOT close your existing session yet.

Open a new PuTTY window and test login with new key:

```
If login succeeds:  proceed to revoke old key
If login fails: debug before touching old key
```

Common issues if login fails:
```
# Check permissions on server
ls -la ~/.ssh/
# authorized_keys must be 600
# .ssh directory must be 700

# Check SSH daemon config
sudo nano /etc/ssh/sshd_config
# Verify: PubkeyAuthentication yes

# Check auth log for errors
sudo tail -f /var/log/auth.log
```

---

## Step 7 - Revoke old key(s) and shred

Once new key login is confirmed working - revoke and shred everything.

Never just delete key files, always `shred`.

Server side - remove from authorized_keys:
```
# Open authorized_keys
nano ~/.ssh/authorized_keys

# Delete the line containing your OLD public key
# Keep only the new Ed25519 key line
# Save and exit
```

Server side - shred all old key files:
```
# Show
ls -la ~/.ssh/

# Shred old RSA keys if present
shred -u ~/.ssh/id_rsa 2>/dev/null || true
shred -u ~/.ssh/id_rsa.pub 2>/dev/null || true

# Shred old ECDSA/P-521 keys if present
shred -u ~/.ssh/id_ecdsa 2>/dev/null || true
shred -u ~/.ssh/id_ecdsa.pub 2>/dev/null || true

# Shred old Ed25519 if rotating an existing one
shred -u ~/.ssh/id_ed25519_old 2>/dev/null || true
shred -u ~/.ssh/id_ed25519_old.pub 2>/dev/null || true

# Verify only new key remains
cat ~/.ssh/authorized_keys

# Verify old files are gone
ls -la ~/.ssh/
```

Also shred new private key from server - it should only live in VeraCrypt:

```
# Private key is now in VeraCrypt on Windows
# No reason to keep it on server
shred -u ~/.ssh/id_ed25519
```

Windows side - shred old .ppk from VeraCrypt. I do not recommend using Eraser, or shredding .exe's these are often compromised. Write your own shredder powershell file for Windows.

```powershell
# Mount VeraCrypt container_serverkeys.vc
# Use your custom shredder on:
#   old_id_rsa.ppk
#   old_id_ecdsa.ppk
#   any old raw private key files

# Wipe clipboard
Set-Clipboard -Value " "

# Unmount container immediately after
```

Revocation checklist:

```
Server side:
  [ ] Old key removed from authorized_keys
  [ ] Old private key shredded
  [ ] Old public key shredded
  [ ] New private key shredded from server
  [ ] ls ~/.ssh/ verified clean
  [ ] Only new Ed25519 public key remains

Windows side:
  [ ] Old .ppk shredded from VeraCrypt
  [ ] Old raw private key shredded
  [ ] PuTTY session updated to new .ppk
  [ ] Clipboard wiped after all operations
  [ ] VeraCrypt container unmounted

Cold storage:
  [ ] Mount cold storage container
  [ ] Replace old keys with new ones
  [ ] Shred old key files in cold storage
  [ ] Verify new keys copied correctly
  [ ] Unmount immediately
  [ ] Note rotation date
```

## Finish

Check AIDE if any files were changed:

```
sudo aide --check --config=/etc/aide/aide.conf
```

Then clean all logs:

```
#!/bin/bash
# deepclean.sh

set -e
shopt -s nullglob

# Fail2ban
# --------
# If you have fail2ban, might want to make extra backup. (do the same for snort, if applicable)
# sudo fail2ban-client status sshd > /var/log/fail2ban_review_sshd_$(date +%Y%m%d).log
# sudo fail2ban-client banned >> /var/log/fail2ban_review_banned_$(date +%Y%m%d).log

# Encrypted backup
# -----------------
# Uncomment if you want to encrypt all log files, and store it as GPG file:
# === Archive and encrypt logs before clearing ===
# DATE=$(date +%m)
# ARCHIVE=~/logs_backup_$DATE.tar.gz
# ENCRYPTED=$ARCHIVE.gpg
# sudo tar -czf $ARCHIVE /var/log 2>/dev/null
# gpg --recipient yourkey@email.com --encrypt $ARCHIVE
# shred -u $ARCHIVE
# echo "=== Encrypted log archive saved: $ENCRYPTED ==="

echo "=== Disk usage before cleanup ==="
df -h /

echo "=== Cleaning package cache and autoremove ==="
sudo apt clean
sudo apt autoremove -y

echo "=== Cleaning journal logs  ==="
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s

echo "=== Cleaning old .gz archives ==="
# sudo find /var/log -type f -name "*.gz" -mtime +60 -delete
sudo find /var/log -name "*.gz" -delete
sudo find /var/log -name "*.1" -delete
sudo find /var/log -name "*.old" -delete

echo "=== Cleaning tmp files  ==="
sudo find /tmp -type f -atime +7 -delete
sudo find /var/tmp -type f -atime +7 -delete

sudo truncate -s 0  /var/log/lynis.log
sudo truncate -s 0  /var/log/mail.log

echo "=== Deep clean... ==="

# Shred sensitive files.
shred -u /var/log/auth.log
touch /var/log/auth.log

sudo truncate -s 0 /var/log/syslog
sudo truncate -s 0 /var/log/kern.log
sudo truncate -s 0 /var/log/dpkg.log         # Debian/Ubuntu
sudo truncate -s 0 /var/log/apt/history.log
# sudo truncate -s 0 /var/log/messages       # RHEL/CentOS
# sudo truncate -s 0 /var/log/secure         # RHEL/CentOS
sudo rm -rf /var/cache/logwatch/*

echo "=== Disk usage after cleanup ==="
df -h /

RED='\033[0;31m'
NC='\033[0m' # No Color
echo -e "${RED}=====================================================================${NC}"
echo -e "${RED}    REMEMBER to manually type: history -c${NC}"
echo -e "${RED}    DO THIS NOW in every open terminal!${NC}"
echo -e "${RED}    This prevents from sensitive data living in memory               ${NC}"
echo -e "${RED}    To prevent this: use a LEADING SPACE before a sensitive command  ${NC}"
echo -e "${RED}=====================================================================${NC}"

# Shred it first.
shred -u ~/.bash_history
touch ~/.bash_history

# Cleaning BASH history and memory.
truncate -s 0 ~/.bash_history
# Clearing less
truncate -s 0 ~/.lesshst

# Clearing MySQL
truncate -s 0 ~/.mysql_history

# Clearing Wget
truncate -s 0 ~/.wget-hsts
truncate -s 0 ~/wget-log

for f in ~/.bash_history-*.tmp; do [ -f "$f" ] && > "$f"; done
for f in ~/wget-log.*; do [ -f "$f" ] && > "$f"; done

sudo systemctl stop auditd
sleep 1
sudo find /var/log/audit/ -type f -exec chmod 640 {} \;
sudo find /var/log/audit/ -type f -exec truncate -s 0 {} \;
sudo systemctl start auditd
sleep 1
sudo systemctl restart rsyslog
# Finish
history -c
truncate -s 0 ~/.bash_history
```

Then restart cron and `logwatch`

```
sudo systemctl start cron
```

## Step 8 - Harden SSH Config

While you're at it, verify `/etc/ssh/sshd_config`:

```
sudo nano /etc/ssh/sshd_config
```

Recommended settings:
```
# Disable password authentication entirely
PasswordAuthentication no
ChallengeResponseAuthentication no

# Only allow public key auth
PubkeyAuthentication yes

# Disable root login
PermitRootLogin no

# Only allow Ed25519
HostKeyAlgorithms ssh-ed25519
PubkeyAcceptedKeyTypes ssh-ed25519

# Limit auth attempts
MaxAuthTries 3

# Disconnect idle sessions
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable unused features
X11Forwarding no
AllowTcpForwarding no
```

Apply changes:
```
# Test config before restarting
sudo sshd -t

# If no errors restart SSH
sudo systemctl restart sshd

# Verify still running
sudo systemctl status sshd
```

> Warning: Always test in a new session before closing existing one.

---

## Step 9 - Store Keys in VeraCrypt

Mount your `container_serverkeys.vc` and store:

```
container_serverkeys.vc/
  ├── id_ed25519           - raw private key (OpenSSH format)
  ├── id_ed25519.pub       - public key
  ├── id_ed25519.ppk       - PuTTY format private key
  └── authorized_keys.bak  - backup of server authorized_keys
```

Unmount container immediately after.

---

## Step 10 - Update Cold Storage Backups

```
1. Mount cold storage VeraCrypt container
2. Replace old key files with new ones
3. Verify files copied correctly
4. Unmount immediately
5. Note rotation date
```

---

## Why Not Generate Keys on Windows?

```
Server generation advantages:
  ├── /dev/urandom - high quality entropy source
  ├── No Windows entropy concerns
  ├── Key never touches Windows filesystem unencrypted
  └── Immediately goes into VeraCrypt on copy

Windows generation risks:
  ├── Lower entropy quality
  ├── Key exists on Windows filesystem briefly
  ├── Pagefile could capture key material
  └── Even with encryptpagingfile, brief exposure window
```

---

*Guide written for PuTTY 0.83 on Windows with Ubuntu/Debian server.*
*Always verify PuTTY binary hash against official chiark.greenend.org.uk release.*
