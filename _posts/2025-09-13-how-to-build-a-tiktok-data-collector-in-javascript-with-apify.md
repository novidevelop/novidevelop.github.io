---
layout: post
title: "How to Build a TikTok Data Collector in JavaScript with Apify"
date: 2025-09-13 00:15:12 +0700
categories: [ tiktok, data-extraction, tutorial, trends ]
tags:
  - "TikTok Scraper"
  - "TikTok data extraction"
  - "Download TikTok videos"
  - "apify"
  - "web-scraping"
---

In our [previous post](/2025/09/13/how-to-scrape-tiktok-data-a-guide-to-the-ultimate-tiktok-scraper.html), we explored how to use the **[TikTok Scraper](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m)** directly from the Apify Console. While using the console is perfect for manual runs, the true power of automation is unlocked when you integrate scraping into your own applications.

Today, we'll take the next step and show you how to build a simple data collector in JavaScript. Using the **Apify API client for JavaScript**, you can run the TikTok Scraper, configure its input, and retrieve the data programmatically—all within a Node.js environment.

### Prerequisites

Before we start, make sure you have the following:

* An **Apify account**. If you don't have one, you can [sign up for free](https://apify.com?fpr=7hce1m).
* Your **Apify API token**. You can find this in your Apify Console under **Settings > Integrations**.
* **Node.js and npm** installed on your machine.

---

### Step 1: Set Up Your Node.js Project

First, let's create a new project. Open your terminal, create a new folder, and initialize it as a Node.js project.

```bash
mkdir tiktok-collector
cd tiktok-collector
npm init -y
````

Next, we need to install the Apify API client, which allows our script to communicate with the Apify platform.

```bash
npm install apify-client
```

-----

### Step 2: The Data Collector Script

Now, create a new file named `index.js` and paste the following code into it. We'll break down what each part does right after.

```javascript
import { ApifyClient } from 'apify-client';

// Initialize the ApifyClient with your Apify API token
// Replace the '<YOUR_API_TOKEN>' with your token from [https://console.apify.com/settings/integrations](https://console.apify.com/settings/integrations)
const client = new ApifyClient({
    token: '<YOUR_API_TOKEN>',
});

// Prepare the input for the TikTok Scraper Actor
const input = {
    "startUrls": [
        "[https://www.tiktok.com/@billieeilish/video/7050551461734042926](https://www.tiktok.com/@billieeilish/video/7050551461734042926)",
        "[https://www.tiktok.com/@gordonramsayofficial](https://www.tiktok.com/@gordonramsayofficial)"
    ],
    "keywords": [
        "Artificial Intelligence",
        "podcast"
    ],
    "maxItems": 10,
    "location": "US"
};

async function runScraper() {
    console.log('🚀 Starting the TikTok Scraper...');

    // Run the Actor and wait for it to finish
    const run = await client.actor("novi/tiktok-scraper-ultimate").call(input);

    console.log('✅ Scraper finished. Fetching results...');
    
    // Fetch and print Actor results from the run's dataset
    console.log(`💾 You can view the full dataset here: https://console.apify.com/storage/datasets/${run.defaultDatasetId}`);
    const { items } = await client.dataset(run.defaultDatasetId).listItems();
    
    console.log('--- Results ---');
    items.forEach((item) => {
        // Print out the video URL and the author's nickname
        console.log(`- Video URL: ${item.share_url}, Author: ${item.author.nickname}`);
    });
    console.log('----------------');
}

runScraper();
```

#### Code Breakdown

1.  **Initialize the Client**: We first import `ApifyClient` and create a new instance. You must replace `<YOUR_API_TOKEN>` with your actual token. **Remember to keep your API token secret\!**
2.  **Prepare Actor Input**: The `input` object is a direct programmatic equivalent of filling out the input form in the Apify Console. You can add any valid input fields here, such as `startUrls`, `keywords`, `maxItems`, or `location`.
3.  **Run the Actor**: The `client.actor("novi/tiktok-scraper-ultimate").call(input)` line is the core of our script. It tells the Apify platform to run the actor named `novi/tiktok-scraper-ultimate` with the input we defined. We use `await` because this process is asynchronous and can take some time.
4.  **Fetch the Results**: Once the actor run is complete, the `run` object contains information about the run, including the `defaultDatasetId` where the results are stored. We use this ID to fetch the `items` from the dataset.
5.  **Process the Data**: The final `forEach` loop iterates through the results. In this example, we're just printing the video URL and author's nickname, but this is where you could save the data to a database, perform analysis, or feed it into another system.

-----

### Step 3: Run Your Collector

Save the `index.js` file and run it from your terminal:

```bash
node index.js
```

You should see an output similar to this:

```
🚀 Starting the TikTok Scraper...
✅ Scraper finished. Fetching results...
💾 You can view the full dataset here: [https://console.apify.com/storage/datasets/YOUR_DATASET_ID](https://console.apify.com/storage/datasets/YOUR_DATASET_ID)
--- Results ---
- Video URL: [https://www.tiktok.com/@billieeilish/video/7050551461734042926](https://www.tiktok.com/@billieeilish/video/7050551461734042926), Author: Billie Eilish
- Video URL: [https://www.tiktok.com/@gordonramsayofficial/video/SOME_VIDEO_ID](https://www.tiktok.com/@gordonramsayofficial/video/SOME_VIDEO_ID), Author: Gordon Ramsay
...and so on
----------------
```

### What's Next?

You've successfully built and run a TikTok data collector from your own machine\! This script is the foundation for countless automation possibilities. You could:

* Store the results in a database like **MongoDB** or **PostgreSQL**.
* Wrap this logic in an **Express.js** server to create your own API.
* Set up a **cron job** on a server to run this script on a schedule, continuously monitoring keywords or profiles.

By integrating the TikTok Scraper into your applications, you move from manual data collection to powerful, automated data pipelines. Happy coding\!
