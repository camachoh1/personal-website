---
title: "Zen and The Art of Open-Source Debugging"
description: "A Firefox Reader Mode bug, test failures, and what I learned about debugging at scale."
pubDate: "Apr 25 2025"
heroImage: "/blog2/engine.jpg"
tags: ["firefox", "opensource", "debugging", "engineering"]
---

_The following is the second entry in a series of articles where I share my experience contributing to Mozilla’s Firefox browser. If you missed it, you can read the first part [here](https://haroldcamacho.dev/blog/i-worked-on-that-building-my-first-contribution-to-firefox/)._

---

Robert M. Pirsig wrote in _Zen and the Art of Motorcycle Maintenance_ when describing a mechanic at work:

> By far the greatest part of his work is careful observation and precise thinking. That’s why mechanics sometimes seem so taciturn and withdrawn when performing tests. They don’t like it when you talk to them because they are concentrating on mental images, hierarchies, and not really looking at you or the physical motorcycle at all. They are using the experiment as part of a program to expand their hierarchy of knowledge of the faulty motorcycle and compare it to the correct hierarchy in their mind. They are looking at underlying form.

---

This paragraph came vividly back to me as I found myself stuck, staring at my failing tests, trying to understand what I’d missed despite following all the steps suggested by the Mozilla engineer mentoring my contribution.

---

### Understanding the Task

It all started with a bug that required me to refactor Firefox’s reader mode code related to language detection and text direction.

![ReaderMode](/blog2/reader_mode.1.jpeg)

Regular Browsing Mode vs. Reader Mode

_Reader Mode is a built-in Firefox feature that strips away clutter like ads, sidebars, and popups so you can focus on the main content of a web page. It’s especially useful for reading articles and blog posts in a clean, distraction-free layout._

Originally, the code used a `Promise` whose `resolve` callback was stored separately in its own variable. This promise was passed into `NarrateControls` to asynchronously load article data. Although it might sound complex, my task was straightforward: consolidate the `Promise` and its separate `resolve` callback into a single, neatly packaged object using [`Promise.withResolvers()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers). Instead of two separate variables (`_languagePromise` and `_foundLanguage` for the resolve), I'd have a single `_languageDeferred` object, accessing its promise or resolve method through clear dot notation (`_languageDeferred.promise` and `_languageDeferred.resolve()`).

There were a couple more adjustments. I needed to rename `article.language` to a more descriptive `article.detectedLanguage` and replace a temporary hard-coded solution for text direction detection with `Services.intl.getScriptDirection`. This function determines whether a language is written left-to-right (`ltr`) or right-to-left (`rtl`) based on a given language string. While most languages—including English, Spanish, and French—are read left to right, others like Arabic and Persian are read right to left, and Reader Mode needs to account for that.

---

### Too Early to Celebrate

After exploring the relevant files — primarily `AboutReader` and `ReaderMode`—I quickly made these updates. Feeling confident, I ran the recommended tests provided by the bug mentor (`toolkit/components/reader/`), and they all passed. It felt great to see those green checkmarks! I committed my changes and submitted my patch through Phabricator, Mozilla's collaborative code review tool. After a quick review and minor tweaks suggested by the mentor, my patch was approved, pushed, and was on its way to being landed. It was time to celebrate!

But then, just a few hours later, I got an unexpected email from Bugzilla: my patch had been backed out due to test failures during the push process. Feeling a bit confused about the email contents, I checked the logs but didn’t immediately understand the issue. Reaching out for guidance again, my mentor pointed out a detail we’d missed — another component (`genai`) had a file `LinkPreviewChild` which relied on destructuring the `article` object and was still using the old `article.language` instead of the updated `article.detectedLanguage`. Once those changes were made I needed to run the `browser_link_preview` tests and re-submit. Easy fix, right?

---

### Branching Troubles

Not quite. When I tried to find the necessary files, they weren’t there in my current feature branch. I realized that my feature branch had become outdated relative to the default branch. To sync things up, I attempted a merge — but accidentally introduced thousands of unintended changes. My heart sank. After spending some time panicking and then carefully reversing the merge, I managed to properly rebase my branch onto default. Now the necessary files were available, and I quickly updated them as suggested.

---

### Keep Calm and Debug

![Errors](/blog2/errors.png)

Running the tests again, I faced another challenge: my previously passing tests were now failing. Panic started to set in again. I reached out once more, and my mentor gently suggested that my branch might still not be fully up-to-date. After double-checking and carefully rebasing again, the tests for my initial patch passed — but the new tests (`browser_link_preview`) continued to fail.

That’s when I remembered the mechanic from Pirsig’s book — quietly thinking, visualizing the hierarchy, seeing not just the motorcycle, but its underlying form. I took a deep breath and approached the tests slowly, carefully reading each line and tracing connections. By examining the implementation of one failing test in detail, I spotted something crucial:

```
await SpecialPowers.pushPrefEnv({
    set: [["browser.ml.linkPreview.allowedLanguages", ""]],
});
```

I realized that `linkPreview` was where allowed languages were defined—so perhaps `detectedLanguage` was used there, too. Following this hunch, I reviewed the `LinkPreview` file closely and indeed found a function still referencing the outdated `language` property instead of the new `detectedLanguage`. Updating this single line of code resolved the issue, and all tests passed immediately. Victory!

### The Quiet Mechanic

Reflecting back, I saw clearly why the mechanic from the book always seemed so quiet and withdrawn during tests. Debugging complex software requires precisely the same sort of careful, patient, and systematic thinking. I had to visualize the hierarchy of the Firefox codebase (at least for the components I was working on), carefully observe interactions between components, and methodically examine each test. In large codebases like Firefox, making a simple change can have far-reaching effects. The real work lies in knowing exactly where to apply those few crucial lines of code.

---

This was another entry sharing my experience contributing to Mozilla Firefox. Thank you for reading!

Once again, if you’re interested in exploring further, you can see the full bug report and my Phabricator patch [here](https://bugzilla.mozilla.org/show_bug.cgi?id=1323331).

_P.S.: In the bug report, you can even find my confused questions during the debugging process — it’s all part of the journey, and a beautiful aspect of open-source contributions is that the process is transparent and available for everyone to see._

Until next time!
