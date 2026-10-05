# adspower proxy: how to give every browser profile its own sticky IP, set up the credentials, and keep costs down

Half the people searching "adspower proxy" are stuck at the same wall: the profiles are set up, the fingerprints look fine, and the accounts still get linked within a week. The other half just want to know which fields to fill in and what it costs. Both problems come down to the same thing, so let's start there.

AdsPower handles one half of the disguise. It generates an isolated fingerprint per browser profile: canvas, WebGL, fonts, timezone, user agent, screen metrics. All local, all controlled by the app. What AdsPower cannot invent for you is the IP address that profile goes out into the world on. If two profiles leave through the same IP, no amount of fingerprint separation matters, because the shared address is the link. Platforms ban on that correlation routinely.

So the proxy sitting behind each profile, not the browser, is usually what decides whether the operation survives.

## One profile, one sticky IP

This is the whole strategy in five words, and almost every failure traces back to breaking it.

Every AdsPower profile should exit through its own dedicated address, and that address should stay the same across sessions. Logging in from a US residential IP on Monday and a German datacenter IP on Tuesday is a pattern no real user produces, and fraud systems treat sudden location and network-type jumps as a strong signal.

Practical rules that follow from this:

- Never let two profiles share a session ID. That is the fastest route to a linked-account ban.
- Rotating proxies are for scraping, not for logged-in accounts. Per-request IP changes are exactly wrong when you need continuity.
- Mobile and static residential addresses carry more trust than datacenter ranges, because datacenter subnets are publicly documented and trivially flagged.
- Set the proxy once per profile and leave it alone. Changing location mid-account is worse than picking a decent location and staying there.

