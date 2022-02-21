---
title: Removing Google Analytics
slug: removing-google-analytics
description: In which I remove Google Analytics because the privacy policy requirements, legal landscape, and ethical considerations add up to too much to keep it
author: Sam Boland
date: '2022-02-21T21:45:36.113Z'
tags: []
pullRequest: null
---

## Quick Post on Privacy Page

So it turns out that I need to implement a privacy policy, since I'm using Google Analytics. I'm going to implement a simple one by creating a new `/privacy` page, and linking to it from the footer that shows on all pages.

I used a few privacy policy generators online to get a feel for how to do this, and most were too detailed for my needs. It seems like I need, at least, the following:

- Explanation of what sort tracking technology I use. Cookies, beacons, etc.
- A method to opt out.
- A contact method if users have questions.

## You know what - I'm dropping Google Analytics

Google analytics is very cool, and I like seeing information about how people around the world look at this site, but there are just so many hoops to jump through. And, it turns out, there was a very [recent ruling in Austria](https://www.pymnts.com/google/2022/ruling-google-analytics-violates-privacy-law/) on January 13, 2022, that declares that the use of Google Analytics in any capacity violates the GDPR.

I'm no lawyer, but I don't want to get sued by someone crawling webpages and looking for people to make examples of.

And, like I said in my original post about it, I always had ethical reservations. I just...I like the pretty numbers. And the maps. I gave in I'm sorry.

Anyways, I am now removing Google Analytics. If you are reading this, then I've pushed the code change that does so.
