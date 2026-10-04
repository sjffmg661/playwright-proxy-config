# playwright proxy: split credentials, per-context rotating IPs, and the config mistakes that quietly send traffic direct

The most common Playwright proxy failure doesn't throw an exception. The browser starts, the script runs to completion, the data comes back — and every request went out over your own IP address. Nothing in the visible behavior suggests the proxy was ignored, which is why the same three bugs keep showing up in forum threads.

This walks through how Playwright actually handles proxies, the three configs that fail silently, and how to point a browser automation script at rotating residential IPs without babysitting a local relay. The proxy side of the examples uses 9Proxy, since its authentication model lines up unusually well with what Playwright's `proxy` option accepts.

## How Playwright takes a proxy

Playwright's proxy support lives in two places, and the difference matters more than it looks:

- **Globally, at browser launch** — every context in that browser uses the same exit IP.
- **Per browser context** — each context gets its own proxy, and therefore its own exit IP.

python
# Global: one IP for the whole browser
browser = chromium.launch(proxy={
    "server": "http://gateway.example.com:17521",
    "username": "user",
    "password": "pass",
})


python
# Per context: a different exit IP per context
browser = chromium.launch()
ctx = browser.new_context(proxy={
    "server": "http://gateway.example.com:17521",
    "username": "user",
    "password": "pass",
})


Both are documented and both work. The per-context form is what people actually want for scraping and multi-account work, because it produces a different exit IP without a second browser process.

## Three ways the config fails without an error

### 1. Credentials embedded in the proxy URL

This is the single most common mistake, and it comes from muscle memory built on `requests` and Scrapy, where `scheme://user:pass@host:port` is normal:

python
# Broken: the credentials are dropped
browser.new_context(proxy={"server": "http://user:pass@gw.example.com:17521"})


Playwright does not parse credentials out of the server string. It wants them as separate fields. If your gateway requires authentication, the request either fails with `407 Proxy Authentication Required` or quietly never leaves, depending on how the upstream reacts to an unauthenticated handshake.

python
# Correct
browser.new_context(proxy={
    "server": "http://gw.example.com:17521",
    "username": "user",
    "password": "pass",
})


The annoying part is that a script that mints that URL from a provider string — `f"http://{user}:{pw}@{host}:{port}"` — looks correct in every log line and passes code review.

### 2. Authenticated SOCKS5

Playwright's `server` field accepts `http://`, `https://` and `socks5://`. Its `username` and `password` fields are documented against HTTP proxies. Authenticated SOCKS5 is not supported, and the upstream feature request for it has been open since 2021. Passing credentials alongside a `socks5://` server gets you a browser that starts fine and traffic that doesn't authenticate.

If your provider gives you a SOCKS5 endpoint with a username and password, you have two workable options: run a local relay that authenticates upstream so Playwright can point at an unauthenticated `socks5://127.0.0.1:<port>`, or simply use the provider's HTTP/HTTPS endpoint instead, where credentials are supported.

### 3. Thinking the proxy is per page

The proxy is set on the context, not the page. Two pages created inside the same context share the same exit IP, the same cookie jar and the same storage. If you're expecting `page`-level rotation, you'll get one IP for a whole batch of requests and wonder why the rate limiter kicked in at request 200.

Rotation means a new context:

python
for i in range(10):
    ctx = browser.new_context(proxy=next_proxy())
    page = ctx.new_page()
    page.goto("https://example.com")
    ctx.close()          # drops cookies, cache and connections


### Verify with the exit address, not the page

Never debug this by checking whether a page loaded. Check where the request came from:

python
page.goto("https://httpbin.org/ip")
print(page.inner_text("body"))


A silent fall-through to the direct connection looks exactly like success until you read that IP back.

## A new IP is not a new identity

Per-context proxies isolate a specific list of things. It's worth being precise about which, because the list is shorter than most guides imply.

