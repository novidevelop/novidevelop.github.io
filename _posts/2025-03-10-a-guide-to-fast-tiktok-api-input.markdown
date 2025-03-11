---
layout: post
title: "Tutorial: A guide to Fast TikTok API's SEARCH Input"
date: 2025-03-10 01:00:00 +0000
categories: [ tiktok, data-extraction, tutorial ]
---

I've been digging into the Fast TikTok API lately, and the `SEARCH` feature is a real game-changer. It lets you tap into
the massive stream of TikTok content in a structured way. I wanted to share a practical guide, based on my experience,
to help you get the most out of it. This isn't just a rehash of
the [official tutorial](https://novidevelop.github.io/tiktok/scraper/2025/02/24/apify-tiktok-search-api-download-videos-by-keyword.html) (
though you should definitely check that out too!), but rather a hands-on walkthrough focused on the search input.

## Why the Search Feature Rocks (and Why You Should Care)

The Fast TikTok API opens up a lot of possibilities, from tracking trends to understanding what resonates with
audiences. The `SEARCH` functionality is your key to unlocking this. Instead of manually scrolling through TikTok, you
can programmatically find videos based on keywords, location, popularity, and more. This can save you *hours* and give
you data-driven insights you wouldn't get otherwise.

**Below is a screenshot of the Fast TikTok API input interface:**

![Fast TikTok API Search Feature]({{ site.baseurl }}/images/search-overview.png)

As you can see, it's pretty straightforward. The interface is well-organized, but there are a few quirks and tricks that
I'll point out. Let's dive into the important fields for when you've selected `SEARCH` as your "Scrapping Type."

## Key Input Fields for SEARCH - Demystified

Once you've chosen `SEARCH` from the "Choose Scrapping Type" dropdown, you'll be working with these key fields:

1. **`type` (Choose Scrapping Type):** This is your starting point. Make sure it's set to `SEARCH`!

2. **`keyword` (Keyword):** The bread and butter of your search. What are you actually looking for? You can use single
   words ("viral") or phrases ("funny cats", "data science trends"). Don't be afraid to experiment!

3. **`region` (Target country):** Want to focus on a specific country? Use the two-letter country code here (e.g., `US`
   for United States, `GB` for Great Britain, `JP` for Japan). Leave it blank for a global search.  *Pro tip:
   Sometimes, leaving it blank gives you surprisingly different results than explicitly selecting "US," even if most of
   your target audience is in the US. It's worth testing both ways.*
    * **Note**: The UI image shows "United Kingdom" as selected. The `region` code is `GB`.

4. **`limit` (limit):** You'll find this one under the expandable "Number of videos per search" section (click the
   little `>` arrow). This is a *soft limit* – think of it as a "minimum" rather than a strict maximum. The API will
   stop *after* it finds *at least* this many videos. This is super important for controlling how long your script runs
   and how much data you pull.
    * **Example:** Setting it to `20` means you'll get *at least* 20 videos, but it might be 23 or 25.

5. **`isUnlimited` (Is Unlimited):** Also hiding under that "Number of videos per search" section. This is the "go wild"
   option. If you set it to `true`, the API will try to grab *everything*.  **Be warned:** this can take a *long* time,
   and you might run into TikTok's rate limits. I usually start with a `limit` to test my query and only
   use `isUnlimited` when I'm sure I need *all* the data.

