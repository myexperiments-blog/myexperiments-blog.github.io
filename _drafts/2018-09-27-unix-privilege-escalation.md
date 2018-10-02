---
layout: post
title:  "Local unix privilege escalation"
date:   2018-09-27 09:00:00
categories: System Pentest
author: Antoine Brunet
permalink: unix-privilege-escalation.html
article_folder: "/tor-middle-box"
comments: true
description: ""
keywords: ""
---

# Summary

- [Introduction](#introduction)
- [Unix privilege escalation scopes](#unix-privilege-escalation-scopes)
    - [Kernel](#kernel)
        - [Kernel privilege escalation overview](#kernel-privilege-escalation-overview)
        - [Good practices to avoid kernel privilege escalation](#good-practices-to-avoid-kernel-privilege-escalation)
        - [Kernel information gathering](#kernel-information-gathering)
    - [Process](#process)
        - [Process privilege escalation overview](#process-privilege-escalation-overview)
        - [Good practices to avoid process privilege escalation](#good-practices-to-avoid-process-privilege-escalation)
        - [Process information gathering](#process-information-gathering)


# Introduction

When an attacker succeed to establish the initial foothold (gain access to a user with restricted privileges), he will seek for a way to increase his privileges (Gain access to an other user with more privileges). We call this action a privilege escalation. It can happen in all sort of application or systems, in this article we only describe the unix system escalation.

Before to go forward we must think like an attacker for hunderstanding the importance of securing a machine to a privilege escalation.

What an attacker may want from our system ?
He may want to :
- steal some sensitive data
- use the hacked system to access to an other one
- want to use our system for pivot to others.

In all theses cases after he will gain access to the machine he will need to secure his access. He do not want to reused the vulnerability each time he want to connect to your system it will be to noisy and it will let anyone who find the vulnerability access to the machine for this reason most of the time attacker's patch the vulnerability they used. So he will have to install a backdoor. A rootkit will be the best choice for him to secured his access. But for install a rootkit or any kind of hidden backdoor he will need to be administrator before. So he will need to find a privilege escalation attack on the system from the user he first get for gain superior privilege.

This article will try to give a complete overview of all unix privilege escalation technics.
Do not hesitate to share with us your techniques in the comments.

# Unix privilege escalation scopes

This article separate the local unix privilege escalation in different scopes: kernel, deamon and process, password mining, sudo, cron and file permission. Foreach I will gave you  commands for information gasthering, I will explain how to gain an escalation privilege with the information you collect, and finnaly I will give you the remediation.

## Kernel

### Kernel privilege escalation overview

...

### Good practices to avoid kernel privilege escalation

There is no way to completely avoid a kernel privilege escalation. But some good practices are good to know. The first one is to always being aware about security vulnerabilities discovered and keeping your system up to date, patch it when a patch is realized.

For a kernel privilege escalation the attacker will most of the time use a kernel exploit. For that he will need four conditions:

1. A vulnerable kernel
1. A matching exploit
1. The ability to transfer the exploit onto the target
1. The ability to execute the exploit on the target

We are never protected against a kernel vulnerabilities every days some 0 days are find and sold on the dark web. So even if you or your company are aware about IT security and are following the CVS publication, you cannot be sure that your system is hundred purcent secured.

For this reason you must influence the last two points. So a good practice will be to focus on restricting or removing programs that enable file transfers, such as FTP, SCP, wget, curl or any program which can permit to realized file transfers. If you need this program restrict them to specific users, IP/domains, more they will be restricted and better it will be.

It is also a good practice to remove or restrict the access of all the compilers such as gcc.

An other good practice will be to limit directories that are writeable and
executable, particularly by service users.

Finally, but it is most of the time hard to do because it has a cost, it is a really good practice to externalized logs in an other machine.

### Kernel information gathering

Some basic command to collect some clue for realized a unix kernel exploitation

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

## Process

### Process privilege escalation overview

Most of the time an attacker succeed to establish the initial foothold by using a miss configured or vulnerable running service, but it does not stop there.

An attacker can use a process which is only accessible from the local machine for doing privilege escalation. For example, using a vulnerable MySQL database which is running as root.

### Good practices to avoid process privilege escalation

Most of the time the method attack will be similar than for a kernel privilege escalation. The attacker will use an exploit, and for that he will need the four conditions we see [upper](#good-practices-to-avoid-kernel-privilege-escalation). Also all the good practices site for the kernel can be transposed for the running processes.

A good practice for all your sensitive services (a service which can be access from the outside is definitly sensitive) is to run them into a chroot jail, at least creating a specific user with no shell. Please do not run them as root, it is the worst that you can do.

### Process information gathering

Some basic command to collect some clue for realized a privilege escalation by passing daemon and process exploitation.

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
| `perl -v` | Perl version |
| `java -version` | Java version |
| `python --version` | Python version|
| `ruby -v` | Ruby version |