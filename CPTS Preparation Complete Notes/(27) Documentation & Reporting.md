
**Well, include everything from a to z, organized.**
## Why it matters (the theory)

Your notes + raw tool output are the **raw material for the report** — and the report is usually the only thing the client ever sees. So:

- Stay **organized** and have a **repeatable process** (saves time at report-writing time).
- Be **verbose** — over-noting never hurts. Notes save you if there's a network issue or the client asks "did you scan X host on Y day?".
- Clear notes let a **teammate cover for you** (e.g. a client meeting) and speak accurately about what was and wasn't done.

## What to note (the sample structure)

Not every project needs all of these — adapt it. Common sections:

- **Attack Path** — the full path of how you got in / compromised hosts, with screenshots + command output (easy to paste into the report later).
- **Credentials** — one central place for all creds/secrets you grab.
- **Findings** — a subfolder per finding, with the write-up + evidence (screenshots, output).
- **Vulnerability Scan Research** — what you researched/tried from scans (so you don't redo work).
- **Service Enumeration Research** — services checked, failed exploits, promising leads.
- **Web Application Research** — interesting web apps, default creds tried, etc.
- **AD Enumeration Research** — step-by-step AD enum done + areas to revisit.
- **OSINT** — interesting info collected (if in scope).
- **Administrative Information** — contacts (PMs, client POCs), objectives/flags from the RoE, running to-do list.
- **Scoping Information** — in-scope IPs/CIDRs, URLs, provided creds (so you don't stray out of scope).
- **Activity Log** — high-level log of everything you did (for event correlation).
- **Payload Log** — every payload used (+ file hash and where it was uploaded).

## Notetaking tools

- **Obsidian** — great for **local** storage (best when handling client data).
- **Outline** — great for **cloud** (also has a self-hosted version).
- Both export to **Markdown** (easy to move between tools).

Rule of thumb: **local tool for real engagements** (client data policies), **cloud is fine for training/CTF/labs**. Check your company policy first.

## Logging (keep the raw output)

- **Log ALL scans and attack attempts** and keep raw tool output — it backs up your notes if you miss something or the client questions your work.
- Also track **what you tried that did NOT work** — useful for event correlation, and for writing a "here's what we tested" narrative when a report has few findings.
- **Tmux logging** saves everything typed/shown in a pane to a file (full setup below).

## Evidence — what to capture

- Prefer **terminal text output over screenshots** — easier to redact, neater in the doc, and the client can copy-paste to reproduce.
- Use **screenshots** for GUI things.
- Capture **significant steps** as you go — and even **failed tests** (proves you were thorough).

## Folder structure (storage)

A clean baseline layout to keep evidence organized:

```
ACME-IPT/
├── Admin
├── Deliverables
├── Evidence
│   ├── Findings
│   ├── Logging output
│   ├── Misc Files
│   ├── Notes
│   ├── OSINT
│   ├── Scans
│   │   ├── AD Enumeration
│   │   ├── Service
│   │   ├── Vuln
│   │   └── Web
│   └── Wireless
└── Retest
```

Make it all in one command:

```bash
mkdir -p ACME-IPT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}
```

## What NOT to archive (do no harm)

- Don't crash hosts, change passwords (unless allowed), or make hard-to-reverse changes.
- Don't exfiltrate real **PII / sensitive data** (GDPR etc. compliance headaches). If you find a share of sensitive files, **screenshot the directory listing**, not the file contents.
- Get **written client approval** before creating accounts or changing system settings.

## Track your changes (accounts + payloads)

For any **payload**: when used, which host, the file path on target, a file hash, and whether it was cleaned up.

For any **account you create / system change**: host IP + hostname, timestamp, description of change, location on host, app/service touched, account name (and password, in case you must hand it over).

---

# TMUX — full setup + logging (step by step)

Tmux logging saves everything in a pane to a log file — the standard way to log an assessment. Default **prefix key = `Ctrl + B`** (press it, release, then press the next key).

## Setup (one time)

1. Clone the Tmux Plugin Manager into your home folder:

```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

2. Create the config file:

```bash
touch ~/.tmux.conf
```

3. Put this inside `~/.tmux.conf`:

```bash
# List of plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-logging'

# bigger scrollback so retroactive logging keeps more
set -g history-limit 50000

# Initialize TMUX plugin manager (keep at bottom)
run '~/.tmux/plugins/tpm/tpm'
```

4. Load the config into your current session:

```bash
tmux source ~/.tmux.conf
```

5. Start a new session:

```bash
tmux new -s setup
```

6. Install the plugins — inside the session press: `Ctrl + B`, then `Shift + I` (capital i). Wait ~5 seconds.

## Logging a session

Start logging: `Ctrl + B`, then `Shift + P`. The bottom of the window shows logging is on + the output file.

Stop logging: `Ctrl + B`, then `Shift + P` again (or `exit` the session).

Important: the **log file only fills up once you stop logging or exit** the session.

## Handy extras

Forgot to log and already deep in? **Retroactive logging** (saves the whole pane buffer):

Press `Ctrl + B`, then `Alt + Shift + P`. (How far back it goes depends on `history-limit` — that's why we set 50000.)

**Screen-capture one pane** to a file (clean, avoids grabbing text from the other pane):

Press `Ctrl + B`, then `Alt + P`.

**Clear a pane's history:** `Ctrl + B`, then `Alt + C`.

## Working with panes

Split vertically: `Ctrl + B`, then `Shift + %`

Split horizontally: `Ctrl + B`, then `"`

Move between panes: `Ctrl + B`, then `O`

## Key combos — quick reference

| Action | Keys (after Ctrl + B) |
|--------|----------------------|
| Install plugins | Shift + I |
| Start / stop logging | Shift + P |
| Retroactive (save whole buffer) | Alt + Shift + P |
| Screen-capture one pane to file | Alt + P |
| Clear pane history | Alt + C |
| Split pane vertical | Shift + % |
| Split pane horizontal | " |
| Move between panes | O |

## Nice extra plugins

- **tmux-sessionist** — manage sessions from inside a session (switch, create, kill, etc.).
- **tmux-pain-control** — easier keybindings for moving/resizing/splitting panes.
- **tmux-resurrect** — restore all sessions/windows/panes (and running programs) after a reboot.

---

# Color-coded terminal + redaction (for the report)

## Color-coding (makes evidence readable)

In your terminal evidence, color-code so the reader instantly sees what matters:

- the **command you ran** in one color (e.g. blue)
- the **interesting output** in another (e.g. red — like a successful login)

This helps non-experts (and the client's team) pick out the key parts and reproduce it.

**How to actually do it:**

- **tmux logs do NOT save colors** — they're plain text (that's good, it's your raw record). You add color later, only to the snippets you put in the report.
- You color it **by hand, per line**, in the report tool:
  - Word → select the text → Home → Font Color → blue for the command, red for the key output.
  - Markdown → use HTML spans: `<span style="color:blue">your command</span>` (only in renderers that allow HTML).
- Only color the **important lines** (the command + the one output line that proves it), not the whole thing.
- To keep a tool's OWN colors (like nmap's) → pipe through **aha**: `nmap -sV <ip> | aha > nmap.html` → opens as colored HTML.

## Redaction (hide secrets safely)

- Redact **credentials and PII** in any evidence.
- Use **solid black bars** over the text — edit the **image directly** (not a shape in Word, which can be deleted). 
- Do NOT use **blur/pixelation** — it can be reversed (tools like Unredacter recover the original).
- For a web page, don't rely on CSS tricks (black text on black bg) — it's visible via highlight / page source.

What to redact and how, with examples:

**Cleartext password** → replace with a placeholder.

Before:
```
evil-winrm -i 10.129.43.33 -u pwn -p 'Pwn123!'
```
After:
```
evil-winrm -i 10.129.43.33 -u pwn -p '<REDACTED>'
```

**Password hash** → keep the first + last 3-4 characters, cut the middle (shows a hash was there without exposing it).

Before:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:5835048ce94ad0564e29a924a03510ef:::
```
After:
```
Administrator:500:aad3...04ee:5835...10ef:::
```

**Screenshot with a secret** → cover it with a **solid black bar baked into the image** (draw a filled rectangle in GIMP/Greenshot). NOT blur (reversible), NOT a Word shape (deletable).

**PII** (names, emails, IDs) → `<REDACTED>` in text, solid black bar in images.

## Terminal output tips

- Prefer **terminal text over screenshots** (easier to redact, neater, copy-pasteable, keeps the report file small).
- You may **cut unnecessary output** and mark it with `<SNIP>` — but **never alter** the actual command or output (the client must be able to reproduce it exactly).
- **Strip formatting** before pasting into Word — pasted "smart quotes"/apostrophes are non-UTF-8 and can break the command when the client copies it.

---

**Core idea:** notes + logs are the raw material for the report (the thing the client pays for), so be organized and verbose. Keep a set note structure (Attack Path, Creds, Findings…), a tidy folder layout, and log everything with **tmux logging** (`Ctrl+B Shift+P`). Capture clean evidence — prefer terminal text, color-code command vs key output, redact secrets with **solid black bars** (never blur), hashes trimmed, cleartext as `<REDACTED>` — and never alter output (use `<SNIP>`).


----

Penetration testing reports depend on the type of assessment, such as vulnerability assessments, internal or external penetration tests, web application testing, cloud testing, IoT testing, or hardware testing. A vulnerability assessment mainly identifies and validates weaknesses without exploiting them, while a penetration test attempts to exploit vulnerabilities to understand their real impact. Reports usually go through a draft review with the client before the final report is issued. After remediation, only the original findings and affected systems should normally be retested within an agreed timeframe. Other deliverables may include attestation letters for third parties, presentation slides, spreadsheets of findings, and urgent vulnerability notifications for critical issues that need immediate attention.


---
---

# AD Attack Chain (methodology) + Report Basics — CPTS Notes

Two parts: (1) a full internal AD attack chain written as a reusable methodology — how you go from "no creds on the network" to "Domain Admin," step by step, with the tool for each step and WHY it works. (2) a short summary of what this reporting section teaches.


# Part 1 — The AD attack chain (no creds → Domain Admin)

The whole idea: each step gives you something (a hash, a password, more access) that unlocks the next step. You keep hopping to more powerful accounts until you own the domain.

## Step 1 — Grab a first hash with no creds (Responder)

- Where you are: on the internal network, **no account at all**.
- Do: run **Responder** to poison LLMNR/NBT-NS name requests → catch a domain user's (`bsmith`) **NTLMv2 hash**.
- Why: it's the easiest way to get *something* when you have zero creds — machines leak hashes trying to find network names.

Responder runs on **YOUR attack box** (not the victim) — you don’t log in or SSH anywhere, you just connect to the network (HTB = the VPN / `tun0`), run `sudo responder -I tun0`, and **wait** for a victim to leak its hash to you.

## Step 2 — Crack the hash → first foothold (Hashcat)

- Where you are: you have a hash (can't log in with it directly here).
- Do: crack `bsmith`'s hash offline with **Hashcat** → get the **cleartext password**.
- Why: a cleartext password = a real login = a **foothold as a normal domain user** (low privilege, but you're *in*).

## Step 3 — Map the domain, find the path (BloodHound.py)

- Where you are: normal domain user.
- Do: run **BloodHound.py** to collect users, groups, ACLs, sessions, admin rights → see visual **attack paths**.
- Found: several accounts have **SPNs** (so they're Kerberoastable), and `mssqlsvc` is **local admin on SQL01**.
- Why: don't attack blindly — find which accounts are weak AND worth it.

## Step 4 — Kerberoast the juicy service account (GetUserSPNs.py / Impacket)

- Where you are: normal user + a target (`mssqlsvc`).
- Do: use **GetUserSPNs.py** to request `mssqlsvc`'s Kerberos **TGS ticket** (Kerberoasting).
- Why: **any** domain user can request a service ticket, and it's encrypted with the service account's password hash → so it's crackable offline.

## Step 5 — Crack the ticket (Hashcat)

- Do: crack the `mssqlsvc` ticket with **Hashcat** → cleartext password.
- Why: now you hold creds for an account that is **local admin on SQL01** (a step up from a normal user).

## Step 6 — Loot SQL01 for better creds (CrackMapExec)

- Where you are: `mssqlsvc` = local admin on SQL01.
- Do: log into SQL01 with **CrackMapExec** and **dump LSA secrets** → find the cleartext password of `srvadmin` (an **autologon** account whose creds sit in the registry).
- Why: privileged machines store other accounts' creds. `srvadmin` is more powerful than `mssqlsvc`.

## Step 7 — Hop to a server with a privileged user (RDP)

- Where you are: `srvadmin` = local admin on **ALL servers except Domain Controllers**.
- Do: **RDP** into MS01 → notice user `pramirez` is also logged in. BloodHound shows `pramirez` has **DCSync rights**.
- Why: srvadmin's wide admin access lets you pick a box where an even more privileged user is logged in and steal from them.

## Step 8 — Steal the privileged user's ticket (Rubeus)

- Where you are: admin on MS01, where `pramirez` is logged in.
- Do: use **Rubeus** to dump `pramirez`'s Kerberos **TGT**, then **pass-the-ticket** to become `pramirez`.
- Why: you don't need pramirez's password — their ticket in memory is enough to act as them. And pramirez can DCSync.

## Step 9 — DCSync → domain compromise (Mimikatz)

- Where you are: acting as `pramirez` (has DCSync rights).
- Do: run **Mimikatz** `dcsync` → pull the **Administrator NTLM hash**.
- Why: DCSync pretends to be a Domain Controller and asks for password data — this gives you **any user's hash = full domain compromise**.

## Step 10 — Dump the whole domain (secretsdump)

- Do: use **secretsdump** to dump **all domain hashes (NTDS)**, then crack offline with Hashcat.
- Why: full control + password analysis for the report.

## The chain in one line

No creds → **Responder** hash → **Hashcat** crack → domain user → **BloodHound** map → **Kerberoast** mssqlsvc → crack → SQL01 local admin → **LSA dump** srvadmin → admin on all servers → steal **pramirez** TGT (Rubeus) → **DCSync** (Mimikatz) → **Domain Admin** → dump NTDS.

Tools per job:

| Job | Tool |
|-----|------|
| Capture a hash (no creds) | Responder |
| Crack hashes/tickets | Hashcat |
| Map the domain / attack paths | BloodHound.py |
| Kerberoasting | GetUserSPNs.py (Impacket) |
| Dump creds from a host (LSA) | CrackMapExec |
| Steal / pass Kerberos tickets | Rubeus |
| DCSync / dump creds | Mimikatz |
| Dump the whole NTDS database | secretsdump |

---

# Part 2 — The reporting part

Don't overthink this. A pentest report = **telling the story of what you found and how to fix it**, written for **TWO readers**:

1. The **money boss** (not technical) → reads the **Executive Summary**, in plain "explain to your mom" words.
2. The **tech team** (who actually fix it) → reads the **Findings**, with technical proof + how to fix.

That's the whole module in one line.

## A report = 4 chunks

| Chunk | What it is (plain words) | Who reads it |
|-------|--------------------------|--------------|
| Executive Summary | "Here's what's wrong and how bad, in simple words." | The boss / budget person |
| Attack Chain | "The story of how I broke in, step by step." (Part 1 above) | Both |
| Findings | "Each problem + proof + how to fix it." | The tech team |
| Appendices | Extra lists (scope, methodology, creds found, changes made). | Auditors / tech team |

## The 3 rules that matter most

1. **Executive Summary = no jargon.** If your mom wouldn't get it, rewrite it. Say "I got into the HR and banking systems," NOT "I achieved Domain Admin via Kerberoasting."

2. **Be specific.** Say "25 accounts," not "several accounts."

3. **Don't dump everything.** No 50 pages of raw output. Show the important bits, redact the secrets.

## Small extra bits the module mentions

- **Prioritize** — focus on the high-impact stuff (RCE, sensitive data). Still report the small issues, but group them instead of wasting time on each.

- **Translate tech into plain English** for the Executive Summary:
  - VPN / SSH = a way to remotely control a computer securely
  - Hash = scrambled output used to check a file or password
  - Password spraying = trying one common password against many accounts
  - Password cracking = turning a scrambled password back into readable text
  - OSINT = gathering public info about a company without touching their network

- **Summary of Recommendations** — a short list of fixes (short / medium / long-term), each tied to a finding = the client's to-do list.

- **Appendices** — the extras: always Scope + Methodology + Severity Ratings; sometimes creds you found, changes you made, payloads used, OSINT, password-cracking stats.

- **Report type changes the focus** — internal (full attack chain) vs external (OSINT + exposed services) vs web app (OWASP Top 10) vs red team (story style). Keep a template for each.

---

---


# How to Write Up a Finding 

The Findings section is the **meat** of the report — each problem you found, how you exploited it, and how to fix it. Write it detailed so the client's tech team can **reproduce** the issue and then **test that their fix worked**.

Tip: if you reuse "stock" findings from a database, always **tweak them to the client's exact situation** — don't misrepresent. "Default credentials" means something very different on a printer vs the building's HVAC vs a key web app.

## What every finding must include

| Part | What goes here |
|------|----------------|
| Description | What the issue is + which platforms it affects |
| Impact | What happens if it's left unfixed |
| Affected systems | The hosts / networks / apps hit (or the whole domain) |
| Recommendation | How to fix it |
| References | Links to learn more + how to resolve |
| Reproduction steps + evidence | Step-by-step proof you collected |

Optional extras (if your template uses them):

- CVE number
- OWASP / MITRE IDs
- CVSS score (or similar)
- Ease of exploitation + how likely an attack is

## How to show reproduction steps well

Assume the reader has **never seen the tool** and won't understand its wall of output. So:

- **One step = one figure (screenshot).** Don't cram multiple steps into one image — the reader can't follow it.

- If setup is needed (e.g. a Metasploit module), show the **full config first** (so they see what it should look like), then a **second figure** of running it.

- **Write a short narrative between figures** — say what's happening and what you were thinking. Don't just stack screenshots with captions.

- **Offer alternative tools** at the end (just name them + a link — don't run the whole exploit twice).

## Make evidence copy-pasteable AND undeniable

- **Copy-pasteable:** if it's a web payload, give the **text** of the request, not just a Burp screenshot — the client needs to copy it to reproduce.

- **Defensible (leaves no room for debate):**
  - To prove **cleartext transmission**, don't just screenshot the login popup (that only shows basic auth exists). Show the creds in a **Wireshark packet capture** — that's the real proof.
  - To prove it's **their host** (not a random image), capture the **URL in the address bar** or `ifconfig`/`ipconfig` output.

- **Screenshot hygiene:** turn OFF your bookmarks bar, disable unprofessional browser extensions (or use a dedicated testing browser), and **redact credentials** (the report gets passed around).

## Writing good remediation recommendations

Be **specific and actionable** — do the homework for them.

Example 1 — registry fix:

- Bad: "Reconfigure your registry settings to harden against X."
- Good: list the **full path to each registry hive** + the exact value to change, AND add a caution ("test on a small group first, registry changes are risky").

Example 2 — don't force expensive tools:

- Bad: "Buy [expensive commercial tool] to fix this."
- Good: give the **vendor's free workaround** as the main fix, and mention commercial tools only as an optional (possibly cost-prohibitive) alternative.

Why bother being this detailed:

- You learn more (and can answer the client's questions confidently).
- The client appreciates it → more likely to rehire + recommend you.
- Adding caution warnings shows you have their best interests in mind (some clients blindly do whatever you say and will blame you if it breaks something).

## Picking good references

Each finding should link 1+ sources for further reading. A good reference is:

- **Vendor-agnostic** — not a product sales pitch (vendor articles push their tool instead of just fixing it).
- **Thorough + free** — no paywalls, no "only half the answer."
- **To the point** — no fluff, not "go read the entire NIST 800-53."
- **Clean, safe site** — no ad spam / sketchy pop-ups.
- **Best of all:** write your own blog post and link that.

## What a BAD finding looks like (avoid)

- Sloppy formatting (e.g. broken reference link)
- Missing CVSS score (if your template uses it, fill it in)
- Description doesn't explain the issue or root cause
- Impact is vague and generic
- Remediation is unclear / not actionable

The reader seeing "it's red = bad" is not enough. Each finding must explain **why they should care** and **exactly what to do about it** (they may have never heard of Kerberoasting or whatever the attack is).

## Formatting choice

Two common layouts, both fine:

- **Tables** (looks clean, but a pain in Word / to automate)
- **Heading levels** (separate each part with a heading)

What actually matters = **readability** and that it's easy to see **where one finding ends and the next begins**. Colors/layout/order can be whatever works.

## Tool mention

**WriteHat** (by Black Lantern Security) = a report-writing tool for building a findings database and generating reports. Most reporting tools are similar — worth playing with one to see how they work.

---
---

# Reporting Tips and Tricks

## 1. Write the Report During Testing

Do not wait until the assessment ends to write the report. Record findings, attack chains, commands, screenshots, and evidence as you work. This saves time and prevents missing important details.

## 2. Use Report Templates

Create a separate template for each assessment type. Use placeholders for client name, dates, scope, and testing details. **Never reuse another client's report directly**, because old names or confidential information may remain.

## 3. Microsoft Word Tips

|Feature|Why use it?|
|---|---|
|Font and table styles|Keep formatting consistent and update it everywhere at once|
|Captions|Automatically renumber figures and tables|
|Page numbers and Table of Contents|Help readers navigate the report|
|Bookmarks|Link to specific sections and help automate templates|
|Custom numbering|Number findings and appendices consistently|
|`Ctrl + S`|Save regularly|
|`Ctrl + A`, then `F9`|Update document fields, including the Table of Contents|
|`F4`|Repeat the previous action|

Use consistent formatting and check that screenshots, links, captions, and page numbers work correctly.

## 4. Automate Reporting

Word macros can automatically fill in client details, dates, and scope, remove sections that do not apply, and correct common formatting mistakes. Automation saves time and makes reports more consistent.

## 5. Maintain a Findings Database

Keep reusable, sanitized examples of common findings and remediation advice. Customize them for each client rather than copying them blindly.

Reporting tools include **Ghostwriter, Dradis, and WriteHat** (free options), and **AttackForge and PlexTrac** (paid options).

## 6. Important Reporting Tips

- Tell a clear story: explain how vulnerabilities connect and what impact they create.
    
- Include enough evidence to prove and reproduce findings, but avoid unnecessary screenshots and output.
    
- Clearly label and explain screenshots.
    
- Redact passwords, hashes, tokens, and other sensitive information.
    
- Keep grammar, spelling, terminology, and formatting consistent.
    
- Use professional hostnames and usernames in screenshots.
    
- Back up notes and evidence regularly.
    
- Have someone else review the report whenever possible.
    

## 7. Communicate With the Client

At the start of an assessment, send a notification containing the tester's name, scope, source IP, testing dates, and contact details. Send a stop notification at the end of each testing day.

Keep the client informed about critical findings, unexpected hosts, possible scope changes, and major compromises. Maintain detailed notes and tool logs so you can prove what was tested and when.

## 8. Draft, Final Report, and QA

The report should go through **Quality Assurance (QA)** before delivery. A reviewer should check technical accuracy, evidence, grammar, and formatting.

Send the client a draft to review and discuss. Make agreed changes, then deliver the final report. Archive the testing data according to the organization's retention policy.

## 9. Retesting After Remediation

After the client fixes vulnerabilities, retest **the original findings on the originally affected systems** within an agreed timeframe. Do not automatically repeat the entire assessment or add newly discovered systems to the retest scope.

Record each finding's status, such as resolved, unresolved, or partially resolved. If the environment has changed significantly, explain that limitation clearly in the report.


You can use the report template provided with this module, find sample pentest reports on GitHub, or use the report templates included in tools such as [WriteHat](https://github.com/blacklanternsecurity/writehat), [Pwndoc](https://github.com/pwndoc/pwndoc), or something similar. 

---
---


## Skill Assessment


```
sudo Responder -I tun0 -wrfv

evil-winrm -i 172.16.5.5 -u BACKUPGENT -p 'Recovery7'

secretsdump.py BACKUPAGENt:Recovery7@172.16.5.5

sudo bloodhound-python -u 'BACKUPAGENT' -p 'Recovery7' -d inlanefreight.local ns 172.16.5.5 -c All

bloodhound

```

[Documentation & Reporting HackTheBox AcademyWriteup | by Logen Autry | FuguSecurity | Medium](https://medium.com/fugusecurity/documentation-reporting-hackthebox-academywriteup-b41718bb26b6) 