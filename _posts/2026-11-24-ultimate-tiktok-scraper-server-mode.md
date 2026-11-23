---
layout: post
title: "Ultimate TikTok Scraper in Server Mode: One GET Endpoint for Any TikTok URL"
date: 2026-11-24 07:10:00 +0700
permalink: /apify/ultimate-tiktok-scraper-server-mode/
categories: [Apify, Developer Guides, TikTok]
tags: [Apify Actors, Server Actors, Apify Standby, TikTok Scraper, Node.js, API]
author: Novi Develop
description: "Ultimate TikTok Scraper now has a server mode. Send any TikTok video, profile, hashtag, music or search URL (or a keyword) to GET /videos and get JSON back. Node.js and curl examples, error handling, and a paste-a-link lookup service."
---

# Ultimate TikTok Scraper in Server Mode: One GET Endpoint for Any TikTok URL

[Ultimate TikTok Scraper](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m) takes almost any TikTok link and figures out what it points to: a single video, a profile, a hashtag, a sound, a search page, or the trending feed. It now runs in server mode too, which makes it a good backend for anything where a user pastes a link and expects data back right away.

This post shows the endpoint, a Node.js client that handles each error the server returns, and a small lookup service you can put behind a Slack command, a browser extension, or an agent tool.

## Server mode in plain words

Server mode is Apify's Standby feature, also called Server Actor. The actor keeps an HTTP server running. You call a URL and get the scraped videos back as JSON in the response body. There's no run to start and no dataset to read later.

Because results only come back in the response, nothing is stored on Apify. If you need the data again tomorrow, save it on your side.

Apify gives each server-mode request about 5 minutes (300 seconds) to finish, so this actor caps one request at 100 items. For bigger jobs, many URLs at once, or `customMapFunction`, use the normal input run. That path still writes to a dataset and hasn't changed. Our [previous post on Server Actors vs run-sync](/apify/server-actors-vs-run-sync-tiktok-x-agent/) covers calling the batch side from code.

The first request after a quiet period may take longer while Apify starts a container. The container then stays warm for about 300 seconds of idle time.

## The endpoint

Server URL:

```
https://novi--tiktok-scraper-ultimate.apify.actor
```

There is one endpoint, `GET /videos`. Its OpenAPI schema is on the actor's Server tab in Apify Console.

Send exactly one of `url` or `keyword`:

- `url`: any tiktok.com URL the actor supports. That covers videos (`https://www.tiktok.com/@billieeilish/video/7050551461734042926`), profiles (`https://www.tiktok.com/@gordonramsayofficial`), hashtags (`https://www.tiktok.com/tag/duet`), music, search pages, and the home/trending feed. Short links work too.
- `keyword`: a search term, such as `podcast`.

Optional parameters:

| Parameter | Values | Default |
|---|---|---|
| `limit` | 1 to 100 | 20 |
| `location` | 2-letter country code. IR, AQ, KP and TD are not supported | `US` |
| `sortType` | `DEFAULT`, `RELEVANCE`, `MOST_LIKED`, `MOST_RECENT` (search and hashtag only) | |
| `dateRange` | `DEFAULT`, `ALL_TIME`, `YESTERDAY`, `WEEK`, `MONTH`, `THREE_MONTH`, `SIX_MONTH` (search and hashtag only) | |

A 200 response is a JSON array of video objects, the same objects a batch run pushes to its dataset.

Errors:

| Status | Body | Meaning |
|---|---|---|
| 400 | `{ "errors": [...] }` or `{ "error": "Unsupported TikTok URL..." }` | Bad parameters, or a URL the actor can't handle |
| 402 | | Your max charge limit is reached |
| 404 | | Unknown path (you called something other than `/videos`) |
| 502 | `{ "error": "Upstream request failed" }` | The fetch behind the request failed |

You're charged per returned item (`result-item`), plus the same per-task events a normal run uses. A single-video lookup, for example, is charged as `single-video-query`. Current prices are on the [Store page](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m).

## curl

Put your Apify token in the `Authorization` header. `?token=` in the query string also works, but headers are less likely to be logged. Don't commit the token.

```bash
export APIFY_TOKEN=...   # Console > API & Integrations
BASE=https://novi--tiktok-scraper-ultimate.apify.actor

# Latest 20 videos from a profile
curl -s --max-time 300 -G "$BASE/videos" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  --data-urlencode "url=https://www.tiktok.com/@gordonramsayofficial" \
  --data-urlencode "limit=20"

# Most liked videos for a hashtag
curl -s --max-time 300 -G "$BASE/videos" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  --data-urlencode "url=https://www.tiktok.com/tag/duet" \
  --data-urlencode "sortType=MOST_LIKED"

# Keyword search, past week, US
curl -s --max-time 300 -G "$BASE/videos" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  --data-urlencode "keyword=podcast" \
  --data-urlencode "location=US" \
  --data-urlencode "dateRange=WEEK"
```

`-G` with `--data-urlencode` turns the values into an encoded query string. A TikTok URL contains `/`, `@`, `?` and `&`, so always encode it before putting it in a query parameter.

## A Node.js client

This uses the built-in `fetch` in Node 18 and later, so there's nothing to install. `URLSearchParams` handles the encoding.