| Property | Is it separate per context? |
| --- | --- |
| Cookies, localStorage, IndexedDB | Yes |
| Cache and service workers | Yes |
| Permissions and grants | Yes |
| Exit IP (when proxy is set per context) | Yes |
| Canvas hash | No — shared |
| WebGL renderer, font set, audio profile | No — shared |
| Screen size, `hardwareConcurrency`, `deviceMemory` | No — shared |
| TLS handshake shape | No — shared |
| Timezone and locale | Not automatically |

Five contexts on five residential IPs are still one machine appearing from five countries. That's fine for a lot of jobs — price monitoring, SERP checks, geo-content verification. It's a real problem when the target correlates sessions by fingerprint instead of by IP.

There's a second, sneakier issue: if your tooling resolves timezone and locale at browser launch, it resolves them once, for the launch-time exit IP. Context one leaves from Germany with a German timezone. Context two leaves from Japan carrying the same German timezone. If you rotate per context, set `timezone_id` and `locale` per context, by hand, to match the proxy's country.

## Residential or datacenter for a Playwright job

A correctly authenticated datacenter proxy still gets blocked a lot of the time, because anti-bot systems score the ASN and the TLS/JA3 fingerprint before your page even renders. Residential exits come from consumer networks, so the IP reputation check passes on the first try and your retry logic stops being the main consumer of CPU.

Datacenter IPs are cheaper and perfectly fine for targets that don't care — internal test suites, public APIs, staging environments. The moment you're hitting something with Cloudflare, Akamai or a custom bot layer in front of it, the ASN is doing the blocking, not your headers.

## How 9Proxy's two models map to two Playwright workloads

9Proxy sells residential access with 20M+ IPs across 90+ countries, targeting down to country, state, city, ZIP code and ISP, over HTTP/HTTPS and SOCKS5. It splits into two billing models, and they correspond to two different automation patterns.

**Residential by GB** — pay per gigabyte, generate unlimited endpoints. Authentication is username/password or IP whitelisting, and everything runs from the web dashboard with no app installed. Session mode is your choice: rotating (fresh IP each request or session) or sticky (same IP for a configured window). GB balances are valid for 180 days, unlimited on Enterprise.

**Residential by IPs** — pay per IP, unlimited bandwidth per IP. Unused IPs don't expire, sessions last from a few hours up to about 24 hours of natural residential uptime, and there's an Auto Rotation Proxy for rotating on selected ports at custom intervals. This model requires the 9Proxy desktop app (Windows, macOS, Linux), which does local port forwarding and optional proxy authentication.

For Playwright specifically, that distinction decides your architecture. If your scripts run inside Docker, GitHub Actions or any headless CI machine, you want the GB-based model: credentials-only auth, no desktop app, no `systemctl` step in your image. The IP-based model is a better fit for a workstation or a dedicated worker box where you want unlimited traffic per IP and long-lived sessions.

Two operational details are worth knowing before you buy. Auto-Refresh detects and replaces offline IPs within about 60 seconds, which matters when your script pulls an IP from a pool and holds a context open. And the "Today List" lets you reuse IPs that were active in the previous 24 hours at no additional cost, which cuts spend for jobs that re-hit the same regions repeatedly.

## Every current 9Proxy package

Prices below reflect the IP-based and bundle adjustment that took effect on June 1; GB-based pricing was not changed in that update. If you've seen older reviews quoting $20 for 100 IPs or a $25 Starter bundle, those numbers predate the adjustment.

