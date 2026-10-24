# htb-odori-writeup
Penetration testing and digital forensics lab
# Hack The Box — Odori

## 1. Overview

This repository documents the analysis of the **Odori** machine on Hack The Box. The lab focuses on network reconnaissance, SMB enumeration, Windows disk forensics, credential analysis, Linux service enumeration, and privilege escalation.

The investigation demonstrates how exposed backup resources, stored credentials, restricted service configurations, and insecure privileged automation can introduce security risks.

### Objectives

- Identify exposed network services.
- Enumerate SMB shares and accessible files.
- Investigate a BitLocker-protected Windows disk image.
- Examine Windows registry hives and credential artifacts.
- Analyze SSH/SFTP restrictions.
- Monitor Linux processes and investigate scheduled tasks.
- Understand the security implications of unsafe privileged automation.

### Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Network scanning and service detection |
| SMBClient | SMB share enumeration and file retrieval |
| John the Ripper | Offline password auditing |
| BitLocker2John | BitLocker hash extraction |
| Dislocker | Accessing BitLocker-protected volumes |
| chntpw | Windows SAM account enumeration |
| SHA256sum | File integrity verification |
| Mimikatz | Windows credential and DPAPI artifact analysis |
| SSH / SFTP | Remote access and file transfer |
| pspy | Linux process monitoring |
| Python | Analysis of application and archiving workflows |

**Environment:** Kali Linux, Windows disk image, Linux server.

---

## 2. Network Reconnaissance

### Objective

Identify active services on the target and determine which services may provide useful information for further investigation.

### Command

```bash
nmap -sC -sV --min-rate=1000 TARGET_IP
```

### Command Breakdown

- `nmap` — launches the network scanner.
- `-sC` — runs Nmap's default script set.
- `-sV` — attempts to identify service versions.
- `--min-rate=1000` — requests a minimum probe sending rate of 1,000 packets per second.
- `TARGET_IP` — the target machine's IP address.

### Findings

The scan identified the following open TCP ports:

| Port | Service | Purpose |
|---|---|---|
| 22 | SSH | Remote administration |
| 139 | NetBIOS/SMB | Legacy SMB-related networking |
| 445 | SMB | File sharing and network resource access |

The service detection output indicated OpenSSH on Ubuntu and Samba services.

### Analysis

The exposed SMB service was selected for further enumeration because network shares may reveal accessible files or information about backup processes.

SSH was also noted as a potential remote access service for later investigation.

**Result:** The initial scan established the available network attack surface and identified SMB as a priority for further analysis.

---

## 3. SMB Enumeration

### Objective

Enumerate available SMB shares and determine whether the server exposes files that may provide useful investigative leads.

### 3.1 List available shares

```bash
smbclient -L //TARGET_IP
```

This command requests a list of SMB shares advertised by the remote server.

The `-L` option lists available shares rather than opening an interactive session with a specific share.

### 3.2 Connect to the backup share

```bash
smbclient //TARGET_IP/backup
```

This command opens an interactive SMB session with the `backup` share.

Useful commands inside the SMB client:

```text
ls
get info.txt
```

- `ls` — lists files and directories in the current remote directory.
- `get info.txt` — downloads the selected file to the local working directory.

The `get` command is used because `smbclient` does not provide the standard shell `cat` command for reading remote files.

### Findings

The backup share contained an information file named `info.txt`.

This established that the share was accessible and that its contents could be retrieved for offline examination.

### Analysis

Backup shares should be reviewed carefully because they can contain configuration files, historical data, application artifacts, or other information that was not intended for general access.

**Result:** SMB enumeration identified an accessible backup resource and provided an initial lead for subsequent investigation.

---

## 4. Windows Disk Forensics

### Objective

Examine the Windows disk image, access its BitLocker-protected volume, and identify forensic artifacts relevant to account and credential analysis.

### 4.1 Extract BitLocker-related hashes

```bash
bitlocker2john -i file02.vmdk | tee hash
```

**Explanation:**

- `bitlocker2john` — extracts password-auditing data from a BitLocker volume.
- `-i file02.vmdk` — specifies the input disk image.
- `|` — sends the command output to another program.
- `tee hash` — displays the output and saves it to a file named `hash`.

The resulting file can be used as input for offline password auditing.

