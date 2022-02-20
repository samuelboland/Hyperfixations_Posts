---
title: Coverage reports in Next.js with Istanbul and Cypress
slug: coverage-reports-js-istanbul-cypress
description: 'Despite being a QA engineer, I don''t write any tests at my job. At my current company, our QA work is entirely manual, and has a large operational component.'
author: Sam Boland
date: '2022-02-06T06:41:05.410Z'
tags: []
pullRequest: None
---

- [Introduction](#introduction)
  - [Problem Statement](#problem-statement)
  - [Writing Plan](#writing-plan)
  - [Definitions](#definitions)
- [Code Coverage Reports with Cypress in NextJS](#code-coverage-reports-with-cypress-in-nextjs)
- [Conclusion](#conclusion)

## Introduction

Despite being a QA engineer, I don't write any tests at my job. At my current company, our QA work is entirely manual, and has a large operational component. I don't want to go too much into the details here, you can read about it on my company's blog if you care about it: <https://engineering.appfolio.com/appfolio-engineering/2021/5/27/quality-assurance-at-appfolio-property-manager-2021>

Despite this, or maybe because of this, I want to learn more automated testing. I haven't been writing tests so far, so I dove in to it over the weekend.

### Problem Statement

I wanted these tests to be written in Cypress, run on CircleCI, automatically determine code coverage, and upload the code coverage results to Codecov.

### Writing Plan

I started writing this post and then realized that it's going to get quite long. I'm going to break this up into a few pieces. I'm also not going to specify what the future posts will be right now, since I never know what I'm going to end up writing beforehand. I do, however, plan to cover the following topics:

- Instrumenting my code <-- **you are here**!
- Generating code coverage reports with Cypress
- Setting up CircleCI to run these tests automatically

  - spoilers: it involves setting the `esmExternals` flag to `false` in my `next.config.js`.

- Uploading them to Codecov
- How I tried to use Cypress to test Google-provided OAuth and failed
- anything else I think of that's relevant

You may notice that I don't intend to cover installing Cypress, or the basics of Cypress tests. That's because it's been done. Not that the other topics haven't, but I'm more interested in talking about them. There are official Next.js docs for Cypress, here: <https://nextjs.org/docs/testing#cypress>

And one more thing: I will present this in a logical ordering for the reader. I did not figure this all out in that order. If you are reading this and feel out of your depth, wondering how someone could possibly just figure this out in some logical ordering: I didn't. It was a long and messy process, involving a lot of Google searches, Github issues, and Stack Overflow answers.

### Definitions

First, let's go over some definitions:

**CI**_:_ Continuous Integration. The practice of running a suite of tests on each code change to ensure that nothing in your application broke due to changes that were included in that commit (or commits).

**CircleCI**: A CI provider. While I _can_ run the tests myself on my local machine (and I do), I also wanted to learn about integrating with CircleCI, since my company uses it a lot. This runs my test suite every time I `git push`. And it's free for small projects like mine!

**Code Coverage**: How much of your actual codebase is being covered by your automated test suite. This requires "instrumentation."

**Instrumentation:** The process of automatically adding some form of markers to your code that can tell you whether a particular line of code was triggered. I use the _Istanbul_ package to do this.

**Codecov**: A third party code coverage analyzer and documenter.

This part, especially the part about CircleCI, proved much more difficult than I initially thought. I spent my entire weekend figuring this out.

By the way, please know that I was figuring out most of this as I went along. I had heard of Cypress, but that was it. This is also my first time using Nextjs, or building my own test suite on CircleCI, so if I made any mistakes....sorry!

## Code Coverage Reports with Cypress in NextJS

Let's start with something that I can do locally: code coverage reports. You can read more about how this works under the hood at Istanbul's site, here: <https://istanbul.js.org/>

The official Cypress docs for this topic are here: <https://docs.cypress.io/guides/tooling/code-coverage#Install-the-plugin>

Let's install it first:

```
    npm install istanbul --save-dev
```

In order to get automatic instrumentation to work, I had to add a custom `.babelrc` file in the root folder of my project, and supply the following configuration:

```jsx
    // .babelrc

    {
        "presets": ["@babel/preset-react", "next/babel"],
        "plugins": ["istanbul"]
      }
```

Note that doing this _disables_ the Rust compiler for Nextjs. This makes site compilation slower. I have not found a way to get instrumentation with Cypress and the Rust compiler working together. I'm sure someone will make it work at some point, though. If you're reading this much later than February 2022, I'd suggest searching for newer documentation that make this obsolete.

Then you need to set up your Cypress config. Update your `cypress/plugins/index.js` to include the cypress code coverage task.

```jsx
    // cypress/plugins/index.js

    module.exports = (on, config) => {
        require('@cypress/code-coverage/task')(on, config);
        return config;
```

Also, update your `cypress/support/index.js` as the docs specify:

```jsx
import '@cypress/code-coverage/support';
```

I diverged from the official docs at this point, and added some customization to my istanbul configuration. For some reason, the actual tool that does the implementation isn't named `istanbul`, it's named `nyc`.

Here's a segment of my `package.json`:

```jsx
    // package.json
    (...)
    "nyc": {
            "report-dir": "coverage",
            "reporter": [
                "cobertura",
                "html"
            ],
            "include": [
                "Components/**",
                "lib/**",
                "pages/**"
            ],
            "all": true
        },
    (...)
```

This puts my reports inside of the `coverage` directory in the root of my project, returns reports in machine-readable and human-readable formats, specifies which directories I want it to look at, and tells is to also generate code coverage for files that do not have tests yet. (That's the `"all":true` part.)

Now you just need to run your tests. I added some shortcuts to my `package.json` scripts to help with that:

```jsx
    // package.json
    (...)
    "script": {
        "cov": "open -a 'Google Chrome' coverage/index.html ",
        "cypress:open": "cypress open",
        "cypress:ci": "cypress run --quiet"
    }
```

I added a shortcut to open the code coverage report. This will only work on OS X.

## Conclusion

I believe that the above is all that you need to run coverage reports locally. I'll admit, I didn't write this immediately after I finished, like I should have, so I may have missed something. If I did, well, I don't have comments or anything yet so I suppose you'll just have to be annoyed by it for now. Sorry!
