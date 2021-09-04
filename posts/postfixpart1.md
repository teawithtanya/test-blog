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

## Linux VM

I started with Azure VM using Linux (ubuntu 20.04) and assigned a static public IP.

## DNS

+ Added A record for mail.example.com pointing to public IP
+ Added MX record for example.com pointing to mail.example.com
+ Added a PTR (reverse DNS) record mapping the public IP address to mail.example.com.

```powershell

$pip = Get-AzPublicIpAddress -Name "mail-ip" -ResourceGroupName "mail_group"
$pip.DnsSettings = New-Object -TypeName "Microsoft.Azure.Commands.Network.Models.PSPublicIpAddressDnsSettings"
$pip.DnsSettings.DomainNameLabel = "examplemail"
$pip.DnsSettings.ReverseFqdn = "mail.example.com."
Set-AzPublicIpAddress -PublicIpAddress $pip
```

+ Added a TXT record, with key example.com and value v=spf1 mx ~all

# Next

Continue reading about my setup

<a href="{{ '/posts/thirdpost/' | url }}">Adventures with Postfix: Part 2</a>
