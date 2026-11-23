---
layout: post
title: "Advanced Search TikTok API in Server Mode: Multi-Keyword Search Over One HTTP Call"
date: 2026-11-24 07:00:00 +0700
permalink: /apify/advanced-search-tiktok-api-server-mode/
categories: [Apify, Developer Guides, TikTok]
tags: [Apify Actors, Server Actors, Apify Standby, TikTok Search, Python, API]
author: Novi Develop
description: "Advanced Search TikTok API now runs in server mode. Call GET or POST /search, get TikTok videos for up to 10 keywords back in the response, and handle meta.partial, charge limits and customMapFunction from Python and curl."
---

# Advanced Search TikTok API in Server Mode: Multi-Keyword Search Over One HTTP Call

In yesterday's post on [Server Actors vs run-sync](/apify/server-actors-vs-run-sync-tiktok-x-agent/) we said our TikTok scrapers would stay batch Actors for now. That didn't last long: three of them now also run in server mode, and this post covers the first, [Advanced Search TikTok API](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m). The batch run works exactly as before, so server mode is just a second way to call the same actor.

This guide covers the endpoints, a curl and Python client that handles the actor's edge cases, and a small keyword monitor you can drop into a backend or hand to an agent.

## What server mode means here

Apify calls this feature Standby, or Server Actor. The actor keeps an HTTP server running. You send a request to a URL and the TikTok videos come back as JSON in the response. You don't start a run, wait for it, or read a dataset afterwards.

The results only live in that HTTP response. Nothing is written to a dataset, so if you want history, you store it yourself.

Use server mode when a person or an agent is waiting for the answer: a dashboard refresh, an "is anyone posting about us today?" check, or a tool call in the middle of a chat. Use the normal batch run when the job is big, scheduled, or needs results you can re-read later. Apify gives a server-mode request about 5 minutes (300 seconds) to finish, so the actor caps how much one request can fetch. Anything past those caps belongs in a batch run.

The first request after the server has been idle can be slower, because Apify has to start a container. After that, the container stays warm for about 300 seconds of idle time.

## Endpoints and parameters

The server lives at:

```
https://novi--advanced-search-tiktok-api.apify.actor
```

`GET /` returns usage info: the endpoints and their limits. The full OpenAPI schema is on the actor's Server tab in Apify Console.

`GET /search` takes query parameters:

| Parameter | Notes |
|---|---|
| `keyword` | Repeatable: `keyword=nike&keyword=adidas` |
| `keywords` | Comma-separated alternative: `keywords=nike,adidas` |
| `limit` | Videos per keyword, 30 to 500, default 30 |
| `region` | 2-letter country code, default `GB`. IR, AQ and KP are not supported |
| `sortType` | `0` relevance (default), `1` most liked, `2` most recent |
| `publishTime` | `ALL_TIME` (default), `YESTERDAY`, `WEEK`, `MONTH`, `THREE_MONTH`, `SIX_MONTH` |
| `includeSearchTerms` | `true` adds `__search_terms` to each video |
| `fetchUserBio` | `true` adds `author.bio`. Slower, and charged extra as `user-bio-lookup` per unique author |

`POST /search` takes a JSON body with the same fields as the actor's normal input: `keywords` (array, max 10), `keyword`, `limit`, `region`, `sortType`, `publishTime`, `includeSearchTerms`, `fetchUserBio`, plus `customMapFunction`. That last one is POST only.

A few limits apply to every request. You can send at most 10 keywords and get at most 500 videos per keyword. The server fetches up to 4 keywords in parallel. `isUnlimited` from the batch input is not supported in server mode.

## The response

A successful call returns 200 with this shape:

```json
{
  "items": [ /* TikTok video objects */ ],
  "meta": {
    "keywords": [{ "keyword": "nike", "count": 100 }],
    "limitPerKeyword": 100,
    "partial": false,
    "timedOut": false,
    "chargeLimitReached": false
  }
}
```

`items` holds the same video objects a batch run would push to its dataset. They are raw TikTok "aweme" objects. The fields we use most are `aweme_id`, `desc`, `create_time`, `author.unique_id`, `author.nickname`, `statistics.play_count`, `statistics.digg_count`, `statistics.comment_count`, `statistics.share_count`, `video` (play addresses, including the no-watermark one) and `music`.

Two `meta` flags need their own handling in your code:

- The server sets itself an internal deadline of about 3.5 minutes. If it runs out of time, it returns whatever it has collected with `partial: true` and `timedOut: true`. You still get a 200.
- If you hit the max charge limit you set on Apify, you also get a 200 with partial results and `chargeLimitReached: true`.

Errors come back as `400 { "error": "..." }` for bad input (too many keywords, invalid JSON) and `500 { "error": "Failed to retrieve search list." }` when the search itself fails.

You pay per returned item (`result-item`), and only for items actually returned. Prices depend on your plan, so check the [Store page](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) for current numbers.

## Calling it with curl

Authenticate with your Apify token in the `Authorization` header. The server also accepts `?token=`, but a header is less likely to end up in logs. Keep the token in an environment variable and out of your repo.

```bash
export APIFY_TOKEN=...   # Console > API & Integrations

curl -s --max-time 300 \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://novi--advanced-search-tiktok-api.apify.actor/search?keyword=nike&keyword=adidas&limit=30&region=US&sortType=1&publishTime=WEEK"
```

