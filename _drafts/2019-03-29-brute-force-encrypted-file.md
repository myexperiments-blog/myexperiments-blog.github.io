---
layout: post
title:  "Finding cipher algorithm of an encrypted file"
date:   2019-03-29 15:00:00
tags: cryptography
author: Antoine Brunet
permalink: finding-cipher-algorithm-encrypted-file.html
comments: true
description: "I was solving a CTF challenge when I came in contact with an encrypted file I downloaded from a FTP that I had just compromised.
I decided to write a quick write up about how I managed to find the cipher algorithm used to encrypt the file in the goal of brute forcing it."
keywords: "brute force, openssl, CTF, challenge, write up, encrypted file, cryptography"
---

# CTF context

I was solving a CTF challenge when I came in contact with an encrypted file I downloaded from a FTP that I had just compromised.
I decided to write a quick write up about how I managed to find the cipher algorithm used to encrypt the file in the goal of brute forcing it.

Actually, there is no process or magic trick to truly define the cipher algorithm used from an encrypted file. What I managed to do is improve your chance to find the good one straight away (In my CTF I was super lucky, in real life things can be much more difficult).

# Operating mode

I downloaded the encrypted file `secret.enc`. After reading, it appears to be a base64 encoding.

```
$ cat secret.enc
U2FsdGVkX189L7GA0iY9tjKWp+KoX6ugH/Aw6Wb1Qtlg3gm9OU0xwFCTaI60oL2DRTfiDMroSFTRYgD7Bor+8Ca/Z3ogamDQfi2RyZLOwLsy2oj7IkMZf7lCqS5izjQ1
```

I decode it with the following command:

```
$ base64 -d secret.enc
Salted__=/���&=�2����_���0�f�B�`�	�9M1�P�h�����E7�
                                                        ��HT�b����&�gz j`�~-�ɒ���2ڈ�"C�B�.b�45
```

As we can see it's encrypted. So I placed the decoded message in a new file that I call `encrypted.enc` and use the `file` command to receive more information.

```
$ base64 -d secret.enc > encrypted.enc
$ file encrypted.enc
    encrypted.enc: openssl enc'd data with salted password
```

I was now informed that the file has been encrypted with Openssl with a salted password.

To simplify the brute force process, I had to find the algorithm used during the encryption phase.

The command `openssl enc -ciphers` will display a list of all the algorithms supported by Openssl, it helped me to define a first list of ciphers.
I redirect them in a file I called `ciphers.list` and after a count it appears that Openssl supports 111 different ciphers.

My goal now was to reduce that list as much as I can.

In order to reduce that list it is necessary to get the length in bytes of the encrypted file.

```
$ wc -c encrypted.enc
    96 encrypted.enc
```