If your profiles sit behind residential addresses that are sticky rather than rotating per request, you have removed the most common cause of linking. DataImpulse handles this on a pay-as-you-go model rather than a monthly subscription, which matters more than it sounds when you are testing ten profiles before scaling to two hundred. That is roughly how the pricing works: 👉 [check the current DataImpulse plans and traffic rates](https://bit.ly/dataimPulse).

## Which proxy type belongs behind an AdsPower profile

DataImpulse sells four network types, and they are not interchangeable for profile work. Here is what each one actually gets you in AdsPower, with the published per-GB rates.

| Proxy type | Published rate | Bulk rate (1 TB+) | Best suited to | Where it falls short |
| --- | --- | --- | --- | --- |
| Residential | $1.00/GB | $0.80/GB | Localized research, ordinary multi-profile work, longer browsing sessions | Still rotating underneath; the IP changes when the session expires |
| Datacenter | $0.50/GB | $0.45/GB | Speed-focused testing, public pages, technical validation | Recognizable as a datacenter range, so weak for account work |
| Mobile (3G/4G/5G/LTE) | $2.00/GB | $1.60/GB | The most scrutinized accounts, app and mobile-web flows | Highest per-GB cost, so watch your traffic budget |
| Premium Residential | $5.00/GB | Custom | Higher-priority residential traffic, dedicated account manager | $5/GB is real money at any volume |

For most AdsPower setups, residential sits in the sensible middle: real ISP-assigned addresses at a rate you can afford to run a dozen profiles on. Mobile is worth the premium when a single account matters more than the traffic bill. Datacenter is a testing tool, not a place to keep logged-in accounts. Premium residential is for teams whose target sites are picky enough that failed requests cost more than the bandwidth.

Coverage varies by type. AdsPower's own guide lists 214 locations for residential, 210 for premium residential, 191 for mobile, and 123 for datacenter, while DataImpulse advertises 195+ countries overall. Either way, plan your profile geography against the specific list for the proxy type you pick, not the headline number.

## Adding a DataImpulse proxy to an AdsPower profile

The manual path takes about a minute per profile. In AdsPower, click **New Profile**, scroll to or select the Proxy section, and fill in the fields with your DataImpulse credentials:


Proxy type:   HTTP  (AdsPower also accepts HTTPS or SOCKS5)
Host:         gw.dataimpulse.com
Port:         823
Username:     YOUR_LOGIN__cr.us;city.newyork;sessid.profile01
Password:     YOUR_PASSWORD


Port 823 handles HTTP/HTTPS traffic; use port 824 if you switch the protocol to SOCKS5. The host and port pair is the same gateway for every DataImpulse traffic type, which means the only thing that differentiates one profile from another is the username string. Get that string right and the rest is copy-paste.

Click **Check Proxy** before saving. AdsPower will report whether the connection works and what country the exit IP sits in. If the check fails, the cause is almost always a typo in the username parameters or a wrong port for the protocol you selected. Fix it there rather than saving a broken profile and debugging later.

One detail worth knowing: some ad-blocking or fingerprint-faking extensions interfere with proxy checks. If the check passes inconsistently, run it with extensions off.

## The username string is where targeting and stickiness live

Static proxy setups bury this in a dashboard. DataImpulse appends it to the login instead, which turns out to be convenient for AdsPower because each profile can carry its own location and session rule inside a single field.

The pattern is:


YOUR_LOGIN__cr.us;city.newyork;sessid.profile01


`__cr.us` sets the country, `city.newyork` narrows it to a city, and `sessid.profile01` pins a sticky session. The same session ID returns the same address, which is the behaviour you want for a profile that has to keep a consistent identity.

A few things this syntax runs into in practice:

- **Session length is finite.** Sticky sessions run from 1 to 120 minutes, and if you do not specify an interval the default lands around 30 minutes. That is long enough for a login-and-browse session, not long enough to treat the address as permanently yours.
- **Targeting beyond country is billed.** Country selection is included in the base rate. City, state, ZIP and ASN filters are charged as an add-on for standard residential traffic. Reports describe that surcharge differently, some as a flat extra and some as double the per-GB rate, so confirm the current treatment in your dashboard before you build a budget around city-level targeting.
- **Sticky ports are a separate range.** For longer-lived sticky connections, DataImpulse assigns ports between 10000 and 20000 instead of the standard gateway ports.
- **State and ZIP and ASN filters exist**, which is more granularity than most antidetect setups need. Country plus city covers the vast majority of profile work.

The honest limitation for account managers: this is a rotating residential network with sticky sessions, not a pool of permanently static residential IPs. If a platform demands that an account never changes address at all, a static ISP package is the technically correct tool, including the one AdsPower sells inside the app. DataImpulse's advantage is price and flexibility, not permanent per-account addresses.

## Setting up dozens of profiles without hand-typing credentials

Two profiles is manual work. Fifty profiles is a CSV import.

AdsPower imports proxies in bulk from a CSV, one row per profile, and the format is straightforward once you have exported a single manually-created profile to copy the column headers:


name,proxy_host,proxy_port,proxy_user,proxy_pass,proxy_type
account01,gw.dataimpulse.com,823,YOUR_LOGIN__cr.us;city.newyork;sessid.acc01,YOUR_PASSWORD,http
account02,gw.dataimpulse.com,823,YOUR_LOGIN__cr.gb;city.london;sessid.acc02,YOUR_PASSWORD,http


The critical part is that every row has a different `sessid`. Generate them from your own account naming convention (acc01, acc02, and so on) so you can trace an address back to a profile later. Reusing a session ID across rows quietly puts two accounts on one IP, and you will not notice until the bans arrive.

Verify credentials before importing at scale. A single curl call confirms the exit country and city:


curl -x "http://YOUR_LOGIN__cr.us;sessid.acc01:YOUR_PASSWORD@gw.dataimpulse.com:823" http://ip-api.com/json


If that returns a US address, the username syntax is right and the import will work. If it errors, the problem is in your string, not in AdsPower. Doing this once saves you from debugging fifty broken profiles.

For a large rollout, buy a small traffic block first, test ten profiles against your actual target platforms, and only then scale. Traffic you have already purchased does not expire, so a small test block is not money wasted. 👉 [Start with the $5 intro package and test before you scale](https://bit.ly/dataimPulse).

## What DataImpulse costs: every package on the pricing page

DataImpulse does not sell subscriptions. You buy a block of traffic and spend it whenever you want, at a rate that drops as the block gets bigger. Every row below is a prepaid traffic package, not a monthly bill.

| Proxy type and package | Traffic included | Price | Effective rate | Billing |
| --- | --- | --- | --- | --- |
| Residential — Intro | 5 GB | $5 | $1.00/GB | One-time, pay-as-you-go —  [buy the residential intro block](https://bit.ly/dataimPulse) |
| Residential — Standard | 50 GB | $50 | $1.00/GB | One-time —  [buy 50 GB of residential traffic](https://bit.ly/dataimPulse) |
| Residential — Volume | 1 TB | $800 | $0.80/GB | One-time —  [buy the 1 TB residential tier](https://bit.ly/dataimPulse) |
| Residential — Custom | 5 TB+ | Quote | Negotiated | Enterprise quote |
| Datacenter — Intro | 10 GB | $5 | $0.50/GB | One-time —  [buy the datacenter intro block](https://bit.ly/dataimPulse) |
| Datacenter — Standard | 100 GB | $50 | $0.50/GB | One-time —  [buy 100 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Datacenter — Volume | 1 TB | $450 | $0.45/GB | One-time —  [buy the 1 TB datacenter tier](https://bit.ly/dataimPulse) |
| Datacenter — Custom | 5 TB+ | From $2,250 | Negotiated | Enterprise quote |
| Mobile — Intro | 2.5 GB | $5 | $2.00/GB | One-time —  [buy the mobile intro block](https://bit.ly/dataimPulse) |
| Mobile — Standard | 25 GB | $50 | $2.00/GB | One-time —  [buy 25 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Mobile — Volume | 1 TB | $1,600 | $1.60/GB | One-time —  [buy the 1 TB mobile tier](https://bit.ly/dataimPulse) |
| Mobile — Custom | 5 TB+ | From $8,000 | Negotiated | Enterprise quote |
| Premium Residential — Intro | 1 GB | $5 | $5.00/GB | One-time —  [buy the premium residential intro block](https://bit.ly/dataimPulse) |
| Premium Residential — Standard | 10 GB | $50 | $5.00/GB | One-time —  [buy 10 GB of premium residential traffic](https://bit.ly/dataimPulse) |
| Premium Residential — Custom | 5 TB+ | From $20,000 | Negotiated | Enterprise quote |

The minimum first purchase is $5 across every proxy type. That is the whole entry cost, and it buys 5 GB of residential traffic, 10 GB of datacenter, or 2.5 GB of mobile, depending on which lane you start in.

Two conditions are worth flagging before you pay. There is no free trial with no payment required; testing starts at that $5 minimum. And the refund policy on Intro packages is a 7-day money-back window on card payments, conditional on having consumed less than 80% of the traffic, with crypto purchases excluded. Read the current terms yourself rather than taking a review's summary of them.

## Where DataImpulse makes sense and where it does not

The independent picture is fairly consistent. TechRadar's review of the service praises the $1/GB residential baseline, non-expiring traffic, the ethically sourced pool, and 24/7 human support, and lists AdsPower among the platforms it publishes integration guides for. The same review's main criticism is that DataImpulse offers no managed scraping API or built-in browser, and expects you to handle the toolkit yourself. For an AdsPower user, that criticism barely applies, since AdsPower is the toolkit.

Cost per profile is where the pay-as-you-go model shows up. A profile that logs in twice a day and browses for twenty minutes does not move much data, so residential traffic at $1/GB goes a long way. The teams that get burned are the ones running video-heavy workflows or leaving profiles open on media sites, where bandwidth disappears fast. For that pattern, mobile at $2/GB will hurt, and premium residential at $5/GB will hurt more.

If you want a quick decision rule:

- **Social, marketplace, and ad accounts:** residential, sticky session per profile, mobile if the account is high value.
- **Bulk testing of fingerprint plus proxy combinations:** datacenter at $0.50/GB, accepting that the IP class is weak.
- **Anything needing a permanently fixed residential IP:** a static ISP package, which is a different product category.

Also worth knowing: AdsPower sells its own proxies inside the app, and buying them there auto-fills the connection details, which removes a step and a class of typo. That option does not replace third-party providers. Custom proxy entry, saved proxies, and partner imports all still work, and teams commonly run both. The in-app route is faster to deploy; a third-party plan is usually cheaper per gigabyte at volume and gives you one traffic balance across every browser you use, not just AdsPower.

## Common failures and how to fix them

Most AdsPower proxy problems fall into a short list:

**"Check Proxy" fails immediately.** Wrong port for the protocol. HTTP and HTTPS use 823, SOCKS5 uses 824. Mixing those up produces a connection error that looks like an authentication problem.

**The check passes but the location is wrong.** Your country parameter did not apply, usually because of a missing underscore or a stray semicolon in the username string. The format is two underscores before `cr`, semicolons between parameters.

**The IP changes between sessions.** You left the session ID out, or you edited an existing `sessid` value. A new session ID means a new address, so treat it as permanent once an account is on it.

**Two accounts banned in the same week.** Almost always the same session ID on both profiles. Audit your usernames for duplicates before assuming the platform is at fault.

**Heavy targeted advertising after a browser session.** Some sites push back hard on city-level targeting on residential traffic, and the added charge for those filters can catch you off guard. Confirm the billing treatment before designing a plan around city precision.

DataImpulse publishes a 99.51% success rate on its network and staffs human support around the clock, which is relevant here because proxy troubleshooting is exactly the kind of problem that needs a person, not a help centre article. 👉 [Compare all DataImpulse proxy plans and rates](https://bit.ly/dataimPulse).

## Short answers to the usual questions

**Does AdsPower need a proxy at all?** It runs without one, but every profile will share your real IP, which defeats the purpose of separate fingerprints. AdsPower states plainly that a proxy does not guarantee account safety, and that is true of any provider. What it does is remove the most obvious link between profiles.

**Can one DataImpulse plan cover many profiles?** Yes. Traffic is pooled per account, so the same balance feeds every profile. Only the username string differs, which is what makes the CSV import approach workable.

**Do unused gigabytes expire?** No. Purchased traffic stays in your balance until it is consumed. That is the reason a small test block is low risk, and the main structural difference from subscription providers who reset your allowance monthly.

**Is $1/GB cheap or normal?** For residential traffic, it sits at the bottom of the published range. Reviews comparing 2026 rates put mid-market residential at $3 to $4 per GB and enterprise at $5 to $8. The number that actually determines your bill is cost per successful request, which depends on how often your targets block you, so measure it on your own websites before scaling up.

**Should the proxy location match the profile's timezone?** Yes. DataImpulse can put a profile on a US-New York address while the fingerprint says Asia/Singapore, and that mismatch is a signal. Align timezone, language, and geolocation with the exit IP rather than configuring them independently.

## The short version

The proxy is the part of an AdsPower setup that fails silently and costs you accounts when it does. Fix the part you control: one sticky IP per profile, one unique session ID each, targeting written into the username, and a residential or mobile network class rather than datacenter ranges. Verify with Check Proxy, verify again with curl before a bulk import, and start on a $5 traffic block so your test phase costs less than lunch.
