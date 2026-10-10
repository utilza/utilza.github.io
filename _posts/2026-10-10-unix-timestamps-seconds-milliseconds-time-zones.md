---
layout: post
title: "Unix timestamps: seconds, milliseconds, and the time zone they don't have"
description: "Why a Unix timestamp shows the wrong date, how to tell seconds from milliseconds, and how to convert timestamps to local time without off-by-hours bugs."
---

Most timestamp bugs come from two facts that are easy to forget. A Unix timestamp has no time zone, and different languages count it in different units. Mix those up and a date lands in 1970, in the year 57000, or a few hours off.

## What the number means

A Unix timestamp counts seconds since 00:00:00 UTC on 1 January 1970. `1700000000` is 2023-11-14 at 22:13:20 UTC, everywhere on Earth, at the same instant. The number doesn't change with your location. Only the way you display it does.

That is why storing timestamps is usually safe and displaying them is where things break. The conversion to a wall-clock time needs a time zone, and code that doesn't name one silently uses whatever the server or browser is set to.

## Seconds or milliseconds?

Languages don't agree on the unit:

| Source | Unit |
| --- | --- |
| `time()` in PHP, `date +%s` in a shell | seconds |
| `time.time()` in Python | seconds, as a float |
| `Date.now()` in JavaScript, `System.currentTimeMillis()` in Java | milliseconds |

The quickest test is to count digits. A current timestamp in seconds has 10 digits. In milliseconds it has 13.

Pass milliseconds to a function that expects seconds and you get a date tens of thousands of years in the future. Pass seconds to JavaScript's `new Date()`, which expects milliseconds, and you get a date in January 1970:

```js
new Date(1700000000)        // 1970-01-20T16:13:20.000Z
new Date(1700000000 * 1000) // 2023-11-14T22:13:20.000Z
```

A [Unix timestamp converter](https://utilza.app/developer-tools/unix-timestamp-converter) reads the unit from the digit count (up to 11 digits as seconds, 12 to 14 as milliseconds, and longer values as micro- or nanoseconds), tells you which one it picked, and lets you override it. Very small millisecond values, such as `5000`, look like seconds, so check the unit it chose.

## The time zone comes from somewhere

When code turns a timestamp into a date without naming a zone, the zone comes from the environment:

- A server's system time zone, often UTC on cloud machines.
- The browser's time zone, which is the visitor's, not yours.
- A database session setting.

So the same row can show 22:13 in the server log, 01:43 the next day for a visitor in Tehran, and 17:13 for one in New York. None of them is wrong, but a report that mixes them is.

The safest habit is to keep timestamps in UTC everywhere they're stored and sent, and convert to a named zone only when you display them. Name the zone with an IANA identifier such as `America/New_York` or `Asia/Tehran`. Abbreviations like `EST` or `IST` are ambiguous and don't account for daylight saving time.

## Daylight saving time makes some hours odd

In zones with daylight saving time, one local hour is skipped each spring and one repeats each autumn. A local time like 02:30 can be impossible on one date and happen twice on another. Code that turns local times back into timestamps has to decide what to do with those cases.

A [time zone converter](https://utilza.app/converters/timezone-converter) uses the IANA database built into your browser, works out each zone's offset for the date you pick, and flags times that don't exist or occur twice. You can add several cities at once, which helps when the question is really "what time is this for the team in Berlin?"
