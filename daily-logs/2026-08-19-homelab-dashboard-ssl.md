## Docker Setup

06:30 PM starting Docker — I spun up Homer inside Docker on my VPS so I could have a tiny, static bookmarks dashboard: quick, local, and accessible from my phone.

The docker run I used:

```
docker run -d --name homer \
  -p 8080:8080 \
  -v /home/user/homer/config.yml:/www/assets/config.yml \
  b4bz/homer:latest
```

Notes and struggles:
- I originally tried to bind the container to port 80, but Docker couldn't bind because Nginx from a previous project already had port 80. I had to stop Nginx (`sudo systemctl stop nginx`) so Docker could start, then switch to running Homer on 8080 and proxy through Nginx instead.
- I mounted the wrong local folder path for Homer's `config.yml` at first. Docker created an empty directory on the host instead of using my config. Fixed it by using an absolute path (`/home/user/homer/config.yml`) instead of a relative one.

## The Reverse Proxy Trap

I thought Nginx would be out of the way, but it was still serving something on 0.0.0.0:80 from a previous static site. Docker complained about binding to port 80, which led to the stop-gap fix:

```
sudo systemctl stop nginx
```

Then I changed approach: run Homer on host port 8080 and have Nginx proxy_pass to it.

Example Nginx proxy config (sites-available/myhomelab.conf):

```
server {
    listen 80;
    server_name myhomelab.duckdns.org;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

I then re-enabled Nginx with the proxy config so Nginx acted as the reverse proxy instead of serving files directly.
