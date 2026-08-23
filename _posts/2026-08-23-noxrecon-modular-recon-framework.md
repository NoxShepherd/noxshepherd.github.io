---
title: "NoxRecon - Building My Own Modular Reconnaissance Framework"
date: 2026-08-23 10:00:00 +0700
categories: [Projects, Offensive Security]
tags:
  - noxrecon
  - python
  - reconnaissance
  - osint
  - nmap
  - subfinder
  - httpx
  - nuclei
  - amass
  - automation
author: NoxShepherd
toc: true
comments: true
---

# Introduction

After spending months learning reconnaissance tools one by one — `nmap`, `subfinder`, `httpx`, `nuclei`, `amass` — I noticed a pattern in my own workflow.

Every time I started a new lab or an authorized assessment, I typed the same commands in the same order:

```bash
dig <target> ANY
whois <target>
subfinder -d <target> -silent
nmap -Pn --open -sV -T4 <target>
httpx <target>
nuclei -target <target>
```

Then I copy-pasted everything into a Markdown report by hand.

That repetition was the moment I realized: instead of just **using** tools, why not build the glue between them myself? Not because better frameworks don't exist — they do — but because building my own would force me to understand every step of the recon pipeline.

> 💡 **Why build your own?**
>
> "Don't just run the tool. Understand what the tool is doing." Building a framework around existing tools is the fastest way I've found to learn *how* each tool fits into a methodology — what it can't do, when it fails silently, and how its output feeds the next stage.

