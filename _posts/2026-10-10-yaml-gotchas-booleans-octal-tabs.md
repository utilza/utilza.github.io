---
layout: post
title: "YAML gotchas: when no means false and 0755 means 493"
description: "The YAML parsing surprises behind broken configs: tabs, unquoted booleans, octal numbers, version strings, and the differences between YAML 1.1 and 1.2."
---

YAML looks forgiving, which is the problem. It guesses the type of every unquoted value, and the guess depends on which version of the spec your parser follows. A config file can pass in one tool and turn into something else in another.

These are the surprises I run into most, with the fix for each.

## Tabs are not allowed for indentation

YAML indents with spaces only. A single tab in the indentation is a syntax error, and editors that mix tabs and spaces make it invisible. The error message usually points at a line near the tab, not at the tab itself.

Set your editor to insert spaces in `.yml` and `.yaml` files, and turn on visible whitespace when a file won't parse. A [YAML validator](https://utilza.app/developer-tools/yaml-validator) reports the line and column of the error, gives hints for common causes like bad indentation and stray colons, and flags tab indentation directly. It runs in the browser, so a config with secrets in it isn't uploaded.

## no, yes, on, and off

YAML 1.1 treats a long list of words as booleans: `yes`, `no`, `on`, `off`, `y`, `n`, and their capitalized forms. Many widely used parsers, including PyYAML, still follow 1.1. So this:

```yaml
countries:
  - GB
  - NO
  - SE
```

becomes `["GB", false, "SE"]`. That is the well-known Norway problem. The same rule turns `on:` at the top of a GitHub Actions workflow into the key `true` when you read the file with PyYAML.

YAML 1.2, published in 2009, reduced booleans to `true` and `false`. Parsers that follow 1.2 read `NO` as a string. Since you don't always control the parser, the safe fix is to quote any string that could be read as something else:

```yaml
countries: ["GB", "NO", "SE"]
```

## Numbers that aren't numbers

Unquoted values that look like numbers become numbers, and that breaks several kinds of data.

File modes: in YAML 1.1, `mode: 0755` is an octal number, so the program receives 493. YAML 1.2 writes octal as `0o755` and reads `0755` as the decimal 755. Either way, quote it if the program expects the string.

Version numbers: `version: 1.10` is the float 1.1. `version: 3.0` is the float 3. Quote versions.

Leading zeros: a ZIP code or ID like `01234` can lose its leading zero, or be read as octal.

Times: some 1.1 parsers read `22:30` as a sexagesimal number, 1350.

## Strings with special characters

A colon followed by a space starts a key, and `#` after a space starts a comment. So this breaks:

