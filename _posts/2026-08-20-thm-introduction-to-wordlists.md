---
title: "TryHackMe - Introduction to Wordlists"
date: 2026-08-20 00:01:00 +0700
categories: [TryHackMe, Offensive Security]
tags:
  - tryhackme
  - wordlists
  - reconnaissance
  - osint
  - cewl
  - crunch
  - ffuf
  - hydra
  - brute-force
  - kali-linux
author: NoxShepherd
toc: true
comments: true
---

# Introduction

While learning Offensive Security, I have been trying to understand not only how to use security tools, but also why I should use them.

One of the topics I recently learned about is **wordlists**.

Before working on this room, I mostly thought of a wordlist as a simple text file containing thousands or millions of possible passwords.

After completing the **Introduction to Wordlists** room on TryHackMe, I realized that wordlists can be used for much more than password testing.

They can be used for:

- Password testing
- Username enumeration
- Directory discovery
- File discovery
- Subdomain enumeration
- Web fuzzing
- API endpoint discovery
- Targeted credential testing

The most interesting part for me was learning how information collected during reconnaissance can be turned into a custom wordlist.

Instead of blindly trying millions of possibilities, I can use information that is actually related to the target.

This write-up documents what I learned and what I did during the lab, starting from information gathering and OSINT, then building custom wordlists, and finally using them with tools such as `ffuf` and `Hydra`.

> **Disclaimer**
>
> All activities described in this write-up were performed inside an authorized TryHackMe laboratory environment for educational purposes. These techniques should never be used against systems without explicit authorization.

---

# What Is a Wordlist?

A wordlist is simply a text file containing a collection of possible values.

For example:

```text
admin
administrator
password
welcome
login
company
```

A security tool can read these values one by one and use them as input during an automated test.

I like to think of it as having a large collection of possible keys and asking a tool:

> "Try these one by one and see if any of them work."

Depending on the situation, those values could be:

- Passwords
- Usernames
- Directory names
- File names
- Subdomains
- API endpoints
- Parameters

This is why wordlists are commonly used throughout penetration testing and security assessments.

---

# Why Not Just Use a Huge Wordlist?

At first, I had the same question:

> "If there are already huge wordlists like `rockyou.txt`, why should I bother creating my own?"

The answer is **relevance**.

A generic wordlist contains a large number of possibilities, but most of them may have nothing to do with the target.

For example, imagine that during reconnaissance I discover that a company uses terms such as:

```text
Helios
Helios Finance
Helios Portal
```

Those terms might appear somewhere in:

- Directory names
- Usernames
- Passwords
- Subdomains
- Internal applications

A generic wordlist might not contain those exact terms.

A custom wordlist allows me to take what I already know about the target and turn that information into additional candidates.

So instead of blindly trying millions of possibilities, I can create a smaller list that is much more relevant to the environment I am testing.

---

# Lab Environment

For this lab, I used the environment provided by TryHackMe.

The target was:

```text
tryfinanceme.local
```

The lab also provided another hostname:

```text
social.tryfinanceme.local
```

I added both entries to my `/etc/hosts` file:

```bash
echo '10.48.177.124 tryfinanceme.local social.tryfinanceme.local' >> /etc/hosts
```

This allowed my machine to resolve the lab hostnames correctly.

At this point, I had a working target and could start the reconnaissance process.

---

# Step 1 - Gathering Information

I did not immediately start brute-forcing the target.

Instead, I wanted to understand what information was already available.

This is where **reconnaissance** becomes important.

The basic idea is simple:

> Before trying to attack something, understand what you are dealing with first.

For this lab, I was interested in collecting things such as:

- Company-specific terms
- Employee names
- Email addresses
- Technology names
- Project names
- Words used throughout the website
- Information contained in public documents

All of these could potentially become useful wordlist entries later.

---

# Step 2 - Using CeWL to Collect Words

The first tool I used was **CeWL**.

CeWL is useful for crawling a website and extracting words that appear throughout its pages.

I ran:

```bash
cewl -d 2 -m 3 --lowercase --with-numbers -e \
--email_file emails.txt \
-w cewl_words.txt \
http://tryfinanceme.local
```

The important options are:

| Option | Purpose |
|:--|:--|
| `-d 2` | Crawl up to two levels deep |
| `-m 3` | Collect words with at least three characters |
| `--lowercase` | Convert extracted words to lowercase |
| `--with-numbers` | Include words containing numbers |
| `-e` | Extract email addresses |
| `--email_file` | Save discovered emails to a file |
| `-w` | Save extracted words to a wordlist |

After the crawl, I had two useful files:

```text
cewl_words.txt
emails.txt
```

