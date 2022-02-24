---
title: Add Google Analytics
slug: add-google-analytics
description: In which I make a deal with the devil and add Google analytics to my site
author: Sam Boland
date: '2022-02-13T18:21:37.745Z'
tags: []
pullRequest: https://github.com/samuelboland/Hyperfixations/pull/31
---

  
## Table of contents

## Introduction

### Ethical Reservations

To be honest, I'm not entirely sure how I feel about this, but I do want to be able to see how many people visit the site, and Google Analytics is a core part of setting a lot of professional sites. I'm going to try it, but I might remove it later.

### What is Google Analytics?

For those who don't know, [Google Analytics](https://analytics.google.com/) is, I think, the most popular method of learning some information about who visits your website. You add a bit of JS that loads on each page, gathers some information on viewers, and reports it to Google. You can then view this information.

## Dev Log

### Initial Research

Let's see how to integrate this. I'm going to start at the source, and just set things up by following the obvious route through Google's analytics page. I'll look up a separate tutorial if I need to.

First, I created a "Property" on the Google Analytics dashboard. I set it up to track `Hyperfixations.io`, and I enabled all tracking metrics.

It eventually lead me to a code block that it wants me to place in my website's `<head`>. However, before I do that, I see something that looks like an access key included in the URL it's giving me. I don't want to share that in my public repo! I'm going to add it to my secrets. For more information on how to do that, see my previous post on **Authenticating with the Github API**. And I'm not going to make the same mistake as last time - I'm going to add it to my CircleCI Config right away.

I'm going to name it `GOOGLE_ANALYTICS_TOKEN`

Ah, wait, it actually has to be named `NEXT_PUBLIC_GOOGLE_ANALYTICS_TOKEN`. The reason for this is that Nextjs only exposes environment variables at build time on the . But we need this to be available at runtime. You can do this by prepending `NEXT_PUBLIC` to the environment variable.

### Wait a minute - this is a Single Page Application

I don't think that just putting this code in my `<head>` tag will work, and here's why: Nextjs is a **Single Page Application**, which means that it doesn't actually load an entirely new page when you click a link. From the browser's perspective, you're (mostly) still on the same page. The Nextjs router handles that under the hood.

That means that the Google Analytics code will not be reloaded on each page load.

That means that it won't work out of the box.

It's time to look up how other people have done this.

### Informing Google Analytics when I change pages in Nextjs