We have a file of 96 bytes. This number of bytes is divisible by 8, so there is a good chance that the cipher algorithm used to encrypt the file is using block cipher techniques ([source](https://en.wikipedia.org/wiki/Block_size_(cryptography))). So I decided to remove all the stream ciphers from my file `ciphers.list`. After removing them, my list was reduced to 102 ciphers.

Let's now create some samples in order to find a pattern between the encoded message and the encoded sample, as we know our encrypted file is 96 bytes long, the clear message is normally lower or equal to 96 bytes.

Let's create some clear text sample with values between 0 and 96.

```sh
$ for sample in $(seq 0 8 96); do python -c "print 'A'*$sample" > $sample; done
```

So now let's encrypt our samples with all our cipher algorithms defined in the file `ciphers.list`.  
I use this script for achieving this task:

```sh
#!/bin/bash
for cipher in $(cat ciphers.list); do
    for sample in $(seq 0 8 96); do
        openssl enc -$cipher -e -in $sample -out $sample$cipher.enc -k whyDudewhy
    done
done
```

After I execute this script I received a bunch of encrypted files.
I start by finding the file with 96 bytes length with the following command:

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
   96 64-camellia-128-cbc.enc
   96 64-camellia-128-ecb.enc
   96 64-camellia128.enc
   96 64-camellia-192-cbc.enc
   96 64-camellia-192-ecb.enc
   96 64-camellia192.enc
   96 64-camellia-256-cbc.enc
   96 64-camellia-256-ecb.enc
   96 64-camellia256.enc
   96 64-seed-cbc.enc
   96 64-seed-ecb.enc
   96 64-seed.enc
   96 72-aes-128-cbc.enc
   96 72-aes-128-ecb.enc
   96 72-aes128.enc
   96 72-aes-192-cbc.enc
   96 72-aes-192-ecb.enc
   96 72-aes192.enc
   96 72-aes-256-cbc.enc
   96 72-aes-256-ecb.enc
   96 72-aes256.enc
   96 72-bf-cbc.enc
   96 72-bf-ecb.enc
   96 72-bf.enc
   96 72-blowfish.enc
   96 72-camellia-128-cbc.enc
   96 72-camellia-128-ecb.enc
   96 72-camellia128.enc
   96 72-camellia-192-cbc.enc
   96 72-camellia-192-ecb.enc
   96 72-camellia192.enc
   96 72-camellia-256-cbc.enc
   96 72-camellia-256-ecb.enc
   96 72-camellia256.enc
   96 72-cast5-cbc.enc
   96 72-cast5-ecb.enc
   96 72-cast-cbc.enc
   96 72-cast.enc
   96 72-des3.enc
   96 72-des-cbc.enc
   96 72-des-ecb.enc
   96 72-des-ede3-cbc.enc
   96 72-des-ede3-ecb.enc
   96 72-des-ede3.enc
   96 72-des-ede-cbc.enc
   96 72-des-ede-ecb.enc
   96 72-des-ede.enc
   96 72-des.enc
   96 72-desx-cbc.enc
   96 72-desx.enc
   96 72-rc2-128.enc
   96 72-rc2-40-cbc.enc
   96 72-rc2-40.enc
   96 72-rc2-64-cbc.enc
   96 72-rc2-64.enc
   96 72-rc2-cbc.enc
   96 72-rc2-ecb.enc
   96 72-rc2.enc
   96 72-seed-cbc.enc
   96 72-seed-ecb.enc
   96 72-seed.enc
```
{:class="big-list-overflow"}

As we can see, I got some matches. These matches come from the clear text 64 bytes and the 72 bytes.
I decided to only take the cipher algorithms which match with both clear texts.
Now my list of cipher algorithms is reduced to only 14.

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

So, even if 14 is much better than 111, it is still long when you have to brute force them with a long password list.
I decided to find the more common cipher in this list, and I choose `aes-256-cbc`. AES is clearly the most common algorithm in this list, the 256 key lengths is famous for being really secure and CBC (Cipher Block Chaining) is the default cipher used by Openssl for the AES algorithm as shown here:

```
$ openssl enc -aes256 -e -in text.clear -out blabla.enc
    enter aes-256-cbc encryption password:
          ^
```

For executing the brute force I had to install [bruteforce-salted-openssl](https://github.com/glv2/bruteforce-salted-openssl).

When you use the tool, keep in mind to set the message digest to `sha256`, which is the default message digest of Openssl ([source](https://www.openssl.org/docs/man1.1.1/man1/dgst.html)). If the file has been encrypted with a different message digest our tool will be unable to know that it found a good result, so keep your fingers crossed.

I used this bruteforce-salted-openssl command: `bruteforce-salted-openssl -t 15 -f rockyou.txt -c aes-256-cbc -d sha256 encrypted.enc` to brute force the file.

After only a second the tool was able to find the password => `bubbles`.

Finally, I was able to decrypt the encoded message with the following Openssl command:

```
$ openssl enc -aes-256-cbc -d -in encrypted.enc -out clear.txt -k bubbles
$ cat clear.txt
    42 is the answer to the ultimate question of life the universe and everything
```