---
layout: post
title:  "Walkthough Guide: How to Setup Kerberos Authentication on Linux"
date:   2023-11-12 13:22:26 +0800
categories: authentication kerberos
---

The objective of this guide is to setup Kerberos on Linux, and authenticate on a server via SSH using Kerberos.

This can be useful to:
* Understand the basics of Kerberos installation, setup, administration on Linux.
* Setting up test environments for developing Kerberized applications.

# Pre-Requisites

* A basic understanding on Linux & bash.
* Familiarity with the [Kerberos Protocol](https://kerberos.org/software/tutorial.html).
* Three Linux hosts.
  * One will be hosting the KDC, we'll call it kdc.example.com
  * One will act as a SSH server, we'll call it server.example.com
  * One will act as the SSH client, well call it client.example.com

An easy way is to get the Linux hosts is to  create virtual machines on a cloud provider. In this guide, we will be using Ubuntu, but this should work for some other distributions (you'll just need to find the correct packages for Kerberos).

# Setting up the KDC

In this guide, we will choose EXAMPLE.COM as the Kerberos realm.

SSH to the host on which you want to install the KDC.

First of all, let's change the hostname to `kdc.example.com`.

```
$ sudo hostnamectl set-hostname kdc.example.com  
```

Then we'll install the KDC and admin-server packages:

```
$ sudo apt install -y krb5-kdc krb5-admin-server
```

You'll be propmted for a few things:
* Default realm: set it to EXAMPLE.COM (in caps)
* Kerberos Servers: kdc.example.com
* Administrative Server: kdc.example.com

Now let's create the Kerberos realm, database, and restart the services:

```
$ sudo krb5_newrealm
$ sudo systemctl restart krb5-kdc
$ sudo systemctl restart krb5-admin-server
```

We can now create a user (let's call it john) and Service Principal for the SSH Server:

```
$ sudo kadmin.local
addprinc john
addprinc -randkey host/server.example.com
ktadd -k /tmp/server.keytab host/server.example.com
quit
```

We'll then need to copy the keytab in /tmp/server.keytab to the server host at /etc/krb5.keytab (using scp for example).

# Setting up the SSH Server

Let's change the hostname and setup a few local DNS records:

```
$ sudo hostnamectl set-hostname server.example.com
```

Edit `/etc/hosts` and add these two lines (replace with the real IP addresses):

```
1.2.3.4 kdc.example.com
2.3.4.5 server.example.com
```

Install the kerberos client package:

```
$ sudo apt install -y krb5-user
```

You'll be propmted for a few things:
* Default realm: set it to EXAMPLE.COM (in caps)
* Kerberos Servers: kdc.example.com
* Administrative Server: kdc.example.com

Edit `/etc/krb5.conf`, and under the `[domain_realm]` section, add these two lines:

```
.example.com = EXAMPLE.COM
example.com = EXAMPLE.COM
```

Check that the keytab is valid and that we can authenticate with it:

```
$ sudo kinit -kt /etc/krb5.keytab
```

Then destroy the credential cache:

```
$ sudo kdestroy
```

Edit the `/etc/ssh/sshd_config` file, and enable these two options if they are not already:

```
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
```

Restart the SSH daemon:

```
$ sudo systemctl restart sshd
```

Finally, add the user account:

```
$ sudo useradd -m -s /bin/bash john
```

# Setting up the SSH Client

Let's change the hostname and setup a few local DNS records:

```
$ sudo hostnamectl set-hostname client.example.com
```

Edit `/etc/hosts` and add these two lines (replace with the real IP addresses):

```
1.2.3.4 kdc.example.com
2.3.4.5 server.example.com
```

Install the kerberos client package:

```
$ sudo apt install -y krb5-user
```

You'll be propmted for a few things:
* Default realm: set it to EXAMPLE.COM (in caps)
* Kerberos Servers: kdc.example.com
* Administrative Server: kdc.example.com

Then kinit:

```
$ kinit john
```

Finally, ssh to the server host, and voila!

```
$ ssh john@server.example.com
```

# Debugging Tips

Run the SSH client with more verbosity:

```
$ KRB5_TRACE=/dev/stdout ssh -vvv john@server.example.com
```

On the SSH server, consider running a SSH daemon on a different port to see the logging output more easily:

```
$ sudo /usr/sbin/sshd -p 2222 -d -d -d
```
