---
layout: post
title: "Download Trending TikTok Videos Easily with Fast TikTok API"
date: 2025-02-23 08:15:00 +0700
categories: [ tiktok, scraper ]
---

Are you interested in TikTok's trending videos?  Want to download them for your own use?  This guide will show you how to use the **Fast TikTok API** from Apify to easily get a list of trending TikTok videos and download them.

This blog post is for beginners and will guide you step-by-step. We will use Javascript code to do this.

## What is Fast TikTok API?

[Fast TikTok API](https://apify.com/novi/fast-tiktok-api) is a tool on Apify that lets you get data from TikTok, like trending videos, video details, and more.  Apify is a platform that helps you collect data from the web.  This API makes it simple to find out what's popular on TikTok right now.

In this guide, we will use this API to:

* Get a list of trending TikTok videos.
* Download these videos to your computer.

Let's get started!

## What You Need

Before you start, make sure you have these things:

1.  **Apify Account:** You need an account on [Apify](https://apify.com/).  Don't worry, they have a free plan to get started.
2.  **Apify API Token:**  Once you have an Apify account, you'll need your API token. This is like a password for the API. You can find it in your Apify account settings.
3.  **Node.js and npm:**  We will use Javascript, so you need to have Node.js and npm (Node Package Manager) installed on your computer. You can download them from [nodejs.org](https://nodejs.org/).
4.  **Code Editor:**  You'll need a code editor to write and run the Javascript code.  [VS Code](https://code.visualstudio.com/) is a popular and free option.

## Step-by-Step Guide

Follow these steps to download trending TikTok videos:

**Step 1: Get your Apify API Token**

1.  Go to the [Apify website](https://apify.com/) and sign up for an account or log in if you already have one.
2.  Once you are logged in, go to your **Account Settings**.
3.  Find your **API token**. It's usually under a section called "Integrations". Copy a token at "Personal API tokens".  We will need it in our code.

![Get Personal API Token]({{ site.baseurl }}/images/get-personal-api-token.jpg)

**Step 2: Install Node.js and npm**

If you don't have Node.js and npm installed, follow these steps:

1.  Go to the [Node.js website](https://nodejs.org/) and download the installer for your operating system (Windows, macOS, Linux).

2.  Run the installer and follow the instructions.  This will install both Node.js and npm.

3.  To check if they are installed correctly, open your terminal or command prompt and run these commands:

    ```bash
    node -v
    npm -v
    ```

    You should see the version numbers of Node.js and npm if they are installed correctly.

**Step 3: Create a Project Folder**

1.  Create a new folder on your computer for this project. You can name it something like `tiktok-downloader`.

2.  Open your terminal or command prompt and navigate to this folder using the `cd` command. For example:

    ```bash
    cd path/to/your/tiktok-downloader
    ```

**Step 4: Initialize npm Project**

1.  In your terminal, inside your project folder, run this command:

    ```bash
    npm init -y
    ```

    This command creates a `package.json` file in your folder. This file helps manage the project's dependencies (the libraries we will use).

**Step 5: Install Apify Client**

1.  In your terminal, run this command to install the `apify-client` library:

    ```bash
    npm install apify-client
    ```

    This library helps us communicate with the Apify API.

**Step 6: Create `download_tiktok_trends.js` and Paste the Code**

1.  Create a new file in your project folder named `download_tiktok_trends.js`.
2.  Open this file in your code editor and copy and paste the following Javascript code into it:

<!-- end list -->

```javascript
const {ApifyClient} = require('apify-client');
const fs = require('fs');
const https = require('https');
const path = require('path');

// Configuration - Make these configurable or environment variables
const APIFY_TOKEN = ''; // Replace with your Apify API token
const DOWNLOAD_DIR = './download/';
const TIKTOK_REGION = 'GB';
const DOWNLOAD_LIMIT = 40;

const client = new ApifyClient({
    token: APIFY_TOKEN,
});

/**
 * @param {string} videoUrl
 * @param {string} destinationPath
 * @returns {Promise<void>}
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
 * Cleans up (deletes) a file at the given path.
 * @param {string} filePath
 */
const cleanupFile = (filePath) => {
    fs.unlink(filePath, (err) => {
        if (err && err.code !== 'ENOENT') { // Ignore 'file not found' errors
            console.error("Error deleting incomplete file:", err);
        }
    });
};


/**
 * Fetches TikTok trending videos using Apify Fast TikTok API and downloads them.
 * @returns {Promise<void>}
 */
const fetchAndDownloadTrendingVideos = async () => {
    try {
        const actorCallResult = await client.actor('novi/fast-tiktok-api').call({
            "region": TIKTOK_REGION,
            "type": "TREND",
            "limit": DOWNLOAD_LIMIT
        });

        const dataset = client.dataset(actorCallResult.defaultDatasetId);
        const {items} = await dataset.listItems();

        if (!fs.existsSync(DOWNLOAD_DIR)) { // Ensure download directory exists
            fs.mkdirSync(DOWNLOAD_DIR, {recursive: true});
        }

        const downloadTasks = items.map(item => {
            const videoUrl = item.video.play_addr.url_list[0];
            const destinationPath = path.join(DOWNLOAD_DIR, `${item.status.aweme_id}.mp4`);
            return downloadVideo(videoUrl, destinationPath);
        });

        await Promise.all(downloadTasks);
        console.log("Successfully downloaded all videos.");

    } catch (error) {
        console.error("Error during trend processing:", error);
    }
};


// Example usage
fetchAndDownloadTrendingVideos()
    .then(() => console.log("Finished processing trend."))
    .catch(error => console.error("Trend processing failed:", error));
```

```
**Important:**  Find the line in the code that says `const APIFY_TOKEN = ''; // Replace with your Apify API token` and replace the empty string `''` with your actual Apify API token that you copied in Step 1.  Make sure to keep the quotes around your token.
```

**Step 7: Run the Script**

1.  Open your terminal or command prompt, and make sure you are still in your project folder (`tiktok-downloader`).

2.  Run the Javascript code using this command:

    ```bash
    node download_tiktok_trends.js
    ```

    The script will start running. You will see messages in the terminal as it downloads the videos.

3.  Once it's finished, you will find the downloaded TikTok videos in a folder named `download` inside your project folder. Each video file will be named with the TikTok video ID and have a `.mp4` extension.

## Customization (Optional)

You can change some settings in the code to customize it:

* **`TIKTOK_REGION = 'GB';`**: This line sets the TikTok region to "GB" (United Kingdom). You can change this to other region codes like "US" (United States), "VN" (Vietnam), etc., to get trending videos from different regions.  See the [Fast TikTok API documentation](https://apify.com/novi/fast-tiktok-api) for a list of supported regions.
* **`DOWNLOAD_LIMIT = 40;`**: This line sets the number of trending videos to download to 40. You can change this number to download more or fewer videos.

To change these settings, simply edit the values in the `download_tiktok_trends.js` file and run the script again.

## Conclusion

Congratulations! You have successfully used the **Fast TikTok API** and Javascript code to download trending TikTok videos. This is a simple way to explore what's currently popular on TikTok.

Remember to check out the [Fast TikTok API page on Apify](https://apify.com/novi/fast-tiktok-api) to learn more about its features and other things you can do with it.  Apify has many other useful tools for web data extraction, so explore their platform to see what else you can discover!

Now you can use these downloaded videos for your projects, learning, or just for fun.  Happy downloading!
