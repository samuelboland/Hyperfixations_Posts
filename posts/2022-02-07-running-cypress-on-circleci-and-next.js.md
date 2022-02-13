---
title: Running Cypress on CircleCI and Next.js
slug: running-cypress-circleci-js
description: null
author: null
date: '2022-02-07T06:44:31.961Z'
lastmod: '2022-02-13T01:00:21.954Z'
draft: false
tags: []
categories: []
---
- [Introduction](#introduction)
- [esmExternals](#esmexternals)

## Introduction

In my previous post, I explained how I got `Istanbul` to instrument my code, and `Cypress` to create coverage reports when it runs. As a reminder, my remaining goals are:

- Run these tests on CircleCI
- Publish my code coverage reports to Codecov

This post will be about the first part.

**Disclaimer:** It's been a few days since I did this, so I don't remember _all_ of the details. But I remember enough, and my commit messages were helpful. I need to remember to write these more quickly.

Also my laptop battery is low.

## esmExternals

I had a lot of issues with this. Specifically, I was getting errors along the lines of:

```
    [334:1020/170552.614728:ERROR:bus.cc(392)] Failed to connect to the bus: Failed to connect to socket /var/run/dbus/system_bus_socket: No such file or directory
```

or sometimes, depending on my particular configuration:

```
    SyntaxError: Unexpected token 'export'
```

It turned out to be some sort of incompatibility between the CircleCI docker images, Cypress, and the latest Next.js. Specifically, the `esmExternals` feature that was enabled in Next.js 17. I don't know the full details of this change, but it involves the interaction between the old styles `require` and new style `import` methods of using external modules.

To fix this, it all came down to a slight adjustment to my `next.config.js`. Here's what mine looks like:

```js
module.exports = {
    reactStrictMode: true,
    experimental: {
        esmExternals: false,
    },
};
```

The key element of figuring it out was this Github issue: <https://github.com/vercel/next.js/issues/30750> for the `framer motion` package. It was causing the same issue. I only figured this out once I expanded my search beyond just Cypress.

After that, I also had to set up my CircleCI configuration correctly. There are many ways to do this, and I will cover my choices in my next post. If you are interested now, you can see my configuration here: <https://github.com/samuelboland/Hyperfixations/blob/main/.circleci/config.yml>
