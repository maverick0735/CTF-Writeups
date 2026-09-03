**Hack The Box: Included — Machine Writeup**
📌 Overview & Metadata
Field	Detail
Target	Included (Linux)
Difficulty	Easy
Pwn Date	03 Sep 2026
Techniques	TFTP unauthenticated upload, LFI → RCE, credential reuse, LXD group privesc
🧠 Concept Breakdown: TFTP & LFI to RCE

This machine demonstrates the danger of combining two high-risk misconfigurations:

Trivial File Transfer Protocol (TFTP): An unauthenticated protocol often used for booting operating systems or backing up network configurations. If writable, attackers can upload malicious files with no credentials required.
Local File Inclusion (LFI): A web vulnerability that lets an attacker force the server to read and render local files. On its own, LFI is a disclosure bug — but when chained with a service that lets an attacker plant a file (like TFTP), it becomes a path to Remote Code Execution (RCE): the "included" file doesn't have to be a static log, it can be a payload the attacker just uploaded.
🔍 Enumeration
nmap -sv -Pn [dedicated lab ip address]


Example structure to fill in:

**bash**
nmap -sC -sV -p- -oN nmap-initial.txt 10.129.51.83
Ports found: 80
Web application observed at: http://10.129.51.83
Parameter identified as vulnerable to LFI: ?file=
🚪 Initial Access: TFTP Upload & LFI Execution

The attack began by leveraging TFTP to upload a PHP reverse shell. 
I generated a standard PHP PentestMonkey reverse shell payload (via revshells.com), configured to connect 
back to my Kali machine on port 4444.

Using the TFTP client, I connected to the target and uploaded the payload:

**bash**
tftp 10.129.51.83
tftp> put shell.php
tftp> quit

Note: Target IPs varied slightly between 10.129.51.64 and 10.129.51.83 during the session due to HTB instance resets.

With the malicious file staged in the default TFTP directory (/var/lib/tftpboot/), 
I used curl to trigger the web application's LFI vulnerability, instructing the server to include and execute the payload:

**bash**
curl -v "http://10.129.51.83/?file=/var/lib/tftpboot/shell.php"

This successfully spawned a reverse shell, granting initial access as the www-data user. I immediately stabilized the shell:

**bash**
python3 -c 'import pty;pty.spawn("/bin/bash")'
🕵️ Lateral Movement: Web Directory Enumeration

As www-data, I navigated to /home/mike but was met with a "Permission denied" error when attempting to read user.txt.

This required enumerating the local filesystem for credentials. Investigating the webroot (/var/www/html) revealed a .htpasswd file:

**bash**
cat .htpasswd
mike:[REDACTED]

Using these credentials, I pivoted to the user mike via su and captured the user flag:

User Flag: a56ef91d70cfbf2cdb8f454c006935a1
🚀 Privilege Escalation: LXD Container Breakout

With a stable shell as mike, the goal shifted to full system compromise. Basic enumeration showed mike did not have standard root privileges or expansive sudo rights, so I looked for alternative escalation paths.

Checking group memberships revealed a critical vulnerability: mike was a member of the lxd group.

**What is the LXD Group?**

LXD is a system container manager for Linux. Users in the lxd group can manage containers locally without needing sudo
— from an offensive standpoint, this is functionally equivalent to root access. An attacker can deploy a new, heavily 
privileged container and mount the host's entire root filesystem (/) into it. From inside that container, the attacker 
operates as root and can read, modify, or destroy the host's files, bypassing standard permission controls entirely.

**Exploitation**

Referencing known LXD privilege-escalation techniques (HackTricks), 
I built and imported the necessary Alpine Linux images (incus.tar.xz, rootfs.squashfs) via wget from my local HTTP server.
I initially hit a "Permission denied" error because my working directory wasn't writable — resolved by staging the exploit 
files in /tmp or /dev/shm instead.
After importing the images and initializing the container with the host root mounted at /mnt/root, I dropped into a shell:

**bash**
lxc exec privesc /bin/sh

This granted a root shell:

**bash**
id
# uid=0(root) gid=0(root) groups=0(root)

Navigating to the mounted host directory (/mnt/root/root), I captured the final flag:

Root Flag: c693d9c7499d9f572ee375d4c14c7bcf
💡 **Lessons Learned**
Vulnerability Chaining — An LFI alone is a disclosure risk, but chaining it with an unauthenticated TFTP upload converts a 
simple file read into full Remote Code Execution.
Internal Recklessness — Storing plaintext credentials in .htpasswd files within the webroot provides a trivial path for 
lateral movement.
Group Misconfigurations — Adding standard users to administrative or virtualization groups (like lxd or docker) creates a 
direct, unauthenticated path to root. Administrators must treat these group assignments with the same scrutiny as the sudoers file.
