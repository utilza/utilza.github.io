---
layout: post
title: "UUID v4 or UUID v7 for database keys?"
description: "How UUID v4 and v7 differ, why random keys slow down B-tree indexes, what v7 reveals about a row, and when a UUID is the wrong choice for a secret token."
---

UUIDs let any server, client, or offline device create an ID without asking the database for the next number. For years the default choice was version 4, which is random. Version 7, standardized in RFC 9562 in 2024, puts a timestamp at the front. For primary keys, that difference matters more than it looks.

## What's inside each version

Both are 128 bits, written as 36 characters:

```text
v4: 3f2c9a1e-7b4d-4c8e-9a12-5e6f7a8b9c0d
v7: 01928f3a-5c21-7d4e-8b3f-2a1c9e8d7f60
```

A v4 UUID is 122 random bits plus 6 fixed bits for the version and variant. Two v4 IDs created a millisecond apart have nothing in common.

A v7 UUID starts with a 48-bit Unix timestamp in milliseconds, followed by the version bits and random bits. IDs created later sort after IDs created earlier, both as text and as bytes. You can see the version in the first digit of the third group: `4` or `7`.

## Why random keys are slow to insert

Most databases store a primary key in a B-tree index, which keeps keys in sorted order. Sequential keys, like an auto-increment integer, always go at the end of the index. The last page stays in memory, and pages fill up neatly.

Random v4 keys land anywhere in the index. Each insert can touch a different page, which means more disk reads once the index is larger than memory, more page splits, and half-empty pages. On a small table you won't notice. On a table with tens of millions of rows and a high insert rate, it shows up as slower writes and a larger index. MySQL's InnoDB feels it more than most, because it stores the whole row in primary key order.

v7 keys arrive in roughly increasing order, so inserts behave much like an auto-increment column while keeping the benefits of IDs you can generate anywhere.

Some databases now generate v7 natively. PostgreSQL 18, for example, added a `uuidv7()` function. Elsewhere, generate it in the application.

## What v7 gives away

The timestamp in a v7 UUID is readable. Anyone who sees the ID can work out when the row was created, to the millisecond. For orders, posts, and log entries that's usually harmless, and sometimes useful. For a user account where the sign-up date is sensitive, or an ID in a public URL that shouldn't reveal activity patterns, it may not be.

Neither version should be used as a secret. A v7 UUID's first 48 bits are predictable, and even v4 isn't guaranteed to come from a secure random source in every library. For password reset links, API keys, and session IDs, use a random token built for the job.

## Storage

Store UUIDs in a native `uuid` column or as 16 bytes of binary, not as a 36-character string. Text takes more than twice the space, and every index on it grows to match. In MySQL, `BINARY(16)` with `UUID_TO_BIN()` and `BIN_TO_UUID()` is the usual pattern.

## Generate a few to compare

A [UUID generator](https://utilza.app/developer-tools/uuid-generator) creates v4 or v7 UUIDs, 1 to 1,000 at a time, with options for uppercase, hyphens, and braces. Its randomness comes from `crypto.getRandomValues` and `crypto.randomUUID`, never `Math.random`. Generate ten v7 IDs and you'll see the shared prefix and the sorted order immediately, which is a quick way to explain the difference to a team.

For secrets, a [random token generator](https://utilza.app/developer-tools/token-generator) builds tokens from hex, base64url, letters and digits, or a custom character set, choosing each character with `crypto.getRandomValues`.

## Which one to use

- New tables where inserts are frequent: v7.
- IDs where creation time must stay private: v4, or keep an internal v7 key and expose a separate random ID.
- Existing tables with v4 keys that perform fine: leave them.
- Password resets, API keys, sessions: a random token, not a UUID.

