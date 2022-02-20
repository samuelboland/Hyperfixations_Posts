---
title: Separate pages for individual blog posts
slug: separate-pages-individual-blog-posts
description: I want a way for users to click on the title of the blog post and be taken to a page that only shows that post.
author: Sam Boland
date: '2022-02-14T22:30:16.825Z'
tags:
    - posts
    - github
    - api
    - promise
pullRequest: '(https://github.com/samuelboland/Hyperfixations/pull/37)'
---

- [Introduction](#introduction)
- [Explaining dynamic routes in Nextjs](#explaining-dynamic-routes-in-nextjs)
  - [How does it know what to generate beforehand? With getStaticProps](#how-does-it-know-what-to-generate-beforehand-with-getstaticprops)
- [Dev Log](#dev-log)
  - [Populating all possible routes with getStaticPaths](#populating-all-possible-routes-with-getstaticpaths)
  - [Update: This is annoying](#update-this-is-annoying)
  - [Update 2: I FIGURED IT OUT](#update-2-i-figured-it-out)

## Introduction

I want a way for users to click on the title of the blog post and be taken to a page that only shows that post. I want this so that I can eventually lower the amount of text on the posts index page by truncating or even removing the content on each post, or adding pagination, or something. That comes later.

For now, let's figure out how to use dynamic routes in static generation!

## Explaining dynamic routes in Nextjs

Nextjs has a neat feature called [dynamic routes](https://nextjs.org/docs/routing/dynamic-routes).

If you name a route with brackets, like `[post].js`, it will be automatically generated when Nextjs builds your site. It will take the url that you give it as an input, and return some page based on that.

For example, let's say that I created a route like `pages/[posts].js`. Then I try to go to `https://hyperfixations.io/pages/some-blog-post-name-here`. The function that we write inside of `[post].js` will take `some-blog-post-name-here`, and do something with it. Preferably, it will return a page containing just that post.

### How does it know what to generate beforehand? With getStaticProps

Remember, Nextjs is a *static site generator*. So it somehow has to know what paths/urls a user might go to. It does that with [`getStaticPaths()`](https://nextjs.org/docs/basic-features/data-fetching/get-static-paths).

We need to call `getStaticPaths` in the `[posts].jsx` page. The overall format of this function is:

```js
export async function getStaticPaths() {
    // Some code to generate a list of paths goes here, like: 

    myPaths = const (someInputVariable) => {
        someFunctionsHere
    };

    return {
        paths: [
            myPaths
        ]   
    }
}
```

Let's think about what we might want to use for this. We want a user to be able to click into *any* blog post on the site and receive a page with that post, and only that post. That means that we need to generate paths for each blog post. How could we do that?

Well, we already have a way of getting a list of posts. The `fetchAll` function in the `githubApi` file we made in the last post! But, it returns more info than we really need, so maybe we could create a lighter version of it.

Note that I haven't built this yet, what you're reading is my reasoning process.

## Dev Log

### Populating all possible routes with getStaticPaths

I'm going to modify `lib/githubApi.js` and add a `fetchAllPostNames` function.

Here's what I have:

```js
export const fetchAllPostNames = async () => {
    const apiResponse = await fetch(URL, {
        method: 'GET',
        headers: headers,
    }).then((response) => response.json());

    const articles = await Promise.all(
        apiResponse.map((item) => {return item.name})
    );
    return articles;
};
```

If I did it right, this should return an array with the filenames of each post. Let's test it! I'm going to create a `/posts/posts.js` page. Note that I am not yet enclosing it in braces. Once I do that, it will become a dynamic route. For now, I just need a place to test some stuff.

Actually, this isn't working for some reason. After messing around with it, I think that I have to have it inside of an asynchronous function, and a page cannot be async. Let's try putting it in a `getStaticProps`.

### Update: This is annoying

So I just spent a good, like, 3 hours trying to get this to work. The annoying part was finding a way to link to the post from the posts index page. Specifically, a way to give that page the file path to look up to get the right `[slug]`.

When I get the text for the posts index page, I first hit the Github API with a request for all files in the posts folder. That API response includes N items, where N is the number of posts that I have. As part of each of those items, there is a "download url," which I then use to download the actual post data.

But as part of those responses there's *also* a "file name," which I can easily use in getStaticPaths to generate the pages.

What I need is a way to link those two things.

### Update 2: I FIGURED IT OUT

Wow that was extremely annoying. This took nearly two days, and it was all because I don't really know how to work with promises.

The main issue that I had was when I decided to try passing the filename to the index page. Remember, that page shows all of my posts. I also wanted it to be aware of the filename for each file that each post is generated from, so that it could use that name in a link.

This turned out to be harder than I thought, and it was all because of promises. But `Promise.all()` came through to save the day.

I warn you, **the following code is not nice.** I will refactor this at some point. Maybe. If I feel like it.

My github API lib now has three parts:

- fetchAll
- fetchAllPostNames
- fetchOne

`fetchAll` is used in getStaticProps on my index page. It gets the name, frontmatter, and content for all posts.

`fetchAllPostNames` is used in my dynamic `[post].js`. It is used in getStaticPaths there, so that Next knows what routes to pre-generate.

`fetchOne` is also used in my dynamic page. It is used to fetch the frontmatter and content for a single post.

I had to make some changes to `fetchAll` to make it work. It now looks like this:

```js
export const fetchAll = async () => {
    const apiResponse = await fetch(URL, {
        method: 'GET',
        headers: headers,
    }).then((response) => response.json());

    const nameAndUrl = apiResponse.map((item) => {
        return { name: item.name, url: item.download_url };
    });

    const nameAndMarkdown = nameAndUrl.map((item) => {
        const articles = fetch(item.url)
            .then((response) => response.text())
            .then((article) => matter(article))
            .then((markdown) => {
                return { name: item.name, content: markdown.content, frontmatter: markdown.data };
            });
        return articles;
    });

    const resolvedPromises = await Promise.all(nameAndMarkdown);

    return resolvedPromises;
};
```

So there's a lot going on here. Let's go through it piece by piece.

The beginning part, `const apiResponse`, should look familiar. This is a standard fetch.

After that, I map over the resulting response and return an object with the keys "name" and "url." You can see how I'm passing the `name` of the file through the chain so far.

Then, we map over the results of that and use the "url" that I included to fetch the raw post from Github. We convert it to text, then run it through our markdown parser, and finally we return a different object. This time, the object has the keys "name," "content," and "frontmatter."

This seems like it would be enough, but it wasn't. When I tried it, I was getting back an array of `Promise` objects. I needed them to be resolved, and instead give me content and stuff.

After some Google and a lot of pain, I found that I should pass that array to `Promise.all()`. This is an asynchronous function that wait for all promises passed to it to resolve. Or something like that.

And finally, I return my fancy new object, complete with name, content, and frontmatter.

This post is getting quite long, so I'm going to end it here. The code is available on my Github repo.
