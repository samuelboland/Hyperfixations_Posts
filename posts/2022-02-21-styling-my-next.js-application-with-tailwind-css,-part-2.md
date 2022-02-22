---
title: 'Styling my Next.js application with Tailwind CSS, Part 2'
slug: styling-js-application-tailwind-css-part-2
description: null
author: Sam Boland
date: '2022-02-21T23:33:44.387Z'
tags: []
pullRequest: https://github.com/samuelboland/Hyperfixations/pull/52
---

## Introduction

Welcome to the second part of my series on theming my site with Tailwind CSS and DaisyUI.

I am going to try a more compact writing style for this post. I want to see how it feels.

In this post, I will implement a header, footer, and navigation dropdown.

## Dev Log

### Footer

I am using the [footer with copyright text and social icons](https://daisyui.com/components/footer/#footer-with-copyright-text-and-social-icons).

I am removing the YouTube and Facebook links. I will keep the Twitter link, and add one for Github.

For icons, I'm using [React Icons](react-icons.github.io). The footer from DaisyUI uses SVG paths. For consistency, I am removing them. I have chosen Font-Awesome's icon set, and plan to use only Font-Awesome icons.

There is also an icon on the left that looks like a hashtag or pound sign. I will remove that, and replace it with the name of the site, for now. A logo could go there later.

I also set the footer to always stay at the bottom of the page by adding `fixed inset-x-0 bottom-0`.Further, I changed the `<a>` links to be buttons, and used an effect that first saw in the Header, which I discus below, to give them a nice hover effect.

I adjusted the icon size by passing `size={24}`. The default is 1em, which is usually 16px. I cannot set the size in em though, so I have to use px.

The code is now:

```js
import Link from 'next/link';
import { FaGithub, FaTwitter } from 'react-icons/fa';

const Footer = () => {
    return (
        <footer class="footer fixed inset-x-0 bottom-0 items-center bg-neutral p-4 text-neutral-content">
            <div class="grid-flow-col items-center">
                <h1>Hyperfixations |</h1>
                <p>© Sam Boland 2022</p>
            </div>
            <div class="grid-flow-col gap-4 md:place-self-center md:justify-self-end">
                <Link href="https://twitter.com/samcboland">
                    <button class="btn btn-ghost btn-square">
                        <FaTwitter size={24}/>
                    </button>
                </Link>
                <Link href="https://www.github.com/samuelboland/">
                    <button class="btn btn-ghost btn-square">
                        <FaGithub size={24}/>
                    </button>
                </Link>
            </div>
        </footer>
    );
};

export default Footer;
```

### Header

The header is also taken from DaisyUI, but heavily altered, so there isn't much point in specifying which one it was. Here it is now:

```js
import { FaBars } from 'react-icons/fa';
import Link from 'next/link';

const Header = () => {
    return (
        <div class="navbar bg-base-300">
            <div class="flex-1">
                <Link href="/">
                    <a class="btn btn-ghost text-xl normal-case">Hyperfixations.io</a>
                </Link>
            </div>
            <div class="flex-none">
                <button class="btn btn-ghost btn-square">
                    <FaBars size={24} />
                </button>
            </div>
        </div>
    );
};

export default Header;
```

It will change soon, when I introduce the navigation dropdown.

### Navigation Dropdown Menu

[This one](https://daisyui.com/components/dropdown/#dropdown-in-navbar) is an example of a dropdown in a menu. By default, that dropdown uses the `dropdown-end` property. I didn't like how it looked (it covered the button), so I'm using `dropdown-left`. I also added next.js `<Link>` elements, and customized the links.

Here's the navbar with a dropdown in TailwindCSS and DaisyUI:

```js
import { FaBars } from 'react-icons/fa';
import Link from 'next/link';

const Header = () => {
    return (
        <div class="navbar bg-base-300">
            <div class="flex-1">
                <Link href="/">
                    <a class="btn btn-ghost text-xl normal-case">Hyperfixations.io</a>
                </Link>
            </div>
            <div class="dropdown-left dropdown">
                <label tabindex="0" class="btn btn-ghost rounded-btn">
                    <FaBars size={24} />
                </label>
                <ul
                    tabindex="0"
                    class="dropdown-content menu rounded-box mt-4 w-52 bg-base-100 p-2 shadow"
                >
                    <li>
                        <Link href="/">
                            <a>Home</a>
                        </Link>
                    </li>
                    <li>
                        <Link href="/posts">
                            <a>Dev Log</a>
                        </Link>
                    </li>
                    <li>
                        <Link href="/about">
                            <a>About</a>
                        </Link>
                    </li>
                </ul>
            </div>
        </div>
    );
};

export default Header;
```

## Conclusion

That wraps it up. I have implemented a Footer, Header, and a Navigation dropdown in the header. I will continue to iterate on the site design and post articles about my experience.

The page now looks like this:

![Screenshot of my site showing a nice header and footer, but no styling in the body yet](../images/Screen%20Shot%202022-02-22%20at%209.33.32%20PM.png)
