---
layout: post
title: "Tutorial: Scraping TikTok Comments & Customer Sentiment Analysis"
date: 2026-03-12 10:00:00 +0700
description: "Scrape TikTok comments with Python for customer sentiment analysis. Complete tutorial with code to extract, export, and analyze comment data from any video."
categories: [Python, Scraping, TikTok, Data Analysis]
---

# Comprehensive Guide: Scraping TikTok Comments and Real-World Applications

TikTok is not just an entertainment platform, but also a goldmine of data on user preferences, trends, and feedback. To automate the collection of this data, the **TikTok Comment Scraper** on the Apify platform is an extremely powerful tool that processes quickly and notably requires no complex proxy configuration.

In this article, we will learn how to use this scraper with Python and explore a highly potential real-world application idea: **Customer Sentiment Analysis for E-commerce**.

---

## 1. Introduction to TikTok Comment Scraper

The Actor **`xtdata/tiktok-comment-scraper`** on Apify is optimized for blazing-fast data extraction. 
Key highlights:
- Very affordable (only about ~$0.001 for 100 results).
- Extracts user information (nickname, ID) and the number of likes (digg_count) of the comments.
- Supports scraping reply comments as well.

### Basic Input Parameters:
Based on the scraper's architecture, the main configuration parameters include:
- `urls`: (Required) A list of TikTok URLs or video IDs you want to scrape.
- `shouldScrapeReplies`: (Boolean) Set to `True` (Default: `False`) if you want to extract replies to the main comments.
- `maxItems`: (Integer) The maximum number of comments to extract per post (Default limit is 20 results per run).
- `shouldScrapeAll`: (Boolean) If `True`, the tool will ignore `maxItems` and attempt to scrape all comments from the video.

---

## 2. Application Idea: Customer Sentiment & Feedback Analysis

**Problem:** E-commerce brands often spend a significant budget on KOLs/KOCs to review their products on TikTok. However, instead of just counting views or likes, the real value lies in the **user comments**. There's too much data to read manually: Are they praising the excellent design or complaining about the hot fabric material? Or are they complaining about high shipping fees?

**Solution with Scraper:** 
You can use the **TikTok Comment Scraper** to automatically download thousands of comments from product review clips (from your brand or competitors). Then, combine this dataset with Natural Language Processing (NLP) tools or the ChatGPT API to:
1. **Automated Sentiment Classification:** Visualize what percentage of comments are Positive, Negative, or Neutral.
2. **High-Frequency Pain-points Extraction:** Instantly detect common phrases (e.g., "poor quality", "slow delivery", "how to buy").
3. **Competitor Market Research:** Gather all complaints on competitors' TikTok videos to improve your Marketing message (e.g., "Product A is too hot? Buy our Cooling Fabric B now").

---

## 3. Installation Guide and Python Code

### Install the libraries:

```bash
pip install apify-client pandas
```

### Complete Python Script:

Below is a complete sample code to call the scraper using a TikTok video ID and export the results to a CSV file, ready for the Data Analysis step:

```python
from apify_client import ApifyClient
import pandas as pd
from datetime import datetime

# 1. Initialize Apify Client with your API Key
# Login to Apify -> Settings -> API & Integrations to get your token
client = ApifyClient("YOUR_APIFY_API_TOKEN")

actor_id = "xtdata/tiktok-comment-scraper"

# 2. Setup Input 
# You can use direct Video Links or Video IDs
run_input = {
    "urls": [
        "https://www.tiktok.com/@ladygaga/video/7211250685902359850"
    ],
    "maxItems": 100,               # Maximum 100 comments
    "shouldScrapeReplies": False,  # Skip replies for faster speed
    "shouldScrapeAll": False
}

print("Starting TikTok Comment Scraper...")

try:
    # 3. Run the Actor and wait for it to finish
    run = client.actor(actor_id).call(run_input=run_input)
    dataset_id = run.get("defaultDatasetId")
    
    if dataset_id:
        print(f"Success! Dataset ID: {dataset_id}")
        
        # 4. Get the results from the Dataset
        results = client.dataset(dataset_id).list_items().items
        print(f"Extracted {len(results)} comments.")
        
        # Extract the most important data fields
        parsed_data = []
        for item in results:
            # Convert Unix timestamp to readable date format
            create_time = item.get("create_time", 0)
            date_obj = datetime.fromtimestamp(create_time).strftime('%Y-%m-%d %H:%M:%S') if create_time else None
                
            parsed_data.append({
                "comment_id": item.get("cid"),
                "text": item.get("text"),
                "likes": item.get("digg_count", 0),
                "replies_count": item.get("reply_comment_total", 0),
                "username": item.get("user", {}).get("unique_id", "Unknown"),
                "date_created": date_obj
            })
            
        final_df = pd.DataFrame(parsed_data)
        
        # 5. Save to standard UTF-8 CSV file
        final_df.to_csv("tiktok_comments.csv", index=False, encoding='utf-8-sig')
        print("Data successfully exported to tiktok_comments.csv!")
        
    else:
        print("Run finished, but no dataset was generated.")

except Exception as e:
    print(f"An error occurred during scraping: {e}")
```

### How it works:
Once the script completes, it generates a `tiktok_comments.csv` file with standardized columns containing "Content", "Username", and "Likes". With this structure, you just need to load it into Google Sheets, Excel, or feed it into a Natural Language Processing Machine Learning pipeline.

---

## 4. Conclusion

With the **TikTok Comment Scraper** and fewer than 50 lines of Python code, you can easily structurally automate the abundant source of opinions on TikTok. By actualizing ideas like Automated Negative Review Detection (Sentiment Analysis), businesses not only save a tremendous amount of time but also optimize quality based accurately on precise Insight metrics.

Start building your automated data pipeline today!

## Related Articles

* 💬 [Tutorial: Analyzing TikTok Comments with Fast TikTok API]({{ site.baseurl }}/tiktok/data-extraction/tutorial/comments/2025/03/10/fast-tiktok-api-comment-scraping-guide.html) — Alternative approach using the Fast TikTok API's COMMENT input
* 🤖 [Automate TikTok Data Extraction with n8n]({{ site.baseurl }}/n8n/automation/tiktok/scraper/2026/03/12/automate-tiktok-data-extraction-with-n8n.html) — No-code workflow automation for TikTok data

***
**Looking for data extraction tools?**  
Check out our [comprehensive collection of no-code scrapers and APIs]({{ site.baseurl }}/tools/) for TikTok, X.com, and more.