I was right! Take a look at [this post](https://flaviocopes.com/nextjs-google-analytics/).

#### A brief aside on useEffect()

We're going to add a `useEffect()` hook that triggers whenever the Nextjs router changes pages.

As a refresher, [`useEffect()`](https://reactjs.org/docs/hooks-effect.html) is a [React hook](https://reactjs.org/docs/hooks-intro.html). I'm not going to get too deep into how the subject of hooks in modern react, there are other, better, resources for that.

The `useEffect()` hook runs code whenever something that you specify changes. In this case, we will run it whenever we change pages in React. You can specify a condition for it to "watch" in the `dependency array`, which is the second thing you pass to it.

The overall syntax is:

```js
useEffect(() => {
    someFuctionThatDoesStuff
}, [theThingThatItWatches])
```

#### Informing Google Analytics of a page change with a useEffect() hook

The post that I linked above suggests putting this in your `pages/_app.js` folder. That makes sense. In Nextjs, `_app.js` is sort of like a "base page," that all other pages inherit from. It wraps all other pages.

Currently, mine looks like this:

```js
//pages/_app.js
export default function App({ Component, pageProps: { session, ...pageProps }, router }) {
    return (
        <>
            <Head>
                {' '}
                <meta name="viewport" content="width=device-width, initial-scale=1.0" />
                <title>Sam Boland</title>
                <meta name="description" content="Hyperfixations.io" />
            </Head>
            <Layout>
                <AnimatePresence
                    exitBeforeEnter
                    initial="start"
                    onExitComplete={() => window.scrollTo(0, 0)}
                >
                    <Component {...pageProps} key={router.pathname} />
                </AnimatePresence>
            </Layout>
        </>
    );
}

```

Note that this is **not the default provided with nextjs**. I already added some stuff to it to handle some page change animations, and I never wrote how I did it. I added that before I really started recording my work. Sorry! Maybe one day I'll get to that.

You can see a default `_app.js`, and information on why you would want to customize it, at the [Nextjs docs here.](https://nextjs.org/docs/advanced-features/custom-app)

Let's add this stuff! I'm not going to copy/paste my whole new `_app.js`, since it's the same as above but with the new `useEffect` hook added above the return statement.

#### Injecting the Google Analytics Script into my Nextjs app

The previous section ensures that we tell Google Analytics when we change pages, but it doesn't actually add the script. We will do that in a custom `pages/_document.js` file. You can read about this practice [in the Nextjs docs here](https://nextjs.org/docs/advanced-features/custom-document).

This is a fairly common practice when integrating with some third party libraries or tools in Nextjs. I already have a custom one of these, to load some web fonts. You can read about that process in the [docs, as usual.](https://nextjs.org/docs/basic-features/font-optimization)

And that appears to be that.

### Testing Google Analytics integration

Let's fire up `localhost:3000`. I disabled my ad blocker, but I'm not seeing any activity in my analytics dashboard. Let's look in the developer console's network tab.

Ah, there's something wrong with passing my id to it from my environment variable. I'm seeing a query to `https://www.googletagmanager.com/gtag/js?id=undefined`. Note that `undefined`. That's where my ID should be!

#### Troubleshooting the missing ID

My `NEXT_PUBLIC_GOOGLE_ANALYTICS_TOKEN` is defined...OH! I need to restart the server for it to pick up new environment variables. Trying that now.

Actually, I think that I needed to do that for it to pick up changes to `_app.js` and/or `_document.js`, because now I'm getting an error in that code. Specifically in `_app.js` at the `window.gtag` part. It says:

```js
Uncaught (in promise) TypeError: window.gtag is not a function
```

I define `gtag` in the `_document.js`. [Someone on Stackoverflow](https://stackoverflow.com/questions/62791299/typeerror-window-gtag-is-not-a-function) suggests checking to ensure that the `window` object is not undefined before calling it. I suppose that might work, let's give it a shot.

That didn't work. Back to google!

## Dev Log 2 - starting over

### Attempt Number Two

It turns out that Nextjs has official documentation for integrating with Google Analytics in the form of an [example repository](https://github.com/vercel/next.js/tree/canary/examples/with-google-analytics)

Let's try implementing it this way.

#### Extracting the gtag function to a library

Sometimes, when you have a function that you want to reuse in multiple places, it makes sense to pull it out into a separate file. Nextjs suggests putting that into a `/lib` folder. I previously did that when I had a `mongoDb` database.

I'm going to [this file from the example](https://github.com/vercel/next.js/blob/canary/examples/with-google-analytics/lib/gtag.js).

```js
//lib/gtag.js

export const GA_TRACKING_ID = process.env.NEXT_PUBLIC_GA_ID

// https://developers.google.com/analytics/devguides/collection/gtagjs/pages
export const pageview = (url) => {
  window.gtag('config', GA_TRACKING_ID, {
    page_path: url,
  })
}

// https://developers.google.com/analytics/devguides/collection/gtagjs/events
export const event = ({ action, category, label, value }) => {
  window.gtag('event', action, {
    event_category: category,
    event_label: label,
    value: value,
  })
}
```

I got rid of the changes that I made to `_app.js` and `_document.js` earlier. I'm now going to add in the `useEffect()` from the [example repo](https://github.com/vercel/next.js/blob/canary/examples/with-google-analytics/pages/_app.js).

Now I add the script:

```js
 {/* Global Site Tag (gtag.js) - Google Analytics */}
      <Script
        strategy="afterInteractive"
        src={`https://www.googletagmanager.com/gtag/js?id=${gtag.GA_TRACKING_ID}`}
      />
      <Script
        id="gtag-init"
        strategy="afterInteractive"
        dangerouslySetInnerHTML={{
          __html: `
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', '${gtag.GA_TRACKING_ID}', {
              page_path: window.location.pathname,
            });
          `,
        }}
      />
```

Oh, and be sure to import the library and the Nextjs `<Script>` component!

```js
import * as gtag from '../lib/gtag';
import Script from 'next/script';
```

And let's restart again.

And it works!! I see myself popping up on my analytics feed. This is cool!

## CI and Deployment

Let's just make sure that I can run this through CI and deploy it.
