---
layout: post
title: "Why your API rejects a JWT: a checklist for 401 errors"
description: "How to read a JSON Web Token, check its exp, nbf, aud, and iss claims, and find the signature or key mismatch behind a 401 from your API."
---

A 401 with "invalid token" tells you almost nothing. The token might be expired, signed with the wrong key, meant for another service, or simply sent in the wrong header. The good news is that a JWT carries most of the evidence inside itself, so you can usually find the cause in a few minutes.

## Look inside the token first

A JWT is three base64url strings joined by dots: header, payload, and signature. The first two are plain JSON once decoded. Nothing about them is secret, which is why you should never put a password or personal data in a payload.

Paste the token into a [JWT decoder](https://utilza.app/developer-tools/jwt-decoder) and you get the header and payload as JSON, plus a table that turns `exp`, `iat`, and `nbf` into readable dates and says whether the token has expired. It runs in the browser, so the token isn't uploaded. Still, treat a production token like a password: decode it, then let it expire or revoke it.

The header tells you how it was signed:

```json
{ "alg": "RS256", "typ": "JWT", "kid": "2026-09" }
```

The payload holds the claims your API checks:

```json
{
  "iss": "https://auth.example.com",
  "aud": "orders-api",
  "sub": "user_812",
  "iat": 1791590400,
  "nbf": 1791590400,
  "exp": 1791594000
}
```

## 1. Expired, or not valid yet

`exp`, `nbf`, and `iat` are NumericDate values: seconds since 1 January 1970, UTC. If `exp` is in the past, the token is expired and the only fix is a new one. If `nbf` is in the future, the token isn't valid yet.

The second case often comes from clock drift. The server that issued the token runs a few seconds ahead of the server that checks it, and a freshly issued token fails for the first moments of its life. Most JWT libraries accept a small leeway, often called clock skew. Thirty to sixty seconds is a common setting.

## 2. Milliseconds where seconds belong

JavaScript's `Date.now()` returns milliseconds. If someone builds a token with `exp: Date.now() + 3600000`, the value is a thousand times too large. Depending on the library, that token either never expires or fails validation outright.

Count the digits. A current timestamp in seconds has 10 digits; in milliseconds it has 13. A [Unix timestamp converter](https://utilza.app/developer-tools/unix-timestamp-converter) detects the unit from the digit count and shows the date in UTC and your local time, which makes the mistake obvious.

## 3. The signature doesn't match the key

If the dates are fine, check the signature. Three things go wrong here.

The first is an algorithm mismatch. The header says `HS256`, but the API expects `RS256`, or the other way round. HS algorithms use one shared secret. RS, PS, and ES algorithms sign with a private key and verify with a public key. A server configured for one won't accept the other, and it shouldn't.

The second is key rotation. Identity providers publish their public keys as a JWKS and pick one with the `kid` header. If your API cached an old set of keys, a token signed with the new `kid` fails until the cache refreshes.