### 4.2 Perform an offline password audit

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash --format=bitlocker
```

**Explanation:**

- `john` — launches John the Ripper.
- `--wordlist` — specifies a list of candidate passwords.
- `hash` — identifies the file containing the extracted hash data.
- `--format=bitlocker` — selects the BitLocker hash format.

This step tests candidate passwords against the extracted data. A recovered candidate must be validated before being relied upon.

### 4.3 Access the BitLocker volume

```bash
sudo dislocker -r -V /dev/nbd0p3 -u -- /mnt/bitlocker
```

This command uses Dislocker to access a BitLocker-protected volume.

- `sudo` — runs the command with elevated local privileges.
- `-r` — requests read-only access.
- `-V` — specifies the volume to examine.
- `-u` — selects the user-password authentication mode.
- `/mnt/bitlocker` — specifies the mount point for Dislocker output.

The command may request a password to unlock the volume.

### 4.4 Mount the decrypted filesystem

```bash
sudo mount -o loop,ro /mnt/bitlocker/dislocker-file /mnt/vmdk
```

This mounts the decrypted filesystem so its contents can be examined.

- `-o loop` — accesses the filesystem through a loop device.
- `ro` — mounts the filesystem read-only.
- `/mnt/bitlocker/dislocker-file` — the decrypted volume file.
- `/mnt/vmdk` — the destination mount point.

Read-only mounting helps preserve the original evidence during forensic examination.

### 4.5 Examine the filesystem

```bash
ls -lah /mnt/vmdk
```

This lists the contents of the mounted Windows filesystem, including hidden entries, file permissions, and human-readable file sizes.

The investigation identified standard Windows directories, including `Windows`, `Program Files`, `ProgramData`, and `Users`.

### 4.6 Locate registry hives

```bash
sudo ls -lh /mnt/vmdk/Windows/System32/config/SAM
sudo ls -lh /mnt/vmdk/Windows/System32/config/SECURITY
sudo ls -lh /mnt/vmdk/Windows/System32/config/SYSTEM
```

These commands verify the presence and sizes of three important Windows registry hives:

- `SAM` — contains local account information.
- `SECURITY` — stores security-related information.
- `SYSTEM` — contains system configuration, including information relevant to credential analysis.

### 4.7 Create a working directory for forensic artifacts

```bash
mkdir -p ~/DFIR/registry
```

Creates a directory for collected registry files. The `-p` option creates missing parent directories when necessary.

The original investigation copied the registry hives into this working directory for analysis.

### 4.8 Enumerate local Windows accounts

```bash
sudo chntpw -l /mnt/vmdk/Windows/System32/config/SAM
```

The `-l` option lists accounts recorded in the SAM registry hive.

The output included standard Windows accounts and a service account named `svc_backup`.

**Finding:** The disk image contained Windows account artifacts that could be investigated alongside the backup share and other available evidence.

### 4.9 Verify forensic artifacts with SHA-256

```bash
sha256sum ~/DFIR/registry/SAM \
  ~/DFIR/registry/SECURITY \
  ~/DFIR/registry/SYSTEM
```

Calculates a SHA-256 digest for each registry hive.

These hashes can be recorded in forensic notes to help verify that the collected files have not changed between examination stages.

**Result:** The disk image provided access to Windows filesystem and registry artifacts for further credential analysis.

---

## 5. Windows Credential Artifact Analysis

### Objective

Investigate Windows credential storage and understand how protected credentials may be associated with a user's security context.

### 5.1 Locate browser data and credential-related artifacts

```bash
sudo find /mnt/vmdk/Users/Administrator/AppData \
  -type f 2>/dev/null
```

**Explanation:**

- `find` — searches a directory tree.
- `-type f` — limits results to regular files.
- `2>/dev/null` — suppresses error messages, such as permission-related errors.

The investigation identified Microsoft Edge profile artifacts, including `Login Data` and `Local State`.

These files may contain information relevant to browser credential storage and encryption. Their presence alone does not mean that stored passwords are immediately readable.

### 5.2 Understand DPAPI

The investigation also examined the Windows Data Protection API (DPAPI).

DPAPI protects certain Windows secrets using keys associated with a user or system security context.

The analysis focused on the relationship between:

- DPAPI master keys.
- Windows user and system contexts.
- Stored credential artifacts.
- Protected credential records.

Mimikatz was used within the lab to investigate DPAPI-related data.

### Findings

The credential investigation demonstrated that information recovered from a disk image may be useful for understanding how credentials are stored and protected.

However, access to an artifact does not automatically imply that the corresponding secret can be decrypted. The necessary keys, permissions, and security context must also be considered.

**Result:** Windows credential artifacts were identified as a significant part of the investigation and provided context for examining the service account's access to the Linux server.

---

## 6. SSH and SFTP Investigation

### Objective

Investigate remote access to the Linux server and determine how the service account's session was restricted.

### 6.1 Test SSH connectivity

```bash
ssh USERNAME@TARGET_IP
```

This attempts to establish an SSH session using the specified username and target address.

SSH is commonly used for secure remote administration. Successful authentication and the permissions available after login are separate security considerations.

### 6.2 Establish an SFTP session

```bash
sftp USERNAME@TARGET_IP
```

Starts an interactive SFTP session for secure file transfer.

Useful commands inside SFTP:

```text
cd /etc/ssh
get sshd_config
```

- `cd /etc/ssh` — changes the remote working directory.
- `get sshd_config` — downloads the SSH daemon configuration file for local examination.

### 6.3 Examine the SSH configuration

The investigation identified a configuration section associated with the service account's group:

```text
Match group svc_backup
    ForceCommand /opt/restrict /home/%u
    AllowTcpForwarding no
