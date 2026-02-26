---
layout: post
title: "Tutorial: Scrape Twitter Data with Apify's Tweet X.com Scraper"
date: 2025-03-08 10:00:00 +0000
categories: [ twitter, x, scraper, data-extraction, tutorial ]
---

This tutorial guides you through scraping Twitter (now X) data using Apify's **Tweet X.com Scraper**. It's designed for
beginners and provides a step-by-step approach to extracting tweets based on various criteria, such as keywords, user
handles, and date ranges. We'll cover setting up the scraper, configuring the input, and running the scraper to get your
data.

Apify's **Tweet X.com Scraper** is a powerful tool that allows you to extract large amounts of historical data from
Twitter (X) for analysis, research, and competitive intelligence. It's designed for *historical data*, not real-time
monitoring.

**Link to Tweet X.com Scraper:** [https://apify.com/xtdata/twitter-x-scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m).

## Prerequisites

Before you begin, make sure you have the following:

* **Apify Account:** You'll need an Apify account. Sign up for free at [apify.com](https://apify.com/).
*   **Apify API Token:** Once you have an Apify account, you'll need to obtain your API token. You can find it in the [Apify Console](https://console.apify.com/) under "Integrations."
* **Basic understanding of Twitter Advanced Search**: Familiarity with Twitter's advanced search operators will be very
  helpful. See [Advanced Search Guide](https://github.com/igorbrigadir/twitter-advanced-search)

![Get Personal API Token]({{ site.baseurl }}/images/get-personal-api-token.jpg)

## Step-by-Step Guide

### Step 1: Find the Tweet X.com Scraper on Apify

1. Log in to your Apify account.
2. Go to the Apify Store (you can usually find this in the navigation menu).
3. Search for "Tweet X.com Scraper" (or a similar name, depending on the exact Actor name).
4. Click on the Actor to open its details page.

### Step 2: Configure the Input

The Tweet X.com Scraper offers a wide range of input options to tailor your data extraction. Here's a breakdown of the
key settings:

* **Search Terms:** This is the core of your query. You can enter:
    * **Keywords:**  Simple words or phrases (e.g., `artificial intelligence`, `climate change`).
    * **Twitter Handles:**  Usernames (e.g., `@elonmusk`, `@NASA`).
    * **Advanced Search Operators:**  Combine keywords, handles, date ranges, and more using Twitter's advanced search
      syntax (see the [Advanced Search Guide](https://github.com/igorbrigadir/twitter-advanced-search)).  *This is
      highly recommended for precise targeting.*
    * **Start URLs:** Directly provide URLs for tweets, user profiles.
    * **Conversation IDs:** Focus on tweets within a specific conversation.
  
![Search Terms]({{ site.baseurl }}/images/input-search.png)

* **Example Search Term (Fetching tweets from NASA about Mars since Jan 1, 2024):**
    ```
    from:NASA Mars since:2024-01-01
    ```
    * **`from:NASA`**: Specifies the user.
    * **`Mars`**: The keyword to search for.
    * **`since:2024-01-01`**: Sets the start date.

* **Max Tweets:**  Set a limit on the number of tweets to retrieve. *Important for managing costs and preventing
  excessive run times.*

* **Sort:** Choose between "Top" (most relevant) and "Latest" (chronological order).

* **Language:**  Filter tweets by language
  using [ISO 639-1 codes](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes) (e.g., `en` for English, `es` for
  Spanish).

* **Filters:** Refine your results further:
    * **Only Verified Users:**  Target tweets from verified accounts.
    * **Only Twitter Blue:**  Target tweets from Twitter Blue subscribers.
    * **Only Images/Videos/Quotes:** Limit results to tweets containing specific media types.

### Step 3:  Run the Scraper

1. After configuring the input, click the "Start" or "Run" button (the exact wording may vary).
2. The scraper will begin running. You can monitor its progress on the Apify platform.
3. The run time will depend on the complexity of your query and the number of tweets you're requesting.

### Step 4:  Download Your Data

1. Once the scraper finishes, you'll see the results on the Apify platform.
2. You can download the data in various formats, such as JSON, CSV, Excel, or HTML. Choose the format that best suits
   your needs.
3. Click download button to download data you want.

## Example Scenarios and Input Configurations

**Example 1:  Fetching Tweets Mentioning "Apify" and "#webscraping" in the Last Week:**

```json
{
  "searchTerms": [
    "Apify #webscraping since:{{date:YYYY-MM-DD,-7d}} until:{{date:YYYY-MM-DD}}"
  ],
  "maxTweets": 100,
  "sort": "Latest",
  "tweetLanguage": "en"
}
```

**Explanation:**

* `"Apify #webscraping"`: Searches for tweets containing both "Apify" and the hashtag "#webscraping".
* `since:{{date:YYYY-MM-DD,-7d}}`:  Dynamically sets the start date to 7 days ago.
* `until:{{date:YYYY-MM-DD}}`: Dynamically sets the end date to today.
* `"maxTweets": 100`: Limits the results to a maximum of 100 tweets.

**Example 2:  Fetching Tweets from a Specific User (@OpenAI) Containing Images:**

```json
{
  "searchTerms": [
    "from:OpenAI"
  ],
  "maxTweets": 50,
  "sort": "Top",
  "onlyImage": true
}
```

**Explanation:**

* `"from:OpenAI"`: Targets tweets from the user @OpenAI.
* `"maxTweets": 50`:  Limits to 50 tweets.
* `"sort": "Top"`:  Prioritizes the most relevant tweets.
* `"onlyImage": true`: Filters for tweets containing images.

**Example 3: Fetching replies of a tweet with specific keyword**

```json
{
  "includeSearchTerms": false,
  "onlyImage": false,
  "onlyQuote": false,
  "onlyTwitterBlue": false,
  "onlyVerifiedUsers": false,
  "onlyVideo": false,
  "searchTerms": [
    "conversation_id:tweet_id_here keyword_here"
  ],
  "sort": "Latest",
  "tweetLanguage": "en"
}
```

## Important Usage Guidelines

* **Historical Data Only:** This scraper is for retrieving *historical* data, not for real-time monitoring.
* **Concurrency Limits:**  There are limits on concurrent runs. Check the Actor's documentation for details.
* **Minimum Tweets Per Query:** Aim for at least 50 tweets per query.
* **Single Tweet Fetching:**  Generally prohibited without explicit permission.
* **Respect Twitter's Terms of Service:** Always use this tool ethically and responsibly.

## Troubleshooting

* **No result or few result:** Check search term is correct or not. And release some filter option.
* **Incomplete Tweet Collection:** Refer troubleshooting part in actor description.

## Conclusion

You've now learned how to use Apify's Tweet X.com Scraper to extract valuable data from Twitter. Experiment with
different input configurations and advanced search operators to get the most out of this powerful tool. Happy scraping!
Remember to check the official Actor documentation on Apify for the most up-to-date information and features.

Start scraping Twitter data today!  This Actor provides a powerful, flexible, and cost-effective solution for all your
Twitter data extraction needs.

## Scrape any TikTok data you need with dedicated scrapers

If you want to get specific data from TikTok, Twitter, you can use the scrapers below. Each scraper is made to help you get
different kinds of TikTok data, like hashtags, search results, profiles, or everything at once. You can look at them to
see which one you need.

| [🎹️ Fast TikTok API](https://apify.com/novi/fast-tiktok-api?fpr=7hce1m)            | [📹️ TikTok Trend API](https://apify.com/novi/tiktok-trend-api?fpr=7hce1m)         | [🔍️ TikTok Search API](https://apify.com/novi/tiktok-search-api?fpr=7hce1m)             |
|:-------------------------------------------------------------------------|:------------------------------------------------------------------------|:------------------------------------------------------------------------------|
| [🧛️ TikTok User API](https://apify.com/novi/tiktok-user-api?fpr=7hce1m)            | [🧛️ TikTok User Info API](https://apify.com/novi/tiktok-user-info-api?fpr=7hce1m) | [#️ TikTok Hashtag API](https://apify.com/novi/tiktok-hashtag-api?fpr=7hce1m)            |
| [🛍️ TikTok Shop API](https://apify.com/novi/tiktok-shop-scraper?fpr=7hce1m)        | [👤️ TikTok Followers API](https://apify.com/novi/tiktok-followers-api?fpr=7hce1m) | [⚡️ TikTok Scraper (pay-per-result)](https://apify.com/xtdata/tiktok-scraper?fpr=7hce1m) |
| [💬 TikTok Comment API](https://apify.com/novi/tiktok-comment-api?fpr=7hce1m)       | [🎶 TikTok Music API](https://apify.com/novi/tiktok-sound-api?fpr=7hce1m)          | [🎶 TikTok Music Trend API](https://apify.com/novi/tiktok-music-trend-api?fpr=7hce1m)    |
| [🐦 Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m) |                                                                         |                                                                               |


