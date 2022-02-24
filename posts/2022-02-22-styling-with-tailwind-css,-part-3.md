---
title: 'Styling with Tailwind CSS, Part 3'
slug: styling-tailwind-css-part-3
description: Implementing a theme picker with DaisyUI and Next-theme. Surprisingly simple!
author: Sam Boland
date: '2022-02-23T05:35:51.782Z'
tags: []
pullRequest: null
---

## Table of contents

## Introduction

In this article, I intend to implement a theme picker. I want it to be in the header, probably behind some icon indicating art or design.

I will also add the Typography plugin to the entire site, for now.

## Dev Log

I previously used `next-theme` as a proof of concept. In fact, it looks like I left the `<ThemeProvider>` wrapper in my `_app.js`. That saves me a step now.

I'm going to implement this theme switcher in a separate component, just to make it easier to think about.

### Creating a new ThemeChanger component

Creating a new file at `/components/ThemeChanger.jsx`. I will eventually import and place this in my Header.

I'm using the `ThemeChanger` example that I discussed in Part 1 of this series as inspiration.

This turned out to be very simple. I love this `next-themes` package, it's great.

Here's the code:

```js
import { FaPaintBrush } from 'react-icons/fa';
import { useTheme } from 'next-themes';

const ThemeChanger = () => {
    const { theme, setTheme } = useTheme();

    return (
        <>
            <div class="dropdown-left dropdown">
                <label tabIndex="0" class="btn btn-ghost rounded-btn" data-cy="headerDropdownMenu">
                    <FaPaintBrush />
                </label>
                <ul
                    tabIndex="0"
                    class="btn-group dropdown-content menu rounded-box mt-4 w-52 bg-base-100 p-2 shadow"
                >
                    <li class="btn" onClick={() => setTheme('light')}>
                        Light
                    </li>
                    <li class="btn" onClick={() => setTheme('dark')}>
                        Dark
                    </li>
                    <li class="btn" onClick={() => setTheme('cupcake')}>
                        Cupcake
                    </li>
                </ul>
            </div>
        </>
    );
};

export default ThemeChanger;
```

I then imported `ThemeChanger` into my header, and placed it next to the dropdown menu, with a small vertical separator from DaisyUI between them.

Here's what it looks like in the cupcake theme:

![Image of site with new them in place](../images/Screen%20Shot%202022-02-22%20at%209.52.29%20PM.png)

### Applying the Typography plugin to the entire site

I already have this plugin in place for my `[post].js` page. I'm going to just add a `prose` class to the post and site index.

## Conclusion

This was surprisingly easy. I'm getting a bit better at this. :)
