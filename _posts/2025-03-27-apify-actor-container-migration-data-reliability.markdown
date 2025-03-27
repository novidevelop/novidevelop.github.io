---
layout: post
title: "Navigating Container Migration in Apify Actor Runs: Ensuring Data Reliability"
date: 2025-03-27 10:00:00 +0700
categories: [apify, actors, web scraping, data reliability, state persistence, container migration]
tags: [apify, actors, web scraping, data reliability, state persistence, container migration, tiktok scraper, data extraction]
permalink: /blog/apify-actor-container-migration-data-reliability
---

## Navigating the Rapids: Handling Container Migration in Apify Actor Runs

Apify's platform empowers developers to build and deploy powerful web scraping and automation tools called Actors. These Actors, running in containers, can perform complex tasks. However, long-running Actor jobs may encounter a unique challenge: container migration.

**The Challenge: Container Migration and State Persistence**

Apify's infrastructure intelligently manages resources, sometimes necessitating the migration of running containers between servers. This migration, while essential for platform stability and performance, can lead to unexpected issues if not handled correctly.

The primary concern is **state persistence**. If an Actor's state—the data it has processed and the progress it has made—is not properly saved, migration can result in:

* **Duplicated Data:** The Actor might re-process data it has already handled, leading to redundant results.
* **Timeouts and Errors:** Migration can disrupt ongoing operations, potentially causing timeouts or other errors.
* **Increased Costs:** Duplicated work translates to wasted computation time, inflating your Apify usage costs.

**The Solution: Robust State Management**

To mitigate these issues, developers must implement robust state management within their Actors. Apify provides the key-value store, a reliable mechanism for persisting state. Here's how to ensure your Actors handle container migration gracefully:

1.  **Periodic State Saving:** Implement logic to periodically save the Actor's current state to the key-value store. This involves serializing relevant data and storing it with a unique key.
2.  **State Loading on Startup:** Upon Actor startup or restart, check the key-value store for saved state. If found, load and deserialize the data to resume from the last known point.
3.  **Event Handling:** Utilize Apify's event system to listen for the `migrating` event. This allows you to trigger state saving precisely when needed.
4.  **Error Handling:** Implement robust error handling to manage potential issues during state saving and loading.

**Example Code:**

```javascript
import { Actor, log } from 'apify';

await Actor.init();

let crawlingState = await Actor.getValue('my-crawling-state') || { processedUrls: [] };

Actor.on('migrating', async () => {
    log.info('Actor migrating, saving state...');
    await Actor.setValue('my-crawling-state', crawlingState);
    log.info('State saved.');
});

// Your scraping logic here, updating crawlingState.processedUrls

// Example:
crawlingState.processedUrls.push("[https://example.com/page1](https://example.com/page1)");

await Actor.setValue('my-crawling-state', crawlingState);

await Actor.exit();
```

**Explanation of the Code:**

* **`Actor.on('migrating', async () => { ... });`**: This registers a listener for the `migrating` event. When the Actor is about to migrate, this function is executed.
* **`Actor.setValue('my-crawling-state', crawlingState);`**: This line saves the current state of the actor, in this case the `crawlingState` object, to the key-value store. This ensures that if the actor is restarted after migration, it can resume from where it left off.
* **`Actor.getValue('my-crawling-state') || { processedUrls: [] };`**: This line retrieves the saved state from the key-value store. If no saved state exists, it initializes the `crawlingState` object with an empty `processedUrls` array.
* **`log.info()`**: Used for logging information, helpful for debugging.
* **`await Actor.init()` and `await Actor.exit()`**: Required functions to initialize and close the actor.

**Our Actors: Designed for Reliability**

We, at Novi Team, understand the importance of reliable data extraction. That's why we've meticulously designed our Actors, like the **TikTok Profile Scraper**, to handle container migration seamlessly.

Our **[TikTok Profile Scraper](https://apify.com/xtdata/tiktok-user-information-scraper)**, which can scrape multiple urls, userIds and usernames, utilizes best practices for state persistence. It includes:

* **Efficient state saving:** To prevent data loss.
* **Robust error handling:** To maintain smooth operations.
* **optimized code:** To reduce run time, and therefore reduce the likelyhood of a migration happening during a run.

By implementing these strategies, we ensure that our users:

* Avoid duplicated data.
* Experience fewer timeouts and errors.
* Minimize unnecessary costs.

**Conclusion**

Container migration is a reality of cloud-based platforms like Apify. By understanding the challenges and implementing robust state management, developers can build reliable and efficient Actors. Our TikTok Profile Scraper exemplifies this approach, providing users with a seamless and cost-effective data extraction experience.

We encourage you to leverage our tools and best practices to optimize your Apify workflows.

#### If you want to get specific data from TikTok

You can use the scrapers below. Each scraper is made to help you get different kinds of TikTok data, like hashtags,
search results, profiles, or everything at once. You can look at them to see which one you need.

| [🎹️ Fast TikTok API](https://apify.com/novi/fast-tiktok-api)      | [📹️ TikTok Trend API](https://apify.com/novi/tiktok-trend-api)         | [🔍️ TikTok Search API](https://apify.com/novi/tiktok-search-api)             |
|:-------------------------------------------------------------------|:------------------------------------------------------------------------|:------------------------------------------------------------------------------|
| [🧛️ TikTok User API](https://apify.com/novi/tiktok-user-api)      | [🧛️ TikTok User Info API](https://apify.com/novi/tiktok-user-info-api) | [#️ TikTok Hashtag API](https://apify.com/novi/tiktok-hashtag-api)            |
| [🛍️ TikTok Shop API](https://apify.com/novi/tiktok-shop-scraper)  | [👤️ TikTok Followers API](https://apify.com/novi/tiktok-followers-api) | [⚡️ TikTok Scraper (pay-per-result)](https://apify.com/xtdata/tiktok-scraper) |
| [💬 TikTok Comment API](https://apify.com/novi/tiktok-comment-api) | [🎶 TikTok Music API](https://apify.com/novi/tiktok-sound-api)          | [🎶 TikTok Music Trend API](https://apify.com/novi/tiktok-music-trend-api)    |


