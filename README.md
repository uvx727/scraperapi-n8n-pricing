# web scraping n8n workflow: how to build one that survives bot walls, plus what each ScraperAPI plan actually costs

A web scraping n8n workflow needs four things to be reliable: a trigger, a fetch layer that can get past anti-bot protection, a parse step, and somewhere to put the data. The hard part is the second one. n8n's built-in HTTP Request node makes plain requests, so Cloudflare-protected or heavily bot-monitored sites answer with a block page, a CAPTCHA, or nothing at all — and because the failure is silent, your downstream nodes happily keep writing empty rows into Google Sheets until someone notices the dashboard has been stale for a week.

The fix most people land on is a scraping API node. ScraperAPI ships an official n8n node that routes each request through a pool of 40M+ proxies with CAPTCHA handling and JS rendering baked in, and — this matters more than it sounds — returns a clear error when a page genuinely can't be reached instead of empty data. This article walks through how the workflow goes together, where the real costs hide (spoiler: a "100,000 credit" plan does not mean 100,000 pages), and which plan makes sense for which job.

> **Quick verdict:** for most n8n scraping, the **Hobby plan at $49/month** (100,000 credits, 20 concurrent threads) covers standard HTML pages and moderate Amazon monitoring. If you're scraping Google SERPs daily, do the credit math first — SERP queries cost 25 credits each. Every paid plan comes with a 7-day trial with 5,000 credits and no credit card required, so you can test your actual targets before paying anything. 👉 [Start the 7-day free trial with 5,000 credits](https://www.scraperapi.com/?fp_ref=coupons)

## How a scraping workflow is structured in n8n

The skeleton is the same whether you use the official ScraperAPI node or roll your own:

| Stage | Node | Job |
| --- | --- | --- |
| Trigger | Schedule Trigger / Manual Trigger | Decide when the workflow runs |
| Fetch | ScraperAPI node (or HTTP Request) | Get the page content |
| Parse | Built-in HTML/Code node, or `autoparse=true` | Turn raw content into fields |
| Store or alert | Google Sheets, Airtable, Postgres, Slack | Do something with the result |

That structure shows up in nearly every tutorial on the topic, which is fine — it's correct. Where tutorials stop short is everything after the skeleton: the parameter combinations that quietly multiply your credit burn, and the plan-level restrictions that only show up in the docs. Those are covered below.

One option worth knowing about before you build: ScraperAPI maintains pre-built n8n templates, including daily price-drop digests for Amazon, Walmart, and Google that you can import and adapt instead of wiring nodes from scratch.

## Why the HTTP Request node fails on protected sites

The HTTP Request node sends exactly what it says: a plain HTTP request. Bot-protected sites flag it and respond with a block page, a CAPTCHA, or an empty response. The dangerous part isn't the block itself — it's that the failure is silent. Your next node maps "nothing" into empty fields, the workflow shows a green checkmark, and nobody notices until the price tracker has a week of holes in it.

ScraperAPI's own docs put the comparison plainly: the ScraperAPI node routes the same request through rotating proxies with CAPTCHA handling and JS rendering, and returns a clear error when a page can't be reached. For scheduled workflows that feed dashboards, that loud-failure behavior is the difference between a broken pipeline you find in minutes and one you find in a week.

There's a secondary benefit: output. Amazon products and prices, Google Search results, Redfin listings, and Walmart products come back as pre-parsed JSON — price, title, availability, rating fields ready to map straight into Google Sheets, Slack, Airtable, or a database. No HTML-parsing node in the middle. An example from the official page:

json
{"product":{"name":"AirPods Pro 2","price":239.00,"currency":"USD","availability":"In Stock","rating":4.8,"reviews":31204}}


## Setting up the ScraperAPI node

Setup is short. The node works on both n8n Cloud and self-hosted instances, and the scraping itself runs on ScraperAPI's infrastructure, so your n8n host needs no proxy configuration and no browser dependencies.

1. Log into the ScraperAPI dashboard and copy your API key.
2. In your n8n workflow, click **+** to add a node, search for **ScraperAPI**, and install it.
3. Paste the API key in as a credential.
4. Select the **API** resource and enter the target URL.
5. Configure optional parameters — JS rendering, geo-targeting, premium proxies — as needed.
6. Execute. The node returns the scraped content or structured JSON.

