---
layout: default
title:  "Become anonymous online"
date:   2018-07-21 10:00:00
categories: OpenBSD TOR VPN
author: Lovebug
permalink: become-anonymous-online.html
img_folder: "/assets/img/posts/tor-middle-box"
comments: true
description: "You want to be anonymous online ? You don't want to always set up all your environment for it. This article will provide you a solution, which will give you a good way to do it"
keywords: "tor,anonymous,vpn,middle box,anonymous online"
---

# Plan

- Introduction
- setting up a tor middle box
  - set up a PKI
  - set up the vpn server & client
  - set up tor on the server
  - set up the server interfaces & firewall rules
- sources


# Introduction

Being anonymous on Internet is not such easy. There is many different way to do. One of the easiest is to use TOR (The Onion Router).
This article will not discuss about what is the best way to be anonymous, in my opinion, it is mostly depends of your needs, but the safer way will probably be to use [Tails](https://tails.boum.org/).

This article will explain how to setting up a TOR transparent proxy (middle box) in a remote [OpenBSD](https://www.openbsd.org/) server.

## What's a TOR transparent proxy ?

Let's make few reminders.

TOR is a short for The Onion Routing, it is a way to "Protect your privacy. Defend yourself against network surveillance and traffic analysis". You can get further information on the [TOR project website](https://www.torproject.org/about/overview.html.en).

A transparent proxy is a server that sits between your computer and the Internet and redirects your requests and responses without modifying them. A proxy server that does modify your requests and responses is defined as a non-transparent proxy.

So To make it simple a TOR transparent proxy is an intermediary system sitting between you and the Internet, which anonymise all your network traffic.

## What will we setups

I will explain how to setups a remote TOR middlebox accessing through a VPN, as it show on the picture below.

![tor-proxy]({{ site.url }}{{page.img_folder}}/tor-proxy.png){:class="img-responsive"}

We will frist connect to our remote server with a VPN connection. The goal here is to connect securely from everywhere. Also, your service provider haven't any way to know that you are using the Tor protocol (But be aware that your server provider will know it).

After your network traffic redirection through your VPN to your server it will be anonymized with the Tor protocol.

# Setups

Let's start to setups our middlebox.

Here all the steps you must follow to get your middlebox.

- Install and configure [OpenBSD](https://www.openbsd.org/) on your remote server
- Setups a Public Key Infrastructure (PKI)
- Setups the VPN server & client
- Setups Tor on the server
- Setups the server interfaces & firewall rules

First you should have install and configure [OpenBSD](https://www.openbsd.org/) on your remote server. Sadly your server provider will probably not give you a straightforward way to install it, so you will have to make few search. [Here a tutoriel to install OpenBSD on the OVH Cloud](https://www.tumfatig.net/20161124/encrypted-openbsd-6-0-in-the-ovh-cloud/)

## Create your own Public Key Infrastructure (PKI)
You must setting up your own PKI and generating certificates and keys for an OpenVPN server and multiple clients as it is advised in the [OpenVPN documentation](https://openvpn.net/index.php/open-source/documentation/howto.html#pki)

You can read this article of [Freek Dijkstra](http://www.macfreek.nl/memory/Create_a_OpenVPN_Certificate_Authority) which is really helpful and complete on the subject.

An other way to create your own PKI is to use `easy-rsa`, you will find some information in the OpenVPN documentation about that.
