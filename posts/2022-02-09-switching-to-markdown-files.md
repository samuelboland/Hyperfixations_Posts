---
title: Switching to Markdown Files with Frontmatter CMS
slug: switching-markdown-files-frontmatter-cms
description: 'Follow along as I figure out how to query, fetch, and display markdown content from a Github repo.'
author: Sam Boland
date: '2022-02-10T07:45:53.153Z'
tags:
    - vscode
    - CMS
    - frontmatter
    - React
    - Next.js
pullRequest: https://github.com/samuelboland/Hyperfixations/pull/24
---

## Table of contents

## Introduction

In this article, I show my how I created a `posts` page that pulls markdown files from a github repository, pulls front matter and content information out of them, and displays them in a nicely formatted way.

I wrote the bulk of this post as I was working on the code, giving a real-time view into my development process.

### I gave up on Tiptap

Using Tiptap to make a custom rich text editor was fun and illuminating. It was also neat figuring out how to store the generated posts in a managed MongoDB database, and how to retrieve them for static site generation.

However, actually writing things with that editor was honestly sort of annoying. I could have spent time trying to make it better, and perhaps I will one day in the future, but for now that path doesn't interest me.

### Enter Front Matter for vscode

Front Matter is a markdown-based content management system (CMS) entirely within vscode. It's available as a normal extension, and because it operates on standard markdown files, I can use all of the other plugins that vscode has.

It doesn't have every feature that I want, and it sometimes takes a long time to load the sidebar, but it works well enough for now. And the features that it does have are quite nice!

![Screenshot of the Front Matter CMS in my vscode](../images/Screen%20Shot%202022-02-09%20at%2011.54.25%20PM.png)

