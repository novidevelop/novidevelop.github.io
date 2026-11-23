---
layout: post
title: "TikTok Profile Videos API in Server Mode: A Creator's Latest Posts in One Request"
date: 2026-11-24 07:20:00 +0700
permalink: /apify/tiktok-profile-videos-api-server-mode/
categories: [Apify, Developer Guides, TikTok]
tags: [Apify Actors, Server Actors, Apify Standby, TikTok Profile, Influencer Marketing, Python, API]
author: Novi Develop
description: "TikTok Profile Videos API (tiktok-user-api) now runs in server mode. Call GET /posts with a username, userId or profile URL and get the creator's latest videos as JSON. Python and curl examples, 404 and 402 handling, and a sponsored-post checker."
---

# TikTok Profile Videos API in Server Mode: A Creator's Latest Posts in One Request

[TikTok Profile Videos API](https://apify.com/novi/tiktok-user-api?fpr=7hce1m) (slug `tiktok-user-api`) returns the latest videos from a TikTok profile. It now has a server mode: send a username to one URL and the videos come back in the response.

That suits the lookups influencer and social tools make all day. Has this creator posted the sponsored video yet? What has @someone posted this week? Is this account active enough to work with? This guide shows the endpoint, a Python client that handles the actor's errors, and a small sponsored-post checker.

## What server mode is

Server mode is Apify's Standby feature, also called Server Actor. The actor keeps an HTTP server running. You send a GET request and the videos come back as JSON in the response body. You don't start a run or poll a dataset.

The data only exists in that response. Nothing is saved to a dataset, so store what you need on your side.

Apify limits a server-mode request to about 5 minutes (300 seconds), so this actor returns at most 100 videos per call. If you're scraping many profiles or need more than 100 videos from one, use the normal input run. It still writes to a dataset and works as it always did. We covered calling batch actors from code in [Server Actors vs run-sync](/apify/server-actors-vs-run-sync-tiktok-x-agent/).

The first request after the server has been idle can be slow because Apify has to start a container. It then stays warm for about 300 seconds of idle time.

## The endpoint

Server URL:

```
https://novi--tiktok-user-api.apify.actor
```

There's one endpoint, `GET /posts`. The OpenAPI schema is on the actor's Server tab in Apify Console.

Pass exactly one way to identify the creator:

- `username`, for example `nike`
- `userId`, for example `6714563164218679553`
- `url`, a profile URL such as `https://www.tiktok.com/@nike`

And optionally `limit`, from 2 to 100, default 20. You get the creator's latest videos.

A 200 response is a JSON array of video objects, the same TikTok "aweme" objects a batch run writes to its dataset. Useful fields include `aweme_id`, `desc`, `create_time`, `author.unique_id`, `author.nickname`, `statistics.play_count`, `statistics.digg_count`, `statistics.comment_count`, `statistics.share_count`, `video` and `music`.

Errors:

| Status | Body | Meaning |
|---|---|---|
| 400 | `{ "error": ... }` or `{ "errors": [...] }` | Bad parameters, e.g. none or several of username/userId/url, or `limit` out of range |
| 402 | | Your max charge limit is reached |
| 404 | `{ "error": "User not found or could not be resolved." }` | No such creator, or the profile couldn't be resolved |
| 502 | | Upstream fetch failed |

Billing is per returned video (`result-item`). Pricing varies by plan, so check the [Store page](https://apify.com/novi/tiktok-user-api?fpr=7hce1m).

## curl

Authenticate with your Apify token in the `Authorization` header. The server also takes `?token=`, but query strings tend to end up in logs. Keep the token in an environment variable and out of git.

```bash
export APIFY_TOKEN=...   # Console > API & Integrations
BASE=https://novi--tiktok-user-api.apify.actor

# By username
curl -s --max-time 300 -H "Authorization: Bearer $APIFY_TOKEN" \
  "$BASE/posts?username=nike&limit=20"

# By user ID
curl -s --max-time 300 -H "Authorization: Bearer $APIFY_TOKEN" \
  "$BASE/posts?userId=6714563164218679553&limit=10"

# By profile URL (encoded)
curl -s --max-time 300 -G "$BASE/posts" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  --data-urlencode "url=https://www.tiktok.com/@nike"
```

If you store creators in a database, keep their `userId`. A creator can change their username, but the ID stays the same.

## A Python client

```python
import os
import requests

APIFY_TOKEN = os.environ["APIFY_TOKEN"]
POSTS_URL = "https://novi--tiktok-user-api.apify.actor/posts"


class CreatorNotFound(Exception):
    pass


class ChargeLimitReached(Exception):
    pass


def latest_posts(username=None, user_id=None, url=None, limit=20):
    given = {k: v for k, v in
             {"username": username, "userId": user_id, "url": url}.items() if v}
    if len(given) != 1:
        raise ValueError("Pass exactly one of username, user_id or url")
    if not 2 <= limit <= 100:
        raise ValueError("limit must be between 2 and 100")

    resp = requests.get(
        POSTS_URL,
        params={**given, "limit": limit},  # requests URL-encodes the profile URL
        headers={"Authorization": f"Bearer {APIFY_TOKEN}"},
        timeout=300,
    )

    if resp.status_code == 404:
        raise CreatorNotFound(next(iter(given.values())))
    if resp.status_code == 402:
        raise ChargeLimitReached("Raise the max charge limit in Apify to continue")
    if resp.status_code == 400:
        body = resp.json()
        raise ValueError(body.get("error") or body.get("errors"))
    resp.raise_for_status()  # 502 and anything unexpected

    return resp.json()
```

What each failure means for your app:

- 404 is a normal answer. In an influencer tool it usually means a typo or a deleted or renamed account. Mark the creator as "not found" and don't retry in a loop.
- 402 means your own spending cap stopped the call. Stop the batch you're in, since every further call will fail the same way.
- 502 is worth one retry after a short pause.
- A client timeout usually lines up with a cold start plus a large `limit`. Retry with a smaller limit.

## Mini-project: did the creator post the sponsored video?

A common job in influencer campaigns: you've paid creators to post with a campaign hashtag by a deadline, and someone checks each profile by hand. This script does the check from a list.

```python
import time
from datetime import datetime, timezone

CAMPAIGN_TAG = "#summerrun"
CAMPAIGN_START = datetime(2026, 11, 15, tzinfo=timezone.utc)

creators = ["creator_one", "creator_two", "creator_three"]


def find_campaign_post(videos):
    for v in videos:
        created = datetime.fromtimestamp(v.get("create_time") or 0, tz=timezone.utc)
        if created < CAMPAIGN_START:
            continue
        if CAMPAIGN_TAG.lower() in (v.get("desc") or "").lower():
            return v, created
    return None, None


report = []
for handle in creators:
    try:
        videos = latest_posts(username=handle, limit=30)
    except CreatorNotFound:
        report.append((handle, "NOT FOUND", ""))
        continue
    except ChargeLimitReached:
        print("Charge limit reached, stopping early")
        break
    except requests.HTTPError:
        time.sleep(5)
        try:
            videos = latest_posts(username=handle, limit=30)
        except Exception as e:
            report.append((handle, "ERROR", str(e)))
            continue

    post, created = find_campaign_post(videos)
    if post:
        stats = post.get("statistics") or {}
        url = f"https://www.tiktok.com/@{handle}/video/{post.get('aweme_id')}"
        report.append((handle, f"POSTED {created:%Y-%m-%d}",
                       f"{stats.get('play_count') or 0:,} plays  {url}"))
    else:
        report.append((handle, "MISSING", f"checked latest {len(videos)} videos"))

for handle, status, detail in report:
    print(f"{handle:<18}{status:<20}{detail}")
```

A few things to watch:

- `limit=30` covers creators who post several times a day. If a creator posts more often than that, a sponsored video from early in the campaign could fall outside the latest 30. Raise the limit for them (up to 100).
- Hashtags in `desc` are plain text, so a case-insensitive substring check works. If you also need to match a sound, look at the `music` object.
- Each video returned is charged, so don't use `limit=100` for every creator by default. Start small and raise it only where needed.
- Run it from a scheduled job and save the report rows yourself. Server mode keeps no history.

The same `latest_posts` function also works as an agent tool: "Get the latest TikTok videos posted by a creator, by username or profile URL". Pass back the creator's handle, each video's date, caption and play count, and turn a 404 into a short "creator not found" message the model can repeat to the user.

## When to use the batch run

Server mode is built for one creator per request, up to 100 videos. Use the actor's normal input run when you have a long list of profiles, need more than 100 videos from a creator, or want results stored in a dataset. The batch side and its sync and async options are covered in [Server Actors vs run-sync: Wiring TikTok and X Scrapers into an AI Agent](/apify/server-actors-vs-run-sync-tiktok-x-agent/).

Pricing, the input schema and the Server tab are on the [TikTok Profile Videos API Store page](https://apify.com/novi/tiktok-user-api?fpr=7hce1m).
