---
layout: post
title: "Test a sed command before you run sed -i on real files"
description: "How to build and check a sed substitution safely: GNU vs macOS -i, delimiters, basic vs extended regex, and a dry run before editing files in place."
---

`sed -i` edits files in place, with no undo. A pattern that matches a little more than you meant, run across a directory of config files, can do a lot of damage in a second. Most of that risk goes away if you test the expression first and know which `sed` you're running.

## Dry run first, then edit

Without `-i`, sed prints the result and leaves the file alone. Use that as your preview:

```shell
sed 's/http:/https:/g' config.ini | less
```

To see only the lines that would change, combine `-n` with the `p` flag:

```shell
sed -n 's/http:/https:/gp' config.ini
```

When the output looks right, add `-i`. For anything you can't easily restore from version control, keep a backup:

```shell
sed -i.bak 's/http:/https:/g' config.ini
```

`-i.bak`, with the suffix attached, works on both GNU sed and macOS sed. It leaves `config.ini.bak` next to the edited file.

## GNU sed and macOS sed differ on -i

Linux ships GNU sed. macOS ships BSD sed. They disagree on the in-place flag:

```shell
# Linux (GNU sed)
sed -i 's/foo/bar/' file.txt

# macOS (BSD sed)
sed -i '' 's/foo/bar/' file.txt
```

BSD sed requires an argument after `-i`, so `''` means "no backup". On Linux, that same `''` is read as the script, and the command fails or edits the wrong thing. This is why a script written on a Mac breaks in CI, and the reverse. Use `-i.bak` when a script has to run on both, or install GNU sed on macOS as `gsed`.

A few other GNU features also fail on macOS, including `\b` word boundaries and one-line `a`, `i`, and `c` commands.

## Pick a delimiter that isn't in the pattern

The `s` command uses `/` by default, so paths and URLs need escaping:

```shell
sed 's/\/var\/www\/old/\/var\/www\/new/g' site.conf
```

Any character can be the delimiter. Choose one that doesn't appear in the text:

