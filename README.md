# ScraperAPI Async Scraper Complete Guide: How Does Async Scraping Work, When Should You Use It Over Sync, How to Submit Batch Jobs and Webhooks — Plus Full Plan Breakdown and Pricing (With Code Examples for Python, Node.js, and More)

You've got a list of 10,000 URLs to scrape. You fire up your script, set a reasonable timeout, and start sending requests one by one. Three hours later, you've got 6,200 successes, 800 timeouts, and a gnawing feeling that you're leaving data on the table. Sound familiar?

That's exactly the problem ScraperAPI's async scraper was built to solve. Not the "I need results in 200ms" problem — the "I need all of these to succeed, and I don't care if it takes a few minutes" problem. They're different animals, and mixing up which tool you reach for is one of those mistakes that quietly costs you real data.

This guide is going to walk through the whole thing: what the ScraperAPI async scraper actually does, when it beats synchronous scraping, how the endpoints work (with real code), and what the different plans cost if you decide to scale up.

---

**What Is ScraperAPI's Async Scraper, and Why Does It Exist?**

ScraperAPI started as a proxy-rotation API — you send a request, it bounces it through a pool of 40 million+ IPs, handles retries and CAPTCHAs, and hands you back clean HTML. That synchronous flow works great when you need data right now and the target site isn't too hostile.

But there's a ceiling. Synchronous requests are constrained by timeout windows. When you're hitting a website with aggressive bot detection — Cloudflare, Datadome, PerimeterX — the service sometimes needs more than a few seconds to find the right bypass approach. And if your script has a hard timeout, that request dies before it can succeed.

The async scraper flips the model entirely. Instead of "send request, wait for response," it's "submit job, move on, collect results later." Behind the scenes, ScraperAPI keeps retrying and applying increasingly sophisticated bypass techniques for up to 24 hours until the page comes back with a clean 200. You don't have to babysit it. You don't have to build retry logic. You just check back when you're ready — or better yet, set up a webhook so the data gets pushed straight to you.

> **In ScraperAPI's own words**: "We recommend using our Async API when success rate matters more than response time."

That one sentence is actually a pretty precise description of when to use it.

---

**Async vs. Sync: The Practical Decision Tree**

There's no universal winner here. Both modes have their place, and picking the wrong one just makes your life harder.

**Use the synchronous API when:**
- You need the scraped data immediately (real-time pricing lookups, live search results)
- Your target sites are standard, unprotected pages
- You're building something interactive where a user is waiting for a response
- You're doing quick one-off checks

**Use the async scraper when:**
- You're processing large batches of URLs — dozens, hundreds, or thousands at a time
- Your success rate on certain sites has been dropping
- You're hitting Cloudflare or other aggressive anti-bot systems
- You're running scheduled data collection jobs (nightly price checks, weekly product catalog updates)
- You want to decouple data collection from data processing in your pipeline

The async endpoint shines brightest on the hard stuff: Amazon product pages, Google SERP results, LinkedIn profiles, any site where your sync requests have been returning errors or timing out at a rate that's actually hurting your data quality.

---

**The Two Async Endpoints You Need to Know**

ScraperAPI's async system runs on two main endpoints.

**Single job submission** — `https://async.scraperapi.com/jobs`

You POST a single URL and get back a status URL plus a job ID. You check that status URL periodically until the status field flips to `finished`, at which point the full response is in the `body` field.

**Batch job submission** — `https://async.scraperapi.com/batchjobs`

Same idea, but you pass an array of URLs instead of a single string. ScraperAPI spins up individual jobs for each URL and returns a list of status URLs. One batch can include up to **50,000 URLs** — which covers the vast majority of real-world use cases without needing to chain requests.

Results are stored for up to 72 hours (with 24 hours guaranteed), so you don't have to retrieve them immediately — but don't leave it too long.

---

**Getting Started: Basic Async Job in Python**

Let's make this concrete. Here's the minimal code to submit a scraping job and retrieve the result:

