---
layout: post
title: "Email going to spam? Check your MX, SPF, DKIM, and DMARC records"
description: "How to read the DNS records that decide whether mail from your domain is trusted, and the common SPF, DKIM, and DMARC mistakes that send it to spam."
---

When mail from a new domain lands in spam, or a contact form's messages never arrive, the cause is often in DNS. Receiving servers look up a handful of records to decide whether a message really came from your domain. If those records are missing, duplicated, or out of date, the message looks forged.

Gmail and Yahoo have required SPF, DKIM, and DMARC from bulk senders since 2024, and many other providers treat their absence as a bad sign for everyone else.

## MX: where your mail goes

MX records say which servers accept mail for your domain. They don't affect whether your outgoing mail is trusted, but they're the first thing to check when mail sent to you goes missing.

```text
example.com.  3600  IN  MX  1   mx1.mailhost.net.
example.com.  3600  IN  MX  10  mx2.mailhost.net.
```

The lower number is tried first. An [MX lookup](https://utilza.app/dns-tools/mx-lookup) lists the mail servers in priority order with their TTL and the IPv4 and IPv6 addresses each one resolves to. A record with priority 0 and a host of `.` is a null MX: it says the domain doesn't accept mail at all. It belongs on domains that never receive mail, and nowhere else.

## SPF: which servers may send for you

SPF is a TXT record on the domain itself that lists the servers allowed to send mail as `@example.com`:

```text
v=spf1 include:_spf.mailhost.net include:sendgrid.net ~all
```

Three mistakes cause most SPF failures.

A domain must have one SPF record. When you add a new sending service, the instinct is to add a second `v=spf1` record. Two SPF records is an error, and receivers treat the result as a failure. Merge the `include:` entries into one record instead.

SPF allows at most 10 DNS lookups while it's evaluated. Each `include:`, `a`, `mx`, and `redirect` counts, and so do the lookups inside the records you include. A domain with five or six services can pass the limit without anyone noticing.

The ending matters. `-all` tells receivers to reject anything not listed, `~all` asks them to treat it as suspicious, and `+all` lets anyone send as you. Never publish `+all`.

## DKIM: a signature on every message

DKIM signs each message with a private key held by your mail provider. The matching public key is published in DNS under a selector the provider chooses:

```text
selector1._domainkey.example.com  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBgkqh..."
```

Because the selector varies, you have to know it before you can look the record up. Your provider's setup page names it, and so does the `s=` value in the `DKIM-Signature` header of any message it has sent.

The usual DKIM problem is a key that was never published, or was published under the wrong name. Some DNS panels add the domain to every record automatically, so typing `selector1._domainkey.example.com` produces `selector1._domainkey.example.com.example.com`.

## DMARC: what to do when checks fail

DMARC is a TXT record at `_dmarc.example.com`. It tells receivers what to do with mail that fails SPF and DKIM alignment, and where to send reports:

