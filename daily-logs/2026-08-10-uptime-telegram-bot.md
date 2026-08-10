# Uptime Telegram Bot — 2026-08-10

## The Code

I wrote a little Python script on my VPS to ping my personal websites every 5 minutes. If a site looks down, the script POSTs a message to my Telegram account via the Bot API.

Here's the core ping logic I used originally:

```python
import requests
import time

BOT_TOKEN = "123456:ABC-DEF"  # yes, this was hardcoded initially
CHAT_ID = "@myusername"

def send_alert(site):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    requests.post(url, data={"chat_id": CHAT_ID, "text": f"DOWN: {site}"})

def ping_site(site):
    try:
        r = requests.get(site, timeout=10)
        if r.status_code != 200:
            send_alert(site)
    except Exception as e:
        send_alert(site)

if __name__ == "__main__":
    sites = ["http://example.com", "http://my-redirecting-site.com"]
    while True:
        for s in sites:
            ping_site(s)
        time.sleep(300)  # 5 minutes
```

## The Mistakes

10:00 AM — I started the script for the first time.

10:01 AM — It crashed instantly with:

    ModuleNotFoundError: No module named 'requests'

I had installed Python but forgot to pip install requests. Once I installed requests and added it to a requirements.txt, the script ran again.

10:30 AM — I realized I had hardcoded my Telegram Bot token directly into the script (see BOT_TOKEN above). That's a terrible security practice — if I ever push the repo, I just leaked my token. I moved the token to a .env file and used python-dotenv to load it (details in next commit).

The script also kept reporting false "down" states because one of my sites redirected from http:// to https://. I hadn't allowed redirects explicitly and the request behavior intermittently flagged the site as down. I fixed that by adding allow_redirects=True and a proper timeout so redirects don't trigger false positives.

11:00 AM — Relief when the first Telegram message popped up on my phone. I honestly breathed a sigh of relief.

## The Final Cron Job

I run this script every 5 minutes from cron on my VPS. The sequence for this entry:

1) Wrote the initial script but it crashed because I forgot to install requests.
2) Realized token was hardcoded and later moved it to a .env file.

(See next commit for the version that loads the token from .env)
