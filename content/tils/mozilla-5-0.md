+++
date = '2025-01-01T13:03:48+01:00'
title = 'Mozilla/5.0'
+++
While reading access logs of my web server, I noticed that almost all browsers' user agent output started with `Mozilla/5.0`. I found that curious and decided to dive into the reason why.

It turns out that Gecko, the Firefox rendering engine started it. This was okay because that was Mozilla. Web developers liked Gecko and gave the good website code to it. When KHTML was created for Konqueror, they wanted some of that good code and added `Mozilla/5.0` to its user agent string. Apple forked KHTML to create Webkit (Safari and old Chrome) and carried on the tradition. Google continued the tradition when they forked Webkit to create Blink (new Chrome and Edge).

It was interesting following the [story](https://webaim.org/blog/user-agent-string-history/). Just a series of newer browser engines seeking acceptance.
