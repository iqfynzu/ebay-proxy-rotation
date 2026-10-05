# ebay proxies: How to Choose and Set Up IPs for Scraping eBay and Running Multiple Seller Accounts

Open a few hundred eBay search pages from one connection and you'll meet the wall quickly: a 403, or worse, a 200 response with an empty results page that looks fine until your parser returns zero rows. eBay scores the network behind a request, and a single IP doing bot-shaped work doesn't last long. If you're running more than one seller account, the problem is the opposite — the IP needs to stay exactly the same, forever, because a moving exit under a stable login looks like an account takeover.

That's the whole tension with eBay proxies: scraping wants rotation, accounts want permanence. Most of the bad advice out there treats "eBay proxies" as one product. It isn't. Below is how to pick the right type, how to configure sessions that don't fall over, what the traffic actually costs, and where the cheap options stop working.

## First, a naming collision worth clearing up

Search "eBay proxy" and half the results are about proxy bidding — eBay's own automatic bidding system, where you enter a maximum and the platform raises your bid in small increments. That has nothing to do with network proxies. The other half of the results mean a proxy server that routes your traffic through a different IP address.

This article is about the second thing: the network layer that sits between your script or browser and eBay's servers. If you're here for bidding mechanics, that's an eBay feature, not a product you buy.

## What eBay actually looks at

The block doesn't come from one signal. From what's publicly documented and what providers who work with eBay traffic describe, these are the checks that matter:

**IP type and the ASN behind it.** Datacenter ranges are cheap to enumerate. eBay's systems know a hosting ASN when they see one, and login or listing actions from those ranges get extra friction. Residential and carrier IPs sit inside consumer networks, where the same address is shared by real shoppers.

**Exit stability inside a session.** Seller work is long: drafting listings, revising them, answering buyer messages, checking payouts. If the exit IP changes mid-session, eBay sees a stable cookie arriving from a new network and pushes re-verification.

**Browser and device fingerprint.** Canvas, WebGL, fonts, timezone, screen metrics. Two accounts sharing one browser profile stay linked even on completely separate IPs, so profile isolation matters as much as IP isolation.

**Geo coherence.** The exit country gets compared against the account's registered address, shipping origin, and which eBay site you're on. A UK seller account driven from an Asian exit is a mismatch that shows up on every single login.

**Request rate on public pages.** Search results and sold-listing pages are rate limited per exit. One IP firing thousands of requests returns CAPTCHAs, throttled pages, or truncated result sets long before any account is involved.

**Non-IP linkage.** Payment methods, payout bank details, addresses, phone numbers, email patterns. No proxy setup fixes a shared bank account. Separate those before blaming the network.

Worth saying plainly: a proxy changes your apparent network origin. It does not reproduce the shopper's delivery ZIP, does not make a desktop browser look like the mobile app, and does not launder account data you've already shared between accounts.

## Which type for which job

Choosing per job is where most of the money and most of the failures hide.

| Job | Proxy type | Session mode |
| --- | --- | --- |
| Scraping search results and sold listings at volume | Rotating residential | Fresh IP per request, or a short sticky window per listing |
| Paging through one product page or an auction detail | Sticky residential | One exit held for the length of that parse |
| One seller account, long-term | Static residential or ISP | One IP per account, no rotation |
| Testing a regional view (ebay.com vs .co.uk vs .de) | Residential in that country | Sticky, matched to the marketplace domain |
| Landing pages, open directories, high throughput, no account | Datacenter | Fast, cheap, high concurrency |
| Accounts that keep getting flagged, or mobile-only surfaces | Mobile (4G/5G) | Sticky or static, highest cost |

Two rows cause most of the pain. People scrape with a sticky session pinned to one IP and wonder why they get CAPTCHAs after 200 pages. And people run seller accounts on a rotating pool, which is the clearest possible signal that the login is not a person at home.

**How many IPs you need:** for accounts, the unit is the account, and it's one dedicated IP each — never shared, ideally on different subnets, each geo-matched to the account's registered address. For scraping there are no named IPs to count. You buy bandwidth and let the rotation do the work.

## Setting it up: DataImpulse as the working example

DataImpulse sells residential, datacenter, mobile, and premium residential IPs on a pay-as-you-go model — traffic doesn't expire, and there's no subscription. Residential starts at $1/GB, datacenter at $0.50/GB, mobile at $2/GB.

The setup is the same shape on any provider, so here's the concrete version. Create an account, add the proxy type you need, and top up your balance. You get a gateway endpoint and credentials, not a downloadable IP list:


http://LOGIN:PASSWORD@gw.dataimpulse.com:823


Port 823 is the rotating HTTP/HTTPS gateway; port 824 is SOCKS5. Ports in the 10000–20000 range are sticky sessions, and the sticky window runs from 1 to 120 minutes, with 30 minutes as the default if you don't specify. In Python that's a one-line change:

python
proxies = {
    "http":  "http://LOGIN:PASSWORD@gw.dataimpulse.com:823",
    "https": "http://LOGIN:PASSWORD@gw.dataimpulse.com:823",
}
html = requests.get(url, headers=HEADERS, proxies=proxies, timeout=30)


Targeting travels in the username string rather than a dashboard setting, which is convenient when you're scraping several marketplaces in one script. Country is included in the base rate — `user:pass_country-us` or `_country-de`. State, city, ZIP, and specific ASN selection route at double the standard per-GB rate on residential plans, so factor that in if your workflow needs city-level precision. Premium residential includes all targeting with no surcharge.

If your workload is regional price comparison or a light scheduled pull rather than a pipeline, the 👉 [DataImpulse residential intro plan](https://bit.ly/dataimPulse) is 5 GB for $5, which is enough to measure your real cost per successfully parsed page instead of guessing from the sticker price.

## Rotation settings that survive contact with eBay

The instinct when you get blocked is to rotate harder. That's usually backwards — if the CAPTCHA rate is climbing, more churn from the same pool often makes it worse. Practical starting points:

- **Search and category discovery:** rotate every 5–20 requests per exit.
- **Product detail pages:** rotate every 10–50 requests per exit.
- **Concurrency per exit:** start at 1. Raise it only while success stays high and CAPTCHAs stay rare.
- **Sticky window:** hold one exit for about 10 minutes when you're working a single page type; don't mix regions inside one batch if you care about comparable pricing.
- **After a 429:** read `Retry-After` if it's there, otherwise back off exponentially, and lower both your rate and your concurrency before doing anything else.
- **After a 403:** slow down, check your headers, then switch exits. Full browser headers — User-Agent, Accept-Language, Referer — fix more 403s than new IPs do.
- **On a CAPTCHA page:** stop pushing. Stabilize the exits and reduce rotation, because a CAPTCHA usually means request rate, not IP reputation.

Keep exits aligned to the marketplace you're scraping, and keep separate pools per market so your benchmarks stay comparable across runs.

## What the traffic actually costs

DataImpulse's published pricing, by product. Everything is pay-as-you-go, traffic doesn't expire, and no subscription is required.

**Residential proxies** — rotating and sticky, HTTP(S)/SOCKS5, country targeting included, 90M+ IPs across 195 countries.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 5 GB | $5 | $1.00 | [Start with 5 GB of residential traffic](https://bit.ly/dataimPulse) |
| Basic | 50 GB | $50 | $1.00 | [Get the 50 GB residential plan](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $800 | $0.80 | [Buy 1 TB residential at $0.80/GB](https://bit.ly/dataimPulse) |
| Custom+ | 5 TB+ | From $4,000 | Custom | [Request a residential volume quote](https://bit.ly/dataimPulse) |

**Datacenter proxies** — rotating, 99.9% uptime claim, randomized subnets, 20M IPs, sub-100ms response times, sticky sessions up to 30 minutes.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 10 GB | $5 | $0.50 | [Try 10 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Basic | 100 GB | $50 | $0.50 | [Get the 100 GB datacenter plan](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $450 | $0.45 | [Buy 1 TB datacenter at $0.45/GB](https://bit.ly/dataimPulse) |
| Custom+ | 5 TB+ | From $2,250 | Custom | [Request a datacenter volume quote](https://bit.ly/dataimPulse) |

**Mobile proxies** — 3G/4G/5G/LTE, rotating and sticky, 16M+ mobile IPs, sessions active up to two hours.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 2.5 GB | $5 | $2.00 | [Try 2.5 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Basic | 25 GB | $50 | $2.00 | [Get the 25 GB mobile plan](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $1,600 | $1.60 | [Buy 1 TB mobile at $1.60/GB](https://bit.ly/dataimPulse) |
| Custom+ | 5 TB+ | From $8,000 | Custom | [Request a mobile volume quote](https://bit.ly/dataimPulse) |

**Premium residential proxies** — filtered top-tier IPs, all targeting options included, dedicated proxy manager, 24/7 human support.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 1 GB | $5 | $5.00 | [Try 1 GB of premium residential](https://bit.ly/dataimPulse) |
| Basic | 10 GB | $50 | $5.00 | [Get the 10 GB premium plan](https://bit.ly/dataimPulse) |
| Custom+ | 5 TB+ | From $20,000 | Custom | [Request a premium residential quote](https://bit.ly/dataimPulse) |

A few notes that affect the real bill rather than the headline number. Advanced targeting on standard residential routes at 2× the per-GB rate, so a heavy city-targeted scrape is cheaper on premium residential than it looks. Mobile and premium residential discounts only kick in properly at the terabyte tier. And the per-GB price is not the metric that matters — cost per successfully parsed page is, which is why starting with a 5 GB intro and measuring beats guessing.

Third-party write-ups mention a 7-day refund window for new users, with crypto payments excluded; confirm the current terms with support before you rely on it. DataImpulse publishes a 99.51% success rate and is rated 4.8/5 on G2, both of which are the provider's own numbers rather than independent verification. TechRadar's review credits the residential pool with a consistently high scraping success rate and flags the same structural trade-off: there's no managed scraping API, so you write your own retries, parsing, and CAPTCHA handling. If you want someone else to run the pipeline, this isn't that product.

## Free proxies, and why eBay is the worst place to try them

Free proxy lists are mostly datacenter IPs that die within minutes, and the ones that survive have usually been run through banned accounts already. On a marketplace, that matters more than on a normal site: eBay's systems compare account history against the address, so building a seller account on a shared public IP means inheriting whatever the last few hundred users did with it.

There's a narrower honest use: a one-off look at how a listing renders in another country, or checking that your parser's plumbing works before pointing paid IPs at it. Neither is worth building on. For scraping you want rotating residential billed by bandwidth; for accounts you want an address that stays yours.

## Running more than one seller account

The rule that matters: one account, one dedicated IP, never shared, and that IP doesn't move. If you only have a rotating pool, pin a sticky session long enough to cover the whole session — but understand that "long enough" for a seller account means permanently, which is what static residential or ISP IPs are for.

Match the exit to the account's registered country and the marketplace domain it sells on. Keep each account in its own isolated browser profile, because fingerprint sharing links accounts even when every IP is clean. And separate the non-network identifiers before you spend anything on proxies — different payment methods, different payout details, different addresses, different phone numbers. A proxy is the last link in that chain, not the first.

## Questions that come up often

**Do I need proxies to scrape eBay at all?** For a handful of requests, no. For any real volume, yes. eBay scores IP reputation and rate limits aggressively, so one address gets blocked or throttled fast. Rotating residential spreads the load and lets you target the country whose pricing you actually want.

**Will eBay ban me for using a VPN?** eBay may flag an account if it sees frequent location changes or multiple accounts on one address. A stable, geo-matched IP behaves more like a normal home connection than a VPN that hops countries.

**Can't I just use eBay's official API?** For anything the API covers, yes — and that's the cleaner path where supported data exists. Proxies are for the pages and regions the API doesn't expose, and they don't exempt you from eBay's terms or from local data-protection rules.

**What breaks a working setup first?** Usually headers, not IPs. Missing a realistic User-Agent, Accept-Language, and Referer is the most common cause of 403s and empty 200s. If adding the full header set doesn't help, you've likely hit a rate limit — slow down and route through residential instead of rotating harder.

The short version: pick the proxy type per job, keep accounts on addresses that don't move, scrape on ones that do, and measure cost per parsed page instead of per gigabyte. The hardware is the easy part.