For AI agent workflows, the wiring is: a Chat Message Received trigger feeds an AI Agent node; connect an AI Chat Model (OpenAI or similar) to the Agent's Chat Model input, a Simple Memory node to Memory, and the ScraperAPI node to the Tool input. Add a system prompt describing the task, and the agent can pull live web data on demand. ScraperAPI also runs a hosted MCP server (`https://mcp.scraperapi.com/mcp`) that connects through n8n's MCP Client Tool with Bearer auth — same API key, HTTP Streamable transport.

If you'd rather not touch the node at all, the MCP route or the plain HTTP Request node pointed at the ScraperAPI API endpoint both work; the official node is just the tidy version.

## ScraperAPI plans and prices

Here's the full current lineup from the pricing page. Annual billing takes 10% off every paid plan.

| Plan | Monthly price | Annual (per month) | API credits | Concurrent threads | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [Sign up free](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [Get Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [Get Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [Get Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [Get Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [Get Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [Get Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [Contact sales](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Every plan includes JS rendering, premium proxy support, CAPTCHA and anti-bot detection, automatic retries, structured data endpoints, unlimited bandwidth, and a 99.9% uptime guarantee. What actually separates the tiers is volume, concurrency, and two quieter differences: Hobby and Startup are limited to **US & EU geo-targeting** and a 30-day analytics window, while Business and up get country-level targeting and unlimited analytics history.

## The credit math nobody runs before buying

This is where most plan comparisons go quiet, and it's the part that determines whether your plan is cheap or a trap.

One credit is the cost of a standard page request. But heavier targets cost more, and the multipliers stack — not always linearly:

- Standard HTML page: **1 credit**
- Amazon, eBay, Walmart product pages: **5 credits**
- Google and Bing (including subdomains): **25 credits**
- LinkedIn: **30 credits**
- `render=true` (JS rendering): **+10**
- `premium=true` proxies: **+10**
- `ultra_premium=true`: **+30**
- Cloudflare, Datadome, or PerimeterX bypass: **+10**, applied automatically when detected

The catch is the combinations. `premium=true` plus `render=true` costs **+25 total, not +20**. `ultra_premium=true` plus `render=true` costs **+75, not +40**. That last combination is documented but not shown on the pricing page, and it's where people get burned: a Hobby plan's 100,000 credits cover 100,000 standard pages, but only around **1,333 requests** against a Cloudflare-protected site with JS rendering and ultra-premium proxies. Same plan, 75x difference.

For scale reference, a Scaling plan ($475/month, 5M credits) buys roughly 5M standard pages, 1M Amazon pages, 200K Google SERP queries, or 500K JS-rendered pages per month.

Two tools keep this from being guesswork:

- The **Domain Cost Estimator** in your dashboard shows the exact per-URL cost before you commit. Use it on your actual targets, not a sample page.
- You can set a **`max_cost` per scrape** to cap what any single request can consume.

> Run your real targets through the cost estimator before picking a plan. "How many requests do I get" is a question with no single answer — it depends entirely on what you're scraping. 👉 [Check plan prices and the free trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)

Also useful to know: exceeding your concurrent thread limit returns a 429 asking you to back off, with no extra charge. Failed requests are not billed — you only pay for successful results.

## Overdraft mechanics: PAYG is not what it sounds like

A common assumption is that "pay as you go" means you can sign up and pay per request without a subscription. On ScraperAPI it doesn't work that way.

PAYG here is an **overflow feature**, available only on Scaling, Professional, Advanced, and Enterprise. When your included credits run out, PAYG lets you keep scraping at a fixed per-credit rate, up to a monthly spending cap you set yourself. There is no pure per-request tier with no monthly commitment.

For Hobby, Startup, and Business, the options are different: scraping pauses when credits run out (until renewal), or you can opt into **auto-upgrade**, which moves your account to the next plan automatically when credits hit zero. Available on Hobby through Professional.

Translation for plan-pickers: if you genuinely need usage-based overflow protection, the floor is Scaling at $475/month. If your volume is steady and predictable, that feature is irrelevant — Hobby or Startup with a well-estimated credit budget is the better buy.

## Parameter combinations to set carefully in the n8n node

The official node exposes several parameters that look harmless in a dropdown and get expensive or mutually exclusive in practice:

- **RENDER** increases processing time on every call. Only switch it on for pages that actually need JS execution — many "dynamic-looking" pages render fine without it.
- **PREMIUM and ULTRA_PREMIUM cannot be combined.** Pick one.
- **SESSION_NUMBER cannot be used with Premium or Ultra Premium.** If your workflow needs session persistence, it can't also need premium proxies on that request.
- **KEEP_HEADERS** (custom headers/cookies) is currently supported only through the HTTP Request Node path, and is incompatible with Ultra Premium.
- **RETRY_404** doesn't work under Premium or Ultra Premium.
- **autoparse=true** returns parsed JSON and — per the docs — does not increase the request cost. Free money where available.
- **ZIP** scopes Amazon results to a US postal code (Amazon US only).

A third-party breakdown of the API also notes the retry-404 and session restrictions trip people up most often, precisely because nothing in the n8n UI flags the conflicts — you find out from an API error or a bigger bill.

## How it compares to the alternatives

Three other routes people take, with the honest trade-offs:

**n8n's own browser/Playwright-style nodes.** You get a headless Chromium inside the workflow: navigate, wait for selectors, click, screenshot. The costs are real — your n8n host needs Chrome dependencies and enough RAM, community nodes with native dependencies don't run on n8n Cloud, and you own the proxy rotation and every future anti-bot update yourself. This route makes sense when you need full browser interaction (logins, multi-step flows) that a scraping API isn't built for.

**ScrapingBee.** Also has a managed API for n8n, and the company publishes an open-source no-code scraper template that chains n8n with ScrapingBee's API plus an LLM for AI enrichment. Functionally a close sibling to what's described here; current per-request pricing wasn't verified for this article, so compare credit systems directly if you evaluate it.

**Browse AI.** Runs point-and-click robots on its own platform and its own scheduler. If your automation already lives in n8n, that splits your triggers and data across two systems. The ScraperAPI node keeps the whole workflow in one place with your triggers under your control. If you have no n8n investment at all and want visual robots, Browse AI's model is genuinely easier — it's just a different architecture.

## FAQ

**Does the ScraperAPI node work on n8n Cloud?**
Yes. The node works on both n8n Cloud and self-hosted instances, since the scraping runs on ScraperAPI's side. This is one area where it beats community browser nodes, which can't run native dependencies on Cloud.

**How much does n8n web scraping cost with ScraperAPI?**
The free plan gives 1,000 credits (5 concurrent connections) with no card required, and the paid entry point is Hobby at $49/month for 100,000 credits. What that buys depends on your targets: 100,000 standard pages, or 20,000 Amazon pages, or 4,000 Google searches. Use the dashboard's cost estimator on your real URLs before committing.

**Can I scrape Google search results on a schedule?**
Yes, via the structured data options — Google SERP results come back as pre-parsed JSON with rankings and listings ready to map into Sheets or a database. Budget 25 credits per query, and remember each SERP request also counts against your plan's concurrency limit.

**Is there a trial?**
A 7-day trial with 5,000 credits, no credit card required. That's enough to test your exact target URLs with the exact parameter combinations you plan to use — which is exactly the exercise that reveals whether Hobby's 100,000 credits means "100K pages" or "1,300 pages" for your case.

## Picking a plan, and what to do before checkout

A few closing rules of thumb, based on the numbers above:

- **Just experimenting or running one small daily job:** the free plan's 1,000 credits is enough to prototype the workflow end to end.
- **Steady volume on normal HTML pages or moderate Amazon monitoring:** Hobby at $49/month. Turn on auto-upgrade if you'd rather not have scraping pause mid-cycle.
- **Daily Google SERP tracking or heavy JS-rendered targets:** do the credit math first. If the math says 300K+ effective credits, Startup at $149 is the sensible floor; if you also want PAYG overflow, that's Scaling at $475.
- **Any plan, one last check before paying:** confirm your target URLs' costs in the Domain Cost Estimator, and confirm whether you need non-US/EU geo-targeting — if yes, Hobby and Startup won't cover it, so start at Business.

👉 [Start the 7-day trial with 5,000 credits and test your targets first](https://www.scraperapi.com/?fp_ref=coupons)
