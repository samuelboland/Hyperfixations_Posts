---
title: Add syntax highlighting to posts
slug: add-syntax-highlighting-posts
description: null
author: null
date: '2022-02-19T17:59:17.838Z'
lastmod: '2022-02-19T19:19:26.411Z'
draft: true
tags: []
categories: []
pullRequest: null
---

- [Explanation](#explanation)
- [Dev Log](#dev-log)

## Explanation

I want syntax highlighting in my posts, because plain code blocks are hard to read. In a previous post about resolving images in markdown, I used a post from a blog to help me with a part of it. Turns out, that blog also has a post [about syntax highlighting](https://amirardalan.com/blog/syntax-highlight-code-in-markdown), which I'm going to implement.

It leverages the extensibility of `ReactMarkdown` by making a new `SyntaxHighlighter` component, which we can then just plug right in to `ReactMarkdown`.

## Dev Log

Let's start by implementing the `SyntaxHighlighter` component.

Let's see, it looks like they're importing [Prism](https://prismjs.com/), but from [React Syntax Highlighter](https://github.com/react-syntax-highlighter/react-syntax-highlighter) so I need to `npm install react-syntax-highlighter`.

Now I will create a `SyntaxHighlight.jsx` file in my `lib` folder. I'm choosing to put this in `lib` rather than `components` because it is not itself generating any jsx. The distinction is small, and I could go either way.

Then I just add it to my `[posts].js`, like so:

```js
    <ReactMarkdown
        data-cy="postShowBody"
        components={(MarkdownComponents, SyntaxHighlight)}
    >
        {post.content}
    </ReactMarkdown>
```

And it...just works. Wow. Okay.

I'm just. Used to having to fix it. But it just worked. Okay, sweet!

Well that's that. All credit to the devs of these packages and to Amir Ardalan, who's site has great tutorials: <https://amirardalan.com/blog/syntax-highlight-code-in-markdown>

The theme doesn't fit with my page's design, but I'll mess with that.