The first one contained words extracted from the website.

The second one contained email addresses discovered during the crawl.

This was already useful because the email addresses could later help me generate possible usernames.

---

# Step 3 - Looking for Documents

The next thing I checked was whether the target exposed any documents.

During the lab, I found a `/docs/` directory containing PDF files.

I downloaded the available PDFs with:

```bash
wget -r -A pdf http://tryfinanceme.local/docs/
```

Why bother looking at documents?

Because documents can contain information that is not necessarily visible on the main website.

For example:

- Internal terminology
- Employee names
- Project names
- Product names
- Technology references
- Email addresses

In a real engagement, publicly accessible documents can sometimes reveal surprisingly useful information.

For this lab, I extracted useful strings from the documents and added them to my growing collection of candidate words.

---

# Step 4 - Building Username Candidates

After collecting names and email addresses, I wanted to generate possible username formats.

For example, if I discovered:

```text
Alex Johnson
```

there are several common ways an organization might create a username.

## First Name + Last Name

I generated:

```bash
awk '{print tolower($1)"."tolower($2)}' names.txt > users_first.last.txt
```

This produced:

```text
alex.johnson
```

## First Initial + Last Name

I also generated:

```bash
awk '{print tolower(substr($1,1,1))tolower($2)}' names.txt > users_flast.txt
```

This produced:

```text
ajohnson
```

## First Name + Last Initial

Finally:

```bash
awk '{print tolower($1)tolower(substr($2,1,1))}' names.txt > users_firstl.txt
```

This produced:

```text
alexj
```

The reason I created multiple formats is simple:

**I do not want to assume that an organization uses only one username convention.**

Different environments may use:

```text
firstname.lastname
flastname
firstnamel
```

or another naming pattern entirely.

---

# Step 5 - Cleaning the Wordlist

At this point, I had gathered information from several sources.

The problem was that the raw data was messy.

It contained things like:

- Duplicate entries
- Uppercase and lowercase variations
- Special characters
- Short strings
- Windows carriage returns
- Irrelevant values

Using a messy wordlist can waste time.

For example:

```text
Helios
helios
HELIOS
```

are technically different entries to a tool, even though they represent the same word for our purposes.

So I combined the wordlists first:

```bash
cat cewl_words.txt raw_words.txt | sort -u > words_raw.txt
```

The `-u` option of `sort` removes duplicate lines.

Then I normalized and filtered the result:

```bash
cat words_raw.txt | \
tr '[:upper:]' '[:lower:]' | \
tr -d '\r' | \
grep -P '^[a-z0-9][a-z0-9._-]{4,}$' | \
sort -u > words_clean.txt
```

This command performs several operations:

- Converts uppercase characters to lowercase
- Removes Windows carriage returns
- Filters out strings that do not match the expected format
- Removes duplicate entries

After cleaning the list, I checked how many entries remained:

```bash
wc -l words_clean.txt
```

The result was:

```text
161
```

So I ended up with **161 cleaned and unique words**.

That is a much more manageable list compared to blindly using a huge wordlist containing millions of entries.

---

# Step 6 - Preparing the Username List

I also cleaned up the username candidates by merging the different formats:

```bash
cat users_first.last.txt \
users_flast.txt \
users_firstl.txt \
users_from_emails.txt | sort -u > users.txt
```

Now I had a consolidated username list:

```text
users.txt
```

At this point, my wordlists were becoming more structured.

I had:

```text
words_clean.txt
users.txt
```

The first would be used for web enumeration.

The second would be used for login testing.

---

# Step 7 - Generating a Password List With Crunch

The next part was interesting.

During the lab, I learned that the target's password pattern followed:

```text
Helios20NN!
```

where `NN` represents two digits.

Instead of generating thousands or millions of random passwords, I could generate only the values that matched the known pattern.

For this, I used **Crunch**:

```bash
crunch 11 11 -t Helios20%%! -o pass_helios.txt
```

The important part here is:

```text
%%
```

In Crunch, `%` represents a numeric character.

So:

```text
Helios20%%!
```

means:

```text
Helios2000!
Helios2001!
Helios2002!
...
Helios2099!
```

This results in only 100 possible passwords.

I could then use:

```text
pass_helios.txt
```

as my password candidate list.

This was a good example of why understanding the target's password pattern can dramatically reduce the search space.

---

# Step 8 - Final Wordlists

At this stage, I had three important files:

```text
words_clean.txt
users.txt
pass_helios.txt
```

Each one had a different purpose:

| File | Purpose |
|:--|:--|
| `words_clean.txt` | Directory and file discovery |
| `users.txt` | Username candidates |
| `pass_helios.txt` | Password candidates |

