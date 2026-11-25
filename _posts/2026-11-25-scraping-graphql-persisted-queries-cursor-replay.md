---
layout: post
title: "Scraping GraphQL Persisted Queries: Capturing APQ Hashes, Cursor Pagination, and Resumable HTTP Replay"
date: 2026-11-25 08:15:00 +0700
permalink: /web-scraping/scraping-graphql-persisted-queries-cursor-replay/
categories: [Web Scraping, Developer Guides, Python]
tags: [GraphQL, Apollo APQ, Web Scraping, Playwright, Python, SQLite, Apify]
author: Novi Develop
description: "How to scrape GraphQL APIs using Apollo Automatic Persisted Queries without running a full browser loop. Capture SHA-256 query hashes with Playwright, paginate Relay cursors over HTTP, recover from PersistedQueryNotFound rotations, and checkpoint state in SQLite."
---

Modern storefronts and SaaS directories rarely paginate with `?page=2` HTML links anymore. Almost every list view sends `POST` or `GET` requests to a single `/graphql` endpoint, swaps readable query strings for SHA-256 hashes under [Apollo Automatic Persisted Queries (APQ)](https://www.apollographql.com/docs/apollo-server/performance/apq), and tracks position with opaque cursor strings inside the JSON `variables` payload.

Driving a headless browser to click "Load more" for 500 pages wastes RAM, proxy bandwidth, and CPU rendering images you never read. A cleaner split uses Playwright once to capture the live `operationName`, `sha256Hash`, and session headers, then runs the full cursor loop over plain HTTP with SQLite checkpoints.

## How Apollo APQ and Relay cursors look on the wire

When a frontend enables APQ, the client sends a JSON envelope containing the operation name, its runtime `variables`, and a `persistedQuery` block under `extensions` carrying the SHA-256 digest of the GraphQL document:

```json
{
  "operationName": "GetCatalogPage",
  "variables": { "first": 20, "after": "cursor:40" },
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "0eef0a3856c94530..."
    }
  }
}
```

Many deployments also accept APQ over HTTP `GET` with URL-encoded `operationName`, `variables`, and `extensions` query parameters so CDNs can cache identical catalog pages at the edge.

Following the [GraphQL pagination model](https://graphql.org/learn/pagination/) and the [Relay Cursor Connections specification](https://relay.dev/graphql/connections.htm), responses return an `edges` array and a `pageInfo` object holding `hasNextPage` and `endCursor`. You pass `endCursor` back as `variables.after` on the next call until `hasNextPage` turns `false`.

Compared to fetching rendered HTML pages, direct GraphQL replay shrinks payload volume sharply. In our local storefront test fixture serving five product records per page, the full HTML catalog shell weighed `54,871` bytes raw (`5,547` bytes gzipped), while the equivalent `/graphql` APQ JSON response weighed `814` bytes raw (`250` bytes gzipped). That is a 98.5% raw byte reduction and a 95.5% gzip wire reduction before counting images or JavaScript bundles.

## Three failure modes that break naive GraphQL scrapers

- GraphQL endpoints routinely return HTTP `200 OK` even when the request fails. Rate limits, auth timeouts, and missing hashes show up inside a top-level `"errors": [...]` array while `data` is `null` or partial. Checking `resp.status == 200` treats failures as empty pages.
- Every frontend deploy that adds or reorders a single field inside a query changes its SHA-256 digest. When your scraper replays yesterday's hash, the server answers HTTP `200` with `PersistedQueryNotFound` (`extensions.code == "PERSISTED_QUERY_NOT_FOUND"`). Hardcoding hashes in source code means every frontend release pauses your pipeline.
- Long cursor crawls can die halfway through from a proxy reset or worker restart. Because cursors are sequential, losing `endCursor` forces a restart from page one.

## Bootstrapping hashes once with Playwright network interception

You only need a browser when starting a crawl or refreshing an expired hash. Attach a listener via [Playwright's network API](https://playwright.dev/docs/network) before calling `page.goto()`, inspect outgoing `/graphql` requests, and capture the active hash along with required headers:

```javascript
import { chromium } from "playwright";

export async function captureApqDescriptor(targetUrl, targetOperation) {
  const browser = await chromium.launch({ headless: true });
  const page = await browser.newPage();
  let captured = null;

  page.on("response", async (response) => {
    if (!response.url().includes("/graphql") || response.status() !== 200) {
      return;
    }
    const req = response.request();
    let payload = null;
    if (req.method() === "POST") {
      try {
        payload = req.postDataJSON();
      } catch {
        return;
      }
    } else if (req.method() === "GET") {
      const url = new URL(req.url());
      const op = url.searchParams.get("operationName");
      const extRaw = url.searchParams.get("extensions");
      if (op && extRaw) {
        payload = {
          operationName: op,
          extensions: JSON.parse(extRaw),
        };
      }
    }

    const hash = payload?.extensions?.persistedQuery?.sha256Hash;
    if (payload?.operationName === targetOperation && hash) {
      const headers = req.headers();
      captured = {
        endpoint: response.url().split("?")[0],
        operationName: payload.operationName,
        sha256Hash: hash,
        cookie: headers["cookie"] || "",
        authorization: headers["authorization"] || "",
      };
    }
  });

  await page.goto(targetUrl, { waitUntil: "networkidle" });
  await browser.close();
  return captured;
}
```

Wait for `networkidle` or trigger one scroll step if the target list lazy-loads below the fold. Once `captured` holds `endpoint`, `sha256Hash`, and any session header, close the browser and hand those values to an HTTP worker.

## Building a resumable Python replay loop with hash recovery

The Python script below runs the paginated APQ crawl using only the standard library (`urllib.request`, `gzip`, `json`, and `sqlite3`). It checks `payload["errors"]` on every HTTP `200` response, distinguishes `PersistedQueryNotFound` from transient GraphQL errors, invokes a `refresh_hash_fn` callback when a hash rotates, and commits every page alongside the updated `endCursor` in SQLite.

```python
import gzip
import json
import sqlite3
import time
import urllib.request


class PersistedQueryExpired(Exception):
    pass


class GraphQLTransientError(Exception):
    pass


def post_apq_page(
    endpoint: str,
    operation_name: str,
    sha256_hash: str,
    variables: dict,
    headers: dict = None,
) -> tuple:
    req_headers = {
        "Content-Type": "application/json",
        "Accept": "application/json",
        "Accept-Encoding": "gzip",
        "User-Agent": "CatalogReplay/1.0",
    }
    if headers:
        req_headers.update(headers)

    body = json.dumps(
        {
            "operationName": operation_name,
            "variables": variables,
            "extensions": {
                "persistedQuery": {
                    "version": 1,
                    "sha256Hash": sha256_hash,
                }
            },
        }
    ).encode("utf-8")

    req = urllib.request.Request(endpoint, data=body, headers=req_headers, method="POST")
    with urllib.request.urlopen(req, timeout=15) as resp:
        wire = resp.read()
        enc = (resp.headers.get("Content-Encoding") or "").lower()
        raw = gzip.decompress(wire) if enc == "gzip" else wire
        payload = json.loads(raw.decode("utf-8"))

    errors = payload.get("errors") or []
    if errors:
        codes = {e.get("extensions", {}).get("code", "") for e in errors}
        messages = " | ".join(e.get("message", "") for e in errors)
        if "PERSISTED_QUERY_NOT_FOUND" in codes or "PersistedQueryNotFound" in messages:
            raise PersistedQueryExpired(messages)
        raise GraphQLTransientError(messages)

    return payload.get("data", {}), len(raw), len(wire)


def run_resumable_graphql_crawl(
    db_path: str,
    endpoint: str,
    operation_name: str,
    initial_hash: str,
    refresh_hash_fn,
    page_size: int = 5,
    max_pages: int = None,
) -> dict:
    conn = sqlite3.connect(db_path)
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS cursor_state (
            operation_name TEXT PRIMARY KEY,
            query_hash TEXT NOT NULL,
            last_cursor TEXT,
            done INTEGER NOT NULL DEFAULT 0
        )
        """
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS catalog_items (
            sku TEXT PRIMARY KEY,
            title TEXT NOT NULL,
            price REAL NOT NULL,
            in_stock INTEGER NOT NULL
        )
        """
    )

    row = conn.execute(
        "SELECT query_hash, last_cursor, done FROM cursor_state WHERE operation_name=?",
        (operation_name,),
    ).fetchone()

    if row:
        query_hash, cursor, done = row
    else:
        query_hash, cursor, done = initial_hash, None, 0

    stats = {
        "pages_fetched": 0,
        "items_upserted": 0,
        "hash_refreshes": 0,
        "graphql_retries": 0,
        "raw_bytes": 0,
        "wire_bytes": 0,
        "completed": bool(done),
    }

    if done:
        conn.close()
        return stats

    while max_pages is None or stats["pages_fetched"] < max_pages:
        variables = {"first": page_size, "after": cursor}
        attempt = 0
        while True:
            attempt += 1
            try:
                data, raw_len, wire_len = post_apq_page(
                    endpoint, operation_name, query_hash, variables
                )
                break
            except PersistedQueryExpired:
                if stats["hash_refreshes"] >= 2:
                    conn.close()
                    raise
                query_hash = refresh_hash_fn(operation_name)
                stats["hash_refreshes"] += 1
                conn.execute(
                    "INSERT OR REPLACE INTO cursor_state VALUES (?, ?, ?, ?)",
                    (operation_name, query_hash, cursor, 0),
                )
                conn.commit()
            except GraphQLTransientError:
                if attempt >= 3:
                    conn.close()
                    raise
                stats["graphql_retries"] += 1
                time.sleep(0.05 * attempt)

        conn_block = data.get("catalog") or {}
        edges = conn_block.get("edges") or []
        page_info = conn_block.get("pageInfo") or {}

        for edge in edges:
            node = edge.get("node") or {}
            conn.execute(
                "INSERT OR REPLACE INTO catalog_items VALUES (?, ?, ?, ?)",
                (node["sku"], node["title"], float(node["price"]), 1 if node["inStock"] else 0),
            )
            stats["items_upserted"] += 1

        cursor = page_info.get("endCursor")
        has_next = bool(page_info.get("hasNextPage"))
        stats["pages_fetched"] += 1
        stats["raw_bytes"] += raw_len
        stats["wire_bytes"] += wire_len

        conn.execute(
            "INSERT OR REPLACE INTO cursor_state VALUES (?, ?, ?, ?)",
            (operation_name, query_hash, cursor, 0 if has_next else 1),
        )
        conn.commit()

        if not has_next or not cursor:
            stats["completed"] = True
            break

    conn.close()
    return stats
```

We verified this loop end to end against a local 25-product GraphQL server configured to rotate its active `sha256Hash` right before page 3 (`cursor:10`) and inject an HTTP `200` `RATE_LIMITED` error on page 4 (`cursor:15`):

| Run | Pages fetched | Items upserted | Hash refreshes | GraphQL retries | Gzip wire bytes | Completed |
|---|---|---|---|---|---|---|
| Run 1 (stopped after 2 pages) | 2 | 10 | 0 | 0 | 506 | False |
| Run 2 (resumed from SQLite) | 3 | 15 | 1 | 1 | 763 | True |
| Run 3 (already completed) | 0 | 0 | 0 | 0 | 0 | True |

Across all three runs, the client sent exactly 7 HTTP requests (5 data pages, 1 rejected stale-hash call, and 1 rate-limited retry), wrote all 25 distinct `SKU` rows to SQLite with zero duplicates, and skipped every network call on the third invocation once `done` was set to `1`.

## Where direct GraphQL replay stops scaling

Direct HTTP replay breaks down when a target binds every GraphQL request to short-lived per-request cryptographic signatures generated inside obfuscated browser JavaScript, or enforces strict guest-token and session rotation across every few pages. X/Twitter's internal GraphQL API is a familiar example: query IDs rotate regularly alongside `x-client-transaction-id` headers and feature flag dictionaries, so maintaining a custom replayer turns into ongoing reverse-engineering work.

A practical split keeps the lightweight APQ replay pattern on public e-commerce catalogs and B2B directories while offloading heavily signed social GraphQL and mobile RPC endpoints to maintained actors such as [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m), [TikTok User Info API](https://apify.com/novi/tiktok-user-info-api?fpr=7hce1m), or [Fast TikTok API](https://apify.com/novi/fast-tiktok-api?fpr=7hce1m). Keeping custom APQ loops on stable catalogs removes the browser from your inner pagination loop without forcing you to chase weekly signature changes.
