---
layout: post
title: "How to Download TikTok Videos by Keyword using Apify TikTok Search API"
date: 2025-02-24 08:15:00 +0700
categories: [ tiktok, scraper ]
---

Are you looking to download TikTok videos based on specific keywords?  This blog post will guide you through a step-by-step process using the Apify TikTok Search API and a simple JavaScript code snippet.  This method is perfect for content creators, marketers, or anyone interested in analyzing TikTok trends and content.

We will be leveraging the power of the [Apify TikTok Search API](https://apify.com/novi/tiktok-search-api) to efficiently search and download TikTok videos programmatically.  This API allows you to extract data from TikTok, including video URLs, based on your search criteria.

Let's dive into the tutorial!

## Prerequisites

Before you begin, ensure you have the following:

* **Node.js and npm (Node Package Manager) installed:** If you don't have Node.js installed, download it from the [official Node.js website](https://www.google.com/url?sa=E&source=gmail&q=https://nodejs.org/). npm usually comes bundled with Node.js.
* **Apify Account:** You'll need an Apify account to access the TikTok Search API. Sign up for a free account at [Apify](https://apify.com/).
* **Apify API Token:** Once you have an Apify account, you'll need to obtain your API token. You can find it in the [Apify Console](https://www.google.com/url?sa=E&source=gmail&q=https://console.apify.com/) under "Integrations."

![Get Personal API Token]({{ site.baseurl }}/images/get-personal-api-token.jpg)

## Step-by-Step Guide

Follow these steps to download TikTok videos using the provided code and the Apify TikTok Search API.

### Step 1: Set up your Project Environment

1.  **Create a new project directory:** Open your terminal and create a new directory for your project.

    ```bash
    mkdir tiktok-downloader
    cd tiktok-downloader
    ```

2.  **Initialize a Node.js project:**  Initialize your project with npm to manage dependencies.

    ```bash
    npm init -y
    ```

3.  **Install the Apify Client:** Install the `apify-client` library, which is necessary to interact with the Apify API.

    ```bash
    npm install apify-client fs https https path
    ```

### Step 2:  Understand and Implement the Code

Create a new JavaScript file, for example, `download_tiktok_videos.js`, and copy the following code into it. We'll break down each part of the code for better understanding.

```javascript
const {ApifyClient} = require('apify-client');
const fs = require('fs');
const https = require('https');
const path = require('path');

// Configuration
const APIFY_TOKEN = ''; // Replace with your Apify API token
const DOWNLOAD_DIR = './download/';

const client = new ApifyClient({
    token: APIFY_TOKEN,
});

/**
 * Downloads a video from a given URL to a specified destination path.
 * @param {string} videoUrl - The URL of the video to download.
 * @param {string} destinationPath - The local path where the video should be saved.
 * @returns {Promise<void>} - A promise that resolves when the download is complete.
 */
const downloadVideo = (videoUrl, destinationPath) => {
    return new Promise((resolve, reject) => {
        const fileStream = fs.createWriteStream(destinationPath);

        https.get(videoUrl, (response) => {
            if (response.statusCode !== 200) {
                const error = new Error(`Download failed with status code: ${response.statusCode}`);
                cleanupFile(destinationPath); // Clean up incomplete file
                reject(error);
                return;
            }

            response.pipe(fileStream);

            fileStream.on('finish', () => {
                fileStream.close(resolve); // Resolve promise when file stream finishes
                console.log(`Video downloaded successfully to: ${destinationPath}`);
            });

            fileStream.on('error', (err) => {
                cleanupFile(destinationPath);
                reject(err);
            });

            response.on('error', (err) => {
                cleanupFile(destinationPath);
                reject(err);
            });
        }).on('error', (err) => {
            cleanupFile(destinationPath);
            reject(err);
        });
    });
};

/**
 * Cleans up (deletes) a file at the given path if an error occurs during download.
 * @param {string} filePath - The path to the file to be deleted.
 */
const cleanupFile = (filePath) => {
    fs.unlink(filePath, (err) => {
        if (err && err.code !== 'ENOENT') { // Ignore 'file not found' errors
            console.error("Error deleting incomplete file:", err);
        }
    });
};

/**
 * Fetches TikTok videos based on search keywords using Apify Fast TikTok API and downloads them.
 * @param {string} keyword - The keyword to search for on TikTok.
 * @param {string} publishTime -  Time range for video publishing (e.g., "ALL_TIME", "PAST_24_HOURS").
 * @param {number} sortType - Sorting type for videos (0 for relevance, 1 for latest).
 * @param {string} region -  The region to filter videos by (e.g., "US", "VN").
 * @param {number} limit -  Maximum number of videos to download.
 * @returns {Promise<void>} - A promise that resolves when all videos are downloaded.
 */
const searchAndDownloadVideos = async (keyword, publishTime, sortType, region, limit) => {
    try {
        const actorCallResult = await client.actor('novi/tiktok-search-api').call({
            "keyword": keyword,
            "publishTime": publishTime,
            "sortType": sortType,
            "region": region,
            "limit": limit
        });

        const dataset = client.dataset(actorCallResult.defaultDatasetId);
        const {items} = await dataset.listItems();

        if (!fs.existsSync(DOWNLOAD_DIR)) { // Ensure download directory exists
            fs.mkdirSync(DOWNLOAD_DIR, {recursive: true});
        }

        const downloadTasks = items.map(item => {
            const videoUrl = item.aweme_info.video.play_addr.url_list[0];
            const destinationPath = path.join(DOWNLOAD_DIR, `${item.aweme_info.status.aweme_id}.mp4`);
            return downloadVideo(videoUrl, destinationPath);
        });

        await Promise.all(downloadTasks);
        console.log("Successfully downloaded all videos.");

    } catch (error) {
        console.error("Error during fetching data and processing:", error);
    }
};


// Example usage
searchAndDownloadVideos(keyword = "viral",
    publishTime = "ALL_TIME",
    sortType = 0,
    region = "US",
    limit = 40)
    .then(() => console.log("Finished processing and download video."))
    .catch(error => console.error("Video search processing failed:", error));
```

**Code Breakdown:**

* **`require(...)`:**  These lines import necessary libraries:

    * `apify-client`:  For interacting with the Apify API.
    * `fs` (File System): For file system operations like creating directories and writing files.
    * `https`: For making HTTPS requests to download videos.
    * `path`: For handling file paths.

* **`APIFY_TOKEN` and `DOWNLOAD_DIR`:**

    * `APIFY_TOKEN`:  **You need to replace the empty string `''` with your actual Apify API token.** This token authenticates your requests to the Apify API.
    * `DOWNLOAD_DIR`:  This variable defines the directory where downloaded videos will be saved. By default, it's set to `./download/` in the same directory as your script.

* **`downloadVideo(videoUrl, destinationPath)` Function:**

    * This function is responsible for downloading a single video from a given `videoUrl` and saving it to the `destinationPath`.
    * It uses `https.get()` to fetch the video content.
    * It handles potential errors during download, such as network issues or invalid URLs.
    * It uses `fs.createWriteStream()` to efficiently write the video data to a file.

* **`cleanupFile(filePath)` Function:**

    * This utility function is used to delete partially downloaded files if an error occurs during the download process. This prevents incomplete or corrupted video files from being saved.

* **`searchAndDownloadVideos(keyword, publishTime, sortType, region, limit)` Function:**

    * This is the core function that orchestrates the TikTok video search and download process.
    * It takes several parameters to customize your search:
        * `keyword`: The search term to find relevant TikTok videos (e.g., "funny cats", "dance challenge").
        * `publishTime`:  Filters videos based on their publish time.  Possible values include:
            * `ALL_TIME`
            * `PAST_24_HOURS`
            * `PAST_7_DAYS`
            * `PAST_30_DAYS`
            * `PAST_90_DAYS`
            * `PAST_YEAR`
        * `sortType`:  Determines how search results are sorted:
            * `0`:  Sorted by relevance (default).
            * `1`: Sorted by latest.
        * `region`:  Filters videos by region. Use [two-letter country codes](https://www.google.com/url?sa=E&source=gmail&q=https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) (e.g., "US" for the United States, "VN" for Vietnam).
        * `limit`:  The maximum number of videos to retrieve and download per search.
    * It uses `client.actor('novi/tiktok-search-api').call({...})` to call the Apify TikTok Search API with your specified search parameters.  **This line is where the magic happens!** It sends a request to Apify's API to perform the TikTok search.
    * It retrieves the search results from the Apify dataset.
    * It creates the `DOWNLOAD_DIR` if it doesn't exist.
    * It iterates through the search results, extracts the video URL from each item, and calls the `downloadVideo()` function to download each video.
    * `Promise.all(downloadTasks)` ensures that all video downloads are completed before the function finishes.

* **Example Usage:**

    * The code at the end demonstrates how to use the `searchAndDownloadVideos()` function. You can modify the parameters in this example to change your search query.

### Step 3: Configure API Token and Download Directory

1.  **Apify API Token:**  Open the `download_tiktok_videos.js` file in a text editor. Locate the line:

    ```javascript
    const APIFY_TOKEN = ''; // Replace with your Apify API token
    ```

    Replace the empty string `''` with your Apify API token that you obtained from the [Apify Console](https://www.google.com/url?sa=E&source=gmail&q=https://console.apify.com/). **Keep your API token secure and do not share it publicly.**

2.  **Download Directory (Optional):** If you want to change the download directory, modify the `DOWNLOAD_DIR` variable:

    ```javascript
    const DOWNLOAD_DIR = './my-tiktok-downloads/'; // Example: Change download directory
    ```

    Make sure the directory path is valid for your operating system.

### Step 4: Run the Code

1.  **Open your terminal:** Navigate to your project directory (`tiktok-downloader`) in your terminal.

2.  **Run the script:** Execute the JavaScript file using Node.js:

    ```bash
    node download_tiktok_videos.js
    ```

    The script will start searching for TikTok videos based on the example parameters (keyword: "viral", region: "US", limit: 40) and download them to the `download/` directory (or your custom directory). You will see console logs indicating the download progress.

### Step 5: Explore Different Search Parameters (Optional)

Experiment with different search parameters in the `searchAndDownloadVideos()` function call to refine your TikTok video searches. For example:

* **Change the keyword:**

  ```javascript
  searchAndDownloadVideos(keyword = "recipes", // Search for recipe videos
      publishTime = "MONTH",
      sortType = 0,
      region = "US",
      limit = 20)
  ```

* **Search for latest videos in Vietnam:**

  ```javascript
  searchAndDownloadVideos(keyword = "news",
      publishTime = "YESTERDAY",
      sortType = 2, // Sort by latest
      region = "VN", // Vietnam region
      limit = 10)
  ```

Refer to the [Apify TikTok Search API documentation](https://apify.com/novi/tiktok-search-api) for a full list of available parameters and their possible values.

## Conclusion

Congratulations! You have successfully built a simple TikTok video downloader using the Apify TikTok Search API. This blog post provided a beginner-friendly guide to help you understand the code and start downloading TikTok videos based on your keywords.

Feel free to explore the [Apify TikTok Search API](https://apify.com/novi/tiktok-search-api) further and customize the code to fit your specific needs. You can modify search parameters, add more error handling, or integrate this functionality into larger projects. Happy downloading!


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

