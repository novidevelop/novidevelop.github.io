---
layout: post
title: "Scrape X (Twitter) Followers and Following Lists from Node.js: Filters, Budgets and a Clean CSV"
date: 2026-11-26 08:10:00 +0700
permalink: /apify/scrape-x-twitter-followers-nodejs/
categories: [Apify, Developer Guides, Twitter]
tags: [Apify Actors, apify-client, Twitter Scraper, X.com, Followers, Node.js, Lead Generation]
author: Novi Develop
description: "Call Twitter X.com Followers & Following Scraper from Node.js with apify-client: server-side filters, a hard cost cap, merging several source accounts into one CSV, and slimming rows with customMapFunction."
---

# Scrape X (Twitter) Followers and Following Lists from Node.js: Filters, Budgets and a Clean CSV

[Twitter X.com Followers & Following Scraper](https://apify.com/xtdata/twitter-x-followers-following-scraper?fpr=7hce1m) is a new Apify actor that returns the followers and/or the following list of public X accounts as full profile objects. It doesn't need an X login or developer account, and it filters on the server, so rows you don't want are never delivered or billed.

This guide calls it from Node.js. By the end you'll have a script that takes a few source accounts, pulls their followers with filters and a hard cost limit, merges the results per user, and writes a CSV you can import into a CRM.

## Setup

```bash
npm i apify-client
export APIFY_TOKEN=...   # Apify Console → Settings → API & Integrations
```

Use Node 18 or newer. The examples are ES modules (`"type": "module"` in `package.json`, or save the file as `.mjs`).

## The smallest call

```js
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });

const run = await client.actor('xtdata/twitter-x-followers-following-scraper').call({
    usernames: ['apify'],
    getFollowers: true,
    getFollowing: true,
    maxItems: 10,
});

const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(run.status, items.length);           // SUCCEEDED 20
console.log(items.map((u) => `${u.type}\t@${u.username}\t${u.followersCount}`).join('\n'));
```

`call()` starts the run and waits for it to finish. `maxItems` applies per source account **and** per list, so one account with both lists on gives up to 20 rows.

Accounts can be given in three fields, and you can mix them in one run:

| Field | Accepts |
|---|---|
| `usernames` | `apify`, `@apify` |
| `userIds` | `"44196397"` (pass IDs as strings) |
| `startUrls` | `https://x.com/apify`, `https://x.com/apify/followers`, `https://x.com/apify/following` |

## What a row looks like

```json
{
    "type": "follower",
    "id": "2990041791",
    "username": "FarisBakr",
    "name": "Bakr FARIS",
    "description": "Software engineer. Contact: hello@example.com",
    "followersCount": 7385,
    "followingCount": 4047,
    "tweetCount": 86295,
    "verified": false,
    "isBlueVerified": false,
    "location": "Casablanca, Grand Casablanca",
    "createdAt": "Wed Jan 21 15:34:59 +0000 2015",
    "profileUrl": "https://x.com/FarisBakr",
    "website": "https://example.com",
    "emails": ["hello@example.com"],
    "protected": false,
    "sourceUser": "elonmusk"
}
```

(Trimmed. Rows also carry `likeCount`, `listedCount`, `mediaCount`, the profile image and banner URLs, and pinned tweet IDs.)

Three fields save you post-processing:

- `website` is already expanded from the `t.co` short link.
- `emails` is an array of addresses parsed out of the bio. It's empty when there are none.
- `sourceUser` is the account exactly as you passed it, so you can join rows back to your input even when you mixed usernames and IDs.

## Filter on the server, not in your script

You could download everything and run `.filter()` locally. The trouble is that you pay for every delivered row. These input fields drop non-matching users before delivery, so they cost nothing:

```js
const input = {
    usernames: ['competitor_one', 'competitor_two'],
    getFollowers: true,
    maxItems: 200,          // keep up to 200 matching users per account
    minFollowers: 1000,
    onlyWithEmail: true,
    // maxFollowers, onlyVerified, onlyWithWebsite also exist
    maxScanned: 5000,       // stop checking profiles after this many per list
};
```

`maxScanned` exists because a strict filter can match very rarely. Without a cap, a run could page through a 10M-follower account looking for one more match. The default cap is `maxItems × 5`. If you get fewer than `maxItems` results, the cap was reached first, and the run log says so. Raise `maxScanned` when the filter is strict and the source account is large.

## Put a hard ceiling on cost

Pricing is per delivered profile: on Apify's paid plans, $0.0005 per follower and $0.001 per followed account (less on Gold and up). There's no start fee. To make sure a typo in `maxItems` can't run up a bill, pass `maxTotalChargeUsd`:

```js
const run = await client
    .actor('xtdata/twitter-x-followers-following-scraper')
    .call(input, { maxTotalChargeUsd: 2 });
```

The actor stops delivering when the next item would go over the limit, and finishes normally with what it has. Your code doesn't need a special case for it.

## Merge several sources into one row per user

When you scrape several competitors, the same person often follows more than one of them. Those people are usually the most interesting leads, so collapse the rows and keep the list of sources:

```js
function mergeBySource(items) {
    const byId = new Map();
    for (const u of items) {
        const row = byId.get(u.id) ?? {
            id: u.id,
            username: u.username,
            name: u.name,
            followersCount: u.followersCount,
            verified: u.verified || u.isBlueVerified,
            website: u.website ?? '',
            emails: new Set(),
            sources: new Set(),
        };
        (u.emails ?? []).forEach((e) => row.emails.add(e.toLowerCase()));
        row.sources.add(`${u.sourceUser}:${u.type}`);
        byId.set(u.id, row);
    }
    return [...byId.values()]
        .map((r) => ({ ...r, emails: [...r.emails], sources: [...r.sources] }))
        .sort((a, b) => b.sources.length - a.sources.length || b.followersCount - a.followersCount);
}
```

Sorting by the number of sources first puts users who follow several of your inputs at the top.

## Write the CSV

No library needed. Just quote every cell:

```js
import { writeFile } from 'node:fs/promises';

const cell = (v) => `"${String(v ?? '').replaceAll('"', '""')}"`;

async function toCsv(rows, path) {
    const header = ['username', 'name', 'followersCount', 'verified', 'website', 'emails', 'sources'];
    const lines = rows.map((r) => [
        r.username, r.name, r.followersCount, r.verified, r.website,
        r.emails.join('; '), r.sources.join('; '),
    ].map(cell).join(','));
    await writeFile(path, [header.join(','), ...lines].join('\n'));
}
```

## The whole script

```js
import { ApifyClient } from 'apify-client';
// mergeBySource and toCsv from above

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });

