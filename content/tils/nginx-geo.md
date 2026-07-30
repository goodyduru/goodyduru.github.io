+++
date = '2026-07-30T12:35:06+01:00'
title = 'Nginx Geo'
+++
I am a big fan of making my applications as light as possible and that means taking advantage of their service dependencies. For me, that looks like pushing as much data handling as possible to the database (usually Postgres) and letting the reverse proxy (usually Nginx) handle whatever network tasks it can. I read this [article](https://digitalseams.com/blog/most-googlebots-are-fake) on fake Googlebots and I was curious about how to use Nginx to block them. In the old days, I'd search Google, but now I use Gemini and its ilk and here's Gemini's answer:

```sh
# 1. Map Google's official IP ranges (populated via script/cron)
geo $is_real_googlebot {
    default 0;
    include /etc/nginx/googlebot-ips.conf; # Contains ranges like 66.249.66.0/24 1;
}

# 2. Check if the User-Agent claims to be Googlebot
map $http_user_agent $claims_googlebot {
    default 0;
    "~*Googlebot" 1;
}

# 3. Combine checks: Claims Googlebot AND NOT in Google IP list
map "$claims_googlebot:$is_real_googlebot" $fake_googlebot {
    "1:0" 1;
    default 0;
}

server {
    ...
    if ($fake_googlebot) {
        return 403; # Or 444 to close connection without response
    }
}
```
What caught my attention was the keyword __geo__. This is an Nginx [directive](https://nginx.org/en/docs/http/ngx_http_geo_module.html) that lets you define variables based on the client IP addresses. These variables let you take actions such as showing certain pages to certain clients, adding headers, or blocking traffic from specific networks in subsequent config blocks. The possibilities are endless.

Anyway, I thought I'd share this.