6. **`sortType` (Sort Type):**  Expand the "Filters (for SEARCH)" section (again, click that `>`). This lets you choose
   *how* TikTok ranks the results. Your choices are:
    * `0`: **Relevance** (This is TikTok's secret sauce – their default ranking algorithm).
    * `1`: **Most Liked** (Pretty self-explanatory – sorted by the number of likes).
    * `2`: **Most Recent** (Newest videos first). *I often use this to find emerging trends.*

7. **`publishTime` (Publish Time):**  Also under the "Filters (for SEARCH)" section. This lets you narrow down your
   search by the video's publication date. It's great for finding content related to a specific event or time period.
    * `ALL_TIME`: No time limit.
    * `YESTERDAY`: Videos from yesterday.
    * `WEEK`: Videos from the past week.
    * `MONTH`: Videos from the past month.
    * `THREE_MONTH`:  The last three months.
    * `SIX_MONTH`: The last six months.

8. **`url` Start URL to Scrape.**: You won't need this for `SEARCH`. It's for other scraping types like HASHTAG, USER,
   MUSIC, and COMMENT.

9. **`urls` URLS/IDs of videos.**:  This is only used when you're scraping specific videos (`VIDEO` type), so you can
   ignore it for `SEARCH`.

## Putting it All Together: Example Scenarios (and Some Gotchas)

Let's walk through some real-world examples, and I'll share some things I've learned along the way:

**Example 1:  Find 20 of the most relevant "viral" videos in the UK.**

This is a pretty standard search. Here's how you'd set it up:

* `type`: `SEARCH`
* `keyword`: `viral`
* `region`: `GB`
* `limit`: `20`
* `isUnlimited`: `false`
* `sortType`: `0` (Relevance)
* `publishTime`: `ALL_TIME`

**Example 2: Find the 50 most-liked "funny cat" videos from the UK, published in the last week.**

This one's a bit more specific, targeting a niche and a time frame:

* `type`: `SEARCH`
* `keyword`: `funny cats`
* `region`: `GB`
* `limit`: `50`
* `isUnlimited`: `false`
* `sortType`: `1` (Most Liked)
* `publishTime`: `WEEK`

**Example 3: Get all "trending now" videos from any region, sorted by most recent. (The "Firehose" Approach)**

This is where you might use the `isUnlimited` option, but be careful!

* `type`: `SEARCH`
* `keyword`: `trending now`
* `region`: (Leave blank – we want *everything*)
* `limit`: `100` (I'd *definitely* start with a limit to make sure things are working)
* `isUnlimited`: `true` (Only after you've tested with a limit!)
* `sortType`: `2` (Most Recent)
* `publishTime`: `ALL_TIME`

**Here's what a filled-out input form might look like for Example 1:**

![Fast TikTok API Example Search Input]({{ site.baseurl }}/images/search-input.png)

This image would show the `SEARCH` type selected, the `keyword` field filled with "data science," the `region` set to "
US," and the `limit` at 20 (or 30, to match the example text). You'll also see the "Number of videos per search" and "
Filters (for SEARCH)" sections expanded, showing the `isUnlimited`, `sortType`, and `publishTime` settings. This gives
you a clear visual of a real configuration.

**A Word of Warning:** TikTok's search can be a bit... unpredictable. Sometimes, you'll get slightly different results
even with the same parameters. This is just the nature of the platform. Don't be afraid to tweak your settings and
experiment!

## Troubleshooting and Tips

* **Stuck? Start Small:** If you're not getting any results, or you're getting unexpected ones, try simplifying your
  query. Start with a common keyword, a small `limit`, and no region restrictions. Then, gradually add more filters.
* **Keyword Variations are Key:** "Data Science" might give you different results than "data science tips" or "learn
  data science." Try different variations!
* **The `isUnlimited` Dilemma:**  It's tempting to use it all the time, but resist! Start with a reasonable `limit` and
  only use `isUnlimited` if you absolutely need *every single video*.
* Double Check region code. A wrong region code can leads to unexpected results.

## Going Further

This guide covers the basics of using the `SEARCH` input in the Fast TikTok API. There's a lot more you can do with the
API, including downloading videos (without watermarks!), scraping user profiles, and more. I might cover those in future
posts if there's interest! Let me know what you'd like to see. And definitely explore
the [official tutorial](https://novidevelop.github.io/tiktok/scraper/2025/02/24/apify-tiktok-search-api-download-videos-by-keyword.html) –
it's a great resource. Happy scraping!

| [🎹️ Fast TikTok API](https://apify.com/novi/fast-tiktok-api)     | [📹️ TikTok Trend API](https://apify.com/novi/tiktok-trend-api)         | [🔍️ TikTok Search API](https://apify.com/novi/tiktok-search-api)  |
|:------------------------------------------------------------------|:------------------------------------------------------------------------|:-------------------------------------------------------------------|
| [🧛️ TikTok User API](https://apify.com/novi/tiktok-user-api)     | [🧛️ TikTok User Info API](https://apify.com/novi/tiktok-user-info-api) | [#️ TikTok Hashtag API](https://apify.com/novi/tiktok-hashtag-api) |
| [🛍️ TikTok Shop API](https://apify.com/novi/tiktok-shop-scraper) | [👤️ TikTok Followers API](https://apify.com/novi/tiktok-followers-api) |                                                                    |


