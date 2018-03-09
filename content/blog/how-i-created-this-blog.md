+++
author = "Scott Guymer"
categories = [""]
date = "2018-03-09T10:19:18+00:00"
description = "How I set up this blog and what tech I use to manage it."
draft = true
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "How I created this blog"
type = "post"

+++
## HUGO

I decided that I wanted something a little bit different to Jekyll that would possibly give me the opportunity to learn something new and possibly a new language (Go) in the process. After scouring the web for static site generators I stumbled on HuGo and thought it was great. It is Jekyll inspired but has some great features.

## GitHub Pages

I wanted to host this somewhere free and simple. GH Pages seemed like a great way of doing this! I also wanted custom domain name and probably SSL too (more about that later!)

## Forestry.io

I accidentally stumbled across forestry.io and really had to take a look. Its a CMS for static sites. Where each of your edits is a commit to your repository. And its great! I'm using it right now.

## Cloudflare

I wanted SSL... and I wanted custom domain names... GH doesnt support this, and probably never will. I also wanted something that was as free or cheap as i could get it. Cloudflare fits the bill perfectly.

One thing that nearly put me off was the fact that cloudflare wants to take over your entire DNS service, i had to admin I was a bit hesitant. All of the CDNs that I had used professionally had been CNAME based. Bu cloudflare imported my existing DNS entries for my mail and other things and all I had to do was change my name servers over with my domain provider and it fired into life! The SSL took a little longer to provision but was all automatic which was great!