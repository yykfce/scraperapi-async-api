# ScraperAPI Async API: Endpoints, 50,000-URL Batches, Webhooks, Retention and Credit Costs

Async scraping on ScraperAPI starts with a POST to `https://async.scraperapi.com/jobs` and ends with either a status URL you poll or a webhook payload that lands in your own service. The job retries in the background for up to 24 hours before it gives up. One batch request accepts up to 50,000 URLs, and a finished result stays retrievable for up to 72 hours, 24 of which are guaranteed.

What follows is the working layer: request and response shapes you can paste into a script, the parameters that change what a job costs, and the failure states to handle before you point production traffic at it.

> **Quick look.** For a first async pipeline over ordinary pages, Hobby at $49/month (100,000 credits, 20 concurrent threads) is the sensible starting point. Which plan you actually need depends more on the domains you target than on URL count: Google and Bing pages cost 25 credits each, LinkedIn 30, and a page behind Cloudflare, DataDome or PerimeterX adds 10. 👉 [See every plan and the current price list](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The plans, and what a credit actually buys

The pricing page lists six self-serve plans between $49 and $1,975 a month, plus a free tier and an Enterprise tier priced by quote. Annual billing runs about 10% below the monthly figures, and credits reset each cycle without rolling over.

| Plan | Price / month | Credits / month | Cost per 1,000 credits | Fits |
| --- | --- | --- | --- | --- |
| [ Free plan](https://www.scraperapi.com/?fp_ref=coupons) | $0 | 1,000 | — | Smoke-testing the endpoint and one or two domains |
| [ Hobby](https://www.scraperapi.com/?fp_ref=coupons) | $49 | 100,000 | $0.49 | A first async pipeline on ordinary HTML pages |
| [ Startup](https://www.scraperapi.com/?fp_ref=coupons) | $149 | 1,000,000 | $0.149 | Rendered pages or premium proxies at low volume |
| [ Business](https://www.scraperapi.com/?fp_ref=coupons) | $299 | 3,000,000 | $0.10 | Mixed domain lists, including search engines |
| [ Scaling](https://www.scraperapi.com/?fp_ref=coupons) | $475 | 5,000,000 | $0.095 | Steady production batches with room to grow |
| [ Professional](https://www.scraperapi.com/?fp_ref=coupons) | $975 | 10,500,000 | $0.093 | Heavy render/premium mix across many sites |
| [ Advanced](https://www.scraperapi.com/?fp_ref=coupons) | $1,975 | 21,500,000 | $0.092 | Large continuous workloads |
| [ Enterprise](https://www.scraperapi.com/?fp_ref=coupons) | Quote | Custom | — | Committed volume and support terms |

The per-credit column is the part worth staring at. Hobby costs more than three times as much per credit as Startup, and Startup costs roughly 50% more per credit than Business. Hobby is priced for people testing the product, not for people running it.

The multipliers from the pricing page convert credits into real capacity. A standard page is 1 credit, so Hobby's 100,000 credits is 100,000 plain pages. Turn on `render: true` and that drops to 10,000 pages in the same month. Point the same credits at Google and you get 4,000 requests, because Google and Bing cost 25 credits per page.

## Async or Standard: where the line sits

Use the async endpoint once you are submitting around 20 URLs or more, when the target site is slow or unstable, when you want webhook delivery instead of polling, or when you need to pull PDFs and images. Those are the conditions ScraperAPI itself names in its async reference.

The trade is deliberate: async optimises for success rate rather than latency, which is why a job can sit in the queue retrying for a full day. A single URL where you need the HTML inside the same function call belongs on the Standard API at `api.scraperapi.com`, which returns inline and keeps your code shorter. Following links across one site is the Crawler's job, and recurring scheduled runs belong in DataPipeline.

The vendor advertises a 99.99% success rate and capacity above 100 million monthly requests on the async product page. Treat both as vendor figures rather than a property of your target list, and validate the domains you care about before committing to a tier.

## One job, start to finish

Four endpoints cover the whole lifecycle. Submit a single URL with POST to `https://async.scraperapi.com/jobs`, submit a list with POST to `/batchjobs`, read a job with GET on its status URL, and cancel it with DELETE on the same path.

Authentication lives in the JSON body as `apiKey`, in camelCase. This is the first thing that breaks when people copy a Standard API snippet across, because the synchronous endpoint uses `api_key` with an underscore. A wrong key comes back as HTTP 401 at submission time.

python
import requests, time

API_KEY = "YOUR_KEY"

job = requests.post(
    "https://async.scraperapi.com/jobs",
    json={
        "apiKey": API_KEY,           # camelCase, unlike api_key on the Standard API
        "url": "https://example.com/product/123",
        "apiParams": {"render": True, "country_code": "us"},
    },
).json()
# {"id": "...", "status": "running", "statusUrl": "...", "url": "..."}

def poll(status_url, interval=5, max_wait=120):
    deadline = time.time() + max_wait
    while time.time() < deadline:
        data = requests.get(status_url).json()
        if data["status"] == "finished":
            return data["response"]["body"]
        if data["status"] == "failed":
            raise RuntimeError(data.get("failReason"))
        time.sleep(interval)
    raise TimeoutError(status_url)

html = poll(job["statusUrl"])


The `max_wait` in that helper is your client giving up, not the job dying. The job keeps retrying server-side for up to 24 hours unless you cap it with `timeoutSec`. Set your poll loop to something shorter than that and hand the URL to a background worker, or you will hold a request open for a day.

Node is the same shape:

js
const axios = require("axios");

const { data } = await axios.post("https://async.scraperapi.com/jobs", {
  apiKey: "YOUR_KEY",
  url: "https://example.com",
});
console.log(data.statusUrl);


A finished job looks like this, with the payload in `response.body` and the final URL and status echoed in the response headers:

json
{
  "status": "finished",
  "response": {
    "headers": { "sa-final-url": "https://example.com/", "sa-statuscode": "200" },
    "body": "<!doctype html>...",
    "statusCode": 200
  }
}


Two variations matter. If you need to POST to the target site, pass `method: "POST"` together with `headers` and `body` at the top level of the job. If the target returns a PDF or an image, the bytes arrive in `response.base64EncodedBody`, so run it through Base64 decoding and write the file yourself.

The parameter set inside `apiParams` is where most of the useful behaviour lives: `autoparse`, `country_code`, `keep_headers`, `device_type`, `follow_redirect`, `premium`, `ultra_premium`, `render`, `wait_for_selector`, `screenshot`, `retry_404`, `output_format` and `max_cost`. Three more are specific to async and go at the top level: `expectUnsuccessReport`, `timeoutSec` and `meta`.

## Batch jobs: a list in, a list of jobs out

POST your array to `https://async.scraperapi.com/batchjobs` with the field name `urls`. The response is not one job object. It is a list with one `{id, status, statusUrl, url}` entry per submitted URL, which changes how you have to track progress.

Fifty thousand URLs is the hard ceiling per batch request, and it cannot be exceeded. Larger workloads get split into several batch calls, each with its own list of job objects to reconcile.

Polling works fine for a hundred jobs and becomes a problem at ten thousand, because you are making ten thousand serial GET calls before you can move on. This is the point where webhooks pay for themselves. Add a `callback` object:

json
{
  "apiKey": "YOUR_KEY",
  "url": "https://example.com",
  "callback": { "type": "webhook", "url": "https://your-service.example/hooks/scraperapi" },
  "expectUnsuccessReport": true,
  "meta": { "batchId": "run-2026-11", "sourceFile": "urls.csv" }
}


Four behaviours of that callback are easy to miss:

- By default only successful jobs trigger a delivery. Failed jobs stay silent unless you set `expectUnsuccessReport: true`, which is exactly the case where you want a payload, since a failure payload carries `status: "failed"`, an `attempts` count and a `failReason` such as `failed_due_to_timeout`.
- Delivery is retried three times. If all three fail, the job is cancelled rather than queued forever, so your endpoint needs to answer quickly and do the heavy work asynchronously.
- The URL has to be publicly reachable. A localhost listener will never receive anything.
- `meta` is echoed back on every response and every callback. It is the cheapest way to tie a returned job to a batch ID, a source file or a downstream table row without maintaining your own ID map.

## Retention: 72 hours, 24 guaranteed

Results persist for up to 72 hours after a job finishes, with the first 24 hours guaranteed. After the window closes the data is deleted and the only remedy is resubmitting the URLs, which costs credits again.

This changes the design of anything that runs weekly. A pipeline that fires jobs on Friday and reads results on Monday is working inside the guarantee. One that fires on Friday and reconciles the following Friday is not, and a webhook receiver that writes into your own storage removes the question entirely.

## Credits, cost caps and what a failure costs

A normal flat request costs 1 credit, and extra parameters or certain domains raise that. ScraperAPI's own async reference puts concrete numbers on the common cases:

| Request type | Credits |
| --- | --- |
| Standard page | 1 |
| `render: true` | 10 |
| `premium: true` | 10 |
| `ultra_premium: true` | 30 |
| Failed request | 0 |

The same reference states that async jobs which fail after exhausting every retry are not charged, which is the reason a high-retry endpoint is affordable at all: you are paying for results, not for attempts.

`max_cost` inside `apiParams` is the guard rail. Cap a job at, say, 10 credits and a request that would cost more returns a 403 instead of quietly spending 30. For a batch pointed at an unknown domain list, that single parameter is the difference between a predictable invoice and a surprise.

Budgeting is easier with the dashboard's cost estimator, which prices a specific URL before you submit it. Do that for your top ten domains rather than estimating from page counts; a list that is 80% ordinary pages and 20% Google will not behave like the average.

👉 [Check the plan tiers against your own domain mix](https://www.scraperapi.com/?fp_ref=coupons)

## Failure modes and what to do about them

| What you see | What it means | What to do |
| --- | --- | --- |
| Job `finished`, `statusCode` 200 | Success | Read `response.body`, or decode `base64EncodedBody` for binary targets |
| Job `finished`, `statusCode` 403 | The target blocked the request | Resubmit with `premium: true`; escalate to `ultra_premium` if it still blocks |
| Job `failed`, `failReason: failed_due_to_timeout` | 24 hours of retries exhausted | Confirm the URL is reachable; try `render: false`; raise `timeoutSec` only for genuinely slow pages |
| HTTP 401 on submission | Wrong or missing key | Check that the body sends `apiKey`, not `api_key` |
| HTTP 403 on submission | Out of credits, plan limit reached, or `max_cost` cap hit | Check the dashboard and the cap value on that job |
| HTTP 429 on submission | Too many concurrent submissions | Back off, then resubmit in smaller batches |

A 429 is a concurrency refusal, not a billing event: the request is turned away rather than queued and you are not charged for it. Throttling your submitter to the plan's thread count is cheaper than retrying into the same wall.

👉 [Compare plan limits before you scale the submitter](https://www.scraperapi.com/?fp_ref=coupons)

## When async is the wrong tool

Async is a poor fit for anything a user is waiting on. An API endpoint that has to answer in 800ms cannot wait on a job that retries for a day, and a page render that blocks on a webhook will time out long before the callback fires.

It is also overkill for a handful of URLs per run. Below roughly 20 URLs, the Standard API returns inline, keeps your code synchronous, and costs the same 1 credit for a plain page.

If your problem is crawling a site's internal links rather than fetching a known list, the Crawler handles link discovery. If your problem is "run this every morning at 6am", DataPipeline owns the schedule and the storage.

## Picking a plan, in one pass

Match the plan to the domain mix, not to the URL count.

- Ordinary HTML pages, up to a few hundred thousand a month with async retries: **Hobby** at $49 covers it, and the 20 concurrent threads are enough for a single submitter.
- A meaningful share of Google, Bing or LinkedIn requests: run the 25-to-30-credit multiplier over your monthly volume first. That arithmetic usually pushes the decision to **Startup** or **Business**, because Hobby's 100,000 credits are only 4,000 search results.
- Rendered pages or bot-protected targets at volume: the 10-credit multipliers dominate the bill, and per-credit efficiency starts to matter more than the headline price.
- More than 50,000 URLs in one run: split into multiple batch requests and drive everything through webhooks, with `meta` carrying the batch ID.

Before you pay, confirm three things on the pricing page: the trial (7 days with 5,000 credits) and the free tier (1,000 credits, 5 concurrent connections) both let you price your real domains first; credits do not roll over, so size for an average month rather than a peak one; and paid plans are reported to carry a 7-day refund window, which is worth confirming at checkout if you are committing to an annual term.

👉 [Start on the plan that matches your domain mix](https://www.scraperapi.com/pricing/?fp_ref=coupons)
