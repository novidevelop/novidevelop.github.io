---
layout: post
title: "TikTok Shop Scraper: Unearth Trending Products and Analyze the Market"
date: 2025-02-16 14:05:30 +0700
categories: [tiktok, scraper]
---

### Introduction

TikTok Shop has emerged as a significant e-commerce platform, presenting exciting opportunities for businesses and
content creators. To succeed in this rapidly evolving market, understanding trending products and market dynamics is
essential. The TikTok Shop Scraper, an Apify actor, provides a powerful solution for extracting publicly available data
from TikTok Shop, empowering you to make informed decisions.

### What is the TikTok Shop Scraper?

The TikTok Shop Scraper is an actor available on the Apify platform. It allows you to scrape publicly accessible data
from TikTok Shop without needing an official API. This includes information about products, trends, and sellers, giving
you valuable insights into the TikTok Shop ecosystem.

### Key Features and Usage

* Search by Keyword: Find products related to any keyword.  This is invaluable for competitor analysis, identifying niche markets, and discovering products of interest. Simply provide the keyword as input.  For instance, searching for "winter jackets" will return a list of relevant products.
* 
* Sort Results: Sort results by price (ascending or descending), best sellers, or relevance. This enables you to quickly identify trending items or find products within a specific price range. Use the sortType parameter with values like PRICE_ASC, PRICE_DESC, BEST_SELLERS, and RELEVANCE.
* 
* Filter by Region: Specify a two-letter country code (e.g., US, VN) to filter results. This helps you understand regional trends and product popularity. Use the region parameter to set the desired country code.
* 
* Detailed Product Information: Retrieve comprehensive product details, including price, seller information, reviews, and more. The scraper returns a dataset where each product is a separate entry, containing all available information.


### Technical Details and Usage

The scraper is optimized for performance and minimal memory consumption. You interact with it by providing input parameters in JSON format:

* `keyword` (String, Default: "baby"): The search term.
* `limit` (Integer, Default: 20): The maximum number of results.
* `sortType` (String, Default: "RELEVANCE"): Sorting criteria (PRICE_ASC, PRICE_DESC, BEST_SELLERS, RELEVANCE).
* `region` (String, Default: "VN"): Two-letter country code.
* `proxyConfiguration` (Object): Proxy settings. `{ "useApifyProxy": true }` uses Apify's proxy.

#### Example Input JSON:

```json
{
  "keyword": "toys",
  "limit": 50,
  "sortType": "PRICE_ASC",
  "region": "VN",
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

#### Python Code Sample

```python
from apify_client import ApifyClient

# Initialize the ApifyClient with your Apify API token
# Replace '<YOUR_API_TOKEN>' with your token.
client = ApifyClient("<YOUR_API_TOKEN>")

# Prepare the Actor input
run_input = {
    "region": "GB",
    "limit": 20,
    "sortType": "RELEVANCE",
    "proxyConfiguration": { "useApifyProxy": False },
}

# Run the Actor and wait for it to finish
run = client.actor("novi/tiktok-shop-scraper").call(run_input=run_input)

# Fetch and print Actor results from the run's dataset (if there are any)
print("💾 Check your data here: https://console.apify.com/storage/datasets/" + run["defaultDatasetId"])
for item in client.dataset(run["defaultDatasetId"]).iterate_items():
    print(item)

```

#### During the Run & Output

The actor provides status messages during execution. Results are stored in a dataset on the Apify platform, accessible via programming languages like Python, PHP, or Node.js.

#### Limitations and Considerations

* Public Data Only: The scraper accesses only publicly available data.
* TikTok's Policies: TikTok's terms of service and usage limitations apply.
* Rate Limiting: The scraper doesn't currently handle TikTok's rate limiting.
* Customization: The scraper can be customized or extended through Apify.
* Cost: A trial period may be available, followed by a subscription fee. Check the Apify platform for pricing.

### Support and Documentation

For detailed information, refer to the official documentation. For support, use the Apify actor issue tracker or join the Apify Discord server. The scraper is actively maintained, so check for updates and bug fixes.