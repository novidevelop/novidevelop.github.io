---
layout: post
title: "Tutorial: Scraping User Videos with Fast TikTok API"
date: 2025-03-10 01:20:00 +0000
categories: [ tiktok, data-extraction, tutorial, user-profile ]
---

The Fast TikTok API's `USER` scraping type allows you to gather data about a specific TikTok user's profile and videos.
This is invaluable for competitor analysis, influencer identification, and understanding individual content strategies.
This guide will focus on using the API's input UI for the `USER` type. More details
about [The Fast TikTok API](https://apify.com/novi/fast-tiktok-api?fpr=7hce1m).

![Fast TikTok API]({{ site.baseurl }}/images/fast-tiktok-api.png)

## Why Scrape User Data?

Scraping user data provides several benefits:

* **Competitor Analysis:** Track the content, engagement, and growth of your competitors.
* **Influencer Identification:** Find relevant influencers in your niche.
* **Content Strategy Insights:**  Understand what types of videos perform well for specific users.
* **Audience Research:**  Indirectly gain insights into a user's audience by analyzing their content.

## Key Input Fields for USER

When you select `USER` in the "Choose Scrapping Type" (`type`) field, the following input fields are used:

1. **`type` (Choose Scrapping Type):**  Set this to `USER`.

2. **`url` (Start URL to scrape):**  This is where you provide the URL of the TikTok user's profile page. The format
   is `https://www.tiktok.com/@username`. For example, for the user `@tiktok`, the URL would
   be `https://www.tiktok.com/@tiktok`.  *Accuracy is crucial here.* You can find this URL by visiting the user's
   profile on the TikTok website and copying it from your browser.

3. **`limit` (limit):** Located under the "Number of videos per search" section (expand it by clicking the `>`). This is
   a *soft limit*, meaning the API will retrieve *at least* this many videos from the user's profile.

4. **`isUnlimited` (Is Unlimited):**  Also under "Number of videos per search." Setting this to `true` will attempt to
   retrieve *all* videos from the user's profile.  **Exercise caution:** Users with a large number of videos will take a
   long time to scrape, and you may encounter rate limits.

**Input UI Visualization:**

![Fast TikTok API Music]({{ site.baseurl }}/images/fast-tiktok-api-user.png)

Note that `region`, `keyword`, `sortType`, `publishTime`, and `urls` are *not* used with the `USER` type.

## Example Scenarios

**Example 1: Get the first 50 videos from the user @natgeo.**

1. Find the user's profile URL: Go to TikTok and find Charli D'Amelio's profile. The URL
   is `https://www.tiktok.com/@natgeo`.
2. Configure the input:
    * `type`: `USER`
    * `url`: `https://www.tiktok.com/@natgeo`
    * `limit`: `50`
    * `isUnlimited`: `false`

**Example 2:  Try to get *all* videos from a user with a very large number of uploads (use with caution!).**

* `type`: `USER`
* `url`: `https://www.tiktok.com/@someuser` (replace with the actual URL)
* `limit`: `100` (Start with a limit, even if you intend to use `isUnlimited`)
* `isUnlimited`: `true` (Only after testing with a limit)

**Example 3: Get at least 20 videos from a new user @MyNewAccount.**

* `type`: `USER`
* `url`: `https://www.tiktok.com/@MyNewAccount`
* `limit`: `20`
* `isUnlimited`: `false`

## Tips and Considerations

* **Verify the Profile URL:**  Double-check the URL to ensure it's correct. A typo will result in errors.
* **`isUnlimited` Risks:** Be extremely cautious with `isUnlimited`, especially for users with a large number of videos.
  Always start with a `limit` to test.
* **User Privacy:**  Be mindful of user privacy and ethical considerations when scraping user data. Only collect
  publicly available information and use it responsibly.
* **Rate Limits:**  TikTok has rate limits to prevent abuse. If you're making a large number of requests, you may need
  to implement delays or use multiple API keys.


## Scrape any TikTok data you need with dedicated scrapers

If you want to get specific data from TikTok, Twitter, you can use the scrapers below. Each scraper is made to help you get
different kinds of TikTok data, like hashtags, search results, profiles, or everything at once. You can look at them to
see which one you need.

| [🎹️ Fast TikTok API](https://apify.com/novi/fast-tiktok-api?fpr=7hce1m)            | [📹️ TikTok Trend API](https://apify.com/novi/tiktok-trend-api?fpr=7hce1m)         | [🔍️ TikTok Search API](https://apify.com/novi/tiktok-search-api?fpr=7hce1m)             |
|:-------------------------------------------------------------------------|:------------------------------------------------------------------------|:------------------------------------------------------------------------------|
| [🧛️ TikTok User API](https://apify.com/novi/tiktok-user-api?fpr=7hce1m)            | [🧛️ TikTok User Info API](https://apify.com/novi/tiktok-user-info-api?fpr=7hce1m) | [#️ TikTok Hashtag API](https://apify.com/novi/tiktok-hashtag-api?fpr=7hce1m)            |
| [🛍️ TikTok Shop API](https://apify.com/novi/tiktok-shop-scraper?fpr=7hce1m)        | [👤️ TikTok Followers API](https://apify.com/novi/tiktok-followers-api?fpr=7hce1m) | [⚡️ TikTok Scraper (pay-per-result)](https://apify.com/xtdata/tiktok-scraper) |
| [💬 TikTok Comment API](https://apify.com/novi/tiktok-comment-api?fpr=7hce1m)       | [🎶 TikTok Music API](https://apify.com/novi/tiktok-sound-api)          | [🎶 TikTok Music Trend API](https://apify.com/novi/tiktok-music-trend-api)    |
| [🐦 Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper) |                                                                         |                                                                               |

