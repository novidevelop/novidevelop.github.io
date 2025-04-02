---
layout: post
title: "Using Twitter X Scraper with Python"
date: 2025-03-16 08:15:00 +0700
categories: [Python, Scraping]
---

# Using Twitter X Scraper with Python

The **Twitter X Scraper** is a powerful tool for extracting data from Twitter (now X.com). This guide walks you through setting up and using it with Python.

---

## Prerequisites

Before you start, ensure you have:

- Python (version 3.7 or higher).
- An Apify account with an API key.
- Basic Python programming knowledge.

---

## Installation

Install the required Python library:

```bash
pip install apify-client
```

---

## Setting Up the Scraper

1. **Obtain an API Key**  
   Log in to your Apify account and generate an API key from the **Setting -> API & Integrations** section.

    ![Get Personal API Token]({{ site.baseurl }}/images/get-personal-api-token.jpg)

2. **Initialize the Apify Client**  
   Use the API key to authenticate in Python:

   ```python
   from apify_client import ApifyClient

   # Initialize the Apify client
   client = ApifyClient("your_api_key_here")
   ```

---

## Running the Scraper

To scrape data, provide input parameters like search queries. Here’s an example:

```python
# Define the actor ID for Twitter X Scraper
actor_id = "xtdata/twitter-x-scraper"

# Set up input parameters
input_data = {
    "searchQuery": "#Python",
    "maxResults": 100,
    "language": "en"
}

# Run the scraper
run = client.actor(actor_id).call(run_input=input_data)

# Fetch the results
results = client.dataset(run["defaultDatasetId"]).list_items()
for item in results["items"]:
    print(item)
```

---

## Customization Options

Modify the `input_data` dictionary to customize the scraper:

- **Search by User**: Use `@username` in `searchQuery`.
- **Filter by Date**: Add date range parameters.

---

## Handling Results

The scraper outputs JSON data. You can process it or export it using libraries like `pandas`:

```python
import pandas as pd

# Convert results to a DataFrame
df = pd.DataFrame(results["items"])

# Save to a CSV file
df.to_csv("twitter_data.csv", index=False)
```

---

## Best Practices

- **Respect Terms of Service**: Comply with Twitter's guidelines.
- **Rate Limiting**: Avoid excessive requests.
- **Data Privacy**: Handle user data responsibly.

---

Using this guide, you can effectively scrape and process data from Twitter using the Twitter X Scraper and Python. Happy coding!


## Scrape any TikTok data you need with dedicated scrapers

If you want to get specific data from TikTok, Twitter, you can use the scrapers below. Each scraper is made to help you get
different kinds of TikTok data, like hashtags, search results, profiles, or everything at once. You can look at them to
see which one you need.

| [🎹️ Fast TikTok API](https://apify.com/novi/fast-tiktok-api)            | [📹️ TikTok Trend API](https://apify.com/novi/tiktok-trend-api)         | [🔍️ TikTok Search API](https://apify.com/novi/tiktok-search-api)             |
|:-------------------------------------------------------------------------|:------------------------------------------------------------------------|:------------------------------------------------------------------------------|
| [🧛️ TikTok User API](https://apify.com/novi/tiktok-user-api)            | [🧛️ TikTok User Info API](https://apify.com/novi/tiktok-user-info-api) | [#️ TikTok Hashtag API](https://apify.com/novi/tiktok-hashtag-api)            |
| [🛍️ TikTok Shop API](https://apify.com/novi/tiktok-shop-scraper)        | [👤️ TikTok Followers API](https://apify.com/novi/tiktok-followers-api) | [⚡️ TikTok Scraper (pay-per-result)](https://apify.com/xtdata/tiktok-scraper) |
| [💬 TikTok Comment API](https://apify.com/novi/tiktok-comment-api)       | [🎶 TikTok Music API](https://apify.com/novi/tiktok-sound-api)          | [🎶 TikTok Music Trend API](https://apify.com/novi/tiktok-music-trend-api)    |
| [🐦 Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper) |                                                                         |                                                                               |