python
import requests
import time

# Step 1: Submit the job
initial_request = requests.post(
    url='https://async.scraperapi.com/jobs',
    json={
        'apiKey': 'YOUR_API_KEY',
        'url': 'https://quotes.toscrape.com/'
    }
)

job_data = initial_request.json()
status_url = job_data['statusUrl']
print(f"Job submitted. Status URL: {status_url}")

# Step 2: Poll until finished
while True:
    result = requests.get(status_url).json()
    if result['status'] == 'finished':
        html_body = result['response']['body']
        print("Done! HTML length:", len(html_body))
        break
    elif result['status'] == 'failed':
        print("Job failed.")
        break
    else:
        print("Still running... checking again in 5 seconds")
        time.sleep(5)


The response object when status is `finished` contains `response.headers` and `response.body` — the full HTML of the scraped page. From there you parse it however you normally would (BeautifulSoup, lxml, whatever your preference).

---

**Batch Jobs: Submitting Thousands of URLs at Once**

The real power of the async scraper is batch processing. Here's how to submit a list of URLs in Python:

python
import requests

urls = [
    f"https://quotes.toscrape.com/page/{i}/"
    for i in range(1, 11)
]

response = requests.post(
    url='https://async.scraperapi.com/batchjobs',
    json={
        'apiKey': 'YOUR_API_KEY',
        'urls': urls
    }
)

jobs = response.json()
for job in jobs:
    print(f"URL: {job['url']} — Status URL: {job['statusUrl']}")


Each URL gets its own job entry with its own status URL. You can poll them individually, process them in a loop, or — the cleaner option — set up a webhook.

The same endpoint works in Node.js too:

javascript
import axios from 'axios';

const urls = Array.from({ length: 10 }, (_, i) =>
  `https://quotes.toscrape.com/page/${i + 1}/`
);

const { data } = await axios.post('https://async.scraperapi.com/batchjobs', {
  apiKey: 'YOUR_API_KEY',
  urls
});

console.log(data);


And in PHP, for those of you working in that environment:

php
<?php
$payload = json_encode([
    "apiKey" => "YOUR_API_KEY",
    "urls" => [
        "https://wikipedia.org/wiki/Cowboy_boot",
        "https://wikipedia.org/wiki/Web_scraping"
    ]
]);

$ch = curl_init("https://async.scraperapi.com/batchjobs");
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
curl_setopt($ch, CURLOPT_HTTPHEADER, ["Content-Type: application/json"]);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

$response = curl_exec($ch);
curl_close($ch);
print_r($response);


---

**Webhooks: The Cleaner Way to Handle Results**

Polling status URLs is fine for testing. For production, webhooks are what you actually want. Instead of repeatedly checking whether a job is done, you provide a URL endpoint on your own server, and ScraperAPI POSTs the results directly to you the moment each job completes.

The setup adds one field to your request:

python
import requests

initial_request = requests.post(
    url='https://async.scraperapi.com/jobs',
    json={
        'apiKey': 'YOUR_API_KEY',
        'url': 'https://example.com/product/12345',
        'callback': {
            'type': 'webhook',
            'url': 'https://your-server.com/scrape-results'
        }
    }
)

print(initial_request.json())


When the job finishes, ScraperAPI sends the full response payload to `https://your-server.com/scrape-results`. Your server receives the data, processes it, stores it — and you never had to poll anything. This is especially useful when you're running thousands of jobs overnight and want to wake up to a database that's already been populated.

---

**Advanced Parameters: Getting More Out of Each Job**

The async endpoint accepts the same `apiParams` that the synchronous API supports, passed inside the request body:

python
import requests

initial_request = requests.post(
    url='https://async.scraperapi.com/jobs',
    json={
        'apiKey': 'YOUR_API_KEY',
        'apiParams': {
            'country_code': 'us',    # Geotarget your requests
            'render': True,           # Enable JavaScript rendering
            'premium': True,          # Use premium residential proxies
            'autoparse': False,       # Return raw HTML vs. parsed JSON
            'retry_404': False,       # Whether to retry on 404s
            'device_type': 'desktop'  # desktop or mobile user agent
        },
        'url': 'https://www.amazon.com/dp/B08L5NP6NG'
    }
)


