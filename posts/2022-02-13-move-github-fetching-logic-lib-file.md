---
title: Move Github Fetching Logic to lib file
slug: move-github-fetching-logic-lib-file
description: null
author: null
date: '2022-02-13T21:05:12.072Z'
lastmod: '2022-02-14T22:17:44.154Z'
draft: true
tags: []
categories: []
pullRequest: 'https://github.com/samuelboland/Hyperfixations/pull/34'
---

[TOC GOES HERE]

## Introduction

I want to move the logic that I use to fetch the posts from my Github repo from in the `posts/index.js` page, to a separate library. I can then call that library from other places.

I want to do this in preparation for creating separate pages for each post, since I anticipate that I will reuse some of the logic for that.

## Dev Log

### Replicating Original Behavior

First, I want to replicate the original behavior. Eventually I will probably change how my github fetching works to make it more generic, but to start let's just keep it working.

That part was easy. I just extracted all of the original logic from `posts/index.js` and put it into `lib/githubApi.js`. I just had to wrap it inside of a function and return the final value. It also has to be async. Like so:

```js
// lib/githubApi.js
import matter from 'gray-matter';

const githubApi = async () => {
    const GITHUB_API_TOKEN = process.env.GITHUB_API_TOKEN;
    const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations_Posts/contents/posts';

    let headers = new Headers({
        Accept: 'application/json',
        'User-Agent': 'samuelboland',
        Authorization: `token ${GITHUB_API_TOKEN}`,
    });

    const apiResponse = await fetch(URL, {
        method: 'GET',
        headers: headers,
    }).then((response) => response.json());

    const urls = apiResponse.map((item) => item.download_url);

    const articles = await Promise.all(
        urls.map((url) => {
            const markdown = fetch(url).then((response) => response.text());
            return markdown;
        }),
    );
    const data = articles.map((article) => {
        const { data: frontmatter, content } = matter(article);
        return { data: frontmatter, content };
    });
    return data;
};

export default githubApi;

```

Then, in the posts index page, I adjusted `getStaticProps()` to be:

```js
export async function getStaticProps(context) {
    const data = await githubApi();
    return {
        props: { message: data },
    };
}
```

And it works perfectly.

### Organizing my new github Api file

Some of this code will apply whether I am grabbing a single post or all of them. The rest, I will separate into a new `fetchAll` function.

(As I was doing this, I realized that I could make it slightly more compact by removing the step where I pull out the URLs. I can just get those and download them in one step.)

My new `githubApi.js` looks like this:

```js
import matter from 'gray-matter';

const GITHUB_API_TOKEN = process.env.GITHUB_API_TOKEN;
const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations_Posts/contents/posts/';

let headers = new Headers({
    Accept: 'application/json',
    'User-Agent': 'samuelboland',
    Authorization: `token ${GITHUB_API_TOKEN}`,
});

export const fetchAll = async () => {
    const apiResponse = await fetch(URL, {
        method: 'GET',
        headers: headers,
    }).then((response) => response.json());

    const articles = await Promise.all(
        apiResponse.map((item) => {
            const markdown = fetch(item.download_url).then((response) => response.text());
            return markdown;
        }),
    );

    const data = articles.map((article) => {
        const { data: frontmatter, content } = matter(article);
        return { data: frontmatter, content };
    });
    return data;
};
```

I also added a `fetchOne()` function, which is similar. It just has fewer maps. This is right below the `fetchAll()` function.

```js
export const fetchOne = async (slug) => {
    const apiResponse = await fetch(URL + slug + '.md', {
        method: 'GET',
        headers: headers,
    }).then((response) => response.json());

    const article = await fetch(apiResponse.download_url).then((response) => response.text());

    const { data: frontmatter, content } = matter(article);

    const data = { data: frontmatter, content };

    return data;
};
```

Note that this one takes an input, `slug`. I will use that later when I create individual pages for each blog post. I got ahead of myself a bit with this one.

Let's run our tests! Actually, I haven't really covered my tests yet, have I? Well they're not that great. Just Cypress ensuring that my front mage, header/footer, and blog posts page loads. Cypress is pretty nice to work with in most scenarios.

They passed, so I'm going to push this code up. I'm also trying to record the PR that my work is associated with in each post, so look out for that to appear at some point in the future.
