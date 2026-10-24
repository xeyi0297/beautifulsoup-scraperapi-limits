# Web scraping python beautifulsoup: What the classic stack can and can't do, plus ScraperAPI plans and pricing when you need to go further

BeautifulSoup plus requests is still the right starting point for web scraping in Python. It's free, it parses messy real-world HTML without complaining, and a static-site scraper fits in about fifteen lines. The catch is scope: BeautifulSoup only parses HTML that's already in the response. It doesn't fetch pages, doesn't run JavaScript, and won't get you past a CAPTCHA or an IP ban. This article covers the working setup, the exact points where it breaks, and — verified against ScraperAPI's current pricing page — what it costs to extend the same code with a scraping API when your target sites fight back.

> **Quick take:** Scraping static, public pages? The plain BeautifulSoup stack below costs nothing — stay there. Hitting JS-rendered content or bot walls? ScraperAPI handles rendering, proxies and CAPTchas behind one endpoint, with paid plans from **$49/month** (Hobby, 100,000 credits) or $44.10/month billed annually. There's a **7-day trial with 5,000 credits, no credit card required**. 👉 [Start the free ScraperAPI trial and test it against your target sites](https://www.scraperapi.com/?fp_ref=coupons)

## The classic stack, done right

Install three packages and you have the standard setup:

bash
pip install beautifulsoup4 requests lxml


One thing worth knowing before the code: "Beautiful Soup" today means Beautiful Soup 4 — package `beautifulsoup4`, imported as `bs4`. The old BS3 has been unmaintained since 2012 and only works on Python 2. If a tutorial uses `findAll` or `getText` (camelCase), it's ancient; the modern names are `find_all` and `get_text`.

A scraper that survives contact with real websites:

python
import requests
from bs4 import BeautifulSoup

headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}
response = requests.get("https://books.toscrape.com/", headers=headers, timeout=10)
response.raise_for_status()

soup = BeautifulSoup(response.text, "lxml")
for article in soup.select("article.product_pod"):
    print(article.h3.a["title"])
    print(article.select_one("p.price_color").get_text(strip=True))


Three details in that snippet matter more than they look, and a lot of tutorials skip them.

Always pass the parser explicitly — `BeautifulSoup(html, "lxml")` here. `lxml` is markedly faster than Python's built-in `html.parser`, and naming it means every machine parsing the same broken HTML gets the same tree. Omit the parser name and Beautiful Soup picks whichever one happens to be installed, which only bites you on malformed pages. Exactly the pages where you don't want surprises.

The `timeout` argument is not decoration. Without it, a slow server hangs your script indefinitely. And `raise_for_status()` turns a 403 or a 500 into an immediate exception, so you fail on the HTTP error instead of quietly parsing an error page and wondering why you got zero results.

The parsing API itself is small: `find()` returns the first match or `None`, `find_all()` returns everything, and `select()` takes CSS selectors. Those three cover most scraping work. Two quirks worth learning now instead of debugging later:

- It's `class_` with the underscore, because `class` is a reserved word in Python.
- `find()` returning `None` is behind the single most common Beautiful Soup error: `AttributeError: 'NoneType' object has no attribute 'text'`. Check for `None` before chaining.
- Multi-class matching behaves differently across the two APIs. `find_all(class_="badge")` matches any element that has `badge` among its classes; `select(".badge.sale")` requires both classes. And `class_="badge sale"` matches that exact string, order included — fragile, skip it.

That's the parsing layer complete. What's missing is everything between your code and the HTML.

## Where BeautifulSoup stops working

BeautifulSoup fails in two specific ways, and it's worth diagnosing which one you've hit before spending money.

The first is JavaScript-rendered content. If the data shows up in your browser but not in `response.text`, the page builds its HTML client-side. No parser can extract what was never in the response. The free fix is Selenium or Playwright — but those mean running a headless browser locally, which is heavier and slower, and teams typically end up feeding the rendered page source straight back into BeautifulSoup anyway.