A few parameters worth calling out:

- **`render: true`** — Enables headless browser rendering. Essential for sites that load product data or pricing via JavaScript. Costs 10 extra credits per request.
- **`country_code`** — Lets you specify where the request appears to originate from. Useful for geo-sensitive pages (local prices, regional content). Available on Business plans and above for global geotargeting.
- **`premium: true`** — Routes the request through premium residential proxies instead of standard datacenter proxies. Better success rates on stubborn targets. Costs 10 extra credits per request.
- **`max_cost`** — A credit cap you can set per job. If the job would consume more credits than this limit, it returns a 403 error instead of burning your budget. Useful when you're testing expensive targets.

---

**The Credit System: What Actually Gets Billed**

This is the part that surprises people. The per-request cost isn't flat — it depends on where you're scraping and what parameters you've enabled.

A standard, unprotected page costs 1 credit. But:

| Target / Parameter | Credit Cost |
|---|---|
| Standard page | 1 credit |
| Amazon | 5 credits |
| Google / Bing | 25 credits |
| LinkedIn | 30 credits |
| Cloudflare-protected sites | +10 credits (on top of base) |
| `premium=true` | +10 credits |
| `render=true` | +10 credits |
| `screenshot=true` | +10 credits |
| `ultra_premium=true` | +30 credits |

The good news: **you're only charged for successful requests**. If ScraperAPI can't retrieve the page, you don't pay. That's a meaningful guarantee when you're hitting difficult targets.

The catch: those multipliers compound fast. An Amazon product page with JavaScript rendering enabled costs 5 (Amazon) + 10 (render) = 15 credits per request. On the Hobby plan with 100,000 credits, that's roughly 6,600 successful scrapes — not 100,000. Worth running your own numbers before committing to a plan.

---

**Full ScraperAPI Plan Comparison**

All plans include JavaScript rendering, rotating proxy pools, premium proxies, CAPTCHA bypass, custom headers, automatic retries, unlimited bandwidth, and a 99.9% uptime SLA. The differences are volume, concurrency, geotargeting scope, and overflow billing options.

