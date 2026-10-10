---
layout: post
title: "Moving a GitHub Pages site to a custom domain: what I check before launch"
description: "DNS records, HTTPS, redirects, image weight, favicons, share previews, and sitemaps for a GitHub Pages site, with the commands and free tools to check each one."
---

GitHub Pages hosts a static site for free at `username.github.io`. Putting it on your own domain takes a few settings, but the problems tend to show up after launch: a friend in another country still sees the old site, the browser warns about the certificate, a shared link has no preview image, or the home page crawls on a phone.

I go through the steps below before I tell anyone a site is live.

## 1. Point the domain at GitHub

For the apex domain (`example.com`), add four A records at your DNS provider:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

If you want IPv6 as well, add the four AAAA records:

```
2606:50c0:8000::153
2606:50c0:8001::153
2606:50c0:8002::153
2606:50c0:8003::153
```

For `www.example.com`, add a CNAME record that points to `username.github.io` (or `orgname.github.io`). Leave the repository name out of it, and don't use the `*.pages.github.io` address that the Pages settings screen shows.

Then open the repository, go to **Settings > Pages**, and type the domain into **Custom domain**. If you publish from a branch, GitHub commits a `CNAME` file to it. Don't delete that file.

Don't create wildcard records such as `*.example.com`. GitHub's documentation says they put you at risk of a domain takeover even when the domain is verified. Do verify the domain in your account or organization settings, though, so nobody else can attach it to their own Pages site.

## 2. Check that the records have spread

GitHub says DNS changes can take up to 24 hours to propagate. You can check your own view with `dig`:

```shell
dig example.com +noall +answer -t A
dig www.example.com +nostats +nocomments +nocmd
```

Windows doesn't ship `dig`, but PowerShell has an equivalent:

```powershell
Resolve-DnsName example.com -Type A
```

Your answer only tells you what your own resolver has cached. To see what other people get, ask several public resolvers at once with a [DNS propagation checker](https://utilza.app/dns-tools/dns-propagation-checker). If some of them still return the old IP, wait. If all of them return the wrong value, the record itself is wrong and waiting won't fix it.

## 3. Turn on HTTPS

Back in **Settings > Pages**, tick **Enforce HTTPS**. GitHub notes that the option can take up to 24 hours to become available. When the checkbox stays greyed out for longer than that, the DNS records from step 1 are usually the cause.

Once it's on, look at the certificate itself. It should cover both `example.com` and `www.example.com`, and you want to know when it expires. An [SSL certificate checker](https://utilza.app/network-tools/ssl-certificate) shows the names on the certificate, the issuer, and the expiry date without opening developer tools.

## 4. Follow the redirects

When DNS is set up for both the apex and `www`, GitHub Pages redirects one to the other, depending on which one you typed into the Custom domain field. Add the HTTP to HTTPS redirect and a visitor can pass through two hops before reaching the page.

Test the four starting points: `http://example.com`, `http://www.example.com`, `https://example.com`, and `https://www.example.com`. Each one should end on the same URL with a 200 status and no loop. A [redirect checker](https://utilza.app/network-tools/redirect-checker) lists every hop with its status code, which is quicker than reading `curl -IL` output four times.

GitHub Pages has no server config, so you can't write redirect rules for old URLs. If you're moving pages from another host, the `jekyll-redirect-from` plugin creates small HTML pages that send visitors to the new address.

## 5. Cut the image weight

Photos straight from a phone are often several megabytes each. GitHub caps a published Pages site at 1 GB, but load time on mobile becomes a problem long before that.

Resize each image to the largest width it is actually shown at, then save it as WebP or AVIF. I use this [image compressor](https://utilza.app/image-tools/image-compressor) because it runs in the browser, so the files never leave my machine. Compare the result side by side before you replace the original, especially for screenshots with small text.

## 6. Add a favicon and a share preview

Without a favicon, browser tabs show a blank page icon, and every visit requests `/favicon.ico` only to get a 404. Put `favicon.ico` in the root of the site, plus an `apple-touch-icon.png` for iOS home screens. A [favicon generator](https://utilza.app/image-tools/favicon-generator) builds the ICO, SVG, Apple touch icon, and web manifest icons from one image in the browser, and gives you the `<link>` tags to paste into the head.

Shared links need Open Graph tags, or chat apps and social sites show a bare URL. On Jekyll, the `jekyll-seo-tag` plugin writes the title and description tags for you. Set `og:image` to a full URL that starts with `https://`, not a relative path. For a site that isn't on Jekyll, an [Open Graph generator](https://utilza.app/seo-tools/open-graph-generator) writes the tags from a short form.

## 7. Give search engines a sitemap

Jekyll sites can generate a sitemap with a plugin. Both of these are on GitHub Pages' supported list:

```yaml
# _config.yml
url: "https://example.com"
plugins:
  - jekyll-seo-tag
  - jekyll-sitemap
```

Then add a `robots.txt` to the root that tells crawlers where the sitemap is:

```
User-agent: *
Allow: /

Sitemap: https://example.com/sitemap.xml
```

Set `url` to the custom domain. If you leave it as `username.github.io`, the sitemap and canonical tags will point search engines at the old address.

## Final pass

Before announcing the site, I run through this list once more:

- The A, AAAA, and CNAME records return the right values from several resolvers.
- **Enforce HTTPS** is on and the certificate covers both hostnames.
- All four starting URLs end on one address with a 200 status.
- No image on the home page is over a few hundred kilobytes.
- The favicon loads, and a shared link shows a title, description, and image.
- `sitemap.xml` and `robots.txt` load and use the custom domain.

---

*I build [Utilza](https://utilza.app), a set of free browser tools. The checkers linked above are part of it. If one of them gives you a result that looks wrong for a GitHub Pages site, tell me and I'll fix it.*