```

**Explanation:**

- `Match group` — applies subsequent SSH configuration directives to matching group members.
- `ForceCommand` — specifies a command to run instead of the command requested by the client.
- `/opt/restrict` — identifies the configured restriction wrapper.
- `/home/%u` — uses the connecting username in the configured path.
- `AllowTcpForwarding no` — disables SSH TCP forwarding for matching connections.

The actual effective restrictions depend on the complete SSH configuration and the behavior of the configured wrapper.

### 6.4 Inspect the restriction wrapper

The retrieved script contained a command invoking the OpenSSH SFTP server with a specified starting directory.

This was relevant because it showed how SSH configuration and a wrapper script combined to control the service account's session.

### Findings

The service account was configured for a restricted SFTP workflow rather than unrestricted interactive SSH access.

The investigation highlighted the importance of examining both SSH configuration directives and the scripts they invoke.

**Result:** The SSH/SFTP configuration provided information about the service account's access restrictions and the server's remote access design.

---

## 7. Linux Process Enumeration

### Objective

Investigate the Linux server's running processes and identify scheduled applications involved in backup processing.

### 7.1 Monitor process activity

```bash
./pspy64
```

`pspy` monitors process execution and can reveal commands launched by scheduled tasks and other services.

The lab investigation identified a scheduled Python process running:

```text
python3 /opt/archiver/app.py
```

This was a useful lead because scheduled tasks may run with permissions different from those of the account investigating them.

### 7.2 Inspect the archiving application

```bash
cat /opt/archiver/app.py
```

The application scanned the `/backup` directory and compared file modification times against a threshold of three years.

Files older than the threshold were passed to a helper function for archiving.

### 7.3 Inspect the helper module

```bash
cat /opt/archiver/helper.py
```

The helper module constructed an archive filename and invoked the system `tar` executable through Python's subprocess functionality.

It also removed the original file after starting the archiving process.

This behavior warranted further investigation because the application was part of a scheduled workflow and performed file operations.

### 7.4 Review file permissions

```bash
ls -la /opt/archiver/
ls -la /opt/archiver/__pycache__/
```

These commands display file ownership, permissions, sizes, and directory contents.

The investigation also identified a compiled Python module in the application's cache directory.

### Analysis

The important questions at this stage were:

- Which user executed the scheduled application?
- Which files could the service account modify?
- Were executable paths resolved securely?
- Could an unprivileged user influence a privileged operation?
- Did the application validate paths and file operations appropriately?

These checks help identify whether a scheduled task introduces a privilege boundary violation.

**Result:** Process monitoring and source-code review identified the archiving workflow as a critical area for security analysis.

---

## 8. Privilege Escalation — Security Analysis

### Objective

Understand how weaknesses in a privileged archiving workflow can affect the security of the Linux host.

The investigation examined the relationship between the scheduled Python application, its helper module, the compiled Python cache, and the external archiving executable.

### Key observations

1. A scheduled task processed files from a backup directory.
2. The helper module invoked an external executable to create archives.
3. File permissions and executable resolution were important to the security of the workflow.
4. An unsafe interaction between privileged code and user-influenced resources can potentially allow an unprivileged account to influence privileged operations.

### Security implications

Privileged applications should:

- Use trusted absolute paths for external executables.
- Protect scripts, bytecode, and their parent directories from unauthorized modification.
- Validate paths and filenames before processing them.
- Avoid executing user-controlled content with elevated privileges.
- Review scheduled tasks and their dependencies regularly.

**Result:** The final stage demonstrated the importance of secure executable resolution, correct file permissions, and careful separation between privileged services and unprivileged users.

---

## 9. Lessons Learned

This lab provided practical experience in several areas of cybersecurity.

### Network Security

- Identifying exposed services with Nmap.
- Enumerating SMB shares.
- Understanding how exposed backup resources may reveal sensitive information.

### Digital Forensics

- Examining a BitLocker-protected disk image.
- Working with Windows registry hives.
- Identifying credential-related artifacts.
- Using cryptographic hashes to document forensic evidence.

### Linux Security

- Investigating SSH/SFTP restrictions.
- Monitoring scheduled processes.
- Reviewing Python applications and helper modules.
- Assessing file permissions and privileged execution paths.

### Methodology

The investigation followed a structured approach:

1. Reconnaissance.
2. Service enumeration.
3. Evidence collection.
4. Offline forensic analysis.
5. Credential artifact investigation.
6. Linux process and configuration analysis.
7. Security impact assessment.

## 10. Conclusion

The Odori lab demonstrated how network enumeration, digital forensics, credential analysis, and Linux service investigation can be combined during a penetration test.

The most important takeaway was that security weaknesses often arise from the interaction of multiple components rather than a single vulnerable service.

Careful evidence collection, structured analysis, and understanding privilege boundaries are essential skills for penetration testers and security analysts.

**Disclaimer:** This documentation describes work performed in an authorized educational lab environment. Sensitive credentials, challenge flags, and private information are intentionally excluded.
