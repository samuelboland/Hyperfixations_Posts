---
title: Moving to locally-stored Markdown files
slug: moving-locally-stored-markdown-files
description: Moving my posts from a separate Github repo to storing them with my site code.
author: Sam Boland
date: '2022-02-24T04:26:36.757Z'
tags: []
pullRequest: null
---

## Table of contents

## Explanation

This app is not my only side project. I have also been working on a very similar one to help me organize my documents at my job.

That project is very useful for seeing what development would have been like had I taken another path in this app. Sometimes I find that it would have been simpler, and other times, less so.

An example of the former is how I store my posts. In this app, I originally stored them as JSON in a MongoDB instance. Then, I moved to having them on Github in Markdown, but kept them in a separate repository. This lead me down the road of building an API interface to make getting articles easier. That part is documented in previous posts. (See my 2022-02-09 post on "Moving to Github.")

In that other application, I am storing my posts locally. And, honestly, it has been so much simpler to work with.

Currently, the development environment on this app, Hyperfixations.io, is slow. Every time I refresh a page, it has to make an API request to Github again.

I have decided to move my posts for this site into this repository as well.

It feels a bit like giving up, but it's just *so* much easier to work with. And I now have the experience that I gained by creating the API service in the first place.

## Dev Log

This will require some changes to how I get articles. Rather than calling an API, I will have to query the filesystem. Thankfully, Next.js makes this simple.

It will be nice to work with my framework, rather than fight it.

### Moving posts into repository

This part is simple enough, but is a pre-requisite to the rest.

I am creating a `_posts` directory in my repo and copying my posts into it. The underscore puts it near the top of the folder view in vscode, which is useful.

