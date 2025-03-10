---
layout: post
title: "Tutorial: Scraping Specific TikTok Videos with Fast TikTok API"
date: 2025-03-10 01:40:00 +0000
categories: [ tiktok, data-extraction, tutorial, video ]
---

The Fast TikTok API's `VIDEO` scraping type is designed for retrieving data about *specific* TikTok videos. Unlike the
other scraping types that search or browse, this type targets individual videos based on their URLs or IDs. This is
perfect for analyzing particular pieces of content, tracking their performance, or gathering data for research.

![Fast TikTok API]({{ site.baseurl }}/images/fast-tiktok-api.png)

## When to Use the VIDEO Scrape Type

Use the `VIDEO` type when:

* You have a list of specific TikTok video URLs or IDs that you want to analyze.
* You need to track the performance of particular videos over time.
* You're conducting research that requires data from a defined set of videos.

## Key Input Fields for VIDEO

When you select `VIDEO` in the "Choose Scrapping Type" (`type`) field, the key fields are:

1. **`type` (Choose Scrapping Type):** Set this to `VIDEO`.

2. **`urls` (URLs/IDs of videos):** This is where you provide the list of video URLs or IDs. You can enter multiple URLs
   or IDs, one per line. The API accepts both full URLs (
   e.g., `https://www.tiktok.com/@username/video/1234567890123456789`) and just the video ID (
   e.g., `1234567890123456789`).  *Make sure each URL or ID is on a separate line.*

**Important:**  The `limit` and `isUnlimited` fields are *not* used with the `VIDEO` type. The API will retrieve data
for *all* the provided URLs/IDs.

**UI Visualization:**

![Fast TikTok API Music]({{ site.baseurl }}/images/fast-tiktok-api-user.png)

Notice that `region`, `keyword`, `sortType`, `publishTime`, `url` are *not* used with the `VIDEO` type.

## Example Scenarios

**Example 1: Get data for three specific videos.**

* `type`: `VIDEO`
* `urls`:
 `["https://www.tiktok.com/@natgeo/video/7479779154083269930", "https://www.tiktok.com/@natgeo/video/7479447979548658990", "7479118671969930538"]`

**Example 2:  Retrieve data for a single video using its ID.**

* `type`: `VIDEO`
* `urls`:
  ```
  7479779154083269930
  ```

**Example 3:  Retrieve data for a list of videos.**

* `type`: `VIDEO`
* `urls`:
  ```
  https://www.tiktok.com/@user1/video/7123456789012345678
  https://www.tiktok.com/@user2/video/7234567890123456789
  7345678901234567890
  7345678901234567891
  ```

## Tips and Best Practices

* **URL/ID Accuracy:** Double-check every URL and ID to avoid errors.
* **Bulk Input:** Use the "Bulk edit" feature (if available in the UI) to paste a large list of URLs/IDs more easily.
* **Error Handling:** Be prepared to handle potential errors. A video might be deleted, private, or otherwise
  unavailable. Your code should gracefully handle these situations.

## Conclusion

The Fast TikTok API's `VIDEO` input offers a precise way to target and retrieve data from specific TikTok videos. By
providing the correct URLs or IDs, you can efficiently gather the information you need for analysis and tracking. This
method is straightforward but powerful when you have a defined set of videos to examine.