Now that I have a nice Markdown editing environment, I need to source and display my files. To do that, I closely followed [this guide](https://blog.openreplay.com/creating-a-markdown-blog-powered-by-next-js-in-under-an-hour.).

My first pass doesn't quite look as good as the images in that post, because I don't use Tailwind css, but the functionality is there.

![Image of my posts index page with partial styling](../images/Screen%20Shot%202022-02-10%20at%2010.21.19%20AM.png)

### Storing posts on Github in a separate repo

**(Note: starting here, I am writing this document as I'm figuring things out. I think that this will lead to more interesting posts. They might be less coherent though. )
**
One thing that I did really like about storing my posts in MongoDB was separating the _content_ from the _function_. Big fan of [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) here.

I'm taking inspiration from [Netlify CMS](https://www.netlifycms.org) and storing my posts in a github repo, but a separate one from the application. I plan to get the posts out of github with their API.

The repository will be public, at least to start, so I won't need to figure out authentication for now.

## Dev Log

### How will I do this? getStaticProps, probably!

I think that the best way to tackle this is [`getStaticProps`](https://nextjs.org/docs/api-reference/data-fetching/get-static-props). I'm going to create a new page in my site to test this.

Eventually, I need to iterate over all of the files that my query returns and display their contents. But to start, let's just make sure that I can query and display results from the API at all.

`getStaticProps` is a special Next.js function that runs before a page is generated, and performs some action, usually to get something, and then returns it in a JSON-formatted variable named "props." You can then use those props when building a page. A very simple use looks like this:

```js
// Boring example of getStaticProps
const Test = (props) => {
    return <div>{props.message}</div>;
};

export async function getStaticProps(context) {
    return {
        props: { message: 'Hello World!' },
    };
}

export default Test;
```

If you put this somewhere in the `pages` folder in a Next.js project (I have mine in `/pages/posts/test.js`) and navigate to it (`localhost:3000/pages/posts/test`) you will see `Hello World!` on your page.

This isn't very interesting, though, so let's use `getStaticProps` to pull some data from Github.

As a reminder, I'm figuring this out as I'm writing, so I'm jumping from one window to the other.

### Using getStaticProps to get files from a github repository

I just tried changing my `getStaticProps` section to this:

```jsx
export async function getStaticProps(context) {
    const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations';
    const response = await fetch(URL);
    return {
        props: { message: response },
    };
}
```

When I loaded the page, I got an error:

```
Reason: `object` ("[object Response]") cannot be serialized as JSON
```

That makes sense, actually. I'm probably returning a [promise](https://www.newline.co/fullstack-react/30-days-of-react/day-15/) that hasn't resolved yet. According to the MDN Web Docs document on [Using the Fetch Api](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch), and my previous experience that I completely blanked on at the moment, I need to wait for the promise to resolve. Let's try this instead.

```jsx
// Returning the JSON object. This is what I will
// actually use.
export async function getStaticProps(context) {
    const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations';
    const response = await fetch(URL).then((response) => response.json());
    return {
        props: { message: response },
    };
}
```

That gets us a lot closer! Now I get this error:

![Error page showing that I am trying to render an object as a child of a react component, which is not allowed](../images/Screen%20Shot%202022-02-10%20at%2010.57.44%20AM.png)

So we're getting the right data, it's just that it is in the form of a JSON object. We need to do something to it to display it. The simple approach is to stringify it:

```jsx
// gross stringified json
export async function getStaticProps(context) {
    const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations';
    const response = await fetch(URL)
        .then((response) => response.json())
        .then((data) => JSON.stringify(data));

    return {
        props: { message: response },
    };
}
```

That works! But now I just have a bunch of gross text on my screen. It's completely unformatted JSON. Let's make it nicer.

I'm going to take off the final step, `.then((data) => JSON.stringify(data))`, and instead do some interesting stuff to the JSON object.

I'm also going to change my `URL` to:

```jsx
const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations/contents/';
```

This is how to access folders inside of a repository via the API. Just append `/contents/` to it, and then you're at the base level of your repo.

### A brief aside on rate limits

I want to pull information out of the returned response. What does this response look like? Let's find out.

```
curl -i https://api.github.com/repos/samuelboland/Hyperfixations/contents/
```

This returns a bunch of information at the beginning. Some of this is **very important**. In particular, the `x-ratelimit-` parts. You can only request information from the API so often, and then you run out of requests until it resets. My rate limit section looks this this right now:

```
x-ratelimit-limit: 60
x-ratelimit-remaining: 35
x-ratelimit-reset: 1644522492
x-ratelimit-resource: core
x-ratelimit-used: 25
```

I have used 25 of my 60 requests for this time period, whatever that is. The limit resets at [Unix Time](https://en.wikipedia.org/wiki/Unix_time) 1644522492. I didn't mark down when I started, but it looks like this resets once per hour, dated from when I first made a request.

### Structure of the Github api response

Then it includes an _array_ of objects filled with key:value pairs about each file/folder at the top level of the repo. I'm interested in the `posts` and `posts/images` folders. Let's look at one of them:

```
curl -i https://api.github.com/repos/samuelboland/Hyperfixations/contents/_posts
```

Same as above. It returns an _array_ of objects with information about each file in the posts folder. I don't need most of that info, but I see a `download_url`. I tried opening that link, and it's just a raw download of my files! Just like clicking the "raw" button when viewing a file in the frontend github UI.

This should be enough information to get my posts.

### Avoiding rate limits while hacking at things

Chances are high that I won't get this code right the first time. I never do, the point is iteration. That means that I'm going to hit Github's API a lot.

To avoid that, I'm going to take one of the responses that I got earlier and hardcode it. I commented out my `const response = await fetch...` line, and pasted in the results of a previous query.

### Creating a list of download_urls

I know that I'm going to need to map over the response to get the download_url somehow. I'm going to do it in the main body of the page, since that lets me view the results more easily. Let's try this (and remember, I'm passing the response as a prop):

```jsx
const Test = (props) => {
    return (
        <div>
            {props.message.map((item) => {
                console.log(item);
            })}
        </div>
    );
};
```

Okay, this gives me an object per file with all of the key:value pairs in it. Let's dive in deeper.

```jsx
const Test = (props) => {
    return (
        <div>
            {props.message.map((item) => {
                console.log(item.download_url);
            })}
        </div>
    );
};
```

There we go. That gives me a list of URLs for the raw files. Perfect. I will now take out the test code from the main body of the page and return to the `getStaticProps` part.

### Retrieving the markdown files from Github

I adjusted my main body to just do a `{console.log(message)}`, and tried just getting a single file at first:

```jsx
export async function getStaticProps(context) {
    const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations/contents/_posts';

    //const response = await fetch(URL).then((response) => response.json());

    const response = {SOME FAKE DATA HERE}

    const urls = response.map((item) => item.download_url);

    const articles = await fetch(urls[0]).then((response) => response.text());

    return {
        props: { message: articles },
    };
}
```

The `.text` part was new to me, since I had previously always worked in JSON. Good to know! This returns an entire markdown file in my console.log, which is precisely what I wanted.

Now I need to map over _all_ urls instead of just one. First, I tried this:

```jsx
    const articles =
        urls.map((url) => {
            const markdown = fetch(url).then((response) => response.text());
            return markdown;
        }),
    ;
```

This did not work, because it returned promises. I tried adding an `await` before the fetch, but I received a warning that this doesn't do anything.

A quick google search pulled up this useful [Stackoverflow answer](https://stackoverflow.com/a/40140562). I need to wrap the entire function in `await Promise.all()`. That's easy enough:

```jsx
const articles = await Promise.all(
    urls.map((url) => {
        const markdown = fetch(url).then((response) => response.text());
        return markdown;
    })
);
```

That worked perfectly. My console was now logging an array with 5 text objects in it, each object containing a markdown file. Now to display each of them.

### Extracting title, slug, and content from the markdown files

I'm going to use a markdown parser for this. I am drawing heavy inspiration from [this wonderful tutorial](https://blog.openreplay.com/creating-a-markdown-blog-powered-by-next-js-in-under-an-hour), which I also mentioned above.

I'm using `gray-matter` and `markdown-it`. `Gray-matter` parses the frontmatter in my files (which contains metadata) and `markdown-it` parses the content and converts it to HTML for display.

Let's try adapting this code...I want to:

-   Get the frontmatter as an object
    -   Extract the title
    -   Extract the slug
-   Get the content of the post

This should work:

```jsx
const data = articles.map((article) => {
    const { data: frontmatter, content } = matter(article);
    return { data: frontmatter, content };
});

return {
    props: { message: data },
};
```

I'm still using a `console.log(message)` in my main page body to query the structure of this data, and it looks like I can get the pieces that I want with:

```jsx
const title = props.message[0].data.title;
const slug = props.message[0].data.slug;
const content = props.message[0].content;
```

Again I will need to iterate over these, but we're getting very close! I think that the `getStaticProps` is working as I expect. Now let's try displaying each post in a box on the actual page.

### Displaying title, slug, and content on the page

Let's pull out the old reliable `.map` function again, but this time, we need to have it return React elements.

As a small aside: Whenever you use `map` or `foreach` or another method that returns an array, and you want to display that array, React wants you to include a _unique_ `key=` element in the returned JSX. It uses this when manipulating the dom. I don't know the specifics, but it wants it.

We can use the `date` field in the frontmatter for that, since it's down to the...microsecond? What's 3 decimals of a second? `06:34:34.029Z`. Whatever, irrelevant, it will be unique for my use case.

So let's try this:

```jsx
const Test = (props) => {
    return (
        <div>
            {props.message.map((item) => {
                const title = item.data.title;
                const slug = item.data.slug;
                const date = item.data.date;
                const content = item.content;
                return (
                    <div key={date}>
                        <h2>{title}</h2>
                        <div>
                    </div>
                );
            })}
        </div>
    );
};
```

And that worked! It's ugly, and unformatted, and images aren't loading, but I am displaying my title and my content!

![Image showing title and content displayed without formatting](../images/Screen%20Shot%202022-02-10%20at%201.09.07%20PM.png)

Now for the final piece, let's get some proper markdown parsing so that things like headers and bold test display properly.

I also took the chance to make things look just a little bit nicer, in my opinion. Instead of using `props.message` to map, I put it into a separate variable named `posts`.

```jsx
const posts = props.message
return(
    <div>
        {posts.map((item) => {
            ...
        })}
)
```

I also added some sorting. Sorting in Javascript is weird. Take a look at [this page](https://flaviocopes.com/how-to-sort-array-by-date-javascript/) to see what I mean.

I also want my posts to be in reverse chronological order, so I did:

```jsx
const posts = props.message.sort((a, b) => b.date - a.date).reverse();
```

The end result looks like this:

```jsx
const Test = (props) => {
    const posts = props.message.sort((a, b) => b.date - a.date).reverse();

    return (
        <div>
            {posts.map((item) => {
                const title = item.data.title;
                const slug = item.data.slug;
                const date = item.data.date;
                const content = item.content;
                return (
                    <main key={date}>
                        <h1>{title}</h1>
                        <h3>{date}</h3>
                        <div dangerouslySetInnerHTML={{ __html: md().render(content) }} />
                    </main>
                );
            })}
        </div>
    );
};
```

### Some API best practices

I thought that I was done at this point. I naively assumed that my representation of the API was adequate.

There are also some best practices that Github recommends when querying their API, which I learned about when doing research for my next goal, which is to increase my rate limit...limits. It suggests using a header to specify the content type, and passing in your username as the User-Agent. [The documentation can be found here.](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#user-agent-required)

My fetch method now looks like this:

```jsx
const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations_Posts/contents/posts';

let headers = new Headers({
    'Accept': 'application/json',
    'User-Agent': 'samuelboland',
});

const apiResponse = await fetch(URL, {
    method: 'GET',
    headers: headers,
}).then((response) => response.json());
```

## Conclusion

To finalize, my code now looks like this:

```jsx
import matter from 'gray-matter';
import md from 'markdown-it';

const Test = (props) => {
    const posts = props.message.sort((a, b) => b.date - a.date).reverse();

    return (
        <div>
            {posts.map((item) => {
                const title = item.data.title;
                const slug = item.data.slug;
                const date = item.data.date;
                const content = item.content;
                return (
                    <main key={date}>
                        <h1>{title}</h1>
                        <h3>{date}</h3>
                        <div dangerouslySetInnerHTML={{ __html: md().render(content) }} />
                    </main>
                );
            })}
        </div>
    );
};

export async function getStaticProps(context) {
    const URL = 'https://api.github.com/repos/samuelboland/Hyperfixations_Posts/contents/posts';

    let headers = new Headers({
        'Accept': 'application/json',
        'User-Agent': 'samuelboland',
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
        })
    );
    const data = articles.map((article) => {
        const { data: frontmatter, content } = matter(article);
        return { data: frontmatter, content };
    });

    return {
        props: { message: data },
    };
}

export default Test;
```

There are other things that I plan to do with this, like make the formatting nicer, get images to work, make the date not-ugly, truncate the posts on the index page, have individual links for each post, etc etc.

Enjoy! 