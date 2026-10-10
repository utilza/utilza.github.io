---
layout: post
title: "Certificate won't issue? Check your CAA records"
description: "What DNS CAA records do, why they make Let's Encrypt and other certificate authorities refuse to issue, and how to check them on a domain and its parents."
---

You run Certbot or click "issue certificate" in a hosting panel, and the request fails with a message about CAA. Or a certificate renews fine for years and then stops after someone changes DNS providers. Both usually point to the same small record type that most people have never looked at.

## What a CAA record does

A CAA record lists the certificate authorities that may issue certificates for a domain. Every public CA has to check it before issuing. If the record exists and doesn't name that CA, the CA must refuse.

```text
example.com.  3600  IN  CAA  0 issue "letsencrypt.org"
example.com.  3600  IN  CAA  0 issuewild ";"
example.com.  3600  IN  CAA  0 iodef "mailto:security@example.com"
```

The tags work like this:

- `issue` names a CA allowed to issue certificates for the name.
- `issuewild` does the same for wildcard certificates such as `*.example.com`. If it's missing, `issue` applies to wildcards too. A value of `";"` means no CA may issue wildcards.
- `iodef` gives an address for reports. It doesn't allow or block anything.

If a domain has no CAA records at all, any CA may issue. That's the default for most domains, which is why the problem only appears after someone adds a record.

## Why issuance fails

The common cases are simple once you see the records.

The CA isn't listed. Someone added `issue "digicert.com"` when the company bought a certificate, and now Let's Encrypt is blocked. Add a second `issue` record for each CA you use. CAA records add up; they don't replace each other.

The value is misspelled. Each CA publishes the exact domain to use. Let's Encrypt's is `letsencrypt.org`. A typo has the same effect as not listing the CA.

Wildcards are blocked. The `issue` record allows your CA, but an `issuewild ";"` record, or an `issuewild` naming a different CA, stops the wildcard certificate.

The record came from a parent domain. This is the case that confuses people.

## CAA climbs the tree

When a CA checks `shop.example.com` and finds no CAA records there, it checks `example.com`, and so on up to the top. The first name with CAA records decides. So a subdomain with no CAA records of its own can still be restricted by its parent.

If a name is a CNAME, the CA follows the alias and checks the target as well. A subdomain pointed at a hosting platform can pick up that platform's rules.

A [CAA lookup](https://utilza.app/dns-tools/caa-lookup) shows the flags, tag, value, and TTL of each CAA record on the name you enter, and says when there are none. It queries only that name, so when a subdomain shows nothing, look up each parent in turn until you reach the registered domain. A [DNS lookup](https://utilza.app/dns-tools/dns-lookup) checks A, AAAA, CNAME, MX, NS, TXT, SOA, and CAA in one pass, which also tells you whether the name is a CNAME.

From a terminal:

```shell
dig shop.example.com CAA +short
dig example.com CAA +short
```

## DNS errors count too
