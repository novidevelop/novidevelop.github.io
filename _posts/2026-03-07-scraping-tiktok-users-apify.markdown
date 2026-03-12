---
layout: post
title: "How to Scrape TikTok User Profiles Using the Apify SDK"
date: 2026-03-07 10:00:00 +0700
description: "Scrape TikTok user profiles by keyword using the Apify SDK and Node.js. Filter by followers, verification status, and region with the TikTok User Search Actor."
categories: [web-scraping, nodejs]
tags: [apify, tiktok, javascript, automation]
author: "Novi Team - Plato"
---

If you are building a marketing tool, doing social media research, or just want to find specific content creators on TikTok, web scraping is your best friend. However, building a scraper from scratch that handles pagination, proxies, and API changes can be a nightmare.

In this tutorial, I will show you how to effortlessly scrape TikTok user profiles based on keyword searches using the ready-made [TikTok User Search Actor](https://apify.com/novi/tiktok-user-search?fpr=7hce1m) and Node.js.

## What We Are Building

We will write a simple Node.js script that connects to the Apify platform. It will trigger the TikTok User Search Actor, pass our search configuration (like keywords, follower ranges, and profile types), wait for the job to finish, and finally download the extracted data.

## Prerequisites

Before we start, make sure you have:
1. **Node.js** installed on your machine.
2. An **Apify account** and an API Token (you can find this in your Apify Console under Settings > Integrations).

## Step 1: Set Up the Project

First, create a new directory for your project and initialize it:

```bash
mkdir tiktok-scraper-demo
cd tiktok-scraper-demo
npm init -y

```

Next, install the official Apify API client for JavaScript:

```bash
npm install apify-client

```

## Step 2: Write the Scraper Script

Create a file named `index.js` and open it in your favorite code editor. We will use the `apify-client` to interact with the Actor.

Here is the complete code:

```javascript
import { ApifyClient } from 'apify-client';

// Initialize the ApifyClient with your API token
// Best practice: Use environment variables for your token!
const client = new ApifyClient({
    token: 'YOUR_APIFY_API_TOKEN',
});

async function main() {
    console.log('🚀 Starting the TikTok User Scraper...');

    // 1. Prepare the Input Configuration
    // This perfectly matches the custom Input Schema of the Actor
    const input = {
        keywords: ["tech reviewer", "gadget unboxing"],
        maxItems: 50, // Let's grab 50 profiles
        followerRange: "OVER_100K", // Only popular creators
        profileType: "VERIFIED", // Only verified accounts
        searchField: "USERNAME",
        location: "US", // Target the United States
        includeSearchKeywords: true
    };

    try {
        // 2. Start the Actor on the Apify Cloud and wait for it to finish
        // We are calling the specific Actor ID for TikTok User Search
        const run = await client.actor('novi/tiktok-user-search').call(input);

        console.log(`✅ Actor run finished! Run ID: ${run.id}`);
        console.log('📥 Fetching the results...');

        // 3. Fetch the results from the Actor's default dataset
        const { items } = await client.dataset(run.defaultDatasetId).listItems();

        // 4. Do something with the data!
        console.log(`🎉 Successfully extracted ${items.length} user profiles.`);
        
        // Print the first item as a sample
        if (items.length > 0) {
            console.log('Sample User Data:');
            console.log(JSON.stringify(items[0], null, 2));
        }

    } catch (error) {
        console.error('❌ An error occurred:', error.message);
    }
}

main();

```

### Understanding the Input Payload

The magic happens in the `input` object. Because the Actor was built with a highly flexible schema, we can filter users right at the source:

* `keywords`: An array of search terms. The Actor will iterate through these.
* `followerRange` & `profileType`: We map these directly to TikTok's search parameters to ensure we only get high-quality leads.
* `maxItems`: The Actor will handle pagination automatically until it reaches this limit.

## Step 3: Run the Code

Run your script from the terminal:

```bash
node index.js

```

If everything is configured correctly, your script will trigger the Actor on Apify's servers. Once the scraping process is complete, it will pull the dataset directly into your local console.

## Conclusion

Using the Apify SDK and Client abstracts away the heavy lifting of web scraping. By deploying your code as an Apify Actor, you can trigger complex, paginated, and filtered data extraction jobs with just a few lines of code.

Ready to try it yourself? Check out the [TikTok User Search Actor on Apify](https://apify.com/novi/tiktok-user-search?fpr=7hce1m) and start extracting data today! Happy Scraping! 🕷️

---


## Related Articles

Want to explore more TikTok user data tools? Check out these guides:

* 📖 [Introducing the TikTok User Info API]({{ site.baseurl }}/apify/tiktok/data-extraction/2025/09/12/introducing-titkok-user-info-api-profile-data.html) — Overview of the profile data extraction tool
* 🚀 [How to Scrape TikTok User Profiles in 5 Minutes]({{ site.baseurl }}/apify/tiktok/how-to/tutorial/2025/09/12/how-to-scrape-tiktok-user-profiles-in-5-minutes.html) — No-code quickstart guide
* 🤖 [Automate TikTok Data Extraction with n8n]({{ site.baseurl }}/n8n/automation/tiktok/scraper/2026/03/12/automate-tiktok-data-extraction-with-n8n.html) — Workflow automation

***
**Looking for data extraction tools?**  
Check out our [comprehensive collection of no-code scrapers and APIs]({{ site.baseurl }}/tools/) for TikTok, X.com, and more.
