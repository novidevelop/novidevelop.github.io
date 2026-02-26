---
layout: post
title: How to Scrape TikTok User Profiles in 5 Minutes with the TikTok User Info API
date: '2025-09-12'
categories: [apify, tiktok, how-to, tutorial]
tags: [tiktok api, tiktok scraper, web scraping, user data, profile scraper, low-code, automation, data extraction]
---

In our last post, we introduced the **TikTok User Info API**, a powerful tool for extracting public data from TikTok profiles. Now, it's time to get hands-on. This guide will walk you through the entire process of scraping your first set of TikTok user data, from finding the actor to exporting your results.

The best part? You don't need to write a single line of code. We'll focus on the most common use case: extracting profile information from a list of TikTok URLs.

---

### **Step 1: Find the Actor in the Apify Store**

First, you need to navigate to the actor's page in the Apify Store.

➡️ **[Click here to go directly to the TikTok User Info API page.](https://apify.com/novi/tiktok-user-info-api?fpr=7hce1m)**

Once there, click the **"Try for free"** button. If you don't have an Apify account, you'll be prompted to sign up for a free one. The free plan includes platform credits, which are more than enough to run this actor many times.



---

### **Step 2: Configure the Input with Your Target URLs**

After clicking "Try for free," you'll land on the actor's page in the Apify Console. This is your control center. The most important section is the **Input** tab, where you tell the actor which profiles you want to scrape.

The actor accepts input in JSON format. While that might sound technical, it's just a simple way to organize text. To scrape profiles using their URLs, we'll use the `urls` field.

1.  **Clear any existing input** you see in the JSON editor.
2.  **Copy and paste** the code below into the input field.
3.  **Replace the example URLs** with the actual TikTok profile URLs you want to scrape.

```json
{
  "urls": [
    "[https://www.tiktok.com/@nike](https://www.tiktok.com/@nike)",
    "[https://www.tiktok.com/@nickiminai](https://www.tiktok.com/@nickiminai)",
    "[https://www.tiktok.com/@nasa](https://www.tiktok.com/@nasa)"
  ]
}
````

Your screen should look something like this:

-----

### **Step 3: Run the Actor and Monitor its Progress**

With your input configured, you're ready to go.

Click the **"Start"** button.

The actor will change its status to "Running." You can view the **Log** to see its progress in real-time as it finds and scrapes each profile. Once it has finished processing all the URLs, the status will change to **"Succeeded."** For a small list like our example, this will only take a few seconds.

-----

### **Step 4: View and Export Your Scraped Data**

Congratulations, you've successfully extracted TikTok user data\! To see the results, navigate to the **Storage** tab.

Here, you'll find your data neatly organized in a **Dataset**. Click on the **"Export"** button to see all the available formats. You can download your data as JSON, CSV, Excel, XML, and more.

For most users, **CSV** or **Excel** is the perfect choice for analyzing the data in a spreadsheet.

The output for each user will be incredibly detailed. Here is a sample of what the data for a single profile looks like in JSON format:

```json
{
  "uid": "208464585232822272",
  "unique_id": "nike",
  "nickname": "Nike",
  "bio_url": "https://linkin.bio/nike",
  "total_favorited": 18443294,
  "twitter_id": "",
  "twitter_name": "",
  "account_type": 3,
  "ad_virtual": false,
  "advance_feature_item_order": [
    20
  ],
  "apple_account": 0,
  "avatar_168x168": {
    "uri": "tos-useast5-avt-0068-tx/908fe11a62bf22cd36dbea27296d670c",
    "url_list": [
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_168x168.webp?x-expires=1699977600&x-signature=iO8Z1uKpRG5jynA%2BAw6ND6SztR8%3D",
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_168x168.jpeg?x-expires=1699977600&x-signature=H259CrWXtp3plTsl7lZaQoXxAlA%3D"
    ],
    "url_prefix": null
  },
  "avatar_300x300": {
    "uri": "tos-useast5-avt-0068-tx/908fe11a62bf22cd36dbea27296d670c",
    "url_list": [
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_300x300.webp?x-expires=1699977600&x-signature=iIBpadLQcRPvfauyfLZGDd4miBA%3D",
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_300x300.jpeg?x-expires=1699977600&x-signature=4jE9MYyxQqrMAoONVnjT8S6tenw%3D"
    ],
    "url_prefix": null
  },
  "avatar_larger": {
    "uri": "tos-useast5-avt-0068-tx/908fe11a62bf22cd36dbea27296d670c",
    "url_list": [
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_1080x1080.webp?x-expires=1699977600&x-signature=rbAuiQKYSjzHCrJ8DDYuZ2xie%2BM%3D",
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_1080x1080.jpeg?x-expires=1699977600&x-signature=%2BOXf0mltFENXz9mWAq%2BnYFKi%2FSY%3D"
    ],
    "url_prefix": null
  },
  "avatar_medium": {
    "uri": "tos-useast5-avt-0068-tx/908fe11a62bf22cd36dbea27296d670c",
    "url_list": [
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_720x720.webp?x-expires=1699977600&x-signature=GPjdachqxaJQ5n%2Fvp0CWvNlQlZw%3D",
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_720x720.jpeg?x-expires=1699977600&x-signature=PDiBDkKfHRlHfpayHuZuJsd3BTg%3D"
    ],
    "url_prefix": null
  },
  "avatar_thumb": {
    "uri": "tos-useast5-avt-0068-tx/908fe11a62bf22cd36dbea27296d670c",
    "url_list": [
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_100x100.webp?x-expires=1699977600&x-signature=ERmi0FWi6rbmcIi0mfTVLBVmGRc%3D",
      "https://p16-sign-va.tiktokcdn.com/tos-maliva-avt-0068/908fe11a62bf22cd36dbea27296d670c~c5_100x100.jpeg?x-expires=1699977600&x-signature=ocQkV7q0ydjHek3b2seZhAjNTdA%3D"
    ],
    "url_prefix": null
  },
  "aweme_count": 780,
  "bio_secure_url": "https://www.tiktok.com/link/?aid=1233&lang=en&scene=bio_url&target=https%3A%2F%2Flinkin.bio%2Fnike&owner_suid=MS4wLjABAAAA_3ndMt8d_tECTdpKgCxcx238tOnQZX-20wqN01aMui5zQ7hsqSdff-jC5qYC-Cl_",
  "biz_account_info": {
    "added_contact_and_link_list": null,
    "coupon_list": null,
    "leads_gen": {
      "action_name": "",
      "business_data": "",
      "has_leads_gen": false,
      "page_id": 0,
      "schema_url": ""
    },
    "permission_list": [
      "001001",
      "001002",
      "001003",
      "001004",
      "001005",
      "002001",
      "002002",
      "002005",
      "004001",
      "004006",
      "004009",
      "004012",
      "006001",
      "007001",
      "007003",
      "007004",
      "007005",
      "008001",
      "010001",
      "010002",
      "010003",
      "015001",
      "015002",
      "015003",
      "018003",
      "020001",
      "021001",
      "018001"
    ]
  },
  "can_message_follow_status_list": [
    0,
    1,
    2,
    4
  ],
  "category": "Sports, Fitness & Outdoors",
  "commerce_user_info": {
    "ad_revenue_rits": null
  },
  "commerce_user_level": 0,
  "custom_verify": "",
  "display_qna_on_profile": 1,
  "enterprise_verify_reason": "Verified account",
  "favoriting_count": 0,
  "follow_status": 0,
  "follower_count": 5020466,
  "follower_status": 0,
  "following_count": 180,
  "forward_count": 0,
  "ins_id": "nike",
  "is_block": false,
  "is_blocked": false,
  "is_effect_artist": false,
  "is_star": false,
  "live_commerce": false,
  "live_push_notification_status": 2,
  "message_chat_entry": true,
  "mplatform_followers_count": 0,
  "music_tab_info": {
    "show_artist_pick_videos": false
  },
  "original_musician": {
    "digg_count": 0,
    "music_count": 0,
    "music_used_count": 0,
    "new_release_clip_ids": null
  },
  "privacy_setting": {
    "following_visibility": 2
  },
  "profile_tab_type": 0,
  "recommend_reason_relation": "",
  "room_id": 0,
  "sec_uid": "MS4wLjABAAAA_3ndMt8d_tECTdpKgCxcx238tOnQZX-20wqN01aMui5zQ7hsqSdff-jC5qYC-Cl_",
  "secret": 0,
  "share_info": {
    "bool_persist": 1,
    "now_invitation_card_image_urls": null,
    "share_desc": "Check out Nike! #TikTok",
    "share_desc_info": "TikTok: Make Every Second Count",
    "share_image_url": {
      "uri": "tos-useast5-p-0068-tx/105776f7806b459cb32f294ebd046503_1699287598",
      "url_list": [
        "https://p16-sign.tiktokcdn-us.com/obj/tos-useast5-p-0068-tx/105776f7806b459cb32f294ebd046503_1699287598?x-expires=1699826400&x-signature=20%2FtSclpyuGPlE1pLbsOh3Xc4h8%3D",
        "https://p19-sign.tiktokcdn-us.com/obj/tos-useast5-p-0068-tx/105776f7806b459cb32f294ebd046503_1699287598?x-expires=1699826400&x-signature=ew%2F%2BslI6sHcrLICt8iUolIVj%2BmU%3D"
      ],
      "url_prefix": null
    },
    "share_title": "Join TikTok and see what I’ve been up to!",
    "share_title_myself": "",
    "share_title_other": "",
    "share_url": "https://www.tiktok.com/@nike?_r=1&_d=e9cbalc6a27625&sec_uid=MS4wLjABAAAA_3ndMt8d_tECTdpKgCxcx238tOnQZX-20wqN01aMui5zQ7hsqSdff-jC5qYC-Cl_&share_author_id=208464585232822272&sharer_language=en&source=h5_m&u_code=e9cbcf4gk687hf"
  },
  "short_id": "0",
  "show_effect_list": true,
  "show_favorite_list": false,
  "signature": "",
  "signature_language": "un",
  "story_status": 0,
  "supporting_ngo": {},
  "tab_settings": {
    "private_tab": {
      "private_tab_style": 1,
      "show_private_tab": false
    }
  },
  "verification_type": 0,
  "video_icon": {
    "uri": "",
    "url_list": [],
    "url_prefix": null
  },
  "watch_status": false,
  "with_commerce_enterprise_tab_entry": false,
  "with_commerce_entry": false,
  "with_new_goods": false,
  "youtube_channel_id": "",
  "youtube_channel_title": ""
}
```

-----

### **What's Next?**

That's it\! In just four simple steps, you've gone from a list of URLs to a structured dataset ready for analysis. You can now use this process to gather data on influencers, competitors, or any public TikTok profile of interest.

While using URLs is easy and effective, the **TikTok User Info API** offers other ways to target profiles, such as by `usernames` or `userIds`. In our next post, we'll take a deep dive into every input option to help you master this powerful tool.
