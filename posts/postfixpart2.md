---
title: Adventures with Postfix - Part 2.
description: Enabling outgoing email on Postfix.
date: 2021-09-04
tags:
  - postfix
layout: layouts/post.njk
---

This post focuses on my outgoing email setup for a custom domain. Read my previous post for the initial setup with Postfix to forward incoming email.

<a href="{{ '/posts/postfixpart1/' | url }}">Adventures with Postfix: Part 1</a>

## Sender Authentication

For outgoing email from Postfix (using Gmail or another client), I setup authentication to only allow known senders to send email. Postfix works with Cyrus (default) or Dovecat for <a href="http://www.postfix.org/SASL_README.html">SASL authentication</a>.

```bash
# Installs Cyrus SASL
$ sudo apt-get install sasl2-bin libsasl2-modules

# Creates user name & password
$ sudo saslpasswd2 -c -u example.com recipient1

# List users
$ sudo sasldblistusers2

# Restricts access to password file
$ sudo chmod 400 /etc/sasldb2
$ sudo chown postfix /etc/sasldb2
```

Edited SASL config file /etc/postfix/sasl/smtpd.conf

```bash
pwcheck_method: auxprop
auxprop_plugin: sasldb
mech_list: PLAIN LOGIN CRAM-MD5 DIGEST-MD5 NTLM
log_level: 7
```

Enable saslauthd in /etc/default/saslauthd

```bash
# This needs to be uncommented before saslauthd will be run automatically
START=yes

PWDIR="/var/spool/postfix/var/run/saslauthd"
PARAMS="-m ${PWDIR}"
PIDFILE="${PWDIR}/saslauthd.pid"

MECHANISMS="sasldb"

# for chrooted postfix services
OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd -r"
```

```bash
# Some more changes / hacks I found to get things working
$ sudo dpkg-statoverride --force --update --add root sasl 755 /var/spool/postfix/var/run/saslauthd
$ sudo ln -s /etc/default/saslauthd /etc/saslauthd
$ sudo service saslauthd start

# Test sasl auth
$ sudo testsaslauthd -u recipient1 -r example.com -p <password> -f /var/spool/postfix/var/run/saslauthd/mux
```

SASL authentication is PLAIN text. Next, I setup TLS encryption to avoid snooping.

## TLS Encryption

Postfix comes with self-signed certificates to use for TLS Encryption. I even tried with my own generated self-signed certificates. But I found out the hard way, that <a href="https://workspaceupdates.googleblog.com/2020/04/improve-email-security-in-gmail-with-TLS.html">Gmail started enforcing strict email security</a> starting around April 2020. Fortunately, <a href="https://letsencrypt.org/">Let's Encrypt</a> makes it easy and free to get a properly CA signed certificate. I used the following guide for this setup.

👍 [How to secure Postfix using Let’s Encrypt](https://upcloud.com/community/tutorials/secure-postfix-using-lets-encrypt/)

```bash
# Installs certbot to manage generating keys and
# interacting with Let's Encrypt via their ACME process
$ sudo apt install certbot

# Runs certbot in standalone mode since we don't have a web server.
# Ensure Port 80 is open though since certbot uses a temporary web server during the process.
$ sudo certbot certonly --standalone -d mail.example.com
```

## E-mail Relay

Next, I configure Postfix to enable email relay for authenticated users by editing `/etc/postfix/master.cf`.

```bash
smtp       inet n       -       y       -       -       smtpd

submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=may
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_local_domain=$mydomain
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

Some of the above TLS settings seemed to be overridden by defaults in `main.cf`. It's not obvious the right way to avoid repeating this info.

```bash
# Adds the location of letsencrypt certificate and private key
$ sudo postconf -e smtpd_tls_cert_file=/etc/letsencrypt/live/mail.example.com/fullchain.pem
$ sudo postconf -e smtpd_tls_key_fie=/etc/letsencrypt/live/mail.example.com/privkey.pem

$ sudo postfix reload
```

## Verify TLS Setup

I used a random website I found [www.checktls.com](https://www.checktls.com/TestReceiver) to validate my setup.

To validate the actual TLS authentication, I used OpenSSL.

```bash
# Generate base64 encoded text of username and password
$  echo -ne '\000recipient1\000secret' | openssl base64
AHJlY2lwaWVudDEAc2VjcmV0

# Connect to SMTP server over a TLS encrypted connection
$ openssl s_client -connect mail.example.com:587 -starttls smtp
# lots of output
# Use EHLO to check if server supports auth
EHLO localhost

250-mail.example.com
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-AUTH PLAIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING

# Using the supported AUTH PLAIN method with base64 credentials
AUTH PLAIN AHJlY2lwaWVudDEAc2VjcmV0

235 2.7.0 Authentication successful

# Yay! Authentication successful. Time to quit.
quit

221 2.0.0 Bye
closed
```

## References

Some references I used when debugging my setup

+ <http://www.postfix.org/TLS_README.html>
+ <https://kruyt.org/postfix-and-tls-encryption/>
+ <https://bobcares.com/blog/tls-negotiation-failed-the-certificate-doesnt-match-the-host/>
+ <https://serverfault.com/questions/711006/postfix-error-535-5-7-8-error-authentication-failed-authentication-failure>
+ <https://serverfault.com/questions/383212/sasl-plain-authentication-failed-another-step-is-needed-in-authentication>
+ <https://docs.iredmail.org/debug.postfix.html>