The workflow was now starting to come together:

```text
OSINT
  |
  v
Collect Information
  |
  v
Create Wordlists
  |
  v
Clean and Normalize
  |
  v
Generate Usernames
  |
  v
Generate Password Candidates
  |
  v
Use Lists Against the Target
```

---

# Step 9 - Discovering Hidden Directories With ffuf

Now that I had a custom wordlist, I wanted to see whether any hidden directories existed on the web server.

This is where **ffuf** came in.

The concept is actually quite simple.

If my wordlist contains:

```text
admin
login
api
backup
helios
```

ffuf can test them against the target.

The idea is effectively:

```text
http://tryfinanceme.local/admin
http://tryfinanceme.local/login
http://tryfinanceme.local/api
http://tryfinanceme.local/backup
http://tryfinanceme.local/helios
```

It then looks at the HTTP responses to determine whether something interesting exists.

I used my cleaned custom wordlist for the enumeration.

The important discovery was:

```text
/helios/
```

The server returned:

```text
200
```

which means the resource was successfully found.

So I had discovered something that was not immediately obvious from the main website.

---

# Step 10 - Finding the Login Page

After discovering:

```text
/helios/
```

I explored the directory further.

Inside it, I found:

```text
/helios/login.php
```

This gave me a new attack surface:

```text
Web Server
    |
    v
Hidden Directory
    |
    v
/helios/
    |
    v
Login Page
```

At this point, I could move from directory enumeration to credential testing.

---

# Step 11 - Testing the Login With Hydra

For the login page, I used **Hydra**.

The command was:

```bash
hydra -L users.txt \
-P pass_helios.txt \
-f -V -t 4 \
tryfinanceme.local \
http-post-form \
'/helios/login.php:username=^USER^&password=^PASS^:S=THM{'
```

The important parameters are:

| Option | Meaning |
|:--|:--|
| `-L users.txt` | Load usernames from `users.txt` |
| `-P pass_helios.txt` | Load passwords from `pass_helios.txt` |
| `-f` | Stop after finding a valid credential |
| `-V` | Show each attempt |
| `-t 4` | Use four concurrent threads |
| `http-post-form` | Target an HTTP POST login form |

The important part of the command is:

```text
username=^USER^&password=^PASS^
```

Hydra replaces:

```text
^USER^
```

with each username.

And:

```text
^PASS^
```

with each password.

So effectively, Hydra was testing combinations from:

```text
users.txt
+
pass_helios.txt
```

against the login form.

---

# Step 12 - Finding Valid Credentials

After running Hydra, I found a valid credential.

The username was:

```text
alex.johnson
```

I could then use the discovered credential to access the `/helios/` application.

The lab provided the following flag:

```text
THM{w0rdlists_win_rooms}
```

The final chain looked like this:

```text
Reconnaissance
      |
      v
OSINT
      |
      v
Custom Wordlist
      |
      v
Wordlist Cleaning
      |
      v
Username Generation
      |
      v
Password Pattern Generation
      |
      v
ffuf
      |
      v
Hidden Directory
      |
      v
Login Page
      |
      v
Hydra
      |
      v
Valid Credential
      |
      v
Flag
```

---

# What I Learned

The biggest takeaway from this room was not the flag.

It was understanding how different techniques can be connected together.

Before doing this lab, I mostly thought about wordlists as something like:

```text
passwords.txt
```

Now I see them differently.

A wordlist can be the result of an entire reconnaissance process.

For example:

```text
Public Information
       |
       v
OSINT
       |
       v
Names / Emails / Products / Technologies / Projects
       |
       v
Custom Wordlist
       |
       v
Enumeration
       |
       v
Potential Attack Surface
```

The wordlist becomes more valuable when it contains information that is actually relevant to the target.

---

# Generic Wordlists vs Custom Wordlists

There is still a place for generic wordlists.

For example:

```text
rockyou.txt
SecLists
common.txt
```

These are useful because they provide a broad collection of possibilities.

However, they can also contain a lot of noise.

A custom wordlist is different.

Instead of asking:

> "What could possibly exist?"

I am asking:

> "Based on what I already know about this target, what is likely to exist?"

That difference can make enumeration much more efficient.

This does not mean that generic wordlists are useless.

Instead, I see them as complementary.

A good assessment may involve both:

- Generic wordlists for broad coverage
- Custom wordlists for target-specific testing

---

# The Defender Perspective

While working through this room, I also started thinking about the same process from a defender's perspective.

An attacker does not necessarily need access to internal systems to start building useful information.

Publicly available information can already provide valuable clues.

For example:

```text
Employee Names
      +
Company Name
      +
Technology Stack
      +
Project Names
      +
Public Documents
      |
      v
Targeted Wordlist
      |
      v
Credential / Enumeration Attempts
```

This is why information exposure matters.

A single piece of information might not look dangerous.

But when multiple pieces of information are combined, they can become much more useful to an attacker.

This was one of the things I found interesting about this room:

**OSINT and technical exploitation are not completely separate activities.**

Information gathered during reconnaissance can directly influence what happens during exploitation.

---

# Why Custom Wordlists Matter

One of the biggest lessons I took from this room is that the quality of a wordlist can matter more than its size.

A huge wordlist might contain millions of entries, but only a small percentage may be relevant to the target.

A custom wordlist can be much smaller while still being useful because it is based on information collected from the target.

For example:

```text
Generic Wordlist
       |
       v
Millions of Possibilities
       |
       v
Large Search Space
```

Compared with:

```text
Target Information
       |
       v
Custom Wordlist
       |
       v
Smaller Search Space
       |
       v
More Relevant Candidates
```

This is not to say that smaller is always better.

The important point is **relevance**.

---

# Final Thoughts

This room changed the way I look at wordlists.

I started with a simple idea:

> "A wordlist is just a file containing possible passwords."

I finished the lab with a much broader understanding.

A wordlist can be:

- A collection of passwords
- A list of usernames
- A list of directories
- A list of subdomains
- A collection of API endpoints
- A target-specific collection of intelligence

The most valuable part for me was seeing how everything connected.

I gathered information.

I turned that information into words.

I cleaned the words.

I generated username and password candidates.

I used those lists for directory enumeration.

I discovered a hidden application.

Then I used the generated credentials to test the login page.

The important lesson for me is that Offensive Security is not simply about knowing which command to run.

It is about understanding:

> **Why am I running this command?**

> **What information am I looking for?**

> **What does the result tell me?**

> **What should I do next based on that result?**

That mindset is something I want to continue developing as I move deeper into Offensive Security.

---

# Lab Summary

| Item | Result |
|:--|:--|
| Platform | TryHackMe |
| Room | Introduction to Wordlists |
| Main Topic | Custom Wordlists |
| OSINT | Yes |
| CeWL | Yes |
| Crunch | Yes |
| ffuf | Yes |
| Hydra | Yes |
| Clean Wordlist Size | 161 entries |
| Hidden Directory | `/helios/` |
| HTTP Status | `200` |
| Username Found | `alex.johnson` |
| Flag | `THM{w0rdlists_win_rooms}` |

---

# Tools Used

| Tool | Purpose |
|:--|:--|
| **CeWL** | Website crawling and word/email collection |
| **Crunch** | Pattern-based wordlist generation |
| **ffuf** | Directory and file enumeration |
| **Hydra** | Login credential testing |
| **awk** | Username generation |
| **grep** | Filtering wordlist entries |
| **sort** | Sorting and removing duplicates |
| **tr** | Normalizing text |
| **wget** | Downloading target documents |

---

# Key Takeaways

### 1. Wordlists are not only for password cracking

They can also be used for:

- Directory enumeration
- File discovery
- Subdomain enumeration
- Web fuzzing
- Username discovery

### 2. Context matters

A smaller wordlist containing target-specific information can be more useful than a massive generic list.

### 3. OSINT can directly support exploitation

Information gathered during reconnaissance can be transformed into usernames, passwords, directories, and other attack candidates.

### 4. Cleaning matters

Removing duplicates, normalizing case, and filtering irrelevant entries makes the wordlist more efficient.

### 5. Understand the attack chain

The most important thing I learned was not how to run `ffuf` or `Hydra` individually.

It was understanding how the output from one step can become the input for the next step.

```text
Reconnaissance
      |
      v
OSINT
      |
      v
Information Gathering
      |
      v
Custom Wordlist
      |
      v
Enumeration
      |
      v
Discovery
      |
      v
Credential Testing
      |
      v
Access
```

---

# Conclusion

Completing this room gave me a better understanding of how attackers can turn publicly available information into something much more useful.

What initially looked like a simple text file turned out to be an important part of the reconnaissance and enumeration process.

For me, this was another small step toward developing a more practical **Offensive Security mindset**.

I am not just trying to memorize tools or commands.

I want to understand the reasoning behind them.

And that is probably the most important lesson I took from this lab.

---

> **Learn the system. Understand the weakness. Think like an attacker.**
>
> **— NoxShepherd**

---

## Disclaimer

All techniques demonstrated in this write-up were performed against an authorized TryHackMe training environment.

Never perform reconnaissance, enumeration, brute-force attacks, credential testing, or exploitation against systems that you do not own or do not have explicit permission to test.