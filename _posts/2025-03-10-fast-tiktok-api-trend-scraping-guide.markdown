---
layout: post
title: "Tutorial: A Guide to Fast TikTok API's TREND Input"
date: 2025-03-10 01:00:00 +0000
categories: [ tiktok, data-extraction, tutorial, trends ]
---

The Fast TikTok API's `TREND` scraping type is your shortcut to discovering what's currently popular on TikTok. Instead
of manually browsing the "For You" page, you can programmatically retrieve a list of trending videos. This guide will
walk you through using the API's input UI for the `TREND` type, offering practical advice and insights. Check
out [The Fast TikTok API](https://apify.com/novi/fast-tiktok-api).

![Fast TikTok API]({{ site.baseurl }}/images/fast-tiktok-api.png)

## Why Use the TREND Scrape Type?

Tracking trends is crucial for marketers, content creators, and anyone interested in understanding current cultural
moments. The `TREND` feature provides a direct line to this information, allowing you to:

* **Identify emerging trends:** Catch viral content early.
* **Analyze popular content:** Understand what's resonating with audiences.
* **Inform content strategy:** Create content that aligns with current trends.
* **Conduct market research:**  Gain insights into consumer interests.

## Key Input Fields for TREND

When you select `TREND` in the "Choose Scrapping Type" (`type`) field, the relevant input fields are simplified compared
to `SEARCH`. Here's what you need to know:

1. **`type` (Choose Scrapping Type):**  Set this to `TREND`.

2. **`region` (Target country):** This is *crucially important* for `TREND`. TikTok trends vary significantly by region.
   Specify the two-letter country code (e.g., `US`, `GB`, `JP`) for the area you're interested in. Unlike with `SEARCH`,
   you *should* specify a region for `TREND`. If you don't the default is `GB`.

3. **`limit` (limit):**  Found under the expandable "Number of videos per search" section (click the `>` arrow). This
   sets a *soft limit* on the number of trending videos returned. The API will retrieve *at least* this many videos.

4. **`isUnlimited` (Is Unlimited):** Also under "Number of videos per search." Setting this to `true` will attempt to
   retrieve *all* trending videos for the specified region. **Use with caution:** this can take a considerable amount of
   time and may encounter rate limits. It is recommend start with `limit`.

You'll notice that `keyword`, `sortType`, `publishTime`, `url`, and `urls` are *not* used with the `TREND` type. The API
is retrieving currently trending content, so these filtering options aren't applicable.

**Here's how the relevant part of the input UI looks (you'll need to click to expand "Number of videos per search"):**

![Fast TikTok API Music]({{ site.baseurl }}/images/fast-tiktok-api-trend.png)

## Example Scenarios

**Example 1: Get the top 25 trending videos in the United States.**

* `type`: `TREND`
* `region`: `US`
* `limit`: `25`
* `isUnlimited`: `false`

**Example 2:  Retrieve at least 50 trending videos in Germany.**

* `type`: `TREND`
* `region`: `DE`
* `limit`: `50`
* `isUnlimited`: `false`

**Example 3: Try to get *all* trending videos in the United Kingdom (use with caution!).**

* `type`: `TREND`
* `region`: `GB`
* `limit`: `100` (Start with a limit for testing, even if you plan to use `isUnlimited`)
* `isUnlimited`: `true` (Only after initial testing)

## Tips and Considerations

* **Region is Key:**  Trends are highly localized. Always specify a `region` for meaningful results.
* **`limit` vs. `isUnlimited`:**  Start with a reasonable `limit` to get a feel for the data and the API's response
  time. Only use `isUnlimited` if you genuinely need a comprehensive list and are prepared for potentially long
  processing times.
* **Trends are Dynamic:**  Remember that trends change rapidly. The results you get today might be very different from
  the results you get tomorrow.
* **Experiment:** Try different regions and limits to see how the results vary.

## Conclusion

The Fast TikTok API's `TREND` input provides a simple yet powerful way to stay on top of what's happening on TikTok. By
understanding the few key input fields and using them strategically, you can access valuable trend data for your
projects. Happy trend hunting!



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