That returns up to 30 of the most liked videos from the past week for each keyword, so up to 60 items.

## A Python client

This wrapper uses `POST /search` so it can pass `customMapFunction`. The function runs on the server against each video, so the response only carries the fields you need. It reshapes items. Don't use it to filter them out.

```python
import os
import requests

APIFY_TOKEN = os.environ["APIFY_TOKEN"]
SEARCH_URL = "https://novi--advanced-search-tiktok-api.apify.actor/search"

# Runs on the server for every video. Keep it a plain JS function string.
SLIM_VIDEO = """(o) => ({
  id: o.aweme_id,
  desc: o.desc,
  created: o.create_time,
  author: o.author?.unique_id,
  plays: o.statistics?.play_count,
  likes: o.statistics?.digg_count,
  comments: o.statistics?.comment_count,
  shares: o.statistics?.share_count,
})"""


class SearchError(Exception):
    pass


def search_tiktok(keywords, limit=30, region="US", sort_type=0,
                  publish_time="ALL_TIME", slim=True):
    if not 1 <= len(keywords) <= 10:
        raise ValueError("Server mode accepts 1 to 10 keywords per request")
    if not 30 <= limit <= 500:
        raise ValueError("limit must be between 30 and 500 per keyword")

    body = {
        "keywords": keywords,
        "limit": limit,
        "region": region,
        "sortType": sort_type,
        "publishTime": publish_time,
        "includeSearchTerms": True,
    }
    if slim:
        body["customMapFunction"] = SLIM_VIDEO

    resp = requests.post(
        SEARCH_URL,
        json=body,
        headers={"Authorization": f"Bearer {APIFY_TOKEN}"},
        timeout=300,
    )

    if resp.status_code in (400, 500):
        raise SearchError(resp.json().get("error", resp.text))
    resp.raise_for_status()

    data = resp.json()
    meta = data.get("meta", {})
    if meta.get("chargeLimitReached"):
        print("Warning: charge limit reached, results are incomplete")
    elif meta.get("partial"):
        print("Warning: server deadline hit, results are incomplete")
    return data.get("items", []), meta
```

Check `meta` on every call. A 200 does not mean you got everything you asked for. For a dashboard, show a "partial results" note to the user. For an agent tool, put the flag in the tool's output so the model doesn't report a count as complete.

If you get partial results with several keywords and a high `limit`, split the keywords across requests or lower the limit. Remember the server runs up to 4 keywords in parallel, so 10 keywords at 500 each is the request most likely to run out of time.

## Mini-project: a brand share-of-voice check

Here's a small script for competitor checks. It searches a few brand names for the past week, sorted by most liked, and prints how much attention each one got plus the top video for each. It asks for the full video objects (`slim=False`) because it needs `__search_terms` from each one.

```python
from collections import defaultdict

BRANDS = ["nike", "adidas", "puma", "new balance"]

items, meta = search_tiktok(BRANDS, limit=100, sort_type=1,
                           publish_time="WEEK", slim=False)

def plays(v):
    return (v.get("statistics") or {}).get("play_count") or 0

def likes(v):
    return (v.get("statistics") or {}).get("digg_count") or 0

by_brand = defaultdict(list)
for v in items:
    # __search_terms = {"keyword", "sort_type", "publish_time", "region"}
    keyword = (v.get("__search_terms") or {}).get("keyword")
    if keyword:
        by_brand[keyword].append(v)

print(f"{'brand':<14}{'videos':>8}{'total plays':>16}")
for brand in BRANDS:
    vids = by_brand.get(brand, [])
    print(f"{brand:<14}{len(vids):>8}{sum(plays(v) for v in vids):>16,}")

print()
for brand in BRANDS:
    vids = by_brand.get(brand, [])
    if not vids:
        continue
    top = max(vids, key=likes)
    handle = (top.get("author") or {}).get("unique_id")
    url = f"https://www.tiktok.com/@{handle}/video/{top.get('aweme_id')}"
    print(f"{brand}: {likes(top):,} likes  {url}")

if meta.get("partial"):
    print("\nNote: partial results, counts are lower bounds")
```

With `includeSearchTerms` on, each video comes back tagged with the keyword that found it, so you can group four brands from one response instead of making four calls. `meta.keywords` also gives you a per-keyword count if that's all you need.

The same function works as an agent tool. Give it a description like "Search recent TikTok videos for up to 10 keywords and return the top results with play and like counts", expose `keywords`, `publish_time` and `sort_type`, and keep `limit` low (30 is the minimum and usually plenty for a chat answer). Return `meta.partial` along with the items.

If you want to keep a history, write the slim items to your own database after each call. Server mode doesn't store anything for you.

## When to switch back to a batch run

Use the normal input run (it writes to a dataset as always) when:

- you need more than 500 videos for a keyword, or the `isUnlimited` option
- you have more than 10 keywords
- the job runs on a schedule and nobody is waiting on the response
- you want results stored on Apify for later reads

Yesterday's post shows the batch side with `run-sync-get-dataset-items` and the async fallback: [Server Actors vs run-sync: Wiring TikTok and X Scrapers into an AI Agent](/apify/server-actors-vs-run-sync-tiktok-x-agent/).

The actor, its pricing and the Server tab are on the [Advanced Search TikTok API Store page](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m).