| Model | Package | What you get | Effective rate | Price | Validity | Buy |
| --- | --- | --- | --- | --- | --- | --- |
| Residential by IPs | 100 IPs | 100 residential IPs, unlimited bandwidth | $0.24 / IP | $24 | IPs never expire | [ 100 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 500 IPs | 500 IPs, unlimited bandwidth | $0.144 / IP | $72 | IPs never expire | [ 500 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 1,000 + 500 IPs | 1,500 IPs (500 bonus), unlimited bandwidth | $0.084 / IP | $126 | IPs never expire | [ 1,500 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 2,500 IPs | Unlimited bandwidth | $0.084 / IP | $210 | IPs never expire | [ 2,500 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 5,000 IPs | Unlimited bandwidth | $0.072 / IP | $360 | IPs never expire | [ 5,000 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 15,000 IPs | Unlimited bandwidth | $0.048 / IP | $720 | IPs never expire | [ 15,000 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 25,000 IPs | Unlimited bandwidth | $0.035 / IP | $863 | IPs never expire | [ 25,000 IP package](https://bit.ly/9-Proxy) |
| Residential by IPs | 50,000 IPs | Unlimited bandwidth | $0.029 / IP | $1,438 | IPs never expire | [ 50,000 IP package](https://bit.ly/9-Proxy) |
| Business IP | 100,000 IPs | High-volume residential pool | $0.023 / IP | $2,300 | IPs never expire | [ 100,000 IP package](https://bit.ly/9-Proxy) |
| Business IP | 200,000 IPs | High-volume residential pool | $0.021 / IP | $4,140 | IPs never expire | [ 200,000 IP package](https://bit.ly/9-Proxy) |
| Business IP | 500,000 IPs | High-volume residential pool | $0.018 / IP | $8,625 | IPs never expire | [ 500,000 IP package](https://bit.ly/9-Proxy) |
| Residential by GB | 5 GB | Unlimited endpoints | $3.00 / GB | $15 | 180 days | [ 5 GB pack](https://bit.ly/9-Proxy) |
| Residential by GB | 50 + 5 GB | 55 GB with bonus traffic | $2.10 / GB | $105 | 180 days | [ 55 GB pack](https://bit.ly/9-Proxy) |
| Residential by GB | 100 GB | Unlimited endpoints | $1.50 / GB | $150 | 180 days | [ 100 GB pack](https://bit.ly/9-Proxy) |
| Residential by GB | 200 GB | Unlimited endpoints | $1.00 / GB | $200 | 180 days | [ 200 GB pack](https://bit.ly/9-Proxy) |
| Residential by GB | 1,000 GB | Unlimited endpoints | $0.80 / GB | $800 | 180 days | [ 1,000 GB pack](https://bit.ly/9-Proxy) |
| Residential by GB | 2,000 GB | Unlimited endpoints | $0.75 / GB | $1,500 | 180 days | [ 2,000 GB pack](https://bit.ly/9-Proxy) |
| Enterprise GB | 3,000 GB | Team mode (1 owner + 5 members), per-member traffic controls, activity logs | $0.72 / GB | $2,160 | Unlimited | [ 3,000 GB Enterprise pack](https://bit.ly/9-Proxy) |
| Enterprise GB | 6,000 GB | Same Enterprise features | $0.70 / GB | $4,200 | Unlimited | [ 6,000 GB Enterprise pack](https://bit.ly/9-Proxy) |
| Enterprise GB | 10,000 GB | Same Enterprise features | $0.68 / GB | $6,800 | Unlimited | [ 10,000 GB Enterprise pack](https://bit.ly/9-Proxy) |
| Bundle | Starter | 100 IPs + 5 GB | — | $30 | Traffic valid 180 days | [ Starter bundle](https://bit.ly/9-Proxy) |
| Bundle | Popular | 1,500 IPs + 50 GB | — | $180 | Traffic valid 180 days | [ Popular bundle](https://bit.ly/9-Proxy) |
| Bundle | Pro | 5,000 IPs + 500 GB | — | $720 | Traffic valid 180 days | [ Pro bundle](https://bit.ly/9-Proxy) |

## Wiring 9Proxy into Playwright

Use the HTTP endpoint, not SOCKS5. That single decision removes the authenticated-SOCKS5 problem entirely, because Playwright's `username`/`password` fields are the supported path for HTTP proxies.

For the GB-based model, credentials are a sub-account username plus its password, and the username itself carries the targeting and session settings:


<subaccount>-country-<country_code>-st-<state_code>-city-<city_code>-isp-<isp_code>-ssid-<session_id>-sst-<session_time>


That structure is the reason this provider pairs well with Playwright. Each context can get a unique `ssid`, which produces a distinct sticky exit IP — exactly the "one IP per context" model, except nobody has to manage an IP list. Country, city and ISP go in the same string, so geo per context is a formatting change, not a second account.

python
import uuid
from playwright.sync_api import sync_playwright

GATEWAY = "http://<host>:<port>"      # copy from the 9Proxy dashboard
SUBUSER = "<your sub-account username>"
PASSWORD = "<your sub-account password>"

def proxy_for(country="US"):
    session = uuid.uuid4().hex[:10]     # unique ssid -> unique sticky exit IP
    return {
        "server": GATEWAY,
        "username": f"{SUBUSER}-country-{country}-ssid-{session}",
        "password": PASSWORD,
    }

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)

    for country in ("US", "DE", "JP"):
        ctx = browser.new_context(
            proxy=proxy_for(country),
            locale={"US": "en-US", "DE": "de-DE", "JP": "ja-JP"}[country],
            timezone_id={"US": "America/New_York",
                         "DE": "Europe/Berlin",
                         "JP": "Asia/Tokyo"}[country],
        )
        page = ctx.new_page()
        page.goto("https://httpbin.org/ip")
        print(country, page.inner_text("body"))
        ctx.close()

    browser.close()


Note what the loop does beyond proxying: `locale` and `timezone_id` are set per context to match the exit country, because nothing does it for you. The `sst` field controls sticky session length — grab the exact field list and units from the username builder in the dashboard rather than copying a format string from a blog, including this one.

If you'd rather skip password handling on a fixed CI runner, the GB-based model also supports IP whitelisting, which connects without credentials.

## The cost math for a Playwright job

Playwright renders JavaScript, which means page weight. A JS-heavy page is routinely 2–5 MB once scripts, fonts and images are counted, and that's before you take screenshots.

Say you process 100,000 pages a month at roughly 3 MB each — call it 300 GB. On the GB model that lands between the 200 GB ($200) and 1,000 GB ($800) tiers, so somewhere in the $300–$800 range depending on which tier you buy. On the per-IP model, 100 IPs with unlimited bandwidth is $24 flat, and the page weight stops being a line item. Whether that's cheaper depends entirely on how many requests you push through each IP before you burn the session — and on whether the target starts flagging that IP after a few thousand requests.

For Playwright work, the practical split looks like this: high page count with fast rotation favors GB, long-running authenticated sessions and screenshot-heavy jobs favor per-IP. The Starter bundle at $30 exists basically for finding out which one you are.

## Short answers to the questions that come up

**Can Playwright rotate the IP per request?** No. Rotation happens at context level. Create a new context per rotation, or accept one IP for everything inside it.

**Does Playwright support SOCKS5 proxies at all?** Yes — unauthenticated ones. Credentials on a `socks5://` server are not supported; use the HTTP endpoint when you need auth.

**Why did my requests still come from my real IP?** Almost always credentials embedded in the `server` string, or a `socks5://` server with credentials attached. Verify by reading the exit IP back from `httpbin.org/ip`.

**Do unused proxies expire?** IP-based packages don't expire on unused IPs. GB balances run 180 days, unlimited on Enterprise.

**What happens if an IP is dead on arrival?** Auto-Refresh replaces offline IPs within about 60 seconds, and per a third-party review the platform credits IPs that fail to connect within the first minute. Confirm the current policy on the sign-up page before you build retry logic around it.

## Bottom line

The proxy configuration and the proxy provider are two separate problems, and fixing them in that order saves a lot of time. Get credentials out of the URL, use HTTP when you need auth, rotate at the context level, and set timezone and locale per context. Once the script is actually routing through the proxy, plan choice is a simple question: unlimited bandwidth per IP if your contexts are long-lived, GB if you're cycling fast and light.

You can check current pricing and grab the 100 IP entry package here — [👉 see 9Proxy packages and sign up](https://bit.ly/9-Proxy) — or go straight for the model your workload points at.
