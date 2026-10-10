---
layout: post
title: "GitHub Actions cron schedule: why a workflow runs at the wrong time, or never"
description: "How the schedule trigger in GitHub Actions reads cron, which time zone it uses, and the branch, load, and inactivity rules that stop scheduled workflows from running."
---

A scheduled workflow is a few lines of YAML, and it fails quietly. Nothing turns red when a nightly job runs five hours early or stops running after two months. You only notice when the report someone expected never arrives.

These are the causes I have run into, roughly in the order I check them.

## How the schedule trigger reads cron

The `schedule` event takes one or more POSIX cron expressions with five fields:

```text
minute (0-59)  hour (0-23)  day of month (1-31)  month (1-12)  day of week (0-6, Sunday = 0)
```

```yaml
on:
  schedule:
    - cron: "15 4,5 * * *"
```

That line runs at 04:15 and 05:15 every day. Quote the expression. A value that starts with `*` is YAML alias syntax, and the file won't parse.

GitHub Actions doesn't accept the shortcuts some cron daemons allow, such as `@daily` or `@hourly`. Write the five fields out. The shortest interval it allows is once every 5 minutes.

Reading an unfamiliar expression in your head is where most mistakes start. Paste it into a [cron expression parser](https://utilza.app/developer-tools/cron-expression-parser) and it describes the schedule in plain English and lists the next runs, in UTC or your local time. Keep in mind that the parser also accepts `@daily` and the other macros, while Actions does not.

## The time zone is UTC unless you say otherwise

By default, scheduled workflows run in UTC. A job written as `0 9 * * *` by someone in Tehran or New York runs at 09:00 UTC, which is not 09:00 for either of them.

GitHub now lets you set an IANA time zone next to the expression:

```yaml
on:
  schedule:
    - cron: "30 5 * * 1-5"
      timezone: "America/New_York"
```

If you'd rather keep the schedule in UTC, convert the local time first with a [time zone converter](https://utilza.app/converters/timezone-converter), and remember the offset changes when daylight saving time starts or ends. With a `timezone` set, GitHub handles the change for you. A run that falls inside the hour skipped in spring moves to the next valid time, so a 02:30 job runs at 03:00 that day.

## Day of month and day of week together

This one surprises people who have used cron for years. When both the day-of-month and day-of-week fields are restricted, standard cron runs the job when either one matches, not when both do.

```yaml
- cron: "0 6 1 * 1"
```

That line runs at 06:00 on the 1st of every month and also on every Monday. If you meant "the 1st, but only when it's a Monday", cron can't express that alone. Schedule it for every Monday and check the date in a step.

A parser shows the problem straight away, because the list of next runs has far more dates than you expected. If you're building a schedule from scratch, a [cron expression generator](https://utilza.app/developer-tools/cron-expression-generator) produces a five-field line from a few choices, and its output works in GitHub Actions as it is.

## The workflow file has to be on the default branch
