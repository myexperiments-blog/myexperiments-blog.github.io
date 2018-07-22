---
layout: default
title:  "Setting up a remote Tor middlebox"
date:   2018-07-21 10:00:00
categories: VPN TOR OpenBSD
author: Lovebug
permalink: become-anonymous-online.html
article_folder: "/tor-middle-box"
comments: true
description: ""
keywords: "tor,anonymous,vpn,middle box,anonymous online"
---

# I - Introduction

Being anonymous on Internet is not such easy. There is many different way to do. One of the easiest is to use TOR (The Onion Router).
This article will not discuss about what is the best way to be anonymous, in my opinion, it is mostly depends of your needs, but the safer way will probably be to use [Tails](https://tails.boum.org/).

This article will explain how to setting up a TOR transparent proxy (middle box) in a remote [OpenBSD](https://www.openbsd.org/) server.

## 1 - What's a TOR transparent proxy ?

Let's make few reminders.

TOR is a short for The Onion Routing, it is a way to "Protect your privacy. Defend yourself against network surveillance and traffic analysis". You can get further information on the [TOR project website](https://www.torproject.org/about/overview.html.en).

A transparent proxy is a server that sits between your computer and the Internet and redirects your requests and responses without modifying them. A proxy server that does modify your requests and responses is defined as a non-transparent proxy.

So To make it simple a TOR transparent proxy is an intermediary system sitting between you and the Internet, which anonymise all your network traffic.

## 2- What will we setups

I will explain how to setups a remote TOR middlebox accessing through a VPN, as it show on the picture below.

![tor-proxy]({{ site.article_img }}{{page.article_folder}}/tor-proxy.png){:class="img-responsive"}

We will frist connect to our remote server with a VPN connection. The goal here is to connect securely from everywhere. Also, your service provider haven't any way to know that you are using the Tor protocol (But be aware that your server provider will know it).

After your network traffic redirection through your VPN to your server it will be anonymized with the Tor protocol.

# II Setups steps

Let's now setups our middlebox, with the following steps :

- Install [OpenBSD](https://www.openbsd.org/) on a remote server
- Create a Public Key Infrastructure (PKI)
- Setups the [OpenVPN](https://openvpn.net/index.php/open-source/333-what-is-openvpn.html) server & client
- Setups Tor on the server
- Setups the server interfaces & firewall rules

First you should have install and configure [OpenBSD](https://www.openbsd.org/) on your remote server. Sadly your server provider will probably not give you a straightforward way to install it, so you will have to make few search. [Here a tutoriel to install OpenBSD on the OVH Cloud](https://www.tumfatig.net/20161124/encrypted-openbsd-6-0-in-the-ovh-cloud/)

## 1 - Create a Public Key Infrastructure (PKI)
You must setting up your own PKI and generating certificates and keys for an OpenVPN server and multiple clients as it is advised in the [OpenVPN documentation](https://openvpn.net/index.php/open-source/documentation/howto.html#pki)

You can read this article of [Freek Dijkstra](http://www.macfreek.nl/memory/Create_a_OpenVPN_Certificate_Authority) which is really helpful and complete on the subject.

An other way to create your own PKI is to use `easy-rsa`, you will find some information in the OpenVPN documentation about that.

So now you have your PKI you must generate some certificates and keys.

### a - Server side

- ca.crt (Your PKI certificate)
- server.crt (Sign with your PKI)
- server.key (private, don't send it to any clients or other servers)
- Diffie hellman key `openssl dhparam -out dh4096.pem 4096` it is really long to generate one
- TLS auth `openvpn --genkey --secret ta.key`

### b - Client side

- client.crt (Sign wity your PKI)
- client.key (private, don't send it to any clients or servers)

## 2 - Setups OpenVPN

### a - OpenVPN server side

You can download this <a href="{{ site.article_file }}{{ page.article_folder }}/openvpn-server.conf" download >OpenVPN server configuration</a> and run it with the following command: `openvpn --config openvpn.conf --daemon` (Check the process as been launch `ps auxwww | grep openVPN`. If any trouble remove the `--daemon` arguments for easy debugging).

Don't forget to modify the path of you'r certificates and keys on the OpenVPN configuration file.

### b - OpenVPN client side

On the client side you can use this <a href="{{ site.article_file }}{{ page.article_folder }}/openvpn-client.conf" download >configuration file</a>, after creating the openvpn user `sudo adduser --no-create-home --disabled-login --system --group openvpn`.
You must also copy the `ca.crt`, `client.crt`, and you should already have on your client the private key `client.key`.

To make it work you should modify the configuration file by replacing the path of certificates and keys, or you can put directly there content in the OpenVPN configuration path. Don't forget to specify the server hostname or IP.
For exemple with the `ca` you should add:

```
<ca>
VQQGEwJVSzENMAsGA1UECBMEQ2l0eTEPMA0GA1UEBxMGTG9uZG9uMRMwEQYDVQQK
EwpIYWNrVGhlQm94MRYwFAYDVQQDEw1IYWNrVGhlQm94IENBMQwwCgYDVQQpEwNo
dGIxITAfBgkqhkiG9w0BCQEWEmluZm9AaGFja3RoZWJveC5ncjAeFw0xNzA2MjEx
MDQ3MjZaFw0yNzA2MTkxMDQ3MjZaMIGLMQswCQYDVQQGEwJVSzENMAsGA1UECBME
...
</ca>
```

 and run OpenVPN with the following command: `sudo openvpn --config client.conf --daemon`.

So now if everythings good, you should be able to see your remote server IP address when you browse this web page [monip.org](http://monip.org/) from your client.

## 3 - Setups Tor on the server

Install Tor on your remote server: `pkg_add tor`.
Edit the Tor configuration file: `/etc/tor/torrc`, and put the following content:

```
RunAsDaemon 1
Log notice file /etc/tor/tor.log
SOCKSPort 0
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
TransPort 9040
DNSPort 53
```

Let's now run Tor `tor -f /etc/tor/torrc`

## 4 - Setups the server interfaces & firewall rules

This an overview of the networking setup we want for make it work:

![tor-vpn-proxy]({{ site.article_img }}{{page.article_folder}}/tor-vpn-proxy.png){:class="img-responsive"}

All the user network traffic is redirected in the TAP interface witch is the interface used by the VPN. The flow will arrive in the remote server in his TAP interface and it will be redirect with a PF rule set to the Tor service which bind a local address.

### a - Create the tap interface

You can create it with `ifconfig` but after a reboot you will have to recreate it with the same command : `ifconfig tap1 up`, or you can make it persistent to a reboot by creating the hostname file and adding the configuration:  `echo "up" > /etc/hostname.tap1`.

### b - Add Packet Filter rules set

This is the minimal working rules set for you server, take care to let you a ssh access to you server.
modify the `/etc/pf.conf` file with the below rules, and check the syntax with this commande `pfctl -nf /etc/pf.conf`, if no error is report with this command load this configuration with the command `pfctl -f /etc/pf.conf`. 

```
block in log all
pass proto tcp to port 22
pass out all keep state
match in all scrub (no-df random-id)
pass proto udp to port 44101
pass in quick on tap1 rdr-to 127.0.0.1 port 9040
```

### c - Let's do some checks

Everything’s setups, so let’s try if it’s working. Browse this website [monip.org](http://monip.org/) from your client. You should normaly see an IP address, which is not yours either than the remote server one. On the remote server restart the Tor process and check again you’r public IP address from your client, normaly it changed again.

Enjoy your own anonymous VPN :)