```js
// tiktok.js
const BASE = 'https://novi--tiktok-scraper-ultimate.apify.actor';
const TOKEN = process.env.APIFY_TOKEN;

export class TikTokLookupError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
  }
}

export async function getVideos({ url, keyword, limit = 20, location, sortType, dateRange }) {
  if (!!url === !!keyword) {
    throw new Error('Pass exactly one of url or keyword');
  }

  const params = new URLSearchParams({ limit: String(limit) });
  if (url) params.set('url', url);
  if (keyword) params.set('keyword', keyword);
  if (location) params.set('location', location);
  if (sortType) params.set('sortType', sortType);
  if (dateRange) params.set('dateRange', dateRange);

  const res = await fetch(`${BASE}/videos?${params}`, {
    headers: { Authorization: `Bearer ${TOKEN}` },
    signal: AbortSignal.timeout(300_000),
  });

  if (res.ok) return res.json(); // array of video objects

  const body = await res.json().catch(() => ({}));
  switch (res.status) {
    case 400: {
      const msg = body.error ?? (body.errors ? JSON.stringify(body.errors) : '');
      throw new TikTokLookupError(400, msg || 'Invalid request');
    }
    case 402:
      throw new TikTokLookupError(402, 'Apify charge limit reached for this account');
    case 502:
      throw new TikTokLookupError(502, body.error ?? 'Upstream request failed');
    default:
      throw new TikTokLookupError(res.status, body.error ?? `HTTP ${res.status}`);
  }
}
```

How to treat each error:

- 400 is the user's fault, usually a link the actor doesn't support. Show the message and don't retry.
- 402 means you've hit your own spending limit. Retrying won't help. Raise the limit in Apify or tell the user to try later.
- 502 is a failed upstream fetch. One retry after a short wait is reasonable.
- A timeout from `AbortSignal` usually means a big request on a cold container. Lower `limit` before you retry.

## Mini-project: a paste-a-link lookup service

Here is a small HTTP service. Your Slack slash command, browser extension, or agent sends it whatever the user typed. If it looks like a TikTok link, the service passes it as `url`. Otherwise it treats it as a keyword. It returns a short JSON reply your front end can render.

```js
// server.js  (node server.js)
import http from 'node:http';
import { getVideos, TikTokLookupError } from './tiktok.js';

const TIKTOK_HOST = /(^|\.)tiktok\.com$/i;

function parseInput(text) {
  const trimmed = text.trim();
  try {
    const u = new URL(trimmed);
    if (TIKTOK_HOST.test(u.hostname)) return { url: trimmed };
  } catch {
    // not a URL, fall through to keyword
  }
  return { keyword: trimmed };
}

http.createServer(async (req, res) => {
  const reqUrl = new URL(req.url, 'http://localhost');
  if (reqUrl.pathname !== '/lookup') {
    res.writeHead(404).end();
    return;
  }

  const q = reqUrl.searchParams.get('q');
  if (!q) {
    res.writeHead(400, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ error: 'Missing q' }));
    return;
  }

  const limit = Math.min(Number(reqUrl.searchParams.get('limit')) || 10, 100);

  try {
    const videos = await getVideos({ ...parseInput(q), limit });
    res.writeHead(200, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ query: q, count: videos.length, videos }));
  } catch (err) {
    const status = err instanceof TikTokLookupError ? err.status : 504;
    res.writeHead(status, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ query: q, error: err.message }));
  }
}).listen(3000, () => console.log('Listening on http://localhost:3000/lookup?q=...'));
```

Try it:

```bash
APIFY_TOKEN=... node server.js

curl "http://localhost:3000/lookup?q=https%3A%2F%2Fwww.tiktok.com%2F%40billieeilish%2Fvideo%2F7050551461734042926"
curl "http://localhost:3000/lookup?q=podcast&limit=5"
```

The service doesn't check which kind of TikTok URL it got. The actor already sorts out videos, profiles, hashtags, music, search pages and short links, so your code only needs to tell "TikTok link" apart from "anything else".

Some things to add before you ship it:

- Cache results by input for a few minutes. Two people pasting the same viral video shouldn't cost you twice, since every returned item is charged.
- Keep the default `limit` small. For a Slack reply or an extension popup, 5 to 10 videos is plenty, and smaller requests come back faster.
- Before you render anything, log one raw item and pick the fields your UI needs. The server returns the same video objects as the dataset, so the actor's README and a sample batch run show the full shape.
- Keep your Apify token on this server. Never ship it inside a browser extension.

As an agent tool, the same `getVideos` function needs two arguments, `url` or `keyword`, and a description along the lines of "Get TikTok videos for any TikTok link (video, profile, hashtag, sound, search) or a search keyword". Cap `limit` in the tool schema so the model can't ask for 100 videos to answer a simple question.

## When to use a normal run instead

Server mode covers one URL or keyword and up to 100 items per request. Switch to the standard input run when you need:

- many URLs in one job
- more than 100 items
- `customMapFunction`
- results stored in a dataset, or a scheduled job nobody is waiting on

For the batch side, including `run-sync-get-dataset-items` and the async fallback, see [Server Actors vs run-sync: Wiring TikTok and X Scrapers into an AI Agent](/apify/server-actors-vs-run-sync-tiktok-x-agent/). The actor, pricing and Server tab are on the [Ultimate TikTok Scraper Store page](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m).
