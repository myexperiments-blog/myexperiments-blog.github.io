# Local linux privilege escalation

https://www.sans.org/reading-room/whitepapers/testing/attack-defend-linux-privilege-escalation-techniques-2016-37562

unix-privesc-check


- [File Permission](##File-Permission)
- [Cron](##Cron)


## Summary

- [Introduction](#Introduction)
- [Kernel](##Kernel)
- [Deamon & Process](##Deamon-&-Process)
- [Password mining](##Password-mining)
- [Sudo](##Sudo)
- [Cron](##Cron)
- [File Permission](##File-Permission)
- [Other usefull information](#Other-usefull-information)
- [Users & Groups](##Users-&-Groups)

## Introduction

A local linux privilege escalation happens when one user acquires the system rights of another user.
It result that a user can do actions that he normally cannot do.
The goal we try to reach in a privilege escalation attack is most of the time to gain the root access.

This article separate the local linux privilege escalation in different scopes: kernel, deamon and process, password mining, sudo, cron and file permission. Foreach I will gave you the usefull commands for information gasthering, I will explain how to gain an escalation privilege with the information you collect, and finnaly I will give you the remediation.

Do not hesitate to share with us your techniques in the comments.

## Kernel

Kernel exploits are programs that leverage kernel vulnerabilities in order to
execute arbitrary code with elevated permissions. Successful kernel exploits typically
give attackers super user access to target systems in the form of a root command prompt.
In many cases, escalating to root on a Linux system is as simple as downloading a kernel
exploit to the target file system, compiling the exploit, and then executing it.
While information about kernel exploits is well documented, it still remains a
significant problem for Linux users as new kernel exploits are uncovered on a regular
basis. One kernel exploit, Dirty COW, received a great deal of attention because of its
severe and widespread impact on millions of Linux devices.

At the time of this writing, patches that resolve the Dirty COW vulnerability are
available for most mainstream Linux distributions, including Ubuntu, Debian,
RHEL/CENTOS, ARCH, and Gentoo. Simply patching the vulnerable kernel will negate
the Dirty COW exploit entirely; therefore, the easiest way to defend against kernel
exploits is to keep the kernel patched and updated. That said, there are significant
challenges to remediating kernel vulnerabilities through patching alone. First, routine
patching will not impede a cutting edge zero-day exploit. Second, the sheer volume of
potentially vulnerable devices makes this a difficult problem to solve through patching
alone. Consider that Internet of Things and embedded devices are at risk of being
overlooked despite the fact that they are likely targets of exploitation. These specialized
devices may not even have patches to address critical vulnerabilities. In these cases,
administrators should focus on negating the attack vector. Consider that for a kernel
exploit attack to succeed, an adversary requires four conditions:

1. A vulnerable kernel
2. A matching exploit
3. The ability to transfer the exploit onto the target
4. The ability to execute the exploit on the target

In the absence of patches, administrators can strongly influence conditions 3 and
4. Given these considerations, kernel exploit attacks are no longer viable if an
administrator can prevent the introduction and/or execution of the exploit onto the Linux
file system. Therefore, administrators should focus on restricting or removing programs
that enable file transfers, such as FTP, TFTP, SCP, wget, and curl. When these programs
are required, their use should be limited to specific users, directories, applications (such
as SCP), and specific IP addresses or domains. Furthermore, the activities of users with
these application permissions should be logged and monitored. These conditions create
significant obstacles for an adversary to contend with and also provide administrators
early warnings that can enable rapid detection of compromise. Next, administrators
should consider restricting or removing compilers, such as gcc and cc, if they are not
required. Attackers often compile kernel exploits on the target system to ensure their
exploits will work. Removing compilers will impede attackers; however, be warned that
determined attackers may still compile the exploit on a system that is identical or closely
resembles the target. Finally, administrators should limit directories that are writeable and
executable, particularly by service users such as www-apache. If an attacker does not
have the ability to execute his exploit, the attack is rendered ineffective. Proper
implementations of these configurations severely impedes an attacker’s ability to deploy
kernel exploits, ultimately reducing the attack surface and enhancing the Linux system’s
security posture. 

### Kernel information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `uname -a` | Print all available system information   |
| `uname -m` | Linux kernel architecture (32 or 64 bit) |
| `uname -r` | Kernel release |
| `uname -n` or `hostname` | System hostname |
| `cat /proc/version` | Kernel information |
| `cat /etc/*-release` or `cat /etc/issue` | Distribution information |
| `cat /proc/cpuinfo` | CPU information |
| `df -a` | File system information |

### Kernel escalation

### Escalation

### Remediation

## Deamon, process and programs

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `ps aux | grep root` | Display service run as root |
| `ps aux | awk '{print $11}'|xargs -r ls -la -- 2>/dev/null | uniq` | Display process binary permission |
| `cat /etc/inetd.conf` or `cat /etc/xinetd.conf` | List services managed by inetd |
| `dpkg -l` | 	Installed packages (Ubuntu, Debian) |
| `rpm -qa` | Installed packages (RedHat, Fedora Core, Suse Linux, Cento) |
| `pkg_info` | Installed packages (OpenBSD, FreeBSD) |
| `httpd -v` or `apache2 -v` | Apache version |
| `apache2ctl (or apachectl) -M` | List loaded Apache modules |
| `cat /etc/apache2/envvars 2>/dev/null |grep -i 'user|group' |awk '{sub(/.*export /,"")}1'`| Which account is Apache running as |
| `mysql --version` | MYSQL version |
| `psql -V` | Postgres version |
| `perl -v` | Perl version |
| `java -version` | Java version |
| `python --version` | Python version|
| `ruby -v` | Ruby version |


### Escalation

### Remediation

## Password mining

- Logs
- Memory
- History
- Configuration files

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `history` | Displays command history of current user |
| `history | grep -B4 -A3 -i 'passwd\|ssh\|host\|nc\|ping' 2>/dev/null` | Check history for interesting information |
| `grep -B3 -A3 -i 'pass\|password\|login\|username\|email\|mail\|host\|ip' /var/log/*.log 2>/dev/null` | Check log file in `/var/log` for password, login, or email information |
| `find / -maxdepth 4 -name '*.conf' -type f -exec grep -Hn 'pass\|password\|login\|username\|email\|mail\|host\|ip' {} \; 2>/dev/null` | Find the configuration files which contain interesting information |

### Escalation

### Remediation

## Sudo

- Shell escape sequence
- Abuse intended functionality
- LD_Preload / LD_Library_path

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `sudo -l` | List the allowed (and forbidden) commands for the invoking user |
| `sudo -V` | Sudo version |

### Escalation

### Remediation

## Cron

- Path
- Wildcard
- File Overwrite

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `crontab -l` | Display cron of the current user |
| `ls -la /etc/cron*` | Display scheduled jobs overview |

### Escalation

### Remediation

## File Permission

- SUID

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `find / -perm -4000 -type f 2>/dev/null` | Find SUID files |
| `find / -uid 0 -perm -4000 -type f 2>/dev/null` | Find SUID files owned by root |
| `find / -perm -2000 -type f 2>/dev/null` | Find SGID files (sticky bit) |
| `find / ! -path "*/proc/*" -perm -2 -type f -print 2>/dev/null` | Find world-writeable files excluding `proc` file |
| `find / -type f '(' -name *.cert -or -name *.crt -or -name *.pem -or -name *.ca -or -name *.p12 -or -name *.cer -name *.der ')' '(' '(' -user support -perm -u=r ')' -or '(' -group support -perm -g=r ')' -or '(' -perm -o=r ')' ')' 2> /dev/null-or -name *.cer -name *.der ')' 2> /dev/null` | Find keys or certificates you can read |
| `find /home –name *.rhosts -print 2>/dev/null` | Find rhost config files |
| `find /etc -iname hosts.equiv -exec ls -la {} 2>/dev/null ; -exec cat {} 2>/dev/null ;` | Find hosts.equiv, list permissions and cat the file contents |
| `cat ~/.bash_history` | Display current user history |
| `ls -la ~/.*_history` | Dislay the current user various history files |
| `ls -la ~/.ssh/` | Check current user's ssh files |
| `find /etc -maxdepth 1 -name '*.conf' -type f` or `ls -la /etc/*.conf` | List the configuration files in /etc (depth 1, modify the maxdepth param in the first command for change it) |
| `lsof | grep '/home/\|/etc/\|/opt/'` | Display the possibly interesting openfiles |


### Escalation

### Remediation

## Other usefull information

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `cat /etc/shells` | Pathnames of valid login shells |
| `cat /etc/profile` | Display default system variables |
| `env` | Display environmental variables |
| `route` | Display route information |
| `arp -a` | Display arp table |
| `cat /etc/resolv.conf` | Show configured DNS sever addresses |
| `head /var/mail/root` | Try to read root mail |

## Users & Groups

### Information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `whoami` | Print effective userid |
| `id` | Print real and effective user and group IDs |
| `awk -F ':' '{print $1}' /etc/passwd` | List all users on the system |
| `awk -F ':' '{print $1}' /etc/group` | List all groups on the system |
| `last` | Show listing of last logged in users |
| `lastlog` | Foreach users get the last logged in |
| `for i in $(cat /etc/passwd 2>/dev/null| cut -d":" -f1 2>/dev/null);do id $i;done 2>/dev/null` | List uid and groups for each users |
| `grep -v -E "^#" /etc/passwd | awk -F: '$3 == 0 { print $1}'` | List all super users accounts |
| `w` | Show who is logged on and what they are doing |
| `users` or `who -a` | Print the user names of users currently logged in to the current host |

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `whoami` | Print effective userid |
| `id` | Print real and effective user and group IDs |
| `awk -F ':' '{print $1}' /etc/passwd` | List all users on the system |
| `awk -F ':' '{print $1}' /etc/group` | List all groups on the system |
| `last` | Show listing of last logged in users |
| `lastlog` | Foreach users get the last logged in |
| `for i in $(cat /etc/passwd 2>/dev/null| cut -d":" -f1 2>/dev/null);do id $i;done 2>/dev/null` | List uid and groups for each users |
| `grep -v -E "^#" /etc/passwd | awk -F: '$3 == 0 { print $1}'` | List all super users accounts |
| `w` | Show who is logged on and what they are doing |
| `users` or `who -a` | Print the user names of users currently logged in to the current host |