---
layout: post
title: "Scraping Next.js App Router Without a Browser: Parsing __next_f Flight Payloads and RSC Streams"
date: 2026-11-25 00:05:00 +0700
permalink: /web-scraping/scraping-nextjs-app-router-rsc-flight-hydration/
categories: [Web Scraping, Developer Guides, Python]
tags: [Next.js, React Server Components, Web Scraping, Python, JSON-LD, SQLite, Apify]
author: Novi Develop
description: "Extract structured JSON from Next.js App Router sites without Playwright. Parse concatenated self.__next_f.push chunks, use the RSC: 1 header, handle split chunks, and build a SQLite price-change monitor."
---

If your scraper still queries `#__NEXT_DATA__` on a modern Next.js storefront or SaaS site, it probably returns `None` today. Sites built on the Next.js App Router no longer serialize page props into one big JSON script tag. Instead, they stream React Server Components (RSC) over the [Next.js Flight wire format](https://nextjs.org/docs/app/getting-started/server-and-client-components), either embedded across dozens of inline `self.__next_f.push(...)` scripts in the HTML or served directly when the request asks for an RSC payload.

You still do not need Playwright or Puppeteer to read that data when the server renders it for initial paint. Over the last week we built and tested a standard-library Python extractor against live Next.js App Router sites and a local storefront fixture, and ran into a subtle bug that breaks naive regex scrapers whenever Next.js splits a JSON object across two script tags.

## What Next.js App Router sends over the wire

We probed three live Next.js App Router URLs (`https://nextjs.org/blog`, `https://nextjs.org/docs`, and `https://vercel.com/pricing`) with plain HTTP `GET` requests, comparing the standard HTML response against sending the `RSC: 1` header. None of the three pages contained `__NEXT_DATA__`, and only `nextjs.org/docs` carried a single `application/ld+json` block.

| Target URL | HTML raw (bytes) | HTML gzip wire (bytes) | `RSC: 1` raw (bytes) | `RSC: 1` gzip wire (bytes) | `self.__next_f.push` calls |
|---|---|---|---|---|---|
| `https://nextjs.org/blog` | 1,071,304 | 63,427 | 366,862 | 23,873 | 116 |
| `https://nextjs.org/docs` | 444,410 | 58,344 | 293,813 | 39,062 | 44 |
| `https://vercel.com/pricing` | 1,264,013 | 109,840 | 315,772 | 42,321 | 95 |

Sending `RSC: 1` returned `Content-Type: text/x-component` on all three targets and cut gzip wire transfer by 62.4% on `nextjs.org/blog` (23,873 bytes vs 63,427 bytes) and 61.5% on `vercel.com/pricing` (42,321 bytes vs 109,840 bytes). Uncompressed payloads dropped even further, from 1.26 MB down to 315 KB on `vercel.com/pricing`.

One warning before you switch every scraper to `RSC: 1`: some deployments also check router headers such as `Next-Router-State-Tree` or vary cache keys at the CDN layer. Fetching regular HTML and decoding `self.__next_f.push` is slower by a few dozen kilobytes over gzip, yet it matches what an ordinary browser fetch receives on first load and rarely hits edge routing mismatches.

## Inside the Flight payload and the split-chunk bug

Open the HTML of `nextjs.org/blog` and you will find 116 script calls shaped like this:

```html
<script>self.__next_f.push([1,"2:\"/_next/static/immutable/chunks/1m3_xx66brmix.js\"\n"])</script>
```

The browser runs an array-like collector (`self.__next_f = self.__next_f || []`) where the first element (`1`) marks a payload chunk and the second element is a JavaScript string literal fragment of the Flight stream. When we decoded and concatenated all 116 fragments on `nextjs.org/blog`, they formed a 366,816-character stream containing 331 newline-delimited Flight rows: 36 client module import rows prefixed with `I` and 229 JSON element or prop trees.

Live Flight output has two quirks that break simple scrapers:

- Row IDs are hexadecimal or prefixed with `#`, as in `#1:"$Sreact.fragment"` on `nextjs.org/blog` and `nextjs.org/docs`. A parser expecting only base-10 digits before the colon will skip valid rows.
- String fragments can split mid-line and mid-JSON across consecutive `push([1, "..."])` calls.

That second point causes silent data loss. If your code loops through `<script>` tags and runs `re.findall(r'\{.*"price".*\}', script_text)` on each tag independently, any product JSON that spans the boundary between two `push()` calls disappears. We verified this in a local storefront fixture where a two-product JSON payload was split cleanly down the middle across two `<script>` elements: per-script matching found `0` items, while decoding each `[1, chunk]` string with `json.JSONDecoder.raw_decode()` and concatenating the decoded chunks first recovered every SKU intact.

## Extracting Flight trees cleanly in Python

You can parse `self.__next_f.push` streams with Python's standard library alone (`html.parser`, `json`, `re`, and `urllib.request`). Using `json.JSONDecoder().raw_decode()` handles nested brackets inside the `[1, "..."]` array without greedy regex backtracking, and unescapes `\n`, `\"`, and Unicode sequences safely.

```python
import gzip
from html.parser import HTMLParser
import json
import re
import urllib.request

PUSH_START_RE = re.compile(r"self\.__next_f\.push\(\s*\[")
ROW_RE = re.compile(r"^(#?[0-9a-fA-F]+):(.*)$")


class ScriptCollector(HTMLParser):
    def __init__(self):
        super().__init__()
        self._in_script = False
        self._type = ""
        self._id = ""
        self._buf = []
        self.scripts = []

    def handle_starttag(self, tag, attrs):
        if tag.lower() == "script":
            attr_map = {k.lower(): (v or "") for k, v in attrs}
            self._in_script = True
            self._type = attr_map.get("type", "").lower()
            self._id = attr_map.get("id", "")
            self._buf = []

    def handle_data(self, data):
        if self._in_script:
            self._buf.append(data)

    def handle_endtag(self, tag):
        if tag.lower() == "script" and self._in_script:
            self.scripts.append({
                "id": self._id,
                "type": self._type,
                "text": "".join(self._buf),
            })
            self._in_script = False


def fetch_text(url: str, extra_headers=None) -> str:
    headers = {
        "User-Agent": "Mozilla/5.0 (compatible; PriceMonitor/1.0)",
        "Accept-Encoding": "gzip",
    }
    if extra_headers:
        headers.update(extra_headers)
    req = urllib.request.Request(url, headers=headers)
    with urllib.request.urlopen(req, timeout=20) as resp:
        raw = resp.read()
        if (resp.headers.get("Content-Encoding") or "").lower() == "gzip":
            raw = gzip.decompress(raw)
        return raw.decode("utf-8", errors="replace")


def extract_flight_stream(html_text: str) -> str:
    collector = ScriptCollector()
    collector.feed(html_text)
    decoder = json.JSONDecoder()
    chunks = []

    for sc in collector.scripts:
        text = sc["text"]
        if "__next_f.push" not in text:
            continue
        pos = 0
        while True:
            match = PUSH_START_RE.search(text, pos)
            if not match:
                break
            try:
                arr, end_idx = decoder.raw_decode(text, match.end() - 1)
                if isinstance(arr, list) and len(arr) >= 2 and isinstance(arr[1], str):
                    chunks.append(arr[1])
                pos = end_idx
            except json.JSONDecodeError:
                pos = match.end()

    return "".join(chunks)


def parse_flight_rows(flight_text: str):
    rows = []
    decoder = json.JSONDecoder()
    for line in flight_text.splitlines():
        if not line:
            continue
        m = ROW_RE.match(line)
        if not m:
            continue
        row_id, rest = m.group(1), m.group(2)
        if not rest:
            continue
        kind = ""
        payload = rest
        if rest[0] in ("I", "H", "T", "E", "N", "D") and len(rest) > 1 and rest[1] in ('[', '{', '"'):
            kind = rest[0]
            payload = rest[1:]
        try:
            obj, _ = decoder.raw_decode(payload)
            rows.append((row_id, kind, obj))
        except json.JSONDecodeError:
            rows.append((row_id, kind, payload))
    return rows


def find_dicts_with_keys(obj, required_keys: set):
    found = []
    if isinstance(obj, dict):
        if required_keys.issubset(obj.keys()):
            found.append(obj)
        for val in obj.values():
            found.extend(find_dicts_with_keys(val, required_keys))
    elif isinstance(obj, list):
        for item in obj:
            found.extend(find_dicts_with_keys(item, required_keys))
    return found
```

When running against `https://nextjs.org/blog`, `parse_flight_rows(extract_flight_stream(html))` parsed the stream into Python dicts and arrays and pulled 80 `/blog/...` link records directly from the hydrated React tree without parsing a single DOM card selector.

## Real-world project: a SQLite storefront price monitor

Hydration extraction works well for tracking competitor pricing or plan tiers because prices sit inside server-rendered props for SEO, even when the surrounding Tailwind classes change every deploy. Below is a compact monitor that cascades across three layers before giving up:

1. Concatenated `self.__next_f` Flight trees (or fetch `fetch_text(url, {"RSC": "1"})` and pass that stream straight to `parse_flight_rows()`).
2. Legacy `__NEXT_DATA__` JSON for Pages Router sites.
3. `<script type="application/ld+json">` Product offers for storefronts that keep pricing out of React props.

```python
import json
import sqlite3


def extract_products_hybrid(html_text: str):
    stream = extract_flight_stream(html_text)
    if stream:
        products = []
        for _, kind, obj in parse_flight_rows(stream):
            if kind == "" and isinstance(obj, (dict, list)):
                products.extend(find_dicts_with_keys(obj, {"sku", "price", "inStock"}))
        if products:
            return "next_f", products

    collector = ScriptCollector()
    collector.feed(html_text)

    for sc in collector.scripts:
        if sc["id"] == "__NEXT_DATA__":
            try:
                data = json.loads(sc["text"])
                products = find_dicts_with_keys(data, {"sku", "price", "inStock"})
                if products:
                    return "next_data", products
            except json.JSONDecodeError:
                pass

    ld_products = []
    for sc in collector.scripts:
        if sc["type"] == "application/ld+json":
            try:
                block = json.loads(sc["text"])
            except json.JSONDecodeError:
                continue
            for p in find_dicts_with_keys(block, {"sku", "offers"}):
                offers = p.get("offers", {})
                if isinstance(offers, dict) and "price" in offers:
                    ld_products.append({
                        "sku": p["sku"],
                        "name": p.get("name", ""),
                        "price": float(offers["price"]),
                        "inStock": "InStock" in str(offers.get("availability", "")),
                    })
    if ld_products:
        return "json_ld", ld_products

    return "none", []


def poll_storefront(db_path: str, urls: list[str]) -> list[dict]:
    conn = sqlite3.connect(db_path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS price_state (
            url TEXT,
            sku TEXT,
            name TEXT,
            price REAL,
            in_stock INTEGER,
            source_layer TEXT,
            PRIMARY KEY (url, sku)
        )
    """)
    events = []

    for url in urls:
        html_text = fetch_text(url)
        layer, products = extract_products_hybrid(html_text)
        for p in products:
            sku = p["sku"]
            price = float(p["price"])
            in_stock = 1 if p["inStock"] else 0
            prev = conn.execute(
                "SELECT price, in_stock FROM price_state WHERE url=? AND sku=?",
                (url, sku),
            ).fetchone()
            if prev is None:
                events.append({
                    "event": "new_sku",
                    "sku": sku,
                    "price": price,
                    "in_stock": bool(in_stock),
                    "layer": layer,
                })
            elif prev[0] != price or prev[1] != in_stock:
                events.append({
                    "event": "changed",
                    "sku": sku,
                    "old_price": prev[0],
                    "new_price": price,
                    "old_stock": bool(prev[1]),
                    "new_stock": bool(in_stock),
                    "layer": layer,
                })
            conn.execute(
                "INSERT OR REPLACE INTO price_state VALUES (?, ?, ?, ?, ?, ?)",
                (url, sku, p.get("name", ""), price, in_stock, layer),
            )

    conn.commit()
    conn.close()
    return events
```

In our local three-poll verification run across a split-chunk App Router page (`478` bytes HTML vs `218` bytes for `RSC: 1`) and a JSON-LD page, Poll 1 recorded three `new_sku` baselines (`WH-1000XM5` at `$199.0`, `KEY-75-PRO` at `$129.0`, and `MIC-USB-2` at `$89.0`). Poll 2 with unchanged pages produced `[]` (zero duplicate alerts), and Poll 3 caught the `$199.0` to `$149.0` drop on `WH-1000XM5`, the stockout on `KEY-75-PRO`, and the JSON-LD fallback drop from `$89.0` to `$74.0` on `MIC-USB-2`.

## When hydration parsing hits a wall

Two limits matter in production. First, if a site fetches prices inside a client-side `useEffect` after mount, the JSON arrives from a separate XHR call after page load, outside `__next_f` and `RSC: 1`. Inspect the Network tab or run a quick Playwright response listener to see whether the number arrives in HTML or in a follow-up JSON call. Second, some Next.js apps split referenced values across Flight rows using `$L84`-style references (we saw `$L84`, `$L86`, and `$Le4` inside pricing links on `vercel.com/pricing`). When a property contains a `$L` pointer, resolve the referenced row ID in your row dictionary before reading nested fields.

There is also a practical split in how teams use price monitors like this. Direct HTTP hydration parsing works well on public e-commerce sites and SaaS pricing pages, while social commerce feeds guard their mobile APIs with request signatures and session checks. When your SQLite monitor emits a `changed` price-drop event for a product like `WH-1000XM5`, you can follow it with an API call to [Advanced Search TikTok API](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) to check if creators posted promo videos under that product name, or pull catalog listings from [TikTok Shop Scraper](https://apify.com/novi/tiktok-shop-scraper?fpr=7hce1m) to compare storefront discounts against TikTok Shop seller prices. Keeping the lightweight Python script on standard Next.js storefronts and delegating signed mobile endpoints to dedicated scrapers saves browser and proxy overhead where plain HTTP already has the data.
