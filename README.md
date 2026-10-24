# htb-odori-writeup
Penetration testing and digital forensics lab
# Hack The Box — Odori

## Overview

This repository documents a hands-on penetration testing lab focused on network enumeration, Windows disk forensics, credential analysis, and Linux privilege escalation.

**Environment:** Kali Linux, Windows disk image, Linux server.

**Tools:** Nmap, SMBClient, John the Ripper, Dislocker, chntpw, Mimikatz, SSH/SFTP, pspy, Python.

## 1. Network Enumeration

### Nmap scan

```bash
nmap -sC -sV --min-rate=1000 TARGET_IP
```

- `-sC` — runs default Nmap scripts.
- `-sV` — detects service versions.
- `--min-rate=1000` — sets the minimum probe sending rate.

**Findings:** SSH (22), NetBIOS/SMB (139, 445) were identified as exposed services.

## 2. SMB Enumeration

### List available shares

```bash
smbclient -L //TARGET_IP
```

Lists SMB shares available on the target server.

### Connect to the backup share

```bash
smbclient //TARGET_IP/backup
```

Useful commands inside the SMB client:

```text
ls
get info.txt
```

`ls` lists remote files, while `get` downloads a selected file for local analysis.

**Finding:** The backup share contained an information file that provided an initial lead for further investigation.

## 3. Windows Disk Forensics

### Extract BitLocker metadata

```bash
bitlocker2john -i file02.vmdk | tee hash
```

Extracts BitLocker-related password hashes from the disk image for offline analysis.

### Password audit

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash --format=bitlocker
```

Uses John the Ripper and a wordlist to test candidate passwords against the extracted hashes.

### Unlock and mount the volume

```bash
sudo dislocker -r -V /dev/nbd0p3 -u -- /mnt/bitlocker
sudo mount -o loop,ro /mnt/bitlocker/dislocker-file /mnt/vmdk
```

Dislocker provides access to the BitLocker volume. The second command mounts the decrypted filesystem in read-only mode.

### Locate Windows registry hives

```bash
sudo ls -lh /mnt/vmdk/Windows/System32/config/SAM
sudo ls -lh /mnt/vmdk/Windows/System32/config/SECURITY
sudo ls -lh /mnt/vmdk/Windows/System32/config/SYSTEM
```

Checks for the presence of registry files used in Windows account and credential analysis.

### Enumerate local accounts

```bash
sudo chntpw -l /mnt/vmdk/Windows/System32/config/SAM
```

Lists local Windows accounts stored in the SAM registry hive.

### Calculate file hashes

```bash
sha256sum ~/DFIR/registry/SAM \
  ~/DFIR/registry/SECURITY \
  ~/DFIR/registry/SYSTEM
```

Calculates SHA-256 hashes to help verify the integrity of collected forensic artifacts.

## 4. Credential Analysis

The investigation continued with an examination of Windows credential artifacts and DPAPI-protected data.

Relevant artifacts included:

- Windows registry hives: `SAM`, `SECURITY`, and `SYSTEM`.
- Microsoft Edge browser data.
- Windows credential storage.
- DPAPI master keys and protected credential records.

Mimikatz was used in the lab to examine DPAPI-related information and investigate stored credentials.

**Learning outcome:** Understanding how Windows stores credentials and how access to sensitive artifacts can affect system security.

## 5. SSH and SFTP Enumeration

### Test SSH access

```bash
ssh USERNAME@TARGET_IP
```

### Establish an SFTP session

```bash
sftp USERNAME@TARGET_IP
```

SFTP was used to investigate accessible files and examine SSH server configuration.

### Retrieve the SSH configuration

```text
cd /etc/ssh
get sshd_config
```

The configuration was reviewed to understand account restrictions and the forced SFTP command.

**Finding:** The account was subject to a restricted SFTP environment, which became an important part of the investigation.

## 6. Linux Process Enumeration

### Inspect running processes

```bash
./pspy64
```

`pspy` monitors process activity without requiring root privileges in typical configurations.

The investigation identified a scheduled Python application responsible for processing old backup files.

### Examine the application

```bash
cat /opt/archiver/app.py
cat /opt/archiver/helper.py
ls -la /opt/archiver/
ls -la /opt/archiver/__pycache__/
```

These commands inspect the application's source code, helper functions, file permissions, and compiled Python cache files.

**Finding:** The scheduled backup and archiving workflow required further security analysis, especially around privileged execution and file handling.

## 7. Privilege Escalation Analysis

The final stage investigated the interaction between the scheduled archiving task, Python bytecode, and the external `tar` executable.

The key security concepts were:

- Reviewing privileged scheduled tasks.
- Checking file and directory permissions.
- Investigating unsafe executable resolution.
- Understanding how insecure privileged workflows can lead to privilege escalation.

The lab demonstrated why privileged scripts should use trusted absolute executable paths, secure file permissions, and controlled input paths.

## Conclusion

This lab provided practical experience in network reconnaissance, SMB enumeration, Windows disk forensics, BitLocker analysis, credential artifact investigation, Linux process monitoring, and privilege escalation analysis.

The main takeaway is that misconfigured services, exposed backup data, stored credentials, and insecure privileged automation can create opportunities for system compromise.

*For educational purposes and authorized lab environments only.*
