---
title: Resolving remote images in markdown posts
slug: resolving-remote-images-markdown-posts
description: 'My posts are nearly complete, but there''s still a few pieces missing. One of them is the ability to show images!'
author: Sam Boland
date: '2022-02-17T17:51:42.664Z'
tags: []
pullRequest: https://github.com/samuelboland/Hyperfixations/pull/40
---

- [Introduction](#introduction)
- [Dev Log](#dev-log)
  - [Search Posts for Images at build time and replace them with links to image on github](#search-posts-for-images-at-build-time-and-replace-them-with-links-to-image-on-github)
  - [Use Nextjs Image component to display image](#use-nextjs-image-component-to-display-image)
    - [Introducing the Nextjs Image Component](#introducing-the-nextjs-image-component)
    - [Image size is a challenge](#image-size-is-a-challenge)
    - [Using the Remark Ecosystem to render images in markdown in Nextjs](#using-the-remark-ecosystem-to-render-images-in-markdown-in-nextjs)
    - [Replacing Markdown-it with remark and remark-html](#replacing-markdown-it-with-remark-and-remark-html)
    - [Replacing dangerouslySetInnerHtml with ReactMarkdown](#replacing-dangerouslysetinnerhtml-with-reactmarkdown)
- [Wrapping up](#wrapping-up)
  - [Found a bug in testing](#found-a-bug-in-testing)

## Introduction

My posts are nearly complete, but there's still a few pieces missing. One of them is the ability to show images!

I've been embedding images in a lot of my posts, and those images are saved in a folder on my posts repository, just like the actual content. But they don't show up.

This makes sense - the image links in the posts are wrong in a few ways. First, they don't use the next.js `<Image>` tag, which allows for some nice build-time optimization. Second, they don't even exist on the server when I build the site, they exist on my repo.

Next.js is looking for the image, and expects it to have been loaded into a static `/images` route. I expect that the `<Image>` tag would handle that.

So here's my plan:

- Search posts for images at build time
- If image found, change the link to an absolute url
- If image found, wrap in a next.js `<Image>` tag

This is silly, and would be far easier if I just had these files stored locally. Lol.

## Dev Log

### Search Posts for Images at build time and replace them with links to image on github

Let's try this! I'm going to look at the single post page route. Let's try adding in some parsing logic...

Ok, this actually turned out to be rather simple. It's not pretty, but it's simple.

```js
    if (data.content.includes('.png')) {
        const allPngFilesBetweenParens = /\(([^)]+png)\)/g;
        const baseUrl = '(https://github.com/samuelboland/Hyperfixations_Posts/raw/main/images/';

        const images = data.content.match(allPngFilesBetweenParens);
        console.log(images);

        images.map((image) => {
            const baseImage = image.split('/')[2];
            data.content = data.content.replace(image, baseUrl + baseImage);
        });

        //data.content = data.content.replace(allPngFilesBetweenParens, baseUrl);
    }
```

I am really beginning to feel how much Next.js is NOT built for this. It really expects local files. But hey this is a fun exercise.

Let's step through this though.

First, I scan the file to see whether it contains the string `.png`. If so, I do a few steps in this order:

- Define some variables to use later. The first is a [regular expression](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions), which, tbh, is mostly magic to me. I search Google for "regex find all text between parentheses" and then added then used [Regex 101](https://regex101.com/) to figure out where I should add "png" to match only png strings.
- I then get a list of all images that match that regular expression and store them in an array.
- Finally, I map over each of those images, extract the filename, and then replace the original image string with this newly constructed one.

And this works! It makes images appear. But there's a problem: These images are being [hotlinked](https://simple.wikipedia.org/wiki/Hotlinking) directly from Github. This is bad.

### Use Nextjs Image component to display image

#### Introducing the Nextjs Image Component

The [Nextjs image component](https://nextjs.org/docs/basic-features/image-optimization) optimizes images, makes pages load faster, only loads images when they enter the viewport, etc etc. You can read about it at the link.

I want to use this component. But how? There is a section about remote images, and it looks like I have to do something like this:

```jsx
<Image
    src="/me.jpg"
    alt="Picture of me"
    width={500}
    height={500} 
/>
```

I also need to define a "loader," which determines the base URL for the above `src`. That's discussed [here](https://nextjs.org/docs/api-reference/next/image#loader), and looks something like this:

```js
const myLoader = ({ src, width, quality }) => {
  return `https://example.com/${src}?w=${width}&q=${quality || 75}`
}
```

#### Image size is a challenge

But I still need the image dimensions. I wonder if the Github API returns this if I query it for an image file? Let me try!

```bash
curl -i https://api.github.com/repos/samuelboland/Hyperfixations_Posts/contents/images/Screen%20Shot%202022-02-12%20at%207.28.58%20PM.png
```

Hmm, nope. No size information. Some ideas:

- Perhaps I could download it in the API call and then get the image information from there? If I did that, I could just serve the image locally...but idk if that's possible.
- Look online to see if someone knows how to get image size info from Github
- Look online to see if there are any other solutions

Let's go with choice three.

#### Using the Remark Ecosystem to render images in markdown in Nextjs

The "Remark Ecosystem" is something that keeps popping up in my research around markdown in Nextjs, no matter the specific topic, so I think it's time to give it a try.

Remark is a collection of utilities that transforms markdown with plugins. There is also remark-html, which transforms html.

I found a very simple and to-the-point blog post explaining how to add remote images to markdown with the Image component in Nextjs, [here](https://www.theviewport.io/post/using-nextjs-and-nextimage-with-mdx-markdown-processing).

Let's do it!

#### Replacing Markdown-it with remark and remark-html

This part was simple. First, I installed the `remark` and `remark-html` packages. Then, I removed the markdown-it import in my `[post].js`.

I then created a new file in my library folder, `/lib/markdownToHtml.js`. This file looks like this:

```js
import { remark } from 'remark';
import html from 'remark-html';

export default async function markdownToHtml(markdown) {
    const result = await remark()
        .use(html)
        .process(markdown)
        .then((processedMarkdown) => processedMarkdown.toString())
        .then((html) => {
            return html;
        });

    return result;
}
```

It's a little different than the example on the site, because I chain the `.toString()` as part of the promise, and explicitly return the result as well. Idk if I need to do that, but it feels nicer somehow?

Then, in getStaticProps in `[post].js`, I made a small change:

```js
export async function getStaticProps({ params }) {
    const data = await githubApi.fetchOne(params.post);
    data.content = await markdownToHtml(data.content);
    return {
        props: { data },
    };
}
```

I use my new `markdownToHtml` function to parse the markdown content.

Up in the main body of the page, I just remove the call to `md` that I had. My code to render the markdown is now:

```js
<div data-cy="postShowBody" dangerouslySetInnerHTML={{ __html: post.content }} />
```

#### Replacing dangerouslySetInnerHtml with ReactMarkdown

[React markdown](https://github.com/remarkjs/react-markdown) lets us render markdown without any dangerous innerHTML setting, has plugin support, and supports components to *change how the markdown is translated to html*.

I did some searching around, and, somewhat annoyingly, ReactMarkdown changed their API a bit a go. You used to add "renderers" to the ReactMarkdown component, but now you add "components." A lot of examples near the top of Google searches are for the old style.

Thankfully, not all of them are! Check out [this blog post by Amir Ardalan](https://amirardalan.com/blog/use-next-image-with-react-markdown). In it, he describes a how he got this to work on his site.

Basically, you define a `component`, like so:

```js
const MarkdownComponents = {
    // Stole this directly from here: https://amirardalan.com/blog/use-next-image-with-react-markdown
    // Convert Markdown img to next/image component and set height, width and priority
    // example: ![AltText {priority}{768x432}](...
    p: (paragraph) => {
        const { node } = paragraph;

        if (node.children[0].tagName === 'img') {
            const image = node.children[0];
            const alt = image.properties.alt?.replace(/ *\{[^)]*\} */g, '');
            const isPriority = image.properties.alt?.toLowerCase().includes('{priority}');
            const metaWidth = image.properties.alt.match(/{([^}]+)x/);
            const metaHeight = image.properties.alt.match(/x([^}]+)}/);
            const width = metaWidth ? metaWidth[1] : '768';
            const height = metaHeight ? metaHeight[1] : '432';

            return (
                <Image
                    src={image.properties.src}
                    width={width}
                    height={height}
                    className="postImg"
                    alt={alt}
                    priority={isPriority}
                />
            );
        }
        return <p>{paragraph.children}</p>;
    },
};
```

This one is a bit fancy. Naively, all you would need is a component to replace the images. However, that results in putting a `<div>` inside of a `<p>`, which is against html standards and results in errors in the console.

I would suggest reading that site for an explanation, as it explains it better than I would.

## Wrapping up

While I was at it, I also adjusted the posts index page so that it only shows the titles and timestamps of each post, rather than the entire thing. The titles link to the posts.

### Found a bug in testing

It's a very good thing that I have Cypress tests, because they fond a bug! If I attempted to open a post *without* an image, it failed!

If the image processing logic triggered when there wasn't an image, it would break.

I'm tired of this, so I ended up just tossing it into a big old try/catch statement. It is not nice. But it works! Here's the final method I'm using to find images. It's quite unsatisfying, but it's functional.

```js
const findImages = (post) => {
    // Check if file contains an image. If so, extract the base filename of the image,
    // remove the local folder structure, and prepend the fixed repository url to it.
    // Then, replace all instances of the original with the newly created url.

    // Added a try/catch statement because it was triggering whenever the string "png" was present,
    // obviously, so it would sometimes trigger when there wasn't an image to process.
    // This made it mad.
    // This is a gross way to handle this lmao.
    try {
        if (post.includes('.png')) {
            console.log(post);
            const allPngFilesBetweenParens = /\(([^)]+png)\)/g;
            const baseUrl =
                '(https://github.com/samuelboland/Hyperfixations_Posts/raw/main/images/';
            const images = post.match(allPngFilesBetweenParens);
            images.map((image) => {
                const baseImage = image.split('/')[2];
                post = post.replace(image, baseUrl + baseImage);
            });
            return post;
        }
    } catch (err) {
        return post;
    }
    return post;
};
```
