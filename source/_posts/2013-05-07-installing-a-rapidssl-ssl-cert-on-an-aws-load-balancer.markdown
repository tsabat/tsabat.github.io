---
layout: post
title: "Installing a RapidSSL SSL cert on an AWS Load Balancer"
date: 2013-05-07 20:25
comments: true
categories: 
---

Over at [CodePen](http://codepen.io) it came time to renew our SSL cert.  I dutifully follwed the [setup instructions](https://knowledge.rapidssl.com/support/ssl-certificate-support/index?page=content&id=SO21322&actp=search&viewlocale=en_US), but I was greeted with this error:

> Invalid Public Key Certificate

After talking with the support staff at RapidSSL, I was told to reverse the Intermediate CA Bundle.  The example from [their instructions](https://knowledge.rapidssl.com/support/ssl-certificate-support/index?page=content&id=SO21856&actp=search&viewlocale=en_US&searchid=1367983787858) looks like this:

```bash
-----BEGIN CERTIFICATE----
Primary Intermediate CA
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
Secondary Intermediate CA
-----END CERTIFICATE-----

Needs to be switched to..

-----BEGIN CERTIFICATE-----
Secondary Intermediate CA
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
Primary Intermediate CA
-----END CERTIFICATE-----
```

I'm noting this here so in 2016 when we have to renew our SSL Cert, we'll know what to do.
