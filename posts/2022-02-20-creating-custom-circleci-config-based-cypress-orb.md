---
title: Creating a Custom CircleCI Config based on the Cypress Orb
slug: creating-custom-circleci-config-based-cypress-orb
description: 'I redid my circleci config by using parts from the cypress orb. I reduced my build time from 2m45s to 1m 13s, a 55.7% reduction. More than I thought I''d get.'
author: Sam Boland
date: '2022-02-20T20:17:21.716Z'
lastmod: '2022-02-20T21:50:46.776Z'
tags: []
pullRequest: 'https://github.com/samuelboland/Hyperfixations/pull/46'
---

- [Explanation](#explanation)
  - [TL;DR](#tldr)
- [Dev Log](#dev-log)
  - [Extracting the Cypress/Install command](#extracting-the-cypressinstall-command)
    - [Step-by-step](#step-by-step)
  - [Extracting the Cypress/Run command](#extracting-the-cypressrun-command)
    - [Step by step](#step-by-step-1)
  - [Putting it all together](#putting-it-all-together)
- [Conclusion](#conclusion)

## Explanation

(adding this disclaimer as I'm about halfway through: I'm writing this as I go, and I do only minor editing. So like. Now you know.)

The Cypress CircleCI orb was a great start, but it has some issues that I want to fix. Specifically, it wastes time saving the workspace in the `Cypress/Install` step, and then loading it directly after in the `Cypress/Run` step. This takes a significant amount of time, and is completely unnecessary.

The `Persisting to Workspace` step takes 56s. The `Attaching Workspace` step takes 11s. In total, this is 1m 7s. The entire build takes 2m 45s, so this represents **40.6% of the entire job!**

As a refresher, my CircleCI config workflow looks like this right now:

```yml
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
                  cache-key: cache-{{ checksum "package.json"}}
            - cypress/run:
                  cache-key: cache-{{ checksum "package.json"}}
                  store_artifacts: true
                  executor: with-chrome
                  browser: chrome
                  requires:
                      - cypress/install
                  attach-workspace: true
                  start: npm run start
                  wait-on: 'http-get://localhost:3000'
                  post-steps:
                      - store_artifacts:
                            path: coverage
                      - codecov/upload
```

It works, but I want to make it better.

### TL;DR

I redid my circleci config by using parts from the cypress orb. I reduced my build time from 2m45s to 1m 13s, a 55.7% reduction. More than I thought I'd get.
## Dev Log

To fix this, I'm going to build my own custom config. In my previous post, I explained how I altered the CircleCI Orb to run the Cache step after the build step, and to also cache the `.next/cache` folder. I could further alter the orb, but instead, I would rather use it as inspiration and get rid of the code paths that I don't require.

### Extracting the Cypress/Install command

In my previous post, I found that the bulk of the logic in the orb is happening in the `setup` command. This has a few code paths, and selects other commands from the `commands` section of the orb. I could reuse most of those, maybe.

I have to remember to keep the following as well:

```yml
environment:
  CYPRESS_CACHE_FOLDER: ~/.cache/Cypress
```

#### Step-by-step

Let's start with the first piece that runs, which is `restore_cache`. That's easy, that's a built in CircleCI command. That looks like this:

```yml
  - restore_cache:
      keys:
        - cache-{{ checksum "package.json"}}
```

The first step is named `Install` and looks like this:

```yml
- run:
     name: Install
     working_directory: << parametersworking_directory >>
     command: |
       if [[ ! -z "<< parameters.install-command>>" ]]; then
         echo "Installing using custom command"
         echo "<< parameters.install-command >>"
         << parameters.install-command >>
       elif [ "<< parameters.yarn >>" = "true" ; then
         echo "Installing using Yarn"
         yarn install --frozen-lockfile
       elif [ ! -e ./package-lock.json ]; then
         echo "The Cypress orb uses 'npm ci' toinstall 'node_modules', which requiresa 'package-lock.json'."
         echo "A 'package-lock.json' file wasnot found. Please run 'npm install' inyour project,"
         echo "and commit 'package-lock.json' toyour repo."
         exit 1
       else
         echo "Installing dependencies using NPMci"
         npm ci
       fi
```

There are already branches and parameters here that I don't need. I'm not specifying a special working_directory, so I can drop that and just use the default, which should be `''`.

Then in the command, it does a few checks to determine what package manager I'm using. I know that I'm using `npm`, so I don't need to do all of this.

So my new command looks like this:

```yml
- run:
      name: Install
      command: npm install
```

Already much easier to understand.

The next step is `verify cypress`. Let's just use the default:

```yml
- run:
      name: 'Verify Cypress'
      command: npx cypress verify

```

The next step is the build command. This is one where I passed a custom param, so let's be sure to include that. The orb version looks like this:

```yml
  - when:
      condition: << parameters.build >>
      steps:
        - run:
            name: 'Build'
            command: << parameters.build >>
            working_directory: << parameters.working_directory >>
```

My version is this:

```yml
- run: 
      name: 'Build'
      command: 'npm run build'
```

Following that is some conditional logic that determines how it will save the cache. If we're using `yarn` instead of `npm`, it changes. Here's the original:

```yml
- when:
    condition: << parameters.yarn >>
    steps:
      - save_cache:
          key: << parameters.cache-key >>
          paths:
            - ~/.cache
            - .next/cache
- unless:
    condition: << parameters.yarn >>
    steps:
      - save_cache:
          key: << parameters.cache-key >>
          paths:
            - ~/.npm
            - ~/.cache
            - .next/cache
```

Mine is:

```yml
- save_cache:
    key: cache-{{ checksum "package.json"}}
    paths:
        - ~/.npm
        - ~/.cache
        - .next/cache
```

Here's where we diverge from the ordering in the orb. At this point, the orb persists the workspace. I want to avoid that.

### Extracting the Cypress/Run command

I didn't look at this one in my previous post. Here's what my `cypress/run` looks like now:

```yml
 - cypress/run:
       cache-key: cache-{{ checksum"package.json"}}
       store_artifacts: true
       executor: with-chrome
       browser: chrome
       requires:
           - cypress/install
       attach-workspace: true
       start: npm run start
       wait-on: 'http-get:/localhost:3000'
       post-steps:
           - store_artifacts:
                 path: coverage
           - codecov/upload
```

I can keep the `post-steps` the same. I can drop the `attach-workspace` step. I think I can remove `store_artifacts` because that wasn't working, hence the post-step `- store_artifacts` command.

#### Step by step

There's a few steps at the start of this job that pertain to debugging and parallel runs, which aren't something I'm doing. There is also a step in some conditional logic, which I was triggering, that attached my workspace. I'll skip that now.

The next step, with some nested logic, seems to only trigger if `parameters.parallel` is false. The default is `false`, and I never changed it, so I must have used this path. However, I *did* set `attach-workspace` to true. Looking at this now, I think I could have avoided all of this, if that weren't the case. It looks like it calls the build step........

Well too late, this is fun anyways. I'm just going to ignore that.

So with `parameters.parallel` = false, and `parameters.attach-workspace` = true, I think that we actually just short-circuit all of this.

So then we have this block:

```yml
- when:
    condition: <<parameters.start>>
    steps:
      - run:
          name: Start
          command: <<parameters.start>>
          background: true
          working_directory: << parameters.working_directory >>

```

This starts the server. I passed a custom command to it, so let's replicate that here.

```yml
- run: 
      name: 'Start'
      command: 'npm run start'
      background: true
```

the next step calls `wait-on`, which is a useful utility that waits until my server is ready. I was using that, so lets include it. The original looks like:

```yml
- when:
    condition: <<parameters.wait-on>>
    steps:
      - run:
          name: Wait-on <<parameters.wait-on>>
          command: npx wait-on <<parameters.wait-on>>
```

Mine is:

```yml
- run: 
      name: Wait-on 'http-get://localhost:3000'
      command: npx wait-on 'http-get://localhost:3000'
```

Now we actually run tests! The original has a lot of nesting. The branch that I got to looks like this:

```yml
    steps:
      - run:
          name: Run Cypress tests
          no_output_timeout: <<parameters.timeout>>
          # GOOD EXAMPLE conditional text based on boolean parameter
          # --record is needed to pass many other arguments, like "--group" and "--parallel"
      command: |
        npx cypress run \
          <<# parameters.spec>> --spec '<<parameters.spec>>' <</ parameters.spec>> \
          <<# parameters.browser>> --browser <<parameters.browser>> <</ parameters.browser>> \
          <<# parameters.config-file>> --config-file <<parameters.config-file>> <</ parameters.config-file>> \
          <<# parameters.config>> --config <<parameters.config>> <</ parameters.config>> \
          <<# parameters.env>> --env <<parameters.env>> <</ parameters.env>> \
          <<# parameters.record >> --record \
          <<# parameters.group>> --group '<<parameters.group>>' <</ parameters.group>> \
          <<# parameters.parallel>> --parallel <</ parameters.parallel>> \
          <<# parameters.tags>> --tag '<<parameters.tags>>' <</ parameters.tags>> \
          <<# parameters.ci-build-id>> --ci-build-id <<parameters.ci-build-id>> <</ parameters.ci-build-id>> \
           <</ parameters.record>>
           working_directory: << parameters.working_directory >>
```

That's a lot of commands. Let's break it down to what it resolved to. I may be able to get this from CircleCI itself, now that I think about it.

Oh look, it does! If I go to my config from the CircleCI front end, and then hit the "compiled" tab in the top right, I get the compiled version. Which actually would have made this all a lot easier. Not effortless, but easier. Oh well!

The compiled version of this section looks like this:

```yml
- run:
    name: Run Cypress tests
    no_output_timeout: 10m
    command: "npx cypress run \\\n   \\\n   --browser chrome  \\\n   \\\n   \\\n   \\\n  "
    working_directory: ''
```

I can make it look a bit nicer by removing all the newlines. Mine is:

```yml
- run: 
      name: Run cypress tests
      no_output_timeout: 10m
      command: npx cypress run --browser chrome
      working_directory: ''
```

I wonder if this "working_directory" var has been necessary the whole time? I've been omitting it. I'll find out soon enough I guess.

The next piece is storing the artifacts. The original:

```yml
- unless:
    # no custom working directory
   condition: << parameters.working_directory >>
   steps:
    - store_artifacts:
        path: cypress/videos
    - store_artifacts:
        path: cypress/screenshots
```

I can just take the final two pieces, so:

```yml
- store_artifacts: 
      path: cypress/videos
- store_artifacts:
    path: cypress/screenshots
- store_artifacts:
    path: coverage
```

I decided to add the final artifact, `coverage`, since I was doing that in a post_step already.

### Putting it all together

Now I just need to edit my workflow. I'm removing my `cypress/install` and `cypress/run` jobs, and adding my new one. And you know what, just for good measure, I'm adding the `working_directory` stuff in where I missed it. Oh, and I have to remember to add the post_step to upload to codecov.

I then run `circleci config validate`, which comes from their CLI that I downloaded for my last post, and it says that it's valid!

And that's it. If I did this right, I should now have a single step in my CircleCI that both builds and runs cypress, without the workspace handoff. Let's give it a shot!

I'm going to bust the cache too so that it generates a fresh one. I'm doing this by adding a `v1` in the middle of my cache name.

Wow, it worked. That's awesome!

## Conclusion

In the end, my new CircleCI config looks like this:

```yml
# This config is equivalent to both the '.circleci/extended/orb-free.yml' and the base '.circleci/config.yml'
version: 2.1

orbs:
    node: circleci/node@5.0.0
    codecov: codecov/codecov@3.2.2
    cypress: samuelboland/cypress@0.0.12


# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Jobs ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ #

jobs:
    Build_and_run_cypress:
        environment:
            CYPRESS_CACHE_FOLDER: ~/.cache/Cypress

        executor: with-chrome
        steps:
            - checkout
            - restore_cache:
                  keys:
                      - cache-v1-{{ checksum "package.json"}}
            - run:
                  name: 'Install'
                  command: npm install
                  working_directory: ''
            - run:
                  name: 'Verify Cypress'
                  command: npx cypress verify
                  working_directory: ''
            - run:
                  name: 'Build'
                  command: 'npm run build'
                  working_directory: ''
            - save_cache:
                  key: cache-v1-{{ checksum "package.json"}}
                  paths:
                      - ~/.npm
                      - ~/.cache
                      - .next/cache
            - run:
                  name: 'Start'
                  command: 'npm run start'
                  background: true
                  working_directory: ''
            - run:
                  name: Wait-on 'http-get://localhost:3000'
                  command: npx wait-on 'http-get://localhost:3000'
            - run: 
                  name: Run cypress tests
                  no_output_timeout: 10m
                  command: npx cypress run --browser chrome
                  working_directory: ''
            - store_artifacts: 
                  path: cypress/videos
            - store_artifacts:
                path: cypress/screenshots
            - store_artifacts:
                path: coverage

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Executors for Cypress ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ #

# List of executors can be found here: https://github.com/cypress-io/cypress-docker-images/tree/master/browsers
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
            - Build_and_run_cypress:
                  post-steps: 
                      - codecov/upload
```
