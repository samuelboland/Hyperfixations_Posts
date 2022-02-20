---
title: Add caching to CircleCI Build Step
slug: add-caching-circleci-build-step
description: null
author: null
date: '2022-02-19T19:22:03.630Z'
lastmod: '2022-02-20T20:17:40.105Z'
draft: true
tags: []
categories: []
pullRequest: null
---

- [Explanation](#explanation)
- [Dev Log](#dev-log)
  - [Refactoring CircleCI Config](#refactoring-circleci-config)
  - [Wait, let's go another direction](#wait-lets-go-another-direction)
    - [Breaking down the CircleCI Orb, starting with the Jobs](#breaking-down-the-circleci-orb-starting-with-the-jobs)
    - [Finding the `install` command](#finding-the-install-command)
    - [Breaking down the `setup` job](#breaking-down-the-setup-job)
    - [Adding it as a post-step](#adding-it-as-a-post-step)
  - [Figuring out why `.next/cache` is not present in circleCI](#figuring-out-why-nextcache-is-not-present-in-circleci)
- [Conclusion](#conclusion)

## Explanation

CircleCI supports [caching dependencies](https://circleci.com/docs/2.0/caching/). Doing this would greatly reduce build time, but I haven't enabled it yet. If I remember right, I gave up on it when I was setting up my CircleCI pipeline with Cypress because that process was so annoying.

Now that it's been a bit, I feel good returning to this problem.

## Dev Log

I already have a plan for this. Step one is to separate out each piece of my current CCI pipeline into a separate job, rather than defining the entire pipeline in the workflow.

You can think of this like pulling functionality out of a long script into separate functions, but keeping the whole thing working the same. It just makes it easier to read and reason about it.

After that, I will try to add the cache functionality to my build step.

### Refactoring CircleCI Config

My CircleCI Config currently looks like this (comments removed):

```yml
version: 2.1

orbs:
    node: circleci/node@5.0.0
    cypress: cypress-io/cypress@1.29.0
    codecov: codecov/codecov@3.2.2

executors:
  with-chrome:
    docker:
      - image: 'cypress/browsers:node16.5.0-chrome97-ff96'

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Workflows ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ #
workflows:
    Application Test:
        jobs:
            - node/test:
                  pkg-manager: npm
                  test-results-for: other
                  run-command: test:ci
                  post-steps:
                      - store_artifacts:
                            path: coverage
                      - codecov/upload
            - cypress/install:
                build: npm run build
            - cypress/run:
                store_artifacts: true
                executor: with-chrome
                browser: chrome
                requires: 
                    - cypress/install
                        
                attach-workspace: true
                start: npm run start
                wait-on: 
                    "http-get://localhost:3000"
                post-steps: 
                    - store_artifacts: 
                          path: coverage
                    - codecov/upload
```

Note that the jobs are all defined at the bottom in the workflow. I have three jobs:

- `node/test`, which runs my Jest tests. This does not require building the application.
- `cypress/install`, which builds my application.
- `cypress/run`, which runs the app and the cypress tests.

Let's separate these out.

### Wait, let's go another direction

So I just spent a few hours yak shaving, and I think I have an idea. I have been working on the idea of altering the official Cypress CircleCI orb to generate the build cache AFTER the application build, not just after the Cypress build. I also wanted it cache the next.js `.next/cache` folder.

But I'm having trouble getting the workflow to work how I expect it to. For some reason, it just won't execute in the order that I want. I have adjusted the order of the jobs such that the cache step takes place after the build step in the orb config, then I push up the orb, adjust my circleCI to use the newer version, but it still runs in the old order.

Let's try going a little slower and thinking through the *precise* steps that are happening. I thought that I understood this, but apparently I didn't!

#### Breaking down the CircleCI Orb, starting with the Jobs

The orb is composed of 5 parts:

- `display`: Text that explains what the orb is.
- `executors`: A selection of choices for containers to run jobs in.
- `commands`: Custom building blocks that are used to put together jobs.
- `jobs`: The jobs that the orb exposes and allows you to use in your own config.
- `examples`: A bunch of examples, but no functioning code.

The obvious place to start is `jobs`. This exposes both of the jobs that I'm already using: `run` and `install`. I want to modify the `install` job.

This job, like most CCI jobs, is composed of a few pieces.

- `parameters`: Basically variables. Can be supplied in my config. I'm using some, in fact.
- `description`: What it sounds like.
- `executor`: This is where the executor that we picked earlier is passed to.
- `environment`: Sets up an environment variable for the test.
- `steps`: the actual steps that the job is composed of.

The `steps` section has two branches. If a custom build command is triggered, it goes down one branch, otherwise it uses the other.

I am using a custom build command. `npm run build`. So let's look in that branch.

```yml
   - when:
       condition: << parameters.build >>
       steps:
         - install:
             install-command: << parameters.install-command >>
             verify-command: << parameters.verify-command >>
             yarn: << parameters.yarn >>
             post-install: << parameters.post-install >>
             build: << parameters.build >>
             working_directory: << parameters.working_directory >>
             cache-key: << parameters.cache-key >>
```

I originally thought that I had found it here, and I moved `cache-key` to the end. Then I realized that this isn't the order. This is just a job with **one step**, named `- install`. The rest of the stuff below it are parameters being passed to it!

Ok, so where is `install` defined? Let's go back up the file, out of the `jobs` section and into the `commands` section.

#### Finding the `install` command

The `commands` section is comprised of three separate functions:

- `setup`: I don't know yet.
- `write_workspace`: saves the project and the cypress cache, to be re-used by later build steps
- `install`: Here it is!

So what's inside of this? The usual: more stuff! Description, parameters, and steps. Let's look at the `steps`. AND OH JOY ANOTHER NESTED JOB. Layers upon layers of logic. I suppose it's **DRY** though 🙃. The job is, of course, `setup`.

#### Breaking down the `setup` job

Ahh, here's the logic. There's a lot to unpack here. The `steps` have a lot of conditional logic. Here's a collapsed view:

![screenshot showing many collapsed build steps, each containing more additional logic](../images/Screen%20Shot%202022-02-19%20at%204.12.36%20PM.png)

Let's go through this. lol.

- `restore_cache`: This is a native circleCI command.
- the first `run`: This has a shell script that does some conditional logic depending on whether you are using yarn or npm.
- `steps: << parameters.post-install >>`: This resolves to whatever `post-intall` param you pass to the orb from your own config. I'm not passing any.
- the second `run`: This contains the cypress verification step.
- the first `when` and the first `unless`: Here's something. This contains the actual `save_cache` command!
- the final `when`: this contains the build command.

It seems logical to me that I should just have to move the `when/unless` pair to after the build step. I'm going to cut and paste those sections.

Also, those sections contain the paths that I want to cache. I'm going to add `.next/cache` to both. I only need it on the bottom one, but 🤷.

Let's give this a shot. I forked the orb earlier and set up my own CircleCI publishing environment with their CLI, so I'll publish the orb again, bump the version in my config, and try the pipeline. 🤞

That...didn't work. What?

Did I miss a branch? Not that I can see. Hm.

#### Adding it as a post-step

Ok this is a weird way of doing it, but I commented out the `save_cache` logic in the orb, published it, and added it as a `post-step` in my actual CircleCI config.

Now what's weird here is that it's *still* doing "saving cache" before build. Even though I removed those steps.

![image showing that my changes to job order in the CircleCI orb aren't sticking](../images/Screen%20Shot%202022-02-19%20at%205.01.44%20PM.png)

I know that *some* changes are getting through, because I can see the error saying that `.next/cache` can't be found. But the order changes aren't? Is circleCI caching something on their end, or am I misunderstanding how jobs work?

OR! OR! OR am I misunderstanding how orb releases work? I think I am! I've been doing `circleci orb publish promote...` this whole time, but I think that was just promoting the SAME ONE over and over. Maybe I need to do `circleci orb publish...` before I promote it. Let's try that!

LMAO THAT WAS IT

ok

god

ok cool, this is fine. It works. It works!

Oh wait, .next/cache still isn't being cached. Why not? Whatttttttt

### Figuring out why `.next/cache` is not present in circleCI

I'm getting this error in CircleCI:

```bash
`Warning: could not archive /root/.next/cache - Not found`
```

Maybe I'm trying to archive something in the wrong place, idk what the circleci file system looks like.

Let's add something to the build. Maybe a post-step that does an `ls -a`.

Ah, that's it. I need to set it to cache `.next/cache`, not `~/.next.cache`.

Now that I've done that, it works! It's shaved about 1 minute off of my build time.

There are still some gains to be made, though. Most of the build time is now taken up by the `save to workspace` step, which takes nearly an entire minute on its own. And then I just connect to that workspace right away in the following test. If I could skip that, that would give me even more savings.

But that's for another post.

## Conclusion

For the TL;DR enthusiasts:

- I had to adjust the CircleCI orb to cache the `.next/cache` folder, and to perform the caching step *after* the build step.

That's it. That's the whole thing.

This isn't exactly a simple fix. I had to clone the orb, register myself as an author on CircleCI, and then publish it iteratively until I figured it out. I would like to ditch the orb and roll my own custom config soon.
