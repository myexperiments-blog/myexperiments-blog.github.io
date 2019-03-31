---
layout: post
title:  "Brute force an encrypted file"
date:   2019-03-29 15:00:00
tags: cryptography
author: Antoine Brunet
permalink: brute-force-encrypted-file.html
article_folder: "/brute-force-encrypted-file"
comments: true
description: "I was solving a CTF challenge when I had to face to an encrypted file, and I had to brute force it, this article is a quick write up of my process"
keywords: "brute force, openssl, CTF, challenge, writeup, encrypted file, cryptography"
---

# CTF context

I was solving a CTF challenge when I got, from a ftp I just compromised, an encrypted file.
I decided to write a quick write up about how I managed to find the encruption algorithm needed to brute force this file.

# Operating mode

The encrypted file I got, `secret.enc`, seems to be encoded in base 64.

```
$ cat secret.enc
U2FsdGVkX189L7GA0iY9tjKWp+KoX6ugH/Aw6Wb1Qtlg3gm9OU0xwFCTaI60oL2DRTfiDMroSFTRYgD7Bor+8Ca/Z3ogamDQfi2RyZLOwLsy2oj7IkMZf7lCqS5izjQ1
```

Let's decode it.

```
$ base64 -d secret.enc
Salted__=/���&=�2����_���0�f�B�`�	�9M1�P�h�����E7�
                                                        ��HT�b����&�gz j`�~-�ɒ���2ڈ�"C�B�.b�45
```

Has we can see it's encrypted. I put the decoded message in a new file which I call `encrypted.enc` and use the `file` command on it for getting more information.

```
$ base64 -d secret.enc > encrypted.enc
$ file encrypted.enc
    encrypted.enc: openssl enc'd data with salted password
```

So, I was now inform that the file, has been encrypted with Openssl with a salted password.

To simplify the brute force process I had to find the algorithm used during the encryption phase.

The command `openssl enc -ciphers` will display a list of all the algorithms supported by Openssl.
I redirect them in a file I called `ciphers.list` and after a count it appear that Openssl support 111 different ciphers.

I have to reduce this list as much as I can.

A very useful information is the number of bytes in the encrypted file.

```
$ wc -c encrypted.enc
    96 encrypted.enc
```

We have a file with a size of 96 bytes. This number of bytes is divisible by 8, so there is good lucks that the algorithm used to encrypted the file is using block cipher technic ([source](https://en.wikipedia.org/wiki/Block_size_(cryptography))). So I decide to remove all the stream ciphers from my file `ciphers.list`. I find 9 stream ciphers, and I still have 102 ciphers to check.

Let's now create some sample for trying to find a pattern between the encoded message and the encoded sample, has we know our encrypted file is 96 bytes long, so the clear message must be lower or equal to 96 bytes.

Let's create some clear text sample with value comprise between 0 and 96.
We put a 8 for increase by 8 for each iteration (I choose 8 but any number is fine).

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

After I execute this script I got a bunch of encrypted files.
I start by finding the file with 96 bytes length.

```
$ ls *.enc | xargs wc -c | grep '96 '
   96 64-aes-128-cbc.enc
   96 64-aes-128-ecb.enc
   96 64-aes128.enc
   96 64-aes-192-cbc.enc
   96 64-aes-192-ecb.enc
   96 64-aes192.enc
   96 64-aes-256-cbc.enc
   96 64-aes-256-ecb.enc
   96 64-aes256.enc
   96 72-aes-128-cbc.enc
   96 72-aes-128-ecb.enc
   96 72-aes128.enc
   96 72-aes-192-cbc.enc
   96 72-aes-192-ecb.enc
   96 72-aes192.enc
   ...
```

As we can see I get some matches. Some are from clear text 64 bytes long and some other 72 bytes long.
I decided to only take the ciphers which appear to match the 64 bytes long and the 72 bytes long clear text.
So my list of ciphers was reduce to only 14 ciphers.

- aes-128-cbc
- aes-128-ecb
- aes-192-cbc
- aes-192-ecb
- aes-256-cbc
- aes-256-ecb
- camellia-128-cbc
- camellia-128-ecb
- camellia-192-cbc
- camellia-192-ecb
- camellia-256-cbc
- camellia-256-ecb
- seed-cbc
- seed-ecb

So, even 14 is much better than 111, it still long when you have to brute force them with a long password list.
I decide to find the more common cipher in this list, and I choose `aes-256-cbc`. AES is clearly the most common algorithm in this list, the 256 key lengths is famous to be really secure and the cbc cipher is the default cipher used by Openssl for this algorithm as it shown here.

```
$ openssl enc -aes256 -e -in text.clear -out blabla.enc
    enter aes-256-cbc encryption password:
          ^
```

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