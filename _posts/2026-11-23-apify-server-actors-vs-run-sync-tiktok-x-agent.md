---
layout: post
title: "Server Actors vs run-sync: Wiring TikTok and X Scrapers into an AI Agent"
date: 2026-11-23 07:00:00 +0700
permalink: /apify/server-actors-vs-run-sync-tiktok-x-agent/
categories: [Apify, AI Agents, Developer Guides]
tags: [Apify Actors, Server Actors, TikTok Scraper, Twitter X Scraper, MCP, API Tokens]
author: Novi Develop
description: "Apify Server Actors are now GA. Here is when an agent should call a Server Actor, when it should use run-sync-get-dataset-items, and how to handle the 300-second limit for TikTok and X scrapers, with curl and Python examples."
---

# Server Actors vs run-sync: Wiring TikTok and X Scrapers into an AI Agent

Apify made Server Actors generally available this month, announced at its first user conference, Run. The pitch, per [VMblog's coverage](https://vmblog.com/news/apify-unveils-new-ai-agent-capabilities-and-server-actors-to-support-growing-demand/), is "a new way to deploy always-ready services" for "AI agent tool calls, real-time applications, and high-frequency data lookups… without starting a new execution for each interaction."

We maintain a handful of TikTok and X scrapers on the Store, and the question we keep getting is whether those should be Server Actors now. Our answer for most of them is no. This post explains why, shows how to call both kinds of Actor from code, and covers the token cleanup you should do after Apify's November incident.

## What changed

If you used Actor Standby mode, you already know Server Actors. Per [Apify's announcement post](https://blog.apify.com/introducing-server-actors/), it's the same feature under a new name. The difference from a regular Actor is where results go. A regular Actor writes to a dataset, which suits batch work like the post's example of scraping "1,000 Facebook pages overnight." A Server Actor returns results in the HTTP response and stores nothing.

A few details from the [Server Actors docs](https://docs.apify.com/platform/actors/running/server-actors) matter once you start wiring things up:

- Each Server Actor gets a URL of the form `https://<username>--<actor-name>.apify.actor`.
- You authenticate with `Authorization: Bearer <token>`. If the Actor uses the `Authorization` header for its own purposes, send `X-Apify-Authorization` instead. The platform strips the token before the request reaches the Actor.
- The first response has to come back within 5 minutes.
- If no run can pick up your request within 2 minutes, you get HTTP 429. Your account's rate limit is in the `X-RateLimit-Limit` response header.

Pricing comes in two flavours. Multi-tenant Server Actors are pay-per-event only and don't count against your memory or concurrency limits. Single-tenant ones bill platform usage, idle time included, plus events. The Store now has "Multi Tenant Server" and "Single Tenant Server" categories so you can tell them apart.

## Calling a Server Actor

[apify/web-fetch](https://apify.com/apify/web-fetch?fpr=7hce1m) is a good one to try first. It runs as a multi-tenant server, returns a page as markdown, and costs $0.0015 per fetch at the time of writing.

With curl:

```bash
export APIFY_TOKEN=...   # from Console > API & Integrations, never hard-code it

curl -s \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://apify--web-fetch.apify.actor/?url=https://example.com"
```

The path and query parameters depend on the Actor. Check its README or API tab for the exact shape; the host and the Bearer header are the parts that stay the same.

The same call in Python:

```python
import os
import requests

APIFY_TOKEN = os.environ["APIFY_TOKEN"]

def fetch_page(url: str) -> str:
    resp = requests.get(
        "https://apify--web-fetch.apify.actor/",
        params={"url": url},
        headers={"Authorization": f"Bearer {APIFY_TOKEN}"},
        timeout=330,  # server gives itself up to 5 minutes for the first response
    )
    if resp.status_code == 429:
        raise RuntimeError("No Server Actor run free within 2 minutes, back off and retry")
    resp.raise_for_status()
    return resp.text
```

There's no run ID to track and no dataset to read afterwards, which suits an agent tool that grabs one page in the middle of a conversation.

## Calling a batch Actor with run-sync

Our TikTok and X scrapers are regular Actors, so they write to a dataset. The shortest way to get results back in a single request is the [`run-sync-get-dataset-items`](https://docs.apify.com/api/v2/act-run-sync-get-dataset-items-post) endpoint. It starts a run, waits for it, and returns the dataset items in the response body. The Actor ID in the URL uses `username~actor-name`.

Here's [novi/advanced-search-tiktok-api](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m), our TikTok keyword search:

```python
import os
import requests

APIFY_TOKEN = os.environ["APIFY_TOKEN"]
ACTOR = "novi~advanced-search-tiktok-api"
SYNC_URL = f"https://api.apify.com/v2/acts/{ACTOR}/run-sync-get-dataset-items"

run_input = {
    "keywords": ["matcha latte"],
    "limit": 30,           # minimum is 30
    "region": "US",
    "sortType": 0,
    "publishTime": "YESTERDAY",
    "fetchUserBio": False,
}

resp = requests.post(
    SYNC_URL,
    json=run_input,
    headers={"Authorization": f"Bearer {APIFY_TOKEN}"},
    timeout=320,
)
```

The endpoint waits at most 300 seconds, and the docs say: "If the Actor run exceeds 300 seconds, the HTTP response will return the 408 status code." A small search with `limit: 30` usually fits. Ten keywords with a big limit and `fetchUserBio` turned on might not.

### Falling back to an async run

When you get a 408, switch to the async flow: start a run, wait for it to finish, then read the dataset. The [apify-client](https://docs.apify.com/api/client/python/) Python library handles most of this:

```python
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

def search_tiktok(run_input: dict) -> list[dict]:
    resp = requests.post(
        SYNC_URL,
        json=run_input,
        headers={"Authorization": f"Bearer {os.environ['APIFY_TOKEN']}"},
        timeout=320,
    )
    if resp.status_code != 408:
        resp.raise_for_status()
        return resp.json()

    # Too slow for run-sync. Start an async run and poll it.
    run = client.actor("novi/advanced-search-tiktok-api").start(run_input=run_input)
    run = client.run(run["id"]).wait_for_finish(wait_secs=1800)
    if run["status"] != "SUCCEEDED":
        raise RuntimeError(f"Run {run['id']} ended with {run['status']}")

    return list(client.dataset(run["defaultDatasetId"]).iterate_items())
```

A 408 means the HTTP request gave up. Nothing in it says the first run was cancelled, so you could end up paying for two runs. If you already know a job is big, skip run-sync and call `start()` directly. We use run-sync for interactive lookups and go straight to async for anything we'd schedule.

The same pattern works for [xtdata/twitter-x-scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m). Swap the Actor ID and pass its input (`searchTerms`, `twitterHandles`, `maxItems`, `sort`, `tweetLanguage`, `start`/`end`). Keep `maxItems` small on the sync path.

## Handing these tools to an agent

You can wrap the functions above as tools in whatever agent framework you use. The other route is [Apify's MCP server](https://docs.apify.com/platform/integrations/mcp), which exposes Actors as tools an MCP client can call directly. The Apify plugin in Claude's directory was "among the 50 most-used plugins as of October 31, 2026, with more than 57,000 lifetime installs and over 12,000 active accounts," according to the VMblog piece, so a lot of people are already doing this.

Server Actor authors can also expose an MCP endpoint through `webServerMcpPath` and an OpenAPI schema through `webServerSchema`. [apify/web-search](https://apify.com/apify/web-search?fpr=7hce1m) and [apify/rag-web-browser](https://apify.com/apify/rag-web-browser?fpr=7hce1m) both run multi-tenant with MCP support. For our scrapers, an agent going through MCP still gets a batch run underneath, and it should tell the user it's waiting on a run.

## Rotate your tokens

Apify had a security incident this month. Per [BleepingComputer](https://www.bleepingcomputer.com/news/security/apify-rotates-api-tokens-prompts-2fa-resets-after-breach/), there was unauthorized access on November 9 and 10, detected the next day at 12:13 UTC. Apify rotated API tokens and Apify Proxy passwords, reset sessions, revoked GitHub OAuth connections, and asked users to reset 2FA. The report puts the number of customers at around 80,000.

Whether any customer data was accessed is still unclear. The initial report said there was no sign platform data was affected; later secondary coverage claims some was. Reports differ and there's no post-mortem yet, so keep an eye on [status.apify.com](https://status.apify.com).

Your old token is already dead, so every integration needs a new one anyway. Use the chance to stop sharing one account-wide token across everything. [Scoped tokens](https://blog.apify.com/scoped-api-tokens/) have been around since September 2024. In Console, go to API & Integrations and choose "Limit token permissions." You can scope a token to the account or to specific resources: Run and Build permissions on one Actor, or Read on a single dataset or key-value store. Server Actors accept scoped tokens too.

What we do now: one token per integration. The agent that searches TikTok gets a token that can run `novi/advanced-search-tiktok-api` and read its datasets. The web-fetch tool gets its own. If one leaks, you revoke it and nothing else breaks.

## Which one to use

| | Server Actor | run-sync-get-dataset-items | Async run + dataset |
|---|---|---|---|
| Latency | Lowest, no new run per request | One run start, then up to 300 s | Minutes to hours, you poll |
| Hard limit | First response within 5 min; 429 if no run is free within 2 min | 408 after 300 s | None at the HTTP layer |
| Storage | Nothing stored, result is in the response | Dataset is created and returned | Dataset kept for later reads |
| Cost model | Pay-per-event (multi-tenant) or usage incl. idle time plus events (single-tenant) | The Actor's normal pricing | The Actor's normal pricing |
| Good for | One-off lookups, agent tool calls | Small searches an agent can wait on | Bulk scraping, scheduled jobs |

For reference, at the time of writing on the FREE tier: `advanced-search-tiktok-api` charges $0.0004 per result, `twitter-x-scraper` $0.005 per tweet (dropping to $0.00025 on GOLD and above), web-fetch $0.0015 per fetch, and web-search $0.005 per result.

Our rule: if the agent needs one answer about one thing right now, use a Server Actor. If it needs a list of TikTok videos or tweets, use a batch Actor, try run-sync for small jobs, and fall back to async when you hit 408. That's why our TikTok and X scrapers stay batch Actors for now. What they produce is lists that people store and query again later.
