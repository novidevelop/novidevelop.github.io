---
layout: post
title: "Tutorial: Extracting TikTok Videos by Music with Fast TikTok API"
date: 2025-03-10 01:30:00 +0000
categories: [ tiktok, data-extraction, tutorial, music, sound ]
---

[The Fast TikTok API](https://apify.com/novi/fast-tiktok-api)'s `MUSIC` scraping type allows you to retrieve videos that use a specific sound or song. This is
incredibly useful for identifying trends, tracking the use of particular audio clips, and understanding how music
contributes to viral content. This guide focuses on the API's input UI for the `MUSIC` type. Learn more about [The Fast TikTok API](https://apify.com/novi/fast-tiktok-api).

![Fast TikTok API]({{ site.baseurl }}/images/fast-tiktok-api.png)

## Why Scrape by Music?

Music plays a vital role on TikTok. Scraping by music provides insights that other methods might miss:

* **Trend Identification:**  Discover trending sounds and the videos associated with them.
* **Audio Usage Analysis:**  See how a specific song or sound is being used across different videos and contexts.
* **Viral Content Research:**  Understand the relationship between music and virality.
* **Campaign Tracking:**  Monitor the use of a specific sound associated with a marketing campaign.

## Key Input Fields for MUSIC

When you choose `MUSIC` as the "Choose Scrapping Type" (`type`), these are the key fields:

1. **`type` (Choose Scrapping Type):**  Set this to `MUSIC`.

2. **`url` (Start URL to scrape):**  This is the URL of the TikTok music page for the specific sound. The format
   is `https://www.tiktok.com/music/[music-name-and-id]`. For example, a URL might look
   like `https://www.tiktok.com/music/original-sound-6845863000419518214`. *You need the correct URL for the API to
   work.*  You can find this URL by:
    * Finding a video that uses the sound on the TikTok website.
    * Clicking on the spinning record icon or the sound name at the bottom of the video (on the website, *not* the app).
    * Copying the URL from your browser's address bar.

3. **`limit` (limit):**  Found under "Number of videos per search" (click the `>` to expand). This acts as a *soft
   limit*, meaning the API will retrieve *at least* this many videos using the specified music.

4. **`isUnlimited` (Is Unlimited):** Also under "Number of videos per search."  If set to `true`, the API will attempt
   to retrieve *all* videos associated with the music.  **Use with caution!** Some sounds are used in a *huge* number of
   videos, leading to long processing times and potential rate limits.

**UI Snapshot:**

![Fast TikTok API Music]({{ site.baseurl }}/images/fast-tiktok-api-music.png)

The fields `region`, `keyword`, `sortType`, `publishTime`, and `urls` are *not* used with the `MUSIC` type.

## Example Scenarios

**Example 1: Get the first 25 videos using a specific trending sound.**

1. Find a video using the sound.
2. Click the sound's name or the spinning record icon on the TikTok website.
3. Copy the URL from your browser (e.g., `https://www.tiktok.com/music/some-trending-sound-123456789`).
4. Configure the input:
    * `type`: `MUSIC`
    * `url`: `https://www.tiktok.com/music/some-trending-sound-123456789` (replace with the actual URL)
    * `limit`: `25`
    * `isUnlimited`: `false`

**Example 2:  Try to get *all* videos using a very popular sound (use with extreme caution!).**

* `type`: `MUSIC`
* `url`: `https://www.tiktok.com/music/a-very-popular-sound-987654321` (replace with the actual URL)
* `limit`: `100` (Start with a limit, even if you plan to use `isUnlimited`)
* `isUnlimited`: `true` (Only after testing!)

**Example 3: Get at least 150 video using specific sound.**

* `type`: `MUSIC`
* `url`: `https://www.tiktok.com/music/original-sound-example-1234`
* `limit`: `150`
* `isUnlimited`: `false`

## Tips and Precautions

* **Finding the Music URL:**  The most reliable way is through the TikTok website, as described above. Don't try to
  guess the URL.
* **`isUnlimited` – Handle with Care:**  Be very cautious with this option. Always start with a `limit` to gauge the
  volume of videos and the API's response time.
* **Sound Variations:** Sometimes, slightly different versions of the same sound exist. Make sure you're using the exact
  URL for the sound you want to track.

## Conclusion

The Fast TikTok API's `MUSIC` input provides a valuable way to analyze the impact of sound on TikTok. By carefully
obtaining the correct music URL and managing the `limit` and `isUnlimited` settings, you can unlock valuable insights
into trends, audio usage, and viral content. Remember to start small and test your queries thoroughly.
