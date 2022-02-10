---
title: CircleCI Config for Cypress and Jest with Coverage on NextJS
slug: circleci-config-cypress-jest-coverage-nextjs
description: null
author: null
date: '2022-02-09T06:45:37.480Z'
lastmod: '2022-02-10T07:44:51.814Z'
draft: false
tags: []
categories: []
---

## Introduction

A quick post on my CircleCI config. This config allowed me to generate code coverage for Jest and Cypress and upload that coverage to Codecov from CircleCI.

## The Config

The actual file can be found here: https://github.com/samuelboland/Hyperfixations/blob/getCypressWorkingOnCircleCI/.circleci/config.yml

The key components were solved in previous posts, specifically the esm piece in my last one. Once I got that working, this part just fell together.

Sorry that this post is so short, I've got some major refactoring to do. I just spent the last few days trying to get Cypress testing to play nicely, and non-hackily, with authentication, so that I can protect this New Post route.

I've decided it's too much hassle and I'm going to move to a CMS, perhaps a markdown based local one, or perhaps a service like Strapi. This gets rid of the need for authentication completely.
