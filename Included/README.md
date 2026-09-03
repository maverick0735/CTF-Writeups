Hack The Box: Included — Machine Writeup
📌 Overview & Metadata
Target: Included (Linux)

Pwn Date: 03 Sep 2026

🧠 Concept Breakdown: TFTP & LFI to RCE
This machine demonstrates the danger of combining two high-risk misconfigurations:

Trivial File Transfer Protocol (TFTP): An unauthenticated protocol often used for booting operating systems or backing up network configurations. If writable, attackers can upload malicious files.

Local File Inclusion (LFI): A web vulnerability allowing an attacker to read local files on the server. When chained with a service where an attacker can upload a file (like TFTP), LFI can be escalated to Remote Code Execution (RCE) by including the uploaded payload.

🚪 Initial Access: TFTP Upload & LFI Execution
The attack began by leveraging TFTP to upload a PHP reverse shell. I generated a standard PHP PentestMonkey reverse shell payload using revshells.com, configured to connect back to my Kali machine on port 4444.

Using the tftp client, I connected to the target and uploaded the payload:

Bash
tftp 10.129.51.83
tftp> put shell.php
tftp> quit
[Note: Target IPs varied slightly between 10.129.51.64 and 10.129.51.83 during the session due to HTB instance resets]

With the malicious file staged in the default TFTP directory (/var/lib/tftpboot/), I used curl to trigger the web application's LFI vulnerability, instructing the server to execute the payload:

Bash
curl -v "http://10.129.51.83/?file=/var/lib/tftpboot/shell.php"
This successfully spawned a reverse shell, granting initial access as the www-data user. I immediately stabilized the shell using Python:

Bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
🕵️ Lateral Movement: Web Directory Enumeration
As www-data, I navigated to the /home/mike directory but was met with a "Permission denied" error when attempting to read user.txt.

This required enumerating the local file system for credentials. Investigating the webroot (/var/www/html) revealed a .htpasswd file.

Bash
cat .htpasswd
mike:Sheffield19
Using these credentials, I successfully pivoted to the user mike via the su command and captured the user flag:
User Flag: a56ef91d70cfbf2cdb8f454c006935a1

🚀 Privilege Escalation: LXD Container Breakout
With a stable shell as mike, my ultimate goal was complete system compromise. However, basic enumeration revealed that mike did not have standard root privileges or expansive sudo rights. I had to seek out alternative paths for privilege escalation.

Checking group memberships revealed a critical vulnerability: mike was a member of the lxd group.

What is the LXD Group?
LXD is a next-generation system container manager for Linux. Users assigned to the lxd group are granted the authority to manage these containers locally without needing sudo access. From an offensive security perspective, membership in this group is functionally equivalent to having root access. An attacker can exploit this by deploying a new, heavily privileged container and mounting the host machine's entire root filesystem (/) directly into it. Once inside the container, the attacker operates as root and can freely read, modify, or destroy the host's files, completely bypassing standard permission controls.

Recognizing this misconfiguration, I sought out known LXD tricks and referenced HackTricks documentation to execute the container breakout.

I began by attempting to download the necessary Alpine Linux builder images (incus.tar.xz and rootfs.squashfs) from my local HTTP server using wget. Initially, I hit a "Permission denied" error because my current working directory was not writable—a standard operational hurdle requiring a quick shift to a writable directory like /tmp or /dev/shm to stage the exploit files.

After successfully importing the images and initializing the LXC container, I mounted the host's root directory to /mnt/root inside the container and executed a shell:

Bash
lxc exec privesc /bin/sh
This successfully granted a root shell (uid=0(root)). I navigated to the mounted host directory (/mnt/root/root) and captured the final flag:
Root Flag: c693d9c7499d9f572ee375d4c14c7bcf

💡 Lessons Learned
This box perfectly highlights the methodology of a penetration test:

Vulnerability Chaining: An LFI is dangerous, but chaining it with an unauthenticated TFTP upload converts a simple file read into full Remote Code Execution.

Internal Recklessness: Storing plaintext credentials in .htpasswd files within the webroot provides a trivial path for lateral movement.

Group Misconfigurations: Adding standard users to administrative or virtualization groups (like lxd or docker) creates a direct, unauthenticated path to root. Administrators must treat these group assignments with the same strict scrutiny as the sudoers file.
