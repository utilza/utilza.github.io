---
layout: post
title: "Base64 in practice: base64url, padding, data URIs, and broken query strings"
description: "What Base64 is for, why it makes data a third larger, how base64url differs, and the encoding mistakes that corrupt tokens and URLs."
---

Base64 turns bytes into text made of 64 safe characters, so binary data can travel through systems built for text: JSON, email, HTML attributes, HTTP headers. It is an encoding. Anyone can reverse it, and it protects nothing. Most Base64 bugs come from forgetting which variant a system expects or what happens to it inside a URL.

## How it works, briefly

Base64 takes 3 bytes (24 bits) at a time and writes them as 4 characters of 6 bits each. That's why encoded data is about a third larger than the original. A 300 KB image becomes roughly 400 KB of text.

When the input isn't a multiple of 3 bytes, the output is padded with `=` so its length is a multiple of 4:

```text
"Ma"   -> TWE=
"M"    -> TQ==
```

The standard alphabet is `A-Z`, `a-z`, `0-9`, `+`, and `/`.

## Text has to become bytes first

Base64 encodes bytes, not characters. Before encoding text, a program has to choose a character encoding, and it should be UTF-8. JavaScript's built-in `btoa()` only accepts characters in the Latin-1 range and throws an error on Persian, Chinese, or emoji. Code that works in testing with English strings breaks on the first real user name.

Encode to UTF-8 bytes first, then to Base64. A [Base64 encoder and decoder](https://utilza.app/developer-tools/base64-encoder-decoder) does it in that order, so any language works, and it runs in the browser, so pasted data isn't uploaded.

## base64url is a different alphabet

`+` and `/` have special meanings in URLs and file names. base64url replaces them with `-` and `_`, and usually drops the `=` padding. JWTs use base64url for all three parts, and so do many API tokens.

The two look almost the same, so mixing them up is easy:

```text
standard:  a+b/c==
base64url: a-b_c
```

A standard decoder fails on `-` and `_`, and a strict base64url decoder may reject `+`, `/`, or `=`. When a token fails to decode, check which alphabet produced it. The encoder above has a URL-safe option that writes `-` and `_` without padding, and its decoder accepts both alphabets and missing padding.

## The plus sign in a query string

This bug is behind a lot of "invalid signature" and "corrupted token" reports. You put standard Base64 in a query string:

```text
https://example.com/verify?token=ab+cd/ef==
```

In query strings, `+` is read as a space. The server receives `ab cd/ef==`, and decoding fails or produces the wrong bytes. `=` and `/` can cause trouble too.

There are two fixes. Use base64url for values that go in URLs, or percent-encode the value before you add it. A [URL encoder](https://utilza.app/developer-tools/url-encoder-decoder) has a component mode, matching JavaScript's `encodeURIComponent`, that encodes everything except letters, digits, and `- _ . ~`. That turns `+` into `%2B` and `=` into `%3D`.

