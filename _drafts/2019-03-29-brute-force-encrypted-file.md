---
layout: post
title:  "Brute force encrypted file"
date:   2019-03-29 15:00:00
tags: cryptography
author: Antoine Brunet
permalink: brute-force-encrypted-file.html
comments: true
description: ""
keywords: "brute force, openssl, challenge, writeup, encrypted file, cryptography"
---

# Introduction

# The challenge

I got a file `secrets.enc`.

```
$ cat secrets.enc
U2FsdGVkX189L7GA0iY9tjKWp+KoX6ugH/Aw6Wb1Qtlg3gm9OU0xwFCTaI60oL2D
RTfiDMroSFTRYgD7Bor+8Ca/Z3ogamDQfi2RyZLOwLsy2oj7IkMZf7lCqS5izjQ1
```

That looks like the text is encode in base64. Let's decode it.

```
$ base64 -d secrets.enc
Salted__=/���&=�2����_���0�f�B�`�	�9M1�P�h�����E7�
                                                        ��HT�b����&�gz j`�~-�ɒ���2ڈ�"C�B�.b�45
```

It's look like encrypted ! Let's put it in a file and check it with the `file` command.

```
$ base64 -d secrets.enc > encrypted.enc
$ file encrypted.enc
    encrypted.enc: openssl enc'd data with salted password
```

So, it seems to be an openssl encryption with salted with a password.

We first have to guess the algorithm used for the encryption.
Let's count the number of characters in the file.

```
$ wc -c encrypted.enc
    96 encrypted.enc
```

96 is divisible by 8, so there is more chance that the algorithm used is working with blocks (stream is not rejected).
So we will first try some block algorithm.


The command `openssl help` can be usefull for finding those algorithm

```
Cipher commands (see the `enc' command for more details)
aes-128-cbc       aes-128-ecb       aes-192-cbc       aes-192-ecb       
aes-256-cbc       aes-256-ecb       base64            bf                
bf-cbc            bf-cfb            bf-ecb            bf-ofb            
camellia-128-cbc  camellia-128-ecb  camellia-192-cbc  camellia-192-ecb  
camellia-256-cbc  camellia-256-ecb  cast              cast-cbc          
cast5-cbc         cast5-cfb         cast5-ecb         cast5-ofb         
des               des-cbc           des-cfb           des-ecb           
des-ede           des-ede-cbc       des-ede-cfb       des-ede-ofb       
des-ede3          des-ede3-cbc      des-ede3-cfb      des-ede3-ofb      
des-ofb           des3              desx              rc2               
rc2-40-cbc        rc2-64-cbc        rc2-cbc           rc2-cfb           
rc2-ecb           rc2-ofb           rc4               rc4-40            
seed              seed-cbc          seed-cfb          seed-ecb          
seed-ofb 
```

I decide to keep only those one which are really the most common:

- aes-256-ecb
- aes-128-ecb
- aes-256-ofb
- aes-128-ofb
- aes-256-cbc
- aes-128-cbc
- aes-256-ecb
- aes-128-ecb
- aes-256-ofb
- aes-128-ofb
- aria-128-cbc
- aria-256-cbc
- rc4-cbc
- des
- des3
- des-ofb
- des-cbc
- des-ecb

Let's now create some sample for trying to find a pattern, has we know our encrypted file is 96 bytes long, so the clear message must be lower or equal to 96 bytes.

Let's create some clear text sample with value comprise to 0 -> 96
we put a 8 for increase by 8 for each iteration.

```
$ for sample in $(seq 0 8 96); do python -c "print 'A'*$sample" > $sample; done
```

So now let's encrypt with our sample with all our algorithm.
I use this script for acheving this task.

`encrypt.sh`

```
#!/bin/bash
for cipher in $(cat ciphers.list); do
    for sample in $(seq 0 8 96); do
        openssl enc $cipher -e -in $sample -out $sample$cipher.enc -k whyDudewhy
    done
done
```
Execute this script.

So let's now find the file with 96 bytes length.

```
$ ls *.enc | xargs wc -c | grep '96 '
    96 64-aes-128-cbc.enc
    96 64-aes-128-ecb.enc
    96 64-aes-256-cbc.enc
    96 64-aes-256-ecb.enc
    96 72-aes-128-cbc.enc
    96 72-aes-128-ecb.enc
    96 72-aes-256-cbc.enc
    96 72-aes-256-ecb.enc
    96 72-des3.enc
    96 72-des-cbc.enc
    96 72-des-ecb.enc
    96 72-des.enc
    96 encrypted.enc
```

helping by this output we can restrict to theses algorithm:
- aes-128-cbc
- aes-128-ecb
- aes-256-cbc
- aes-256-ecb
- des3
- des-cbc
- des-ecb
- des

We have now to try all those algorithms, or like me you can just try the more common, it appear to time here `aes-256-cbc`.

So we can now install and use [bruteforce-salted-openssl](https://github.com/glv2/bruteforce-salted-openssl).
The openssl default message digest is `sha256` and not `md5` so the bruteforce-salted-openssl command will look like that `bruteforce-salted-openssl -t 15 -f rockyou.txt -c aes-256-cbc -d sha256 encrypted.enc`

we find the password `bubbles`.

let's now decrypt the encoded message `openssl enc -aes-256-cbc -d -in encrypted.enc -out clear.txt -k bubbles`
