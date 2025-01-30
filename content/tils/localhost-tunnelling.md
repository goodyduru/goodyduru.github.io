+++
date = '2025-01-30T19:45:26+01:00'
draft = false
title = 'Localhost Tunneling'
+++
I learned this because of an error. I assumed I couldn't use http://localhost:[port-no] as a redirect URL when testing the app on my machine. I'm ashamed to say that even though I have written two [articles](/posts/practical-oauth-part-one) on [OAuth](/posts/practical-oauth-part-two), I assumed something wrong. Thankfully, I gained new knowledge something for my error.

Before I corrected myself, I was using [localhost.run](https://localhost.run/). Using it requires SSH [reverse tunnels](https://goteleport.com/blog/ssh-tunneling-explained/). Localhost.run is a good service, but it caused a bug in my app. I needed an alternative.

It turns out that you can run SSH localhost tunneling on your computer. All you need to get a publicly accessible web application running on localhost is a server that is reachable from the Internet via SSH and a domain name/subdomain. Thankfully I had both! All I needed to do was set nginx reverse proxy on my server for the domain name to a particular port number (let's say 20001) like this
```txt
server_name subdomain.domain.com;
.....other lines....
location / {
      proxy_pass http://127.0.0.1:20001;                                        # use a port that you like. 30000 is a good one.
    }
......other lines....
```
On my computer, I just needed to run `ssh -N -T -R 20001:localhost:3000 user@my-machine -i /path/to/keys`(my web app is running on port 3000). Like magic, it worked! As a bonus, I added TLS with Let's Encrypt.

Alex Le's [article](https://alexle.net/free-alternative-to-ngrok-with-permanent-subdomain-using-nginx-ssl-and-reverse-ssh-using-your-own-server/) was a great help and I encourage you to read it.
