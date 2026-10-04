# 9proxy proxifier: Route Any Desktop App Through 9Proxy Residential IPs Without Built-In Proxy Support

Some programs just refuse to talk to a proxy. A scraper, a trading terminal, an emulator, a game client, a desktop bot — you paste an IP and port into its settings, and it either ignores them or only understands the Windows system proxy, which is all-or-nothing. Proxifier exists exactly for that gap: it intercepts traffic at the application layer, so anything with a network connection can be pushed through a SOCKS5 or HTTPS proxy.

Pairing it with 9Proxy is a common setup because Proxifier handles the routing and 9Proxy supplies residential IPs that don't get insta-blocked. The catch is that the two tools don't know anything about each other, and most of the frustration people hit comes from three things: using the wrong 9Proxy proxy type for what Proxifier expects, mangling the structured username, and leaving a VPN or system proxy running underneath. Here's how the whole thing actually fits together.

## What Proxifier actually does (and what it doesn't)

Proxifier sits between your applications and the network. It reads each connection attempt, matches it against a rule list, and sends matching traffic to a proxy server you defined — even if the app has no proxy setting anywhere in its interface. Rules work per executable: `chrome.exe` through Proxy A, `bot.exe` through Proxy B, everything else direct.

Two limits worth knowing before you start:

- **No built-in IP pool.** Proxifier needs an upstream proxy. It doesn't provide IPs, only routing.
- **UDP support is limited.** Most proxy setups through Proxifier are TCP-based. Apps that lean on QUIC or raw UDP (some game clients, WebRTC-heavy tools) need extra work or won't route cleanly.

That's the division of labour: Proxifier is the plumbing, 9Proxy is the water.

## Why residential IPs instead of datacenter ones

Proxifier will happily push traffic through a cheap datacenter proxy, and half the sites you hit will still flag it. Datacenter ranges are published, known, and easy to block in bulk. Residential IPs come from real consumer connections, which is why Proxifier plus residential proxies is the standard combination for account-based tasks, ad verification, and anything else where a "this looks like a server" flag costs you the session.

9Proxy's pool covers 20M+ residential IPs across 90+ countries, with targeting down to country, state, city, ZIP code, and ISP. It supports both HTTP/HTTPS and SOCKS5 — the two protocols Proxifier asks you to choose between.

## First decision: which 9Proxy product works with Proxifier

This is where most people get stuck, because 9Proxy sells two different residential products with different plumbing.

|  | Residential Proxy by IPs | Residential Proxy by GB |
| --- | --- | --- |
| Billing | Per IP, unlimited bandwidth while active | Per GB of traffic |
| How you connect | 9Proxy desktop app, IP forwarded to a local port | Straight from the dashboard: host, port, username, password |
| Endpoint format | `localhost:port` (optionally with proxy authentication) | Host + structured username + password |
| IP lifetime | A few hours up to ~24h | Rotates per request or per sticky session |
| Unused balance | Never expires | 180 days, unlimited on Enterprise |
| Good Proxifier fit | Yes — point Proxifier at the forwarded local port | Yes — the option 9Proxy's own Proxifier guide uses |

The short version: **GB-based is the smoother pairing**, because the dashboard hands you a host, port, username and password that drop straight into Proxifier's proxy server fields. IP-based works too, but you have to run the desktop app, forward an IP to a port, and then treat `127.0.0.1:port` as the proxy. That's an extra moving part: every time you swap IPs in the app, the endpoint stays the same but the exit IP changes underneath.