| Plan | Monthly Price | Annual Price | Credits/Month | Concurrent Threads | Geotargeting | Get Started |
|---|---|---|---|---|---|---|
| **Free Trial** | $0 (7 days) | — | 5,000 (one-time) | 5 | Limited | [ Start Free Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Hobby** | $49/mo | $44.10/mo | 100,000 | 20 | US & EU only | [ Get Hobby Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU only | [ Get Startup Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Business** | $299/mo | $269.10/mo | 3,000,000 | 100 | Global | [ Get Business Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Scaling** ⭐ Most Popular | $475/mo | $427.50/mo | 5,000,000 | 200 | Global | [ Get Scaling Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Professional** (Growth) | $975/mo | $877.50/mo | 10,500,000 | 300 | Global | [ Get Professional Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Advanced** (Growth) | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global | [ Get Advanced Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22,000,000+ | 500+ | Global | [ Contact Sales](https://www.scraperapi.com/?fp_ref=coupons) |

**A few things the table doesn't show:**

- **Pay-as-you-go overflow** is only available on Scaling and above. On Hobby, Startup, and Business, hitting your credit limit mid-month means upgrading or stopping. On higher tiers, you keep scraping and pay a fixed per-credit rate.
- **Geotargeting** on Hobby and Startup is capped at US and EU proxies. If you need country-level targeting in Asia, South America, or elsewhere, Business is the minimum.
- **Analytics history** is limited to 30 days on Hobby and Startup. Business and above get unlimited history in the dashboard.
- **The two Growth plans** (Professional and Advanced) were added in May 2026 as a bridge between the Scaling plan and Enterprise — designed for teams that need high volume without going through a custom sales conversation.

The new Professional plan also came with a **bonus 250K credits** as a limited-time offer, and the Advanced plan came with a **bonus 500K credits**, so if you're evaluating those tiers, worth checking whether that offer is still active when you sign up.

---

**Which Plan Makes Sense for Async Scraping?**

The async scraper itself is available on all plans — even the free trial. But how much you can actually accomplish with it depends on two things: your credit balance and your concurrent thread count.

For **testing the async endpoint** and running small batch jobs (a few hundred URLs at a time), the free trial's 5,000 credits is genuinely enough to prove out the workflow. You don't need a card on file.

For **regular async data collection** — say, pulling 10,000 product pages per week — the Hobby plan's 100,000 credits runs thin fast once you factor in credit multipliers. Startup at 1 million credits with 50 concurrent threads is where async batch processing starts to feel roomy.

For **large-scale production async pipelines** — nightly scrapes of millions of URLs, continuous competitor monitoring, AI training data collection — you're looking at Scaling and above. The pay-as-you-go overflow feature alone is worth it, because you stop having to worry about credit budgeting mid-month.

[👉 Try the async scraper free for 7 days — no credit card needed](https://www.scraperapi.com/?fp_ref=coupons)

---

**What Users Actually Say**

ScraperAPI sits around 4.5/5 on Trustpilot and 4.4/5 on G2. The recurring theme in positive reviews is "just works out of the box" and "handles millions of requests asynchronously without me having to manage infrastructure." One reviewer specifically called out the async service as the reason they stayed — synchronous alternatives kept timing out on their target sites, while the async endpoint just kept retrying until it succeeded.

The honest criticism that shows up consistently: credit multipliers aren't intuitive until you've been burned by them once. A few reviewers flagged the experience of watching their credit balance disappear faster than expected on SERP or Amazon scraping. Not a complaint about the service working poorly — just that the pricing model requires a bit of mental math before you start.

The 7-day free trial essentially exists to solve this: you run your actual targets, watch your credit consumption in the dashboard, and you'll know exactly what your real per-month cost will be before you pay anything.

---

**Common Questions**

**Can I use the async endpoint on the free plan?**
Yes. The trial gives you 5,000 credits and access to the async endpoint at `https://async.scraperapi.com/jobs` and `https://async.scraperapi.com/batchjobs`. You can test both single-job and batch workflows during the 7-day window.

**How long does an async job run before timing out?**
ScraperAPI keeps retrying for up to 24 hours. Results are then stored for up to 72 hours. After that, they're deleted and you'd need to resubmit.

**Is there a limit on batch size?**
50,000 URLs per batch job. If your workload exceeds that, split into multiple batches.

**Does the async scraper cost more credits than sync?**
No — the credit cost per request is identical. The difference is reliability and retry behavior, not price.

**Can I cancel a job after submitting?**
Yes — ScraperAPI's API includes job cancellation via a DELETE request to the status URL.

**Is there a discount?**
Annual billing gives you 10% off across all plans automatically — no coupon code needed, just select annual billing at checkout. [👉 Check current plans and offers here](https://www.scraperapi.com/?fp_ref=coupons).

---

**Bottom Line**

The ScraperAPI async scraper is the right choice when you've got a lot of URLs to process and you care more about all of them succeeding than about any individual one returning instantly. It removes timeout anxiety, handles retries automatically, and scales to 50,000 URLs per batch without you writing a single line of retry logic.

The synchronous API is still the right tool when you need data immediately. But for scheduled data collection, large-scale pipelines, or consistently difficult targets like Amazon or Google — the async endpoint is where ScraperAPI actually earns its keep.

Start with the free trial, point it at your real targets, and watch the dashboard. You'll have a clear picture of what plan fits your use case before you spend anything.

[👉 Create a free ScraperAPI account and test the async scraper — 5,000 credits, no card required](https://www.scraperapi.com/?fp_ref=coupons)
