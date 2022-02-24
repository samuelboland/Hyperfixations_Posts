---
title: Authenticating Github API Requests
slug: authenticating-github-api-requests
description: In which I learn how to authenticate API requests to Github
author: Sam Boland
date: '2022-02-13T02:14:49.902Z'
tags: []
pullRequest: https://github.com/samuelboland/Hyperfixations/pull/30
---

## Table of contents

## Introduction

The [Github API](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#requests-from-user-accounts) limits unauthenticated requests to 60 per hour. Authenticated requests, on the other hand, have a limit of 5,000 per hour. This should be more than enough for my uses.

Note that I only use the API when fetching posts. That only happens in two scenarios:

1) I re-render my site in development, or

2) I build the site in production.

Of the two, the only one that could realistically hit the limit is 1, but I still want to try to avoid it.

## Dev Log

### Figuring out how to do it

Let's start by looking at the docs. The [Getting started with the REST API](https://docs.github.com/en/rest/guides/getting-started-with-the-rest-api) documentation seems like a great place to start.

It looks like I need to create a personal access token, and somehow send that, along with my username, whenever I make an API request. I'm not going to cover how to make the token, that's already covered in the Github docs.

The examples on the page are for the `curl` command, which is nice, but I need to know how to do this with the javascript `fetch` method.

[This article](https://learn.co/lessons/javascript-fetch-lab) looks helpful. It appears that I need to add an `Authorization` section to my Headers. And then I need to specify that I am using `token` authentication, and then pass the token. That seems easy. Let's try it!

### Doing It

#### Storing my token

It's not safe to just paste an authentication token in your code somewhere. It's a very bad practice. At my job, if someone accidentally commits a token to version control, our security team jumps on fixing it.

So where do we store it then? That really depends. I haven't read [this blog post](https://www.netmeister.org/blog/sharing-secrets.html) in full, but a quick skim looks good.

I'm going to store the token in two places, depending on the environment:

- Locally: The secret goes into a `.env.local` file, which I added to my `.gitignore`.
- Production: I'm using [Vercel](https://vercel.com/) to manage my deployments. They manage the hard work of storing, encrypting, and exposing the secret to my application, so I will store it there as an environment variable.

I can access both of these locations in the same way, which is ideal. I access it my calling `process.env.VARIABLE_NAME_HERE`. It's convention to name environment variables in all caps.

#### Adding my key to my headers

My current fetch method looks like this:

```js
    let headers = new Headers({
        Accept: 'application/json',
        'User-Agent': 'samuelboland',
    });

    const apiResponse = await fetch(URL, {
        method: 'GET',
        headers: headers,
    }).then((response) => response.json());
```

I need to define my api key, and then add it to my header, like this:

```js
    const GITHUB_API_TOKEN = process.env.GITHUB_API_TOKEN;
    let headers = new Headers({
        Accept: 'application/json',
        'User-Agent': 'samuelboland',
        Authorization: `token ${GITHUB_API_TOKEN}`,
    });
```

Let's see if this works! I'm going to log the API response so that I can see if the rate limiting information has changed. I'm going to do this in the fetch method, like this:

```js
    const apiResponse = await fetch(URL, {
        method: 'GET',
        headers: headers,
    }).then((response) => {
        console.log(response.headers);
        response.json();
    });
```

And look at that:

```js
    'x-ratelimit-limit': [ '5000' ],
    'x-ratelimit-remaining': [ '4997' ],
```

We have 5,000.

#### Addendum: Adding the key to CircleCI

I was experiencing a strange error when building my site on CircleCI for tests, but not when building it on Vercel for deployment:

![image showing my circleCI error message, which says that apiResponse.map is not a function](../images/Screen%20Shot%202022-02-12%20at%207.28.58%20PM.png)

How is `.map` not a function? Oh! Because I didn't add my Github api key to my CircleCI environment variables. This causes the api call to return some sort of failure message, which isn't something that you can `map` over.

Let's add that key to the right place.

And it works!
