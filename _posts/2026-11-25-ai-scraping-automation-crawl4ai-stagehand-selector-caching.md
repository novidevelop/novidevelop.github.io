---
layout: post
title: "Hybrid AI Web Scraping Automation: Crawl4AI, Stagehand, and Self-Healing SQLite Selector Caching"
date: 2026-11-25 10:15:00 +0700
permalink: /web-scraping/ai-scraping-automation-crawl4ai-stagehand-selector-caching/
categories: [Web Scraping, Developer Guides, AI Automation]
tags: [AI Scraping, Crawl4AI, Stagehand, Playwright, Python, SQLite, Automation, Apify]
author: Novi Develop
description: "How to combine LLM schema discovery with deterministic SQLite selector caching in Crawl4AI and Stagehand. Cut LLM calls by 98% across 100 storefront pages while self-healing automatically through mid-crawl CSS refactors."
---

Sending raw HTML to an LLM on every page of a scheduled scrape burns tokens and adds seconds of latency to every request. Pure CSS scrapers sit at the opposite extreme: they run in microseconds per page, then silently write empty rows the afternoon a frontend deploy hashes class names from `.prod-card` to `.tw-card-9a`.

A hybrid pipeline separates discovery from execution. The model inspects one sample page to emit a JSON extraction schema or action plan, which the worker caches locally by domain and page template. Pages 2 through N run through a deterministic parser. If a mid-crawl deploy breaks validation, the worker regenerates the schema once and returns to local parsing.

## How Crawl4AI and Stagehand separate discovery from replay