This post documents **NoxRecon**, my modular reconnaissance framework written in pure Python (standard library only), now open-sourced at [github.com/NoxShepherd/NoxRecon](https://github.com/NoxShepherd/NoxRecon).

> ⚠️ **Disclaimer**
>
> NoxRecon is for **authorized security testing and education only**. Every target mentioned in this post (`scanme.nmap.org`) is a public testing service explicitly provided for scanning practice. Scanning systems without permission is illegal.

---

# What Is NoxRecon?

NoxRecon chains independent recon modules into a single pipeline:

```
Learn → Research → Build a Lab → Experiment
```

…applied to tooling itself. One command runs DNS enumeration, WHOIS, passive subdomain discovery, port scanning, web fingerprinting, cloud bucket checks, geolocation, historical URL discovery, and vulnerability scanning — then aggregates everything into a per-target workspace with JSON outputs and a final human-readable Markdown report.

```
                    ┌──────────────────────────────┐
                    │        noxrecon.py CLI       │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │      Engine (pipeline)       │
                    │   sorts modules by ORDER     │
                    └──────────────┬───────────────┘
                                   │
        ┌──────────┬───────────────┼────────────────┬──────────┐
        ▼          ▼               ▼                ▼          ▼
    ┌───────┐ ┌────────┐    ┌──────────┐    ┌──────────┐ ┌────────┐
    │  dns  │ │whois   │    │portscan  │    │   web    │ │ geoip  │
    └───┬───┘ └───┬────┘    └────┬─────┘    └────┬─────┘ └───┬────┘
        │         │              │               │           │
        ▼         ▼              ▼               ▼           ▼
   results/<target>/... per-module workspace + report/report.md
```

Three design rules guided everything:

1. **Passive-first.** Modules query public datasets (CT logs, Wayback, ip-api.com) before sending a single packet to the target.
2. **Degrade gracefully.** Missing external tool never crashes a run — the affected module falls back to pure-Python implementations or skips with a warning.
3. **Report as you go.** Every module writes structured JSON; the final narrative report is generated from that data, not from console scraping.

---

# Framework Information

| Component | Detail |
|---|---|
| Language | Python 3.8+ (stdlib only — zero pip dependencies) |
| Platform | Linux / macOS |
| Source | [github.com/NoxShepherd/NoxRecon](https://github.com/NoxShepherd/NoxRecon) |
| Core modules | 18 built-in + plugin system |
| Plugins | `nuclei` (vuln scanning), `ffuf` (content fuzzing) |
| Output | Per-target workspace: JSON + `report.md` |

---

# Learning Objectives

- Understand how a modular recon pipeline is architected (loader, engine, workspace, reporter).
- Learn graceful degradation patterns for external-tool dependencies.
- Practice turning raw tool output into a narrative security report automatically.
- See how each recon layer answers a different question about a target.

---

# Phase 1 — Designing the Core

The whole framework hangs on four small pieces in `noxrecon/core/`.

## BaseModule — the contract

Every module subclasses `BaseModule` and declares four class attributes plus a `run()` method:

```python
from noxrecon.core.base import BaseModule


class MyModule(BaseModule):
    NAME = "mymodule"          # used with -m
    DESCRIPTION = "What it does"
    REQUIRES = ("nmap",)       # optional external tools
    CATEGORY = "ports"         # workspace subdirectory
    ORDER = 40                 # position in --all pipeline

    def run(self):
        ...
        self.save_result("mymodule.json", data)
        return data
```

The loader discovers any Python file under `noxrecon/modules/` (and installed plugins under `plugins/`), imports it dynamically, sorts classes by `ORDER`, and executes them in sequence.

> 💡 **Why ORDER matters**
>
> Recon has natural dependencies: you can't reverse-resolve an IP before DNS resolves it, and banner grabbing needs ports from the portscan. Encoding those constraints as numeric ordering keeps the pipeline deterministic without hard-wiring modules to each other.

## Dependency registry

`noxrecon/core/dependency.py` maps tool names to apt packages or Go install commands, then checks PATH plus extra bin dirs (`~/go/bin`, `/usr/local/go/bin`):

```python
REGISTRY = {
    "nmap": ("nmap", None),
    "subfinder": (None, "go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest"),
    "naabu": (None, "go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest"),
    ...
}
```

`python3 noxrecon.py deps` prints a live status table — and because modules declare `REQUIRES`, the checker knows exactly which tool is needed by which module.

## Workspace & reporter

Each run creates `results/<target>/` with one folder per category (`dns/`, `ports/`, `web/`, …), a full `recon.log`, `summary.json`, and the final `report.md`. The reporter renders narrative sections — executive summary, tables, emoji callouts — directly from module JSON, so reports stay consistent regardless of who runs the scan.

---

# Phase 2 — Writing the Modules

## Built-in modules (18)

| Order | Module | Category | What it answers |
|---|---|---|---|
| 10 | `dns` | dns | Where does the target live? (A, AAAA, MX, NS, TXT, SOA, CNAME) |
| 11 | `reversedns` | reversedns | Which hosting infrastructure hides behind the IP? |
| 20 | `whois` | whois | Who registered it, and until when? |
| 30 | `subdomain` | subdomains | What other attack surfaces exist? (crt.sh, certspotter, bufferover, hackertarget, rapiddns, subfinder) |
| 31 | `crtsh` | subdomains | What do certificate transparency logs reveal? |
| 40 | `portscan` | ports | Which services are exposed, at which versions? |
| 41 | `banner` | banner | What do raw banners leak? |
| 42 | `naabu` | ports | What's listening beyond nmap's top-18? |
| 50 | `web` | web | What does the HTTP surface look like? |
| 52 | `waf` | web | Is there a WAF filtering requests before they arrive? |
| 53 | `cdn` | web | Is the origin hidden behind an edge network? |
| 54 | `tech` | web | What stack powers the application? |
| 56 | `ssl` | ssl | Is TLS healthy — expiry, issuer, chain, SANs? |
| 60 | `osint` | osint | What emails and metadata are public? |
| 61 | `cms` | cms | Is a known CMS running? |
| 66 | `vhost` | vhost | Does the server answer to other virtual hosts? |
| 71 | `s3` | s3 | Do cloud buckets named after the target exist? |
| 76 | `geoip` | geoip | Where is it hosted, and who owns the ASN? |
| 81 | `email` | email | What contact addresses are harvested? |
| 82 | `amass` | osint | How does the network graph connect related domains? |
| 83 | `urlenum` | osint | Which historical URLs did Wayback/Common Crawl remember? |

Plus installable plugins: `nuclei` (template-based vuln scanning) and `ffuf` (content fuzzing).

> 💡 **Why so many subdomain sources?**
>
> Each source sees a different slice of reality. crt.sh indexes CT logs, rapiddns scrapes passive DNS, subfinder aggregates dozens of APIs. Merging them and deduplicating consistently finds more than any single source — and recording per-source counts tells me which sources actually earn their network calls.

## The two-pass portscan

The most interesting engineering problem was the port scanner. NSE vulnerability scripts (`--script vuln`) can take minutes on slow hosts — long enough that their `--host-timeout` would wipe out the basic version-scan results too.

The fix: two separate passes, first pass never overwritten.

```text
Phase 1: nmap -Pn --open -sV -T4 --host-timeout 180s   → ports + versions (always kept)
Phase 2: nmap --script vuln on discovered open ports    → CVE references (best effort)
```

If phase 2 times out, the report still shows services and versions — just without CVE annotations. That single design decision removed an entire class of "empty report" failures.

---

# Phase 3 — Testing Against scanme.nmap.org

With the pipeline assembled, I pointed it at `scanme.nmap.org` — Nmap's official public scanning target — using all modules:

```bash
python3 noxrecon.py -t scanme.nmap.org --all
```

Total runtime: about 8 minutes, most of it spent in `amass` graph mapping.

## Results highlights

**DNS & hosting**

| Type | Value |
|---|---|
| A | `45.33.32.156` |
| AAAA | `2600:3c01::f03c:91ff:fe18:bb2f` |
| PTR | `scanme.nmap.org` |
| ASN | AS63949 Akamai Connected Cloud (Linode) |
| Location | Fremont, United States |

**Ports & services**

| Port | Service | Version | Vulnerability check |
|---|---|---|---|
| 22 | ssh | OpenSSH 6.6.1p1 Ubuntu-2ubuntu2.13 | 🟢 none flagged |
| 80 | http | Apache httpd | 🟢 none flagged |

**Other findings**

- **urlenum**: 82 historical URLs recovered via `gau` (Wayback Machine had almost nothing — Common Crawl carried the day).
- **nuclei**: 1 informational finding — SSH authentication method detection on port 22.
- **cloud buckets**: probe buckets named after the target across AWS S3 / Azure / GCP — all private or non-existent.
- **WAF / CDN**: none detected — origin directly reachable, exactly as expected for a deliberately-exposed lab host.

> ✅ **Sanity check passed**
>
> The automated report matched what manual recon would have produced — same IP, same services, same exposure profile — but generated in eight minutes without a single hand-typed command.

## What the report looks like

The generated `report.md` opens with an executive summary, then walks through each layer with explanation paragraphs rather than raw dumps:

```markdown
## Port & Service Discovery

An Nmap version scan (-sV) probed the most common TCP ports and identified
2 open services. A second pass ran the NSE `vuln` script category against
each open port to surface known CVEs.

| Port | Service  | Product / Version   | Vulnerability check |
|------|----------|---------------------|---------------------|
| 22   | OpenSSH  | OpenSSH 6.6.1p1     | 🟢 none flagged     |
| 80   | Apache   | Apache httpd        | 🟢 none flagged     |
```

Because the text is generated from structured findings, every future improvement to a module automatically improves every future report.

---

# Phase 4 — Lessons Learned While Building It

## What went right

- **Stdlib-only Python** meant zero installation friction — clone and run.
- **Graceful degradation** paid off immediately: during one test, crt.sh returned 502 errors for hours; the framework logged the failure, fell back to certspotter, and the run completed normally.
- **Per-target workspaces** made comparing two scans of the same host trivially diffable.

## What surprised me

- **Tool identity crises.** The Python library `httpx` and ProjectDiscovery's `httpx` share a name but share nothing else — my dependency checker happily found the wrong one. Fixed by pinning the Go binary path explicitly.
- **Banner lies.** One target returned `Server: CW` on every web port. Chasing that string taught me about CloudWAF-style protection layers that mask origin server versions — a reminder that version banners are claims, not facts.
- **Timeouts are data.** A module timing out isn't noise to silence; it's a finding about the target (filtered, rate-limited, or protected).

## Red Team Perspective

A modular framework turns recon from a memorized command list into a repeatable methodology. When the pipeline is code, coverage gaps become visible — you can literally see which questions your process never asks.

## Blue Team Perspective

The same automation that helps attackers helps defenders: running this against your own estate shows exactly what a recon pass would hand to an attacker — exposed versions, forgotten buckets, missing WAF — before someone less friendly finds them.

---

# Cheat Sheet

## Installation

```bash
git clone https://github.com/NoxShepherd/NoxRecon.git
cd NoxRecon

sudo bash install_deps.sh     # core: nmap, dig, whois, curl, jq, whatweb + Go tools
sudo bash install_extras.sh   # extras: naabu, waybackurls, gau (+ amass)
```

## Running scans

```bash
# everything
python3 noxrecon.py -t scanme.nmap.org --all

# specific modules only
python3 noxrecon.py -t example.com -m dns -m portscan -m ssl

# interactive wizard
./noxrecon.sh

# dependency status
python3 noxrecon.py deps
```

## Plugin management

```bash
python3 noxrecon.py module install nuclei   # vuln scanning
python3 noxrecon.py module list             # verify
python3 noxrecon.py -t example.com -m nuclei
```

---

# Conclusion

Building NoxRecon taught me more about reconnaissance than simply learning another five tools ever could. Wiring sources together forced me to understand what each one actually measures, where it fails, and how its output should shape the next decision.

The framework is still young — more modules, better reporting, and cleaner fallbacks are on the roadmap — but it already does the job it was built for: turning a pile of disconnected commands into a repeatable, explainable methodology.

Repository: **[github.com/NoxShepherd/NoxRecon](https://github.com/NoxShepherd/NoxRecon)**

---

**Thanks for reading!** If you build something because of this post — or break my framework and find a bug — I'd genuinely love to hear about it.

Happy Hacking! 🚀
