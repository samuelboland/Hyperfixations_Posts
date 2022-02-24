---
title: 'Styling my Next.js application with Tailwind CSS, Part 1'
slug: styling-js-application-tailwind-css-part-1
description: 'Redesigning my site with Tailwind CSS, a series wherein I discuss the trials and tribulations of someone with no design skills trying to make a pretty site.'
author: Sam Boland
date: '2022-02-20T22:50:26.338Z'
tags:
  - tailwind
  - css
pullRequest: 'https://github.com/samuelboland/Hyperfixations/pull/52'
---

## Table of contents

## Introduction

My site is ugly! Here's a pic of the front page now:

![Image of the only-slightly-styled front page](../images/Screen%20Shot%202022-02-20%20at%202.51.46%20PM.png)

I don't like it. I want to change it.

I'm going to try using Tailwind CSS. Now, I've tried using this before, and I didn't really like how long some of the jsx lines got.

To explain a bit: [Tailwind CSS](https://tailwindcss.com/) is a "utility-first CSS framework." It lets you compose styles right in your .jsx files. For example, taken from their front page, you might do something like:

```html
  <div class="pt-6 md:p-8 text-center md:text-left space-y-4">
```

All of those classes are applying CSS to that element under the hood. But, as you can see, some of the classes get real long. I found that hard to reason about, and I found it difficult to work with in a...React-y, way. I always wanted to pull out the styling into some sort of shared components and input them as variables. [Tailwind suggests against that, though.](https://tailwindcss.com/docs/reusing-styles)

In their documentation on "Reusing Styles", they say:

>Tailwind encourages a utility-first workflow, where designs are implemented using only low-level utility classes. This is a powerful way to avoid premature abstraction and the pain points that come with it.

They suggest that if you find yourself in a place where you have a lot of styling that is repeated verbatim near to each other, they suggest:

- Multi-cursor editing
- Loops or maps
- Extracting components

I suppose those are good enough. They do give some reasoning behind this, by the way. They say:

>Yes, HTML templates littered with Tailwind classes are kind of ugly. Making changes in a project that has tons of custom CSS is worse.

I'm willing to give it a real shot.

## Interesting Tailwind Resources

I found this great Github repo called [Awesome Tailwind CSS](https://github.com/aniftyco/awesome-tailwindcss), which is a collection of useful links for Tailwind development. I'm going to go through it and add interesting ones here.

### Libraries of some sort

- [Headless UI](headlessui.dev):"A set of completely unstyled, fully accessible UI components, designed to integrate beautifully with Tailwind CSS."
- [Heroicons](https://heroicons.com/): "Beautiful hand-crafted SVG icons, by the makers of Tailwind CSS." They're also [MIT licensed](https://opensource.org/licenses/MIT)!
- [Tailblocks](https://tailblocks.cc/): A bunch of copy & paste ready component examples. It's very nice.

### Sandbox

- [Tailwind Play](https://play.tailwindcss.com/): A code sandbox for tailwind. It's pretty cool.

### Vscode plugins

- [Headwind](https://github.com/heybourn/headwind): an "opinionated Tailwind CSS class sorter." I like the sound of that.

### Tailwind Plugins

- [Typography](https://github.com/tailwindlabs/tailwindcss-typography): I've used this before when following a tutorial. It adds a new class, `<prose>`, which formats text nicely
- [DaisyUI](https://daisyui.com/): Highly rated component library.
- [Line Clamp](https://github.com/tailwindlabs/tailwindcss-line-clamp): Visually truncates text after a fixed number of lines. I've been thinking about making something like this myself, for my post index page.

There are many, many more plugins and resources for Tailwind, but these are the ones that caught my eye the most.

## Dev Log

### Installing Tailwind and removing previous styling

Let's install it first. The instructions to do so [are here](https://tailwindcss.com/docs/guides/nextjs)).

After that, I need to remove my existing styling. I do that by removing all `.scss` imports, along with any classNames that refer to `{styles.foo}`.

I also removed my `<AnimationWrapper>` components, at least for now. I did not delete the original component though, so I might put it back.

Also, uninstall my old CSS, which was provided by `marx-css` (get it, because it's classless? lol I legitimately enjoy that pun)

```bash
npm uninstall marx-css
```

I followed along with the instructions to install Tailwind, which I won't repeat here.

### Installing DaisyUI

I'm going to try using [DaisyUI](https://daisyui.com/) for the components in my site. It has a lot of components pre-made, and honestly I'm not good at making pretty looking things on my own. It's a major gap in my capabilities.

The instructions to install Daisy are on their site. Simply `npm i daisyui`, and require it in your `tailwind.config.js`.

As part of this, I will also install [Typography](https://tailwindcss.com/docs/typography-plugin).

The DaisyUI [documentation on layout and typography](https://daisyui.com/docs/layout-and-typography/) suggests using this plugin, and that we need to require Daisy *after* Typography.

Just adding what we've done so far gives us a bit of styling. Here's what a post look like right now:

![Image showing some basic styling applied. The background is a dark blue and gray color, and the text is white with a sans-serif font.](../images/Screen%20Shot%202022-02-20%20at%207.24.30%20PM.png)

#### Testing Typography

Using the typography plugin should be as simple as wrapping some text in a tag like this:

```js
<article class="prose"> 
```

I'm going to edit the `[post].js` page to do that.I think that I should it outside of the `<ReactMarkdown>` tags.

After some experimentation, I am also adding a div to center the article class, and centering the article class itself. I don't know if both are necessary.

This is accomplished with the `mx-auto` class.

My `[post].js` now looks like this:

```js
  <section>
      <div class="container mx-auto px-5 py-24">
          <article className="prose mx-auto md:prose-lg lg:prose-xl">
              <h1 data-cy="postShowTitle">{post.data.title}</h1>
              <h2 data-cy="postShowDate">{post.data.date}</h2>
              <ReactMarkdown
              data-cy="postShowBody"
              components={(MarkdownComponents, SyntaxHighlight)}
              >
                  {post.content}
              </ReactMarkdown>
          </article>
      </div>
  </section>
```

And the page, like this:

![Image showing centered text, with a legible paragraph and heading font.](../images/Screen%20Shot%202022-02-20%20at%207.42.28%20PM.png)

### Adjusting the Theme

#### First Attempt: Adding data-theme to `_app.js`: Fail

DaisyUI comes with a few built-in themes. The theme is set in in html tags, and thus can be adjusted at runtime by the user. DaisyUI suggests using [theme-change](https://github.com/saadeghi/theme-change), a little package to help with just this scenario.

First, I want to test that I can apply a theme at all. Currently, it's using the dark theme, since it auto-detects my system settings. I'm going to try adding `data-theme="cupcake"` to the `<Layout>` component in my `_app.js`. As a reminder, this Layout component wraps the rest of my application, so I *think* that it should apply the theme to everything.

That didn't work. Hmmm.

#### Second Attempt: Using Next-themes: Success

The [Next-themes](https://github.com/pacocoursey/next-themes) package supplies a wrapper component that you can place around your application. It aims to make it easier to switch themes.

According to [a discussion on DaisyUI's github](https://github.com/saadeghi/daisyui/discussions/386), these work correctly together. Let's try it.

```bash
npm install next-themes
```

And then wrap `_app.js` in `<ThemeProvider>`. Mine now looks like this:

```js
export default function App({ Component, pageProps: { session, ...pageProps }, router }) {
    <Google Analytics Stuff>
    return (
        <>
          <Seo Stuff>
          <Google Analytics Stuff>
          <ThemeProvider>
              <Layout>
                  <AnimatePresence
                      exitBeforeEnter
                      initial="start"
                      onExitComplete={() => window.scrollTo(0, 0)}
                  >
                      <Component {...pageProps} key={router.pathname} />
                  </AnimatePresence>
              </Layout>
          </ThemeProvider>
        <>
    );
}
```

And of course, remember to `import { ThemeProvider } from 'next-themes'`.

Now, I'm going to copy from the example repo that I linked above, specifically from the [index.jsx](https://github.com/saadeghi/daisyui-nextjs-nextthemes/blob/master/pages/index.jsx) page. I'm going to put this in a new page, just to test the functionality.

It worked!

![Image showing a test page with the DaisyUI Cupcake theme applied, and buttons to change the theme. There are also example buttons to show how the theme changes them.](../images/Screen%20Shot%202022-02-21%20at%2011.35.44%20AM.png)

![The same as the previous image, except using the Retro theme.](../images/Screen%20Shot%202022-02-21%20at%2011.35.55%20AM.png)

I don't need to allow theme changing quite yet, so for now I'll set a default theme in my `_app.js`. Now that I've seen how this works, I think I can make that happen.

#### Applying a theme Globally

This combines my work in the previous sections. I'm getting rid of my test page, and implementing the theme at the `_app.js` level, so that it applies everywhere.

I will add `import { useTheme } from 'next-themes'`, add a state handler with `const { theme, setTheme } = useTheme()`, and then set the theme with `setTheme('cupcake')`. Let's see if that works.

It does!

## Conclusion

This is going to be a long process, so I will split this post into multiple parts. As a refresher, here's what we did:

1. install TailwindCSS and DaisyUI
2. Find some neat tools for Tailwind.
3. Install the official Typography plugin from the Tailwind team and test it out.
4. Figure out how to do themes.
