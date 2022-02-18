---
title: Clearing Tiptap Content on First Focus
description: 'Like my previous post said, I'
'm working on setting up a rich text editor on this site to use to write blog posts. I have it mostly working, but during the process I had a silly idea''': 'How could I get this editor to act like a regular <input> box, at least in terms of how <input> elements can have placeholder text? You can break down that problem into these steps:'''
date: '2022-02-01T06:34:35.029Z'
draft: 'false'
tags: ''
categories: ''
lastmod: '2022-02-18T04:17:50.693Z'
slug: clearing-tiptap-content-focus
---
- [Introduction](#introduction)
  - [How to determine whether an element is focused in React](#how-to-determine-whether-an-element-is-focused-in-react)
  - [Keeping track of focus events with useState()](#keeping-track-of-focus-events-with-usestate)
  - [Dealing with weird focusing issues on page load](#dealing-with-weird-focusing-issues-on-page-load)
  - [Displaying text in the Tiptap content box on load](#displaying-text-in-the-tiptap-content-box-on-load)

## Introduction

Like my previous post said, I'm working on setting up a rich text editor on this site to use to write blog posts. I've got it mostly working, but during the process I had a silly idea: How could I get this editor to act like a regular <input> box, at least in terms of how <input> elements can have placeholder text? You can break down that problem into these steps:

1.  Display placeholder text in the body of the Tiptap editor window
2.  Clear that text on the first focus

    1.  Learn how to determine if the editor is focused
    2.  Keep track of whether the editor is being focused for the first time, or some other time.

### How to determine whether an element is focused in React

Step 2.1 is easy. You can add an `onFocus` tag to any element, like so:

```jsx
    <EditorContent
        onFocus={(e) => {
            console.log("This triggers on focus!");
        }
    />
```

### Keeping track of focus events with useState()

Step 2.2 is similarly easy. Modern React has the `useState` hook, which is perfect for this. I instantiated a variable and a setter function to track this like so:

```javascript
    const [hasBeenFocusedAlready, setHasBeenFocusedAlready] =
        useState(false);
```

You can use this state to determine whether to wipe the content:

```jsx
    onFocus={(e) => {
                        console.log(e);
                        console.log(hasBeenFocusedAlready);
                        hasBeenFocusedAlready ? null : editor.commands.setContent('');
                        setHasBeenFocusedAlready(true);
                    }
```

That handles all of problem 2! This is enough to clear the content on the first focus event, and no others.

### Dealing with weird focusing issues on page load

During this process, I found that sometimes, the field gets focused randomly during page load. To make sure that the `hasBeenFocusedAlready` variable is set right, I use a `useEffect` hook with an empty dependency array. This just forces it to run once the page has finished rendering, and stuff stops loading.

```jsx
useEffect((editor) => {
    setHasBeenFocusedAlready(false);
    setInitialContent('blah');
}, []);
```

### Displaying text in the Tiptap content box on load

Now let's tackle problem 1, **displaying text in the content box on load.** I decided that I wanted to have the ability to pass the desired text as a prop from some other component, so that I can pick what displays in each place that I use this editor.

Further, looking through the docs, and there's a great way to set content _after_ the editor loads: Events. Tiptap exposes a few events during the lifecycle of the component, which you can read about here: https://tiptap.dev/api/events. I chose to hook in to the `onCreate` event. When I initialize the editor, I now do this:

```jsx
const editor = useEditor({
    extensions: [StarterKit],
    onCreate: ({ editor }) => {
        console.log('created');
        editor.commands.setContent(initialContent);
    },
});
```

And I set the initialContent with a useState hook. I'm not sure that I need that to be honest. I might be able to just pass in the `props.content` directly. Either way, it works great.

Putting it all together, we get the following:

```js
    const MyEditor = (props) => {
        const [hasBeenFocusedAlready, setHasBeenFocusedAlready] = useState(false);
        const [initialContent, setInitialContent] = useState(props.content);

        const editor = useEditor({
            extensions: [StarterKit],
            onCreate: ({ editor }) => {
                editor.commands.setContent(initialContent);
            },
        });

        useEffect(() => {
            setHasBeenFocusedAlready(false);
            setInitialContent('blah');
        }, []);

        <EditorContent
            editor={editor}
            onFocus={(e) => {
                console.log(e);
                console.log(hasBeenFocusedAlready);
                hasBeenFocusedAlready ? null : editor.commands.setContent('');
                setHasBeenFocusedAlready(true);
            }
    }
```

I hope that the formatting here hasn't been too atrocious! I have a lot of work left to do to make this a nice editing environment.
