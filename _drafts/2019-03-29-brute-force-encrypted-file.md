---
layout: post
title:  "Write up of a encrypted file brute force"
date:   2019-03-29 15:00:00
tags: cryptography
author: Antoine Brunet
permalink: brute-force-encrypted-file.html
article_folder: "/brute-force-encrypted-file"
comments: true
description: "Solving a CTF challenge I had to face to an encrypted file, and I had to brute force it, this article is a quick write up of my process"
keywords: "brute force, openssl, CTF, challenge, writeup, encrypted file, cryptography"
---

# CTF context

I was solving a CTF challenge when I got from a unsecure ftp an encrypted file.
I decided to write a quick write up about how to brute force this file, because even it is a basic brute force the process for finding the chosen algorithm is not that obvious.

# Operating mode

The encrypted file I got, `secrets.enc`, seems to be encoded in base 64.

```
$ cat secrets.enc
U2FsdGVkX189L7GA0iY9tjKWp+KoX6ugH/Aw6Wb1Qtlg3gm9OU0xwFCTaI60oL2D
RTfiDMroSFTRYgD7Bor+8Ca/Z3ogamDQfi2RyZLOwLsy2oj7IkMZf7lCqS5izjQ1
```

Let's decode it.

```
$ base64 -d secrets.enc
Salted__=/���&=�2����_���0�f�B�`�	�9M1�P�h�����E7�
                                                        ��HT�b����&�gz j`�~-�ɒ���2ڈ�"C�B�.b�45
```

It's encrypted ! Let's put the decoded message in a new file which will be called `encrypted.enc`. We will now used the `file` command for getting more information.

```
$ base64 -d secrets.enc > encrypted.enc
$ file encrypted.enc
    encrypted.enc: openssl enc'd data with salted password
```

So, the `file` command clearly tell us that, this file, has been encrypted with Openssl with a salted password.

To simplify the brute force process we need to find the algorithm chose during the encryption phase, or if we cannot find it just make the list of possible algorithms shorter.

A very useful information will be the number of bytes in the file.

```
$ wc -c encrypted.enc
    96 encrypted.enc
```

We have a file with a size of 96 bytes. This number of bytes is divisible by 8, so there is good lucks that the algorithm used to encrypted the file is using blocks ciphers ([source](https://en.wikipedia.org/wiki/Block_size_(cryptography))).

The command `openssl help` will display a list of all the algorithms supported by Openssl.

```
...
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

I decide to keep only those common algorithms, and I put them in a file (`ciphers.list`):

- aes-256-cbc
- aes-128-cbc
- aes-256-ecb
- aes-128-ecb
- des
- des3
- des-cbc
- des-ecb

Let's now create some sample for trying to find a pattern between the encoded message and the encoded sample, has we know our encrypted file is 96 bytes long, so the clear message must be lower or equal to 96 bytes.

Let's create some clear text sample with value comprise between 0 and 96.
We put a 8 for increase by 8 for each iteration (I chose 8 but any number is fine).

```sh
$ for sample in $(seq 0 8 96); do python -c "print 'A'*$sample" > $sample; done
```

So now let's encrypt our samples with all our algorithms.
I use this script for achieving this task.

```sh
#!/bin/bash
for cipher in $(cat ciphers.list); do
    for sample in $(seq 0 8 96); do
        openssl enc -$cipher -e -in $sample -out $sample$cipher.enc -k whyDudewhy
    done
done
```
After you execute it you should have a bunch of encrypted files.
You can start by finding the file with 96 bytes length and read them for getting those witch are the more similar to our encrypted file.

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
```

We can now try all those algorithms.
But I know that the encrypted file, I found on this unsecure ftp is a message leave by a person to an other one.
I will just start with the more common one in this list which is `aes-256-cbc`.

For executing the brute force I had to install [bruteforce-salted-openssl](https://github.com/glv2/bruteforce-salted-openssl).

Take good care to change the default message digest of bruteforce-salted-openssl which is different from the Openssl default message digest which is `sha256` ([source](https://www.openssl.org/docs/man1.1.1/man1/dgst.html)) and not `md5`. If the file has been encrypted with a different message digest our tool will not be able to know that he find the good result, so keep the fingers crossed.

I used this bruteforce-salted-openssl command: `bruteforce-salted-openssl -t 15 -f rockyou.txt -c aes-256-cbc -d sha256 encrypted.enc`

Bruteforce-salted-openssl used with the `rockyou.txt` passwords list find me the password `bubbles` after only few tries, lucky me.

We can now decrypt the encoded message with the following command:

```
$ openssl enc -aes-256-cbc -d -in encrypted.enc -out clear.txt -k bubbles
$ cat clear.txt
    42 is the answer to the ultimate question of life the universe and everything
```

Done! We get the most useful information in the whole universe.

![universe-answer]({{ site.article_img }}{{page.article_folder}}/universe-answer.png){:class="img-responsive"}