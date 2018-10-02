---
layout: post
title:  "Setting up a remote Tor middle box"
date:   2018-09-17 09:00:00
categories: VPN TOR OpenBSD
author: Antoine Brunet
permalink: tor-middle-box.html
article_folder: "/tor-middle-box"
comments: true
description: "Setting up a TOR transparent proxy (middle box) in a remote OpenBSD server trough a VPN"
keywords: "tor,anonymous,vpn,middle box,anonymous online"
---

# I - Introduction

Being anonymous on Internet is not such easy. There is different ways to do it. One of the easiest is to use Tor (The Onion Router).

I have a remote server which I mostly use as VPN, so I had the idea to use it as a Tor middle box for providing me an anonymous VPN.

In this article I will explain you how to setting up a TOR transparent proxy (middle box) in a remote [OpenBSD](https://www.openbsd.org/) server trough a VPN.

## 1 - What's a TOR transparent proxy ?

Let's start with a few reminders.

TOR is short for The Onion Routing, it is a way to "Protect your privacy. Defend yourself against network surveillance and traffic analysis". You can get further information on the [TOR project website](https://www.torproject.org/about/overview.html.en).

A transparent proxy is a server that sits between your computer and the Internet and redirects your requests and responses without modifying them. A proxy server that does modify your requests and responses is defined as a non-transparent proxy.

So To make it simple a TOR transparent proxy is an intermediary system sitting between you and the Internet, which makes all your network traffic anonymous.

## 2- What will we set up

I will explain how to set up a remote TOR middle box accessing through a VPN, as it appears on the picture below.

![tor-proxy]({{ site.article_img }}{{page.article_folder}}/tor-proxy.png){:class="img-responsive"}

We will first connect to our remote server with a VPN connection. The goal here is to connect securely from everywhere. Also, your service provider does not have any way to know that you are using the Tor protocol (But be aware that your server provider will know it).

After your network traffic redirection through your VPN to your server it will be anonymous with the Tor protocol.

# II Set up steps

Let's now set up our middle box, with the following steps :

- Install [OpenBSD](https://www.openbsd.org/) on a remote server
- Create a Public Key Infrastructure (PKI)
- Set up the [OpenVPN](https://openvpn.net/index.php/open-source/333-what-is-openvpn.html) server & client
- Set up Tor on the server
- Set up the server interfaces & firewall rules

First you should install and configure [OpenBSD](https://www.openbsd.org/) on your remote server. Sadly your server provider will probably not give you a straightforward way to install it, so you will have to do a bit of research. [Here a tutorial to install OpenBSD on the OVH Cloud](https://www.tumfatig.net/20161124/encrypted-openbsd-6-0-in-the-ovh-cloud/)

## 1 - Create a Public Key Infrastructure (PKI)
You must set up your own PKI and generating certificates and keys for an OpenVPN server and multiple clients as it is advised in the [OpenVPN documentation](https://openvpn.net/index.php/open-source/documentation/howto.html#pki)

You can read this article of [Freek Dijkstra](http://www.macfreek.nl/memory/Create_a_OpenVPN_Certificate_Authority) which is really helpful and complete on the subject.

Another way to create your own PKI is to use `easy-rsa`, you can find further information in the OpenVPN documentation.

So now that you have your PKI you must generate some certificates and keys.

### a - Server side

- ca.crt (Your PKI certificate)
- server.crt (Sign with your PKI)
- server.key (private, don't send it to any clients or other servers)
- Diffie hellman key `openssl dhparam -out dh4096.pem 4096` it is really long to generate one
- TLS auth `openvpn --genkey --secret ta.key`

### b - Client side

- client.crt (Sign with your PKI)
- client.key (private, don't send it to any clients or servers)

## 2 - Setups OpenVPN

### a - OpenVPN server side

You can download this <a href="{{ site.article_file }}{{ page.article_folder }}/openvpn-server.conf" download >OpenVPN server configuration</a> and run it with the following command: `openvpn --config openvpn.conf --daemon` (Check if the process has been launched `ps auxwww | grep openVPN`. If there is any trouble, remove the `--daemon` arguments for easy debugging).

Don't forget to modify the path of your certificates and keys on the OpenVPN configuration file.

### b - OpenVPN client side

On the client side you can use this <a href="{{ site.article_file }}{{ page.article_folder }}/openvpn-client.conf" download >configuration file</a>, after creating the openvpn user `sudo adduser --no-create-home --disabled-login --system --group openvpn`.
You must also copy the `ca.crt`, `client.crt`, and you should already have the private key `client.key` on your client.

To make it work you should modify the configuration file by replacing the path of certificates and keys, or you can put directly their content in the OpenVPN configuration path. Don't forget to specify the server hostname or IP.
For example with the `ca` you should add:

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

So now if everything is good, you should be able to see your remote server IP address when you browse this web page [monip.org](http://monip.org/) from your client.

## 3 - Set up Tor on the server

Install Tor on your remote server: `pkg_add tor`.
Edit the Tor configuration file: `/etc/tor/torrc`, and insert the following content:

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

## 4 - Set up the server interfaces & firewall rules

This is an overview of the networking setup we need:

![tor-vpn-proxy]({{ site.article_img }}{{page.article_folder}}/tor-vpn-proxy.png){:class="img-responsive"}

All the user network traffic is redirected in his tap interface which is used by the VPN. All the user’s packets arrive in the remote server on his own tap interface which is in a specific rdomain and associated with a bridge interface. In this bridge we also associate a vether interface which is used as gateway meaning that his ip address is also the default route for this rdomain. Finally all the incoming traffic on the vether interface is redirect with a PF rule set to the Tor service which binds a local address, and sends the packets through the Tor protocol.

Few reminders about OpenBSD interfaces and rdomain:
 - **rdomains** are virtual routing tables. It is a good way to simulate a router, and isolate different networks. You will find further information on the [rdomain man page](https://man.openbsd.org/rdomain.4).
 - **Tap interfaces** are mostly used by VPN, they have an exclusive open property which means that it cannot be opened if it is already in use by another process. You can find further information on the [tap man page](http://man.openbsd.org/tap).
 - **Vether interface** simulates a physical ethernet interface. You can find further information on the [vether man page](http://man.openbsd.org/vether).
 - **Bridge interface** simulates a switch device. You can find further information on the [bridge man page](http://man.openbsd.org/bridge).

### a - Create the interfaces

You have two ways to create those interfaces:
First, you can create interfaces with `ifconfig` but after a reboot you will have to recreate them with the same command set.

```
ifconfig vether1 10.8.2.254/24 rdomain 1 up
ifconfig tap1 rdomain 1 up
ifconfig bridge1 rdomain1 up
ifconfig bridge1 add tap1
ifconfig bridge1 add vether1
route -T 1 add default 10.8.2.254
```

Second, you can make them persistent to a reboot by creating the hostname file and adding their configurations.

Create the file `/etc/hostname.vether1` with the following content:
```
rdomain 1
inet 10.8.2.254/24
!route -T 2 add  default 10.8.2.254
up
```

Create the file `/etc/hostname.hostname.tap1` with the following content:
```
rdomain 1
up
```

Create the file `/etc/hostname.bridge1` with the following content:
```
rdomain 2
add vether2
add tap2
up
```

### b - Add Packet Filter rules set

This are the minimal working rules set for you server, be cautious by allowing ssh access to you server.
Modify the `/etc/pf.conf` file with the below rules, and check the syntax with this command `pfctl -nf /etc/pf.conf`, if no error is reported with this command load this configuration with the command `pfctl -f /etc/pf.conf`.

```
block in log all
pass out all keep state
pass proto tcp to port 22
match in all scrub (no-df random-id)
pass proto udp to port 44101
pass in quick on vether2 inet proto tcp to !(vether2) rtable 0 rdr-to 127.0.0.1 port 9040
pass in quick on vether2 inet proto udp to port domain rtable 0 rdr-to 127.0.0.1 port domain
```

### c - Let's do some checks

Everything is now set up, so let’s see if it’s working. Browse this website [monip.org](http://monip.org/) from your client. You should normally see an IP address, which is not yours or the remote server's. On the remote server restart the Tor process and check your public IP address again, normally it should have changed.

Enjoy your own anonymous VPN :)