The second is bot protection. Cloudflare, Datadome, PerimeterX and friends block default `python-requests` headers on sight, then escalate to CAPTCHAs and IP bans. Beating these yourself means rotating residential proxies, header rotation, retry logic and CAPTCHA solving — an infrastructure project, not a script.

ScraperAPI exists for exactly this second stage. You send your target URL to their endpoint instead of to the site; they handle proxy rotation, headers, retries, ban detection and CAPTCHA bypassing, and return finished HTML for the same BeautifulSoup code you already wrote:

python
import requests
from urllib.parse import urlencode
from bs4 import BeautifulSoup

payload = {"api_key": "YOUR_API_KEY", "url": "https://example.com", "render": "true"}
response = requests.get("http://api.scraperapi.com/", params=urlencode(payload), timeout=60)
soup = BeautifulSoup(response.text, "lxml")


Their own integration docs flag two things that aren't obvious until you trip over them. First, set your request timeout to at least 60 seconds or leave it unset — the API keeps retrying blocked or CAPTCHA'd requests with different proxy configurations for up to 60 seconds before giving up and returning a `500`. A 10-second client timeout will abandon requests that were about to succeed. Second, the good news on billing: requests that end in `500` aren't charged against your credit quota. One caveat if you use their proxy-port mode: your code must skip SSL certificate verification (`verify=False` in requests), per the docs.