[Crawl4AI](https://github.com/unclecode/crawl4ai) handles high-throughput HTML extraction in Python, while [Stagehand](https://github.com/browserbase/stagehand) wraps browser actions and structured reads in TypeScript and JavaScript on top of [Playwright's locator engine](https://playwright.dev/docs/locators).

In [Crawl4AI's no-LLM extraction strategies](https://docs.crawl4ai.com/extraction/no-llm-strategies/), `JsonCssExtractionStrategy.generate_schema()` invokes an LLM once against a sample page and returns a plain JSON dictionary describing `baseSelector` and per-field CSS selectors or attributes. Save that JSON document to disk and pass it to `JsonCssExtractionStrategy(schema)` on every following page so extraction runs locally:

```python
import json
from pathlib import Path
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig, JsonCssExtractionStrategy

SCHEMA_PATH = Path("catalog_schema.json")


def load_or_generate_schema(sample_url: str) -> dict:
    if SCHEMA_PATH.exists():
        return json.loads(SCHEMA_PATH.read_text(encoding="utf-8"))

    schema = JsonCssExtractionStrategy.generate_schema(
        url=sample_url,
        query="Extract each product card: sku attribute, title, price, and stock status",
    )
    SCHEMA_PATH.write_text(json.dumps(schema, indent=2), encoding="utf-8")
    return schema


async def crawl_catalog_pages(urls: list) -> list:
    schema = load_or_generate_schema(urls[0])
    config = CrawlerRunConfig(extraction_strategy=JsonCssExtractionStrategy(schema))
    records = []

    async with AsyncWebCrawler() as crawler:
        for url in urls:
            result = await crawler.arun(url=url, config=config)
            if result.success and result.extracted_content:
                records.extend(json.loads(result.extracted_content))
    return records
```

When a workflow needs browser interaction like clicking pagination controls or opening filter drawers, [Stagehand's caching guide](https://docs.stagehand.dev/v4/best-practices/caching) and [extract documentation](https://docs.stagehand.dev/v4/basics/extract) describe the same split. Point `cacheDir` at a persistent folder so `act()` and `observe()` replay cached locators locally with zero model inference on subsequent runs:

```javascript
import { Stagehand } from "@browserbasehq/stagehand";

export async function runCachedStorefrontStep(targetUrl) {
  const stagehand = new Stagehand({
    env: "LOCAL",
    cacheDir: ".stagehand-cache",
  });

  await stagehand.init();
  const page = stagehand.context.pages()[0];
  await page.goto(targetUrl, { waitUntil: "domcontentloaded" });

  // First run resolves the selector with an LLM and writes it to cacheDir.
  // Later runs replay the cached locator with zero LLM calls until the DOM drifts.
  await stagehand.act("click the Next page button in catalog pagination");

  const items = await stagehand.extract(
    "extract product sku, title, price, and stock label for every visible card"
  );

  await stagehand.close();
  return items;
}
```

Selector discovery and local replay cover single runs, while crawling hundreds of paginated routes also requires a versioned cache store and per-page row validation that triggers one schema refresh if a deploy lands mid-crawl.

## Building a self-healing SQLite selector cache in Python

A local SQLite table keyed by `(domain, template_fp)` gives worker processes atomic schema reads and writes with no external database. Using only the Python 3.9 standard library (`html.parser`, `hashlib`, `json`, `sqlite3`, and `time`), this orchestrator executes Crawl4AI-style JSON-CSS schemas, validates every extracted page, and calls `discover_schema_fn` only on a cold miss or a validation failure:

```python
from html.parser import HTMLParser
import hashlib
import json
import sqlite3
import time
from typing import Callable, Dict, List, Optional, Tuple


class JsonCssExtractor(HTMLParser):
    """Minimal deterministic executor for Crawl4AI-style JSON-CSS schemas."""

    def __init__(self, schema: Dict) -> None:
        super().__init__(convert_charrefs=True)
        self.base_class = schema["baseSelector"].lstrip(".")
        self.fields = schema["fields"]
        self.rows: List[Dict[str, str]] = []
        self._in_card = False
        self._card_depth = 0
        self._current_row: Dict[str, str] = {}
        self._active_field: Optional[str] = None
        self._field_depth = 0
        self._buf: List[str] = []

    def handle_starttag(self, tag: str, attrs: List[Tuple[str, Optional[str]]]) -> None:
        attr_map = dict(attrs)
        classes = set((attr_map.get("class") or "").split())

        if not self._in_card and self.base_class in classes:
            self._in_card = True
            self._card_depth = 1
            self._current_row = {}
            for field in self.fields:
                if field.get("type") == "attribute" and not field.get("selector"):
                    val = attr_map.get(field["attribute"])
                    if val is not None:
                        self._current_row[field["name"]] = val.strip()
            return

        if self._in_card:
            self._card_depth += 1
            if self._active_field is not None:
                self._field_depth += 1
                return
            for field in self.fields:
                sel = (field.get("selector") or "").lstrip(".")
                if sel and sel in classes:
                    if field.get("type") == "attribute":
                        val = attr_map.get(field["attribute"])
                        if val is not None:
                            self._current_row[field["name"]] = val.strip()
                    else:
                        self._active_field = field["name"]
                        self._field_depth = 1
                        self._buf = []
                    break

    def handle_data(self, data: str) -> None:
        if self._active_field is not None:
            self._buf.append(data)

    def handle_endtag(self, tag: str) -> None:
        if not self._in_card:
            return
        if self._active_field is not None:
            self._field_depth -= 1
            if self._field_depth == 0:
                self._current_row[self._active_field] = "".join(self._buf).strip()
                self._active_field = None
                self._buf = []
        self._card_depth -= 1
        if self._card_depth == 0:
            self.rows.append(self._current_row)
            self._in_card = False


def extract_with_schema(html: str, schema: Dict) -> List[Dict[str, str]]:
    parser = JsonCssExtractor(schema)
    parser.feed(html)
    parser.close()
    return parser.rows


def validate_rows(rows: List[Dict[str, str]], required_fields: Tuple[str, ...]) -> bool:
    if not rows:
        return False
    return all(all(row.get(k) for k in required_fields) for row in rows)


class HybridSelectorOrchestrator:
    def __init__(self, db_path: str, required_fields: Tuple[str, ...]) -> None:
        self.required_fields = required_fields
        self.conn = sqlite3.connect(db_path)
        self.conn.execute(
            """
            CREATE TABLE IF NOT EXISTS selector_cache (
                domain TEXT NOT NULL,
                template_fp TEXT NOT NULL,
                schema_json TEXT NOT NULL,
                schema_version INTEGER NOT NULL DEFAULT 1,
                updated_at REAL NOT NULL,
                PRIMARY KEY (domain, template_fp)
            )
            """
        )
        self.conn.commit()

    @staticmethod
    def template_fingerprint(domain: str, route_template: str) -> str:
        raw = f"{domain}:{route_template}".encode("utf-8")
        return hashlib.sha256(raw).hexdigest()[:16]

    def _get_schema(self, domain: str, fp: str) -> Optional[Tuple[Dict, int]]:
        row = self.conn.execute(
            "SELECT schema_json, schema_version FROM selector_cache WHERE domain=? AND template_fp=?",
            (domain, fp),
        ).fetchone()
        if not row:
            return None
        return json.loads(row[0]), int(row[1])

    def _save_schema(self, domain: str, fp: str, schema: Dict, version: int) -> None:
        self.conn.execute(
            """
            INSERT INTO selector_cache (domain, template_fp, schema_json, schema_version, updated_at)
            VALUES (?, ?, ?, ?, ?)
            ON CONFLICT(domain, template_fp) DO UPDATE SET
                schema_json = excluded.schema_json,
                schema_version = excluded.schema_version,
                updated_at = excluded.updated_at
            """,
            (domain, fp, json.dumps(schema), version, time.time()),
        )
        self.conn.commit()

    def extract_page(
        self,
        domain: str,
        route_template: str,
        html: str,
        discover_schema_fn: Callable[[str], Dict],
    ) -> Tuple[List[Dict[str, str]], bool]:
        fp = self.template_fingerprint(domain, route_template)
        cached = self._get_schema(domain, fp)
        used_llm = False

        if cached is None:
            schema = discover_schema_fn(html)
            version = 1
            self._save_schema(domain, fp, schema, version)
            used_llm = True
        else:
            schema, version = cached

        rows = extract_with_schema(html, schema)
        if not validate_rows(rows, self.required_fields):
            schema = discover_schema_fn(html)
            version += 1
            self._save_schema(domain, fp, schema, version)
            used_llm = True
            rows = extract_with_schema(html, schema)
            if not validate_rows(rows, self.required_fields):
                raise RuntimeError(f"Schema self-healing failed for {domain}:{route_template}")

        return rows, used_llm

    def close(self) -> None:
        self.conn.close()
```

When moving this pattern into a live crawler, watch two details:

- Key the cache on a normalized route template such as `catalog-grid` or `product-detail` paired with the domain, so `/catalog/sensors?page=1` and `/catalog/sensors?page=61` resolve to the same cache row while detail pages keep their own schema.
- Validate both row count (`len(rows) > 0`) and required field presence on every page. A partial CSS refactor that renames `.price-num` to `.tw-amt-4f` while keeping the card wrapper intact still returns rows, and checking every required field catches that silent drop immediately before writing incomplete records.

## Benchmark: 100 storefront pages with a mid-crawl layout refactor

We tested both strategies against the same 100-page storefront fixture (6 product cards per page, 600 distinct SKUs from `SKU-0001` through `SKU-0600`) in Python 3.9:

- Pages 1 to 60 render Layout v1 (`class="prod-card"`, `data-sku`, `class="price-num"`, `class="stock-label"`), averaging `9,618.2` HTML bytes per page.
- Pages 61 to 100 simulate a mid-crawl frontend CSS refactor to Layout v2 (`class="tw-card-9a"`, `data-item-id`, `class="tw-amt-4f"`, `class="tw-inv-2b"`), averaging `9,863.2` HTML bytes per page.
- Total corpus size across all 100 pages is `971,620` HTML bytes. Prompt tokens are estimated directly from the HTML payload sent to the model via `len(html) // 4`.

| Metric | Naive per-page LLM | Hybrid SQLite cache + self-heal | Reduction |
|---|---|---|---|
| Pages extracted | 100 / 100 | 100 / 100 | 0 errors |
| Distinct SKUs extracted | 600 / 600 | 600 / 600 | 100% match |
| LLM calls | 100 | 2 | 98.0% |
| HTML bytes sent to LLM | 971,620 | 19,444 | 98.0% |
| Estimated prompt tokens (`len / 4`) | 242,878 | 4,860 | 98.0% |
| Local in-process extraction time | 73.87 ms | 70.67 ms | 0.71 ms / page |

On page 1, the hybrid pipeline missed the cold SQLite cache, called schema discovery once (`v1`), and persisted the schema. Pages 2 to 60 ran deterministically from SQLite (`59` valid cache hits, `0` LLM calls). On page 61, the cached `v1` schema returned `0` rows against the refactored `tw-card-9a` DOM. That failure ran one self-healing schema regeneration, bumped SQLite to `schema_version = 2`, and let pages 61 to 100 finish deterministically with zero further model calls and zero dropped SKUs.

The `70.67 ms` timing measures local in-process execution (SQLite lookup plus `HTMLParser` pass) without remote LLM HTTP round-trip time. Cutting 98 out of 100 remote model requests removes nearly all API latency and token spend while keeping automatic recovery active when a stylesheet deploy lands mid-run.

## Where hybrid selector caching fits and when to use managed actors

Public e-commerce catalogs, B2B directories, job boards, and internal SaaS portals keep their data in server-rendered or hydrated HTML templates that change every few weeks. A handful of template fingerprints cover tens of thousands of URLs, so amortizing one discovery call across an entire crawl keeps compute and API spend predictable.

Social platforms and mobile-first marketplaces such as TikTok, TikTok Shop, and X/Twitter enforce per-request cryptographic signatures, rotating device fingerprints, short-lived guest tokens, and residential IP checks before returning a payload. Self-healing a CSS selector cannot recover data when the upstream response is an empty shell or a bot challenge.

For those targets, running custom Playwright clusters alongside LLM discovery burns engineering hours on anti-bot maintenance. Keeping your Crawl4AI or Stagehand selector-cache workers focused on HTML storefronts while routing signed social feeds to managed actors like [TikTok Shop API](https://apify.com/novi/tiktok-shop-scraper?fpr=7hce1m), [Fast TikTok API](https://apify.com/novi/fast-tiktok-api?fpr=7hce1m), or [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m) keeps both infrastructure cost and maintenance overhead bounded.
