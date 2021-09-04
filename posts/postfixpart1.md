---
title: Adventures with Postfix - Part 1
description: Setting up a forwarding email server using Postfix.
date: 2021-09-04
tags:
  - postfix
layout: layouts/post.njk
---
I wanted to setup an email forwarding / relay server on a custom domain and found these guides helpful.

<a href="https://jichu4n.com/posts/custom-domain-e-mails-with-postfix-and-gmail-the-missing-tutorial/">Custom Domain E-mails With Postfix and Gmail: The Missing Tutorial</a>
<a href="https://upcloud.com/community/tutorials/secure-postfix-using-lets-encrypt/">How to secure Postfix using Let’s Encrypt</a>

The goal is to have emails sent from sender1@hotmail.com >> recipient1@example.com >> someone@gmail.com. This way I can use a custom / vanity domain like example.com while still using my Gmail account to read and respond to emails (or usually just bulk delete them).

## Linux VM

I started with Azure VM using Linux (Ubuntu 20.04) and assigned a static public IP.

## DNS

Added A record for mail.example.com pointing to public IP

Added MX record for example.com pointing to mail.example.com

Added a PTR (reverse DNS) record mapping the public IP address to mail.example.com.

```powershell
# PowerShell code to assign reverse fqdn to Azure Public IP
$pip = Get-AzPublicIpAddress -Name "mail-ip" -ResourceGroupName "mail_group"
$pip.DnsSettings = New-Object -TypeName "Microsoft.Azure.Commands.Network.Models.PSPublicIpAddressDnsSettings"
$pip.DnsSettings.DomainNameLabel = "examplemail"
$pip.DnsSettings.ReverseFqdn = "mail.example.com."
Set-AzPublicIpAddress -PublicIpAddress $pip
```

Added a TXT record, with key example.com and value v=spf1 mx ~all

> This TXT record is known as an <a href="https://en.wikipedia.org/wiki/Sender_Policy_Framework">SPF</a> record. This tells receiving email servers that the servers listed in the MX DNS record of example.com are allowed to send  on behalf of example.com. All others should be marked as suspicious. <a href="https://support.google.com/a/answer/10683907">This Google article</a> explains advanced setup of the SPF record. 

## Postfix

Installed postfix mail server.

```bash
$ sudo DEBIAN_FRONTEND=noninteractive apt-get install postfix
```

Edited /etc/postfix/main.cf. The last 2 lines tell Postfix that emails sent to example.com are forwarded to another domain based on the configured map.

```bash
# Host and site name.
myhostname = mail.example.com
mydomain = example.com
myorigin = example.com

# Virtual aliases.
virtual_alias_domains = example.com
virtual_alias_maps = hash:/etc/postfix/virtual
```

Edited /etc/postfix/virtual.

```bash
# /etc/postfix/virtual

# Forwarding mapping, one from-to address pair per line. The format is:
#     <forward-from-addr> <whitespace> <forward-to-addr>
recipient1@example.com  somebody@gmail.com
recipient2@example.com  someone@gmail.com
```

Updated lookup table.

```bash
$ sudo postmap /etc/postfix/virtual
```

# SRS and SPF

While the setup until now should be sufficient to forward emails sent to receipient1@example.com to Gmail, email forwarding breaks SPF. Gmail won't accept these emails coming from sender1@hotmail.com since mail.example.com is not an authorized server to send emails on behalf of hotmail.com (obviously). <a href="http://www.open-spf.org/SRS/">SRS</a> is the fix. SRS rewrites the sender in a way that works with SPF.

The following steps use <a href="https://github.com/roehling/postsrsd">PostSRSd</a> which implements SRS with Postfix. Since it is not available (at this time) in Ubuntu repos, it needs to be built from source.

```bash
# Setup and download source code from GitHub
$ sudo apt-get install unzip cmake

$ cd /tmp
$ curl -L -o postsrsd.zip \
    https://github.com/roehling/postsrsd/archive/master.zip
$ unzip postsrsd.zip

# Build and install.
$ cd postsrsd-master
$ mkdir build
$ cd build
$ cmake -DCMAKE_INSTALL_PREFIX=/usr ../
$ make
$ sudo make install

# Start SRS
$ sudo service postsrsd start

# Add SRS daemon to startup (Debian/Ubuntu):
$ sudo systemctl enable postsrsd.service
```

Edit /etc/postfix/main.cf

```bash
# PostSRSd settings.
sender_canonical_maps = tcp:localhost:10001
sender_canonical_classes = envelope_sender
recipient_canonical_maps = tcp:localhost:10002
recipient_canonical_classes= envelope_recipient,header_recipient
```

```bash
# Restart Postfix
$ sudo postfix reload
```

# Sanity checks

```bash
# Check for errors
$ postconf 1> /dev/null

# Print config for later diffing
$ postconf > postconf-$(date "+%F")
```

# Hardening

```bash
# Disable VRFY (Verify)
$ sudo postconf -e disable_vrfy_command=yes
$ sudo postfix reload
```

# Take a break

Most common use case of incoming email forwarding set up is done. Continue reading about my outgoing email setup.

<a href="{{ '/posts/thirdpost/' | url }}">Adventures with Postfix: Part 2</a>