If you've hit either wall, the sensible move is to check coverage and cost before committing to anything. 👉 [Open ScraperAPI's pricing page and compare plans](https://www.scraperapi.com/pricing?fp_ref=coupons) — the trial gives you 5,000 credits to test your actual targets.

## ScraperAPI plans and pricing: the full lineup

ScraperAPI bills by monthly API credits and concurrent threads, not seats. Here's everything currently on the official pricing page, including the free plan that only shows up in their FAQ, not in the plan grid:

| Plan | Monthly price | Annual (10% off) | Credits / month | Concurrent threads | Get it |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [Start free](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | $44.10/mo | 100,000 | 20 | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10/mo | 1,000,000 | 50 | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10/mo | 3,000,000 | 100 | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50/mo | 5,000,000 | 200 | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50/mo | 10,500,000 | 300 | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50/mo | 21,500,000 | 500 | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [Contact sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Alongside the free plan, the pricing FAQ lists the 7-day trial with 5,000 credits and no credit card. Every paid plan includes JS rendering, premium residential and mobile IPs, CAPTCHA and anti-bot bypass, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee — the tiers mostly differ in volume and concurrency, plus a few limits covered below.

Annual billing takes 10% off every paid plan. On Hobby that's $58.80/year saved; on Business, $358.80/year. The discount is uniform, so it changes when the upgrade math tips, never which tier fits.

A bit of pricing history that explains the model: before mid-2022, ScraperAPI charged per raw request and Hobby bought 250,000 "API Calls" at $29. The unit changed to credits, capacity dropped to 100,000 and the price rose to $49 — and the headline prices have held steady since, because the multipliers below absorb the rising cost of anti-bot work.

## What a credit actually buys

The plans table's biggest trap: 100,000 credits does not mean 100,000 pages. ScraperAPI charges per request with multipliers tied to how hard the request is.

| Request type | Cost per request |
| --- | --- |
| Standard page (no rendering, standard proxy) | 1 credit |
| Amazon, Walmart, eBay pages | 5 credits |
| JS rendering or premium proxy | 10 credits |
| Premium proxy + rendering | 25 credits |
| Ultra-premium + rendering | 75 credits |
| Google and Bing search results (incl. subdomains) | 25 credits |
| LinkedIn | 30 credits |
| Sites behind Cloudflare / Datadome / PerimeterX, once bypassed | +10 credits per request |

Run the numbers on the Hobby plan and the range is dramatic: its 100,000 credits is roughly 100,000 standard pages, 20,000 Amazon pages, or about 4,000 Google searches. Two customers on the same plan can do anywhere from ~1,333 to 100,000 actual scrapes a month depending on how much rendering and proxy escalation their targets demand.

You don't have to guess your numbers. The dashboard has a Domain Cost Estimator that returns the exact per-request cost for any URL, and every request accepts a `max_cost` parameter to cap what a single scrape can burn. Check your real targets with both before picking a tier — that's the difference between "100,000 sounds like plenty" and knowing your actual monthly page count.

## The fine print that separates the tiers

Beyond credits and price, four plan-level differences decide which tier you actually need, and none of them make the headline table.

**Geotargeting.** Hobby and Startup can only target US and EU locations. Country-level geotargeting — appearing as a user in Japan, Brazil, wherever — starts at Business ($299/mo). If your project needs it, the two cheapest tiers won't do it at any credit count.

**What happens when credits run out.** On Hobby, Startup and Business, scraping pauses until the next billing cycle unless you upgrade. Only Scaling and above get pay-as-you-go overage: keep scraping past your limit at a fixed per-credit rate, up to a monthly spending cap you set. If your volume spikes unpredictably, a cheaper tier can quietly stop mid-month. There's also opt-in auto-upgrade (Hobby through Professional) that bumps you to the next plan automatically — convenient, but it means a bigger charge with no manual gate.

**Concurrency is the real throughput ceiling.** Exceed your thread limit and requests get a `429` — rejected outright, not queued, and not charged. Twenty concurrent threads (Hobby) handles a few thousand pages a month comfortably; it's tight at scale even when you have credits left.

**Analytics history.** Hobby and Startup keep only the last 30 days of dashboard analytics. Business and up keep everything — relevant if you need usage history for budgeting or debugging over longer periods.

A rough decision rule from these verified facts:

- Learning, or scraping a handful of static sites → free plan (1,000 credits, 5 concurrent) and plain BeautifulSoup. Don't pay yet.
- One ongoing project, mostly standard pages, US/EU targets → **Hobby**, $49/mo or $44.10 annual.
- Country-level geotargeting needed, or you want unlimited analytics history → **Business**, the first tier with both.
- Production volume with unpredictable spikes → **Scaling**, $475/mo, the first tier with pay-as-you-go overage.

👉 [Check the Hobby plan and current annual pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) if you already know your monthly page count fits in 100,000 standard requests.

## FAQ

**Is the free trial actually free?**
Yes. Seven days, 5,000 credits, no credit card required. There's also a permanent free plan with 1,000 credits and 5 concurrent connections — between the two, you get real runway to test before spending anything.

**Do failed requests count against my credits?**
No. Requests that end in a `500` after the API's retry cycle aren't billed. Successful requests are charged at the multiplier rates above.

**Do I have to rewrite my BeautifulSoup code?**
No. ScraperAPI replaces the fetching layer only. Your `find_all` and `select` calls stay exactly as they are; just point `requests.get()` at the ScraperAPI endpoint with your target URL as a parameter. Official code examples, including a concurrent-thread version, are on ScraperAPI's GitHub.

**Should I just use Scrapy or Selenium instead?**
They solve different problems. BeautifulSoup is a parser with the smallest learning curve — right for scripts and projects up to thousands of pages. Scrapy is a full crawling framework with scheduling, concurrency and pipelines, worth its steeper setup when a project means "crawl this whole site on a schedule." Selenium and Playwright handle JavaScript execution, not parsing, and their output usually feeds back into BeautifulSoup anyway. The stacks compose: requests + BeautifulSoup for static sites, Playwright + BeautifulSoup when rendering is needed, and a scraping API when anti-bot infrastructure is the actual obstacle.

## Bottom line before you sign up

Diagnose the obstacle first, because the right tool — and the right budget — depends entirely on it. Static pages you can see in "view source": plain requests + BeautifulSoup, cost zero, forever. Data that only exists after JavaScript runs, or sites that block your IP on the first request: that's when a scraping API earns its $49. Two checks before picking a tier: run the Domain Cost Estimator on your real target URLs to convert headline credits into honest page counts, and confirm whether you need country-level geotargeting — if yes, start at Business rather than the cheaper tiers. 👉 [Open the 7-day free trial (5,000 credits, no card needed)](https://www.scraperapi.com/?fp_ref=coupons) and run your own target list through it before deciding.
