---
layout: post
title: "Tutorial: Download TikTok Videos by Hashtag Using Fast TikTok API"
date: 2025-02-22 17:15:30 +0700
categories: [ tiktok, scraper ]
---

This tutorial guides you through using the Fast TikTok API to fetch a list of hashtags and download videos. It's
designed for beginners and provides a step-by-step approach to get you started.

The Fast TikTok API by Apify is a powerful tool that allows you to extract data from TikTok, including hashtags, video
details, and more. In this guide, we will use it to gather videos associated with specific hashtags and download them
for offline use.

**Link to Fast TikTok API:** [Fast TikTok API](https://apify.com/novi/fast-tiktok-api)

## Prerequisites

Before you begin, ensure you have the following installed and set up:

* **Node.js:**  Make sure you have Node.js installed on your system. You can download it
  from [nodejs.org](https://nodejs.org/).
* **Apify Account:** You'll need an Apify account to use the Fast TikTok API. Sign up for free
  at [apify.com](https://apify.com/).
* **Apify API Token:** Once you have an Apify account, you'll find your API token on
  the [Integrations](https://console.apify.com/settings/integrations) page.

## Step-by-Step Guide

Follow these steps to download TikTok videos by hashtag:

### Step 1: Set up Apify Account and Get API Token

1. Go to [apify.com](https://apify.com/) and sign up for a free account.
2. Once logged in, navigate to your Apify Console.
3. Find your API token on the [Integrations](https://console.apify.com/settings/integrations) page. Keep this token
   secure, as you'll need it to access the Apify API.

### Step 2: Install Apify Client

Open your terminal or command prompt and run the following command to install the `apify-client` for JavaScript:

```bash
npm install apify-client --save
```

This command will install the necessary Apify client library in your project.

### Step 3: Create a JavaScript File

Create a new JavaScript file, for example, `tiktok_downloader.js`, and copy the following code into it:

```js
const {ApifyClient} = require('apify-client');
const fs = require('fs');
const https = require('https');
const path = require('path');

const client = new ApifyClient({
    token: '', // Replace with your Apify API token
});

const downloadVideo = async (videoUrl, destinationPath) => {
    try {
        const fileStream = fs.createWriteStream(destinationPath);

        https.get(videoUrl, (response) => {
            if (response.statusCode !== 200) {
                console.error(`Failed to download video. Status code: ${response.statusCode}`);
                fs.unlink(destinationPath, (err) => { //Clean up the file if download failed.
                    if (err) console.error("Error deleting incomplete file:", err);
                });
                return;
            }

            response.pipe(fileStream);

            fileStream.on('finish', () => {
                fileStream.close(() => {
                    console.log(`Video downloaded successfully to: ${destinationPath}`);
                });
            });

            fileStream.on('error', (err) => {
                console.error('Error during download:', err);
                fs.unlink(destinationPath, (unlinkErr) => { //Clean up the file if download failed.
                    if (unlinkErr) console.error("Error deleting incomplete file:", unlinkErr);
                });
            });

            response.on('error', (err) => {
                console.error('Response error:', err);
                fs.unlink(destinationPath, (unlinkErr) => { //Clean up the file if download failed.
                    if (unlinkErr) console.error("Error deleting incomplete file:", unlinkErr);
                });
            });

        }).on('error', (err) => {
            console.error('HTTP request error:', err);
            fs.unlink(destinationPath, (unlinkErr) => { //Clean up the file if download failed.
                if (unlinkErr) console.error("Error deleting incomplete file:", unlinkErr);
            });
        });

    } catch (error) {
        console.error('An unexpected error occurred:', error);
        fs.unlink(destinationPath, (unlinkErr) => { //Clean up the file if download failed.
            if (unlinkErr) console.error("Error deleting incomplete file:", unlinkErr);
        });
    }
}

const crapAndDownloadByHashtag = async (hashtag) => {
    const {defaultDatasetId} = await client.actor('novi/fast-tiktok-api').call(
        {
            "region": "GB",
            "type": "HASHTAG",
            "url": "\"" + hashtag + "\"",
            "limit": 40
        }
    )

    const {items} = await client.dataset(defaultDatasetId).listItems()

    const tasks = []
    items.forEach(item => {
        tasks.push(downloadVideo(item.video.play_addr.url_list[0], "./download/" + item.status.aweme_id + ".mp4"))
    })

    await Promise.all(tasks)
}

crapAndDownloadByHashtag("[https://www.tiktok.com/tag/fyp](https://www.tiktok.com/tag/fyp)")
    .then(() => console.log("Finish"))
```

### Step 4: Modify the Code

1. **API Token:** Open `tiktok_downloader.js` and replace the empty string in `token: ''` with your actual Apify API
   token from Step 1.
2. **Hashtag:** In the last line of the code, `crapAndDownloadByHashtag("https://www.tiktok.com/tag/fyp")`, you can
   change `"https://www.tiktok.com/tag/fyp"` to any TikTok hashtag URL you want to download videos from. For example, to
   download videos from the hashtag `#travel`, you would use `"https://www.tiktok.com/tag/travel"`.
3. **Download Folder:** The script will create a folder named `download` in the same directory as your script to store
   the downloaded videos.

### Step 5: Run the Code

In your terminal, navigate to the directory where you saved `tiktok_downloader.js` and run the script using Node.js:

```bash
node tiktok_downloader.js
```

The script will start fetching video URLs for the specified hashtag and downloading them. You will see console messages
indicating the download progress.

### Step 6: Check Downloaded Videos

Once the script finishes running (you'll see the "Finish" message in the console), navigate to the `download` folder in
the same directory as your script. You will find the downloaded TikTok videos in `.mp4` format in this folder.

## Code Explanation

Let's break down the code to understand what's happening:

* **Import Libraries:**

    * `apify-client`:  The official Apify API client for Node.js, used to interact with the Fast TikTok API.
    * `fs`:  Node.js file system module, used for creating file streams to save videos.
    * `https`:  Node.js https module, used to make HTTP requests to download video URLs.
    * `path`:  Node.js path module, used for handling file paths.

* **Initialize Apify Client:**

  ```js
  const client = new ApifyClient({
      token: '', // Your Apify API token here
  });
  ```

  This initializes the Apify client with your API token, allowing you to authenticate and use Apify Actors (like the
  Fast TikTok API).

* **`downloadVideo` Function:**

  ```js
  const downloadVideo = async (videoUrl, destinationPath) => { ... }
  ```

  This function is responsible for downloading a single video from a given `videoUrl` and saving it to
  the `destinationPath`. It uses `https.get` to fetch the video and `fs.createWriteStream` to save it as a file. Error
  handling is included to manage download failures.

* **`crapAndDownloadByHashtag` Function:**

  ```js
  const crapAndDownloadByHashtag = async (hashtag) => { ... }
  ```

  This is the main function that orchestrates the process:

    1. **Call Fast TikTok API:**
       ```js
       const {defaultDatasetId} = await client.actor('novi/fast-tiktok-api').call({ ... });
       ```
       It calls the `novi/fast-tiktok-api` Apify Actor with configurations to fetch videos based on the
       provided `hashtag`. It specifies the region as "GB", the type as "HASHTAG", the hashtag URL, and a limit of 40
       videos.
    2. **Get Dataset Items:**
       ```js
       const {items} = await client.dataset(defaultDatasetId).listItems()
       ```
       It retrieves the items (video data) from the default dataset created by the Apify Actor.
    3. **Download Videos:**
       ```js
       const tasks = []
       items.forEach(item => {
           tasks.push(downloadVideo(item.video.play_addr.url_list[0], "./download/" + item.status.aweme_id + ".mp4"))
       })
       await Promise.all(tasks)
       ```
       It iterates through the retrieved items, extracts the video URL from `item.video.play_addr.url_list[0]`, and
       calls the `downloadVideo` function for each video to download it. `Promise.all` ensures all downloads happen
       concurrently.

* **Run the Script:**

  ```js
  crapAndDownloadByHashtag("[https://www.tiktok.com/tag/fyp](https://www.tiktok.com/tag/fyp)")
      .then(() => console.log("Finish"))
  ```

  Finally, it calls `crapAndDownloadByHashtag` with the example hashtag URL (`"https://www.tiktok.com/tag/fyp"`) to
  start the process.

## Conclusion

Congratulations\! You've successfully used the Fast TikTok API to download TikTok videos by hashtag. This tutorial
provides a basic framework, and you can further explore the Fast TikTok API documentation to discover more features and
customization options, such as different regions, video limits, and data fields. Happy coding\!
