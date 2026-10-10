---
layout: post
title: "Password-protect a folder with .htpasswd on Apache and Nginx"
description: "How to put HTTP Basic authentication in front of a staging site or admin folder: creating the .htpasswd file, choosing a hash, and the Apache and Nginx config."
---

A staging site, a client preview, or an old admin folder often needs a simple lock: one username, one password, no login page to build. HTTP Basic authentication with an `.htpasswd` file does that with a few lines of server config. It is also easy to set up in a way that leaks the password file or sends the password in the clear, so here is the setup I use.

## What Basic authentication does

When a protected URL is requested, the server answers with `401` and a `WWW-Authenticate` header. The browser shows its own login box, then sends every following request with:

```text
Authorization: Basic dXNlcjpwYXNzd29yZA==
```

That value is just `user:password` in Base64. Anyone who can read the traffic can decode it, so Basic authentication is only acceptable over HTTPS. Set up the certificate first.

## 1. Create the password file

Each line of an `.htpasswd` file holds a username and a password hash:

```text
alice:$2y$10$Vq1eS7dN0lO5m2l8gk1yXe5JQ0wq9mQ8kH0rQ2sYxW1bT3uZ4aP6K
```

With Apache's tools installed, the `htpasswd` command creates it. `-B` selects bcrypt, `-c` creates a new file, and leaving `-c` out adds a user to an existing one:

```shell
htpasswd -c -B /etc/apache2/.htpasswd alice
htpasswd -B /etc/apache2/.htpasswd bob
```

If you're on shared hosting without shell access, or on a machine without Apache's tools, an [htpasswd generator](https://utilza.app/developer-tools/htpasswd-generator) produces the line in the browser. It supports bcrypt, APR1-MD5, and SHA-1, recommends bcrypt with a cost of 10 to 12, and also writes the Apache and Nginx config snippets. The password is hashed locally and not sent anywhere.

Choose bcrypt where the server supports it. SHA-1 is unsalted and fast to crack. APR1-MD5 is salted but old, so use it only as a fallback.

## 2. Keep the file outside the web root

Put `.htpasswd` somewhere the web server won't serve, such as `/etc/apache2/` or a folder above `public_html`. Apache's default config refuses to serve files starting with `.ht`, but a changed config, a different server, or a backup copy named `htpasswd.txt` can expose it. A downloadable hash file invites an offline attack on the password.

## 3. Apache config

In the folder you want to protect, or in its `<Directory>` block in the main config, add:

```apache
AuthType Basic
AuthName "Restricted"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user
```

`AuthUserFile` needs the full path from the server's root, not a path relative to the site. `AuthName` is the text some browsers show in the login box.
