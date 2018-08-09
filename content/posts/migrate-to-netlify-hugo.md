---
title: "Migrate to Netlify Hugo"
date: 2018-08-05
draft: false
---

Log into your FreeBSD host in which you want to add an SMTP relay to.

First we need to hop into the `/etc/mail` directory.

```
cd /etc/mail
```

Then we want to create a copy of the freebsd.mc file to our `HOSTNAME.mc`. This enables us to safely make changes to the sendmail configuration.

```
cp freebsd.mc `hostname`.mc
```

**Warning**: Never make changes to the sendmail.cf file directly. They could get overwritten if someone updates the sendmail configuration properly using the `.mc` files.

## Adding SmartHost to Configuration

Open the `HOSTNAME.mc` file in your favorite editor:

```
vim `hostname`.mc
```

Search for 'SMART_HOST' and uncomment/update the line:

```
define(`SMART_HOST', `relay.domain.tld')
```

Save the file and commit our changes.

## Enable Changes

```
make
make install restart
```

Thats it, you are all set, sendmail is running with a relay to your ISP or corporate relay.
