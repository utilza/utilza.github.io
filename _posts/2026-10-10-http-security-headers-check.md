---
layout: post
title: "HTTP security headers: what to send and how to check them"
description: "The response headers that protect a website, HSTS, CSP, X-Content-Type-Options, frame protection, Referrer-Policy, and Permissions-Policy, and how to confirm the server really sends them."
---

A few response headers block whole classes of attacks for the price of one line of server config each. They're also easy to get subtly wrong: set on the home page but not on errors, overwritten by a later block, or sent by the origin and dropped by a proxy. This is the set I check on every site, and how I check it.

## The headers

### Strict-Transport-Security

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

HSTS tells the browser to use HTTPS for your domain for the next `max-age` seconds, even if someone types `http://` or follows an old link. It only takes effect when received over HTTPS.

Start with a short `max-age`, such as a day, and raise it once you're sure every subdomain serves HTTPS. `includeSubDomains` covers all of them, including ones you forgot about. `preload` asks browsers to ship your domain in their built-in HTTPS list, and removal takes a long time, so add it last.

### Content-Security-Policy

```text
Content-Security-Policy: default-src 'self'; img-src 'self' data:; frame-ancestors 'none'
```

CSP lists where scripts, styles, images, and other resources may load from. It is the strongest defense against cross-site scripting, and the hardest to roll out, because every third-party script, font, and analytics tag needs an entry.

Use `Content-Security-Policy-Report-Only` first. The browser reports what would have been blocked without blocking it, so you can fix the policy before it breaks the site.

### X-Content-Type-Options

```text
X-Content-Type-Options: nosniff
```

Stops browsers from guessing a file's type when the `Content-Type` header says otherwise. Without it, an uploaded text file can be run as a script in some situations. There's no reason not to send it.

### Frame protection

```text
X-Frame-Options: DENY
```

Prevents other sites from loading yours in a frame, which is how clickjacking works. The modern equivalent is CSP's `frame-ancestors` directive. Sending both covers older browsers. Use `SAMEORIGIN` instead of `DENY` if your own site frames its pages.

### Referrer-Policy

```text
Referrer-Policy: strict-origin-when-cross-origin
```

Controls how much of the current URL is sent to other sites when a visitor follows a link. This value sends only the origin to other sites and nothing when going from HTTPS to HTTP. It matters when URLs contain search terms, IDs, or tokens.