const run = await client.actor('xtdata/twitter-x-followers-following-scraper').call({
    usernames: process.argv.slice(2),      // node leads.mjs competitor_one competitor_two
    getFollowers: true,
    maxItems: 200,
    minFollowers: 1000,
    onlyWithEmail: true,
    maxScanned: 5000,
}, { maxTotalChargeUsd: 2 });

if (run.status !== 'SUCCEEDED') {
    throw new Error(`Run ${run.id} ended with ${run.status}`);
}

const { items } = await client.dataset(run.defaultDatasetId).listItems();
const rows = mergeBySource(items);
await toCsv(rows, 'leads.csv');
console.log(`${items.length} rows → ${rows.length} unique users → leads.csv`);
```

`listItems()` returns up to the API's page size in one call. For runs with tens of thousands of rows, page with `{ offset, limit }` or download the dataset as CSV directly with `client.dataset(id).downloadItems('csv')`.

## Slim rows with customMapFunction

If you only need a few fields, `customMapFunction` reshapes each row inside the actor, which keeps the dataset small:

```js
customMapFunction: `(object) => ({
    username: object.username,
    followers: object.followersCount,
    email: object.emails[0] ?? null,
    from: object.sourceUser,
})`,
```

It runs in an isolated process with plain synchronous JavaScript: no `require`, `fetch`, timers or `Buffer`. It's for reshaping only, not for filtering. If it returns nothing or throws, the original row is saved, so use the filter fields above to drop users.

## Things that look like bugs but aren't

- **Protected accounts return zero rows.** Their lists aren't public. They're also not charged.
- **A list ends before `maxItems`.** X sometimes serves fewer users than the profile's follower count suggests.
- **Partial results with a message in the log.** If some source accounts fail and others succeed, the run still succeeds with what it got. If every request fails, the run fails and nothing is charged.
- **The same user appears twice.** That happens across sources or across the two lists, never within one list. `mergeBySource` handles it.

## Related actors

To find source accounts by topic, use [X.com Twitter User Search Scraper](https://apify.com/xtdata/twitter-x-user-search-scraper?fpr=7hce1m). To look up full profiles for a list of handles you already have, use [X.com Twitter User Info Scraper](https://apify.com/xtdata/twitter-x-user-info-scraper?fpr=7hce1m). For tweets by keyword or from specific users, use [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m).