Both are available from the same account. 👉 [👉 Start with a 9Proxy account and see both product lines](https://bit.ly/9-Proxy)

## The Proxifier setup, step by step

This follows 9Proxy's own integration documentation. Do it in this order.

**1. Install Proxifier and clear the path.** Proxifier offers a 31-day free trial, and there's a portable build for Windows. Before anything else, turn off any active VPN, system-level proxy, or other app-level proxy tool. Two layers of interception fighting each other produces connection failures that look like the proxy is broken.

**2. Add the proxy server.** In Proxifier go to `Profile > Proxy Servers`, hit **Add**, and fill in:

| Field | What goes here |
| --- | --- |
| Host | Your 9Proxy host (the docs use an IP such as `38.180.149.107`) |
| Port | Your 9Proxy port (for example `17521`) |
| Type | `HTTPS` or `SOCKS5` — either works, SOCKS5 is the usual pick |

**3. Enable authentication.** Tick **Enable Authentication**. The username is *not* just your account name — it's a structured string that carries your targeting settings. The password is your 9Proxy sub-user password.

**4. Create a proxification rule.** Go to `Profile > Proxification Rules`, click **Add**, browse to the `.exe` you want routed, and set **Action** to the proxy you just created. You can reorder rules to control priority — Proxifier evaluates them top-down, so put the specific app rules above any catch-all direct rule.

That's the whole integration. No plugin, no API key exchange.

## The username string, decoded

This is the part that breaks setups. 9Proxy embeds all targeting inside the username rather than in separate fields, which is unusual if you're used to other providers. The general format:


<sub-user>-country-<country_code>-st-<state_code>-city-<city>-isp-<isp_code>-ssid-<session_id>-sst-<session_time>


| Segment | Example | What it does |
| --- | --- | --- |
| sub-user | `useruser123` | Your sub-account name from the dashboard — always comes first |
| country | `country-US` | Two-letter country code for the exit IP |
| st | `st-ohio` | Optional state/region filter |
| city | `city-newyork` | Optional city targeting (use underscores for spaces) |
| isp | `isp-as22773_Cox_Communications_Inc.` | Optional ISP/ASN filter |
| ssid | `ssid-device1` | Session ID — lets you hold multiple sticky IPs from one config |
| sst | `sst-15` | Sticky duration in minutes before the IP rotates |

A working example from the docs: `subaccount-country-us-sst-15-ssid-device1`, paired with your sub-user password.

**Rotating mode:** omit `sst` and `ssid` entirely. Every request gets a fresh IP. Good for scraping and price checks.

**Sticky mode:** include `sst-15` to hold one IP for 15 minutes. Add `ssid-bot01`, `ssid-bot02` etc. to run several sticky sessions in parallel from the same base config — each unique `ssid` gets its own IP.

For Proxifier specifically, sticky is almost always what you want, because Proxifier routes whole applications, and applications expect a continuous connection. A rotating exit IP in the middle of a login flow is a guaranteed logout. One practical note from the docs: don't stack every filter at once. Adding state *and* city *and* ISP narrows the available pool hard. Country-only targeting responds fastest.

## Routing one app while everything else stays direct

The rule list is where Proxifier earns its keep. A structure that works well:

1. Rule for `yourapp.exe` → 9Proxy proxy, sticky session (`sst-15` with a fixed `ssid`)
2. Rule for your browser if it needs a different region → 9Proxy proxy, different `ssid` and `country`
3. Default rule → Direct

That third line matters. Without a default direct rule, Proxifier can end up trying to route its own update checks and your system traffic through the proxy, which is noise you don't need.

If you're running IP-based packages instead, the equivalent is: open the 9Proxy app, filter by country/state/city/ZIP/ISP, right-click the IP you want, choose port forwarding, and use the resulting `localhost:port` in Proxifier's Host and Port fields. The app also offers proxy authentication in `username:password:localhost:port` form if you'd rather not leave the local port open.

## When Proxifier + 9Proxy misbehaves

| Symptom | Likely cause |
| --- | --- |
| All traffic fails immediately | VPN, system proxy, or another proxy tool still active |
| Authentication errors | Username isn't in structured format, or you used account password instead of the sub-user password |
| Login sessions keep dropping | Rotating mode in use — add `sst` and a fixed `ssid` |
| Country targeting isn't respected | Typo in the country/state/city segment, or too many filters stacked |
| One app works, another bypasses the proxy | That app has no rule, or its rule sits below the direct rule |
| Slow responses | Over-filtered targeting (state + city + ISP) shrinking the pool |

## Full 9Proxy package list and current pricing

9Proxy runs on a balance model rather than monthly subscriptions — you buy a package, and it sits in your account until used. One thing to flag: the company raised prices on IP-based and bundle packages on June 1, 2026, the first increase in its history. GB-based pricing was left untouched. Plenty of third-party reviews still quote the older IP numbers, so if a page says 100 IPs cost $20, it's out of date.

| Type | Package | Total price | Effective rate | Validity | Get it |
| --- | --- | --- | --- | --- | --- |
| IP-based | 100 IPs | $24 | $0.24/IP | IPs never expire | [ Buy 100 IPs](https://bit.ly/9-Proxy) |
| IP-based | 500 IPs | $72 | $0.144/IP | IPs never expire | [ Buy 500 IPs](https://bit.ly/9-Proxy) |
| IP-based | 1,000 IPs + 500 bonus | $126 | $0.084/IP | IPs never expire | [ Buy 1,500 IPs](https://bit.ly/9-Proxy) |
| IP-based | 2,500 IPs | $210 | $0.084/IP | IPs never expire | [ Buy 2,500 IPs](https://bit.ly/9-Proxy) |
| IP-based | 5,000 IPs | $360 | $0.072/IP | IPs never expire | [ Buy 5,000 IPs](https://bit.ly/9-Proxy) |
| IP-based | 15,000 IPs | $720 | $0.048/IP | IPs never expire | [ Buy 15,000 IPs](https://bit.ly/9-Proxy) |
| IP-based | 25,000 IPs | $863 | $0.035/IP | IPs never expire | [ Buy 25,000 IPs](https://bit.ly/9-Proxy) |
| IP-based | 50,000 IPs | $1,438 | $0.029/IP | IPs never expire | [ Buy 50,000 IPs](https://bit.ly/9-Proxy) |
| Business IP | 100,000 IPs | $2,300 | $0.023/IP | IPs never expire | [ Buy 100,000 IPs](https://bit.ly/9-Proxy) |
| Business IP | 200,000 IPs | $4,140 | $0.021/IP | IPs never expire | [ Buy 200,000 IPs](https://bit.ly/9-Proxy) |
| Business IP | 500,000 IPs | $8,625 | $0.018/IP | IPs never expire | [ Buy 500,000 IPs](https://bit.ly/9-Proxy) |
| GB-based | 5 GB | $15 | $3.00/GB | 180 days | [ Buy 5 GB](https://bit.ly/9-Proxy) |
| GB-based | 50 GB + 5 bonus | $105 | $2.10/GB | 180 days | [ Buy 55 GB](https://bit.ly/9-Proxy) |
| GB-based | 100 GB | $150 | $1.50/GB | 180 days | [ Buy 100 GB](https://bit.ly/9-Proxy) |
| GB-based | 200 GB | $200 | $1.00/GB | 180 days | [ Buy 200 GB](https://bit.ly/9-Proxy) |
| GB-based | 1,000 GB | $800 | $0.80/GB | 180 days | [ Buy 1,000 GB](https://bit.ly/9-Proxy) |
| GB-based | 2,000 GB | $1,500 | $0.75/GB | 180 days | [ Buy 2,000 GB](https://bit.ly/9-Proxy) |
| Enterprise GB | 3,000 GB | $2,160 | $0.72/GB | Never expires | [ Buy 3,000 GB](https://bit.ly/9-Proxy) |
| Enterprise GB | 6,000 GB | $4,200 | $0.70/GB | Never expires | [ Buy 6,000 GB](https://bit.ly/9-Proxy) |
| Enterprise GB | 10,000 GB | $6,800 | $0.68/GB | Never expires | [ Buy 10,000 GB](https://bit.ly/9-Proxy) |
| Bundle | 100 IPs + 5 GB | $30 | Starter | 180 days on traffic | [ Buy the Starter bundle](https://bit.ly/9-Proxy) |
| Bundle | 1,500 IPs + 50 GB | $180 | Popular | 180 days on traffic | [ Buy the Popular bundle](https://bit.ly/9-Proxy) |
| Bundle | 5,000 IPs + 500 GB | $720 | Pro | 180 days on traffic | [ Buy the Pro bundle](https://bit.ly/9-Proxy) |

Payment methods include credit and bank cards, crypto (USDT, BTC, ETH, LTC, DOGE and others), Alipay, Apple Pay, Google Pay, and 9Proxy's own wallet balance.

## Which package to buy if you're mainly using Proxifier

Proxifier routes whole applications with continuous sessions, and that shapes the answer.

**If you're running a handful of apps with a lot of traffic each**, IP-based is the cheaper structure at small scale because bandwidth is unlimited. 100 IPs for $24 gets you going, and unused IPs don't rot — a real advantage over subscription plans if your workload is lumpy.

**If you're doing broad geographic testing or high-rotation work**, GB-based is better. The 5 GB pack at $15 is the cheapest way into the platform at all, and it works from the dashboard with no desktop app in the loop — which is the exact shape Proxifier expects.

**If you need both**, the bundles price out well. The Starter bundle at $30 gives you 100 IPs and 5 GB for $6 more than the IP pack alone, which is hard to argue with if you're unsure which model suits your workload.

One caveat worth stating plainly: the 180-day clock on GB traffic is real, and Enterprise is the only tier where it disappears. If your usage is sporadic, don't buy 2,000 GB "to save money per GB" and then watch it expire unused.

## Verifying that it actually worked

Don't trust a green light in the app. Open the routed application, hit an IP-checking service from inside it, and confirm three things: the IP is not yours, the location matches what you put in the username string, and the connection type reads as residential rather than hosting.

If you want the exit IP to hold steady, re-check it after ten minutes. A residential IP going dark mid-session is normal behaviour, not a bug — which is why 9Proxy's IP packages include Auto Refresh to swap dead IPs for fresh ones, and Auto Rotation if you'd rather rotate on a schedule.

## Quick answers

**Does 9Proxy need the desktop app for Proxifier?** Only for IP-based packages. GB-based proxies are created in the dashboard and pasted straight into Proxifier, no app required.

**HTTPS or SOCKS5?** Either works with 9Proxy. SOCKS5 is the common default for app-level routing.

**Can I run different apps through different countries?** Yes. Give each one its own rule in Proxifier and its own `ssid` in the username string.

**Is there a free trial?** 9Proxy has offered limited trials to new users depending on availability, historically arranged through support rather than a self-serve button. Proxifier's own trial runs 31 days, which is enough to confirm the routing side works before you spend anything.

**Does an invite link help?** 9Proxy runs a lifetime affiliate program that advertises a 5% discount for referred users, so signing up through a referral link is the cheaper entry point if one is available to you. 👉 [👉 Create your 9Proxy account through this invite link](https://bit.ly/9-Proxy)

The combination itself is unglamorous but effective: Proxifier decides *which* traffic goes through a proxy, 9Proxy decides *what* that proxy looks like to the world. Get the username string right, keep one VPN out of the way, and use sticky sessions for anything that involves staying logged in — that covers about 90% of the problems people run into with this setup.
