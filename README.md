# ScraperAPI Concurrent Threads: How Many Threads Each Plan Gets, How to Max Them Out, and What 100 vs 500 Threads Actually Buys You

Every ScraperAPI plan puts a hard cap on how many requests you can have in flight at once, and that cap is usually what makes a scraper feel slow, not your credit balance. The numbers: Free 5 threads, Hobby 20, Startup 50, Business 100, Scaling 200, Professional 300, Advanced 500, Enterprise custom. Send more parallel requests than your plan allows and the API answers with a 429, and you are not charged for the rejected request.

Below are the thread counts and prices for all eight tiers, the one public benchmark that compares 100 and 500 threads on the same job, the code settings that decide whether you actually reach your limit, and the credit multipliers that make a fast plan expensive if you scrape heavy pages.

> **Quick pick:** most people validating a target should start on Hobby (20 threads, 100,000 credits, $49/month) or Startup (50 threads, 1,000,000 credits, $149/month). Business ($299/month, 100 threads) is the first tier with country-level geotargeting, so it is the floor if your data has to come from outside the US and EU. New accounts get a 7-day trial with 5,000 credits.
> 👉 [Start the free trial and check current plan pricing](https://www.scraperapi.com/?fp_ref=coupons)

## All eight plans, threads and credits side by side

| Plan | Monthly price | Credits/month | Concurrent threads | Geotargeting | Sign up |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | 1,000 | 5 | None | [Create a free account](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | 100,000 | 20 | US & EU only | [Get Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | $149 | 1,000,000 | 50 | US & EU only | [Get Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | $299 | 3,000,000 | 100 | 50+ countries | [Get Business](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | $475 | 5,000,000 | 200 | 50+ countries | [Get Scaling](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | $975 | 10,500,000 | 300 | 50+ countries | [Get Professional](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | $1,975 | 21,500,000 | 500 | 50+ countries | [Get Advanced](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Custom | 22,000,000+ | 500+ (custom) | 50+ countries | [Talk to sales](https://www.scraperapi.com/?fp_ref=coupons) |

Annual billing takes about 10% off and is applied at checkout: Hobby drops to $44.10/month, Startup $134.10, Business $269.10, Scaling $427.50. Two things the table doesn't show. Country-level targeting (`country_code=de`, for example) starts at Business and doesn't cost extra credits. Pay-as-you-go overage, where you keep scraping past your credit limit at a fixed per-credit rate up to a cap you set, only exists from Scaling upward; below that, running out of credits pauses the account unless you turn on auto-upgrade.

Free is a test account, not a working plan: 1,000 credits a month and 5 concurrent connections. The 5,000-credit allowance applies to your first 7 days.

## What a concurrent thread actually limits

A thread is a slot for one in-flight request to `api.scraperapi.com`. Five threads means five requests running in parallel; the sixth has to wait for a slot to free up. ScraperAPI handles proxy rotation, header rotation, ban detection and CAPTCHA solving on its side, so the thread count is the one speed variable you control from your own code.

The API does not accept batch requests. You cannot post a list of 10,000 URLs and let ScraperAPI figure out the parallelism; each URL is its own HTTP call, and the thread ceiling governs how many of those calls are live at any moment.

What happens when you exceed it? The API returns a 429 asking you to bring concurrency back inside your plan's limit, and no credits are deducted for the rejected request. Some third-party write-ups describe the overflow as queuing on ScraperAPI's side; the practical result is the same either way, because you don't get throughput above your plan's ceiling. The difference matters locally: if your client fires 200 requests against a 50-thread plan and doesn't handle 429s, you end up with a pile of stalled connections on your machine and a job that finishes no faster.

Setting `max_workers=200` on a 50-thread plan is the most common version of this mistake. The extra workers don't buy speed, they just sit in your process waiting.

## The one public benchmark: 100 threads vs 500 threads

ScraperAPI's own write-up on concurrent threads ran the same scraper twice against a sample of 1,000+ URLs collected from CNN's tech section, extracting the H1, title and meta description of each page. The only variable changed between runs was the thread count.

| Threads | Time to scrape ~1,000 pages |
| --- | --- |
| 100 | 100.68 seconds |
| 500 | 23.56 seconds |

Roughly four times faster at 500 threads. The write-up doesn't state which plan the test account was on, and 500 threads maps to the Advanced tier or a custom Enterprise limit, so don't read this as "Business can run 500 threads." What it does show is that the thread setting moves total runtime by a multiple, not a percentage, as long as your targets respond consistently.

You can translate that into expected throughput per tier. The table below assumes an 8-second average response, which sits in the middle of ScraperAPI's typical 4 to 12 second range. These are estimates, not measurements:

| Concurrent threads | Plan | Estimated pages/minute at 8s per request |
| --- | --- | --- |
| 5 | Free | ~37 |
| 20 | Hobby | ~150 |
| 50 | Startup | ~375 |
| 100 | Business | ~750 |
| 200 | Scaling | ~1,500 |
| 300 | Professional | ~2,250 |
| 500 | Advanced / Enterprise | ~3,750 |

If your targets are slower than 8 seconds because they sit behind bot protection, cut those numbers accordingly. If they are simple HTML pages, they can come back faster.

## How to keep every thread busy

Writing the loop correctly is the difference between using 100 threads and using 12. The pattern ScraperAPI documents uses `ThreadPoolExecutor` with `max_workers` set to your plan's thread count exactly:

python
import requests
from concurrent.futures import ThreadPoolExecutor

API_KEY = "YOUR_KEY"
NUM_THREADS = 100   # match your plan, do not exceed it
NUM_RETRIES = 3

def scrape_url(url):
    params = {"api_key": API_KEY, "url": url}
    for _ in range(NUM_RETRIES):
        try:
            r = requests.get("http://api.scraperapi.com/", params=params)
            if r.status_code in (200, 404):
                return {"url": url, "status": r.status_code, "html": r.text}
        except requests.exceptions.ConnectionError:
            continue
    return {"url": url, "status": "failed"}

with ThreadPoolExecutor(max_workers=NUM_THREADS) as ex:
    results = list(ex.map(scrape_url, urls))


A few settings around that loop decide whether the threads stay saturated:

**Don't set a client timeout, or set it to at least 60 seconds.** On a ban, a CAPTCHA or a slow response, ScraperAPI retries internally with a different proxy and header set for up to 60 seconds before returning a 500. A 10-second client timeout kills requests that were about to succeed, and those aborted attempts still consume your threads.

**Disable SSL verification in proxy mode.** Requests through the proxy port need `verify=False`, or they fail on certificate checks.

**Retry 500s, but back off.** ScraperAPI reports that over 97% of requests succeed on the first attempt for most sites, and failed requests return a 500 without being charged. Retrying those three times gets 99.9% through. What breaks performance is re-queueing failures immediately with no delay: they occupy thread slots that healthy URLs could be using. Exponential backoff between attempts fixes it. If you use the official SDK, retries are built in at a default of 3 and can be raised with `retry=5`.

**Respect the 2MB per-request ceiling.** Larger files fail. Images and PDFs are scraped like any other URL, but they count against the same size limit.

**Check your own machine before blaming the plan.** If the dashboard shows you below your thread limit while the code is set correctly, the bottleneck is local: antivirus or firewall inspection, network limits, or the operating system's open-file limit. On Linux that's `ulimit` and `/etc/security/limits.conf`; on Windows it's registry values. Each open connection counts against the file descriptor limit, so a machine capped at 1,024 descriptors will stall well before a 200-thread plan does.

For genuinely large jobs, the Async API accepts batches of up to 50,000 URLs and manages concurrency server-side. You trade real-time results for not having to think about threads at all. 👉 [Compare the plans that include higher thread limits](https://www.scraperapi.com/?fp_ref=coupons)

## Faster threads can burn credits faster

Threads and credits are separate budgets, and the interaction is where budgets go wrong. Credits are charged per request, and the cost per request depends on the target and the parameters:

| Request type | Credits per request |
| --- | --- |
| Standard HTML page | 1 |
| Amazon product page | 5 |
| Google / Bing SERP | 25 |
| LinkedIn | 30 |
| `render=true` (JavaScript rendering) | +10 |
| `premium=true` (premium proxies) | +10 |
| Premium proxy + rendering combined | 25 |
| Ultra-premium proxy + rendering | 75 |

The stacking is not additive in the way people assume: premium plus rendering is 25 credits, not 20, and ultra-premium plus rendering is 75, not 40. Only successful responses are billed (200 and 404); connection errors and timeouts don't consume credits.

Run the multipliers against a plan and the headline credit number stops meaning what it looks like. Business's 3,000,000 credits is 3 million plain HTML pages, 300,000 JavaScript-rendered pages, or 120,000 Google SERP requests. Hobby's 100,000 credits scraping Amazon pages with rendering on (5 + 10 = 15 credits each) covers roughly 6,600 pages, not 100,000.

There is a second consequence, in the other direction. At 100 threads and 8 seconds per request, a Business account can theoretically push about 37,500 requests an hour, which is around 27 million a month. You will hit the concurrency ceiling long before you exhaust the credit pool on cheap pages. Upgrading purely to get more credits, when your jobs are already slow, buys you headroom you weren't using.

## When a higher thread count is the right answer

Three signals say the plan, not the code, is the bottleneck:

- **Thread utilization sits at 100%** in the ScraperAPI dashboard for most of a run. The dashboard keeps 30 days of history on Hobby and Startup, and unlimited history on Business and above.
- **Runtime scales linearly with URL count.** Double the URLs and the job takes twice as long, every time, even though you set `max_workers` to your plan's limit.
- **You have trimmed retries, fixed timeouts and checked local limits, and it is still slow.**

Two more reasons to move up a tier that have nothing to do with speed: you need country-level proxies (Business and above), or you need pay-as-you-go overflow so a credit overrun doesn't pause the account (Scaling and above).

The tier ladder roughly doubles threads at each step, so the jump from Hobby to Startup turns a 375 pages/minute ceiling into 750, and Startup to Business doubles it again. If your monthly volume is under 100,000 plain pages and nothing is time-critical, Hobby's 20 threads will finish the work; paying for 100 threads to run a job that completes overnight anyway is money spent on nothing.

## Which plan fits which job

- **Checking whether ScraperAPI works on your targets at all:** Free, 5 threads, 1,000 credits a month, plus the 5,000-credit trial in week one.
- **Under ~100,000 plain HTML pages a month, no geo requirements:** Hobby at $49.
- **Up to a million credits, still US/EU only:** Startup at $149.
- **Country-level IPs, or 100 threads, or analytics history you can actually audit:** Business at $299.
- **Multi-million page jobs with deadlines, or you want PAYG overflow:** Scaling and above.

## Two checks before you pay

**Test your actual targets first.** Success rates vary by site, and third-party benchmarks that show near-total success on some domains also show zero on others. Run a few hundred of your real URLs on the trial and measure two things: the share that come back 200, and the credits each one costs after multipliers. The dashboard's per-domain cost estimator gives you the second number without guessing.

**Convert credits into pages before comparing tiers.** A plan that looks generous at 3 million credits is a 300,000-page plan once rendering is on, and a 120,000-request plan on Google SERPs. Match the tier to the page count you actually need, then check whether the thread count covers your deadline. The plans and billing docs list a 7-day trial and a 7-day refund window, so the trial run costs you nothing but the afternoon.

👉 [Check current ScraperAPI pricing and pick your thread count](https://www.scraperapi.com/?fp_ref=coupons)
