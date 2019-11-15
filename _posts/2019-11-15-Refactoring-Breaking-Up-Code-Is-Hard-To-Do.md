---
title: "Refactoring: Breaking up code is hard to do"
date: 2019-11-15 06:05:00 -0700
categories: softwareengineering
tags: powershell scriptclass autographps
---
Wow, no posts in *months*, but it's not just because I'm a bad blogger. I was heads-down (as much as you can be on a side-project) refactoring one of my experiments, [ScriptClass](https://github.com/adamedx/scriptclass). It's done now (whew!), so I can share what I've learned.

But first, what is this *ScriptClass* thing? Briefly, it's an attempt to address my contention that while [PowerShell 5.0](https://powershell.org)'s `class` keyword is a great implementation of object-oriented types for .NET, it is not great at all for *PowerShell* (the language). I developed *ScriptClass* because I needed it for the PowerShell projects I truly wanted to build, [AutoGraphPS](https://github.com/adamedx/autographps) and [AutoGraphPS-SDK](https://github.com/adamedx/autographps-sdk).

I can go into exactly *why* I needed *ScriptClass* some other time; for now, I'll share the highlights of getting it into a good state.

## Weekend code war

Other than my bus commute, weekends were the only time for me to make progress on *AutoGraphPS*. Saturdays and Sundays are mostly when the refactoring saga unfolded:

* March 2019: Reached 1.5 years old for *AutoGraphPS* -- yay!
* March (end): I'm finally ready to add a cool (but big!) new feature I've always wanted: *instantiation of Graph types as PowerShell objects.*
* March (end): Hmm, that *ScriptClass* **design flaw** of not hiding internal state from other modules is going to be even more dangerous if I try to add this new feature. Let me think of some ideas and fix *ScriptClass* first before I start on it...
* April: Time to **fix the flaw** -- Ready to refactor! *ScriptClass* is going to stop using `ScriptsToProcess` in the module manifest and explicitly expose only the public variables, aliases, and functions of its interface. Should only take a weekend or two to get all the *ScriptClass* unit tests to pass...
* April (mid): Finally got the tests passing -- that took a little longer than I thought it would. Now we just need to make this new *ScriptClass* work with *AutoGraphPS* and *AutoGraphPS-SDK*. We're in the home stretch, right?
* May (mid): We got *AutoGraphPS-SDK* working... sort of... after multiple hacks that require special *ScriptClass* compatibility changes. But *AutoGraphPS* can't see types defined by *AutoGraphPS-SDK* unless I add a truly horrific hack. I don't think I can ship like this. Also, many other things are making me busy; time to take a break.
* July: Ok, it's July 4, a good time to return from the break, and because I've now forgotten whatever logic drove my initial attempt, I can fully justify starting from scratch. Let's try again, and hopefully I remember enough about my earlier design choices that I don't repeat the bad ones...
* August (mid): *ScriptClass* tests are passing! In this refactor, I actually threw away the entirety of *ScriptClass* and **rewrote from scratch.** From the earliest stages of this attempt I wrote new tests that validated the troublesome cross-module type issue. Only after they passed did I focus on getting the ~200 test cases from the original *ScriptClass* to go green.
* September: *AutoGraphPS* and *AutoGraphPS-SDK* are working with the new *ScriptClass* -- no hacks!!! And the breaking changes were minimal, they were less about the commands exposed by *ScriptClass* proper and more about the impact of module isolation.
* September (end): Refactor complete -- released new versions of *AutoGraphPS* and *AutoGraphPS-SDK*, both based on the refactored *ScriptClass*. It works!
* October: Ok, now I can finally get started on that big feature I wanted to add back in March...

**Total refactor time: ~ 6 months**

And yes, you read that correctly: the feature that originally triggered this half-year journey is *still* not yet implemented!

## Was it worth it?

So after all that, what was the point? At the cost of 6 months of weekends, I've certainly made all of the projects "better," but would I do this again under the same circumstances? It turns out the answer is "yes!" for the following reasons:

* The change to *ScriptClass* was a *necessary* change -- I was going to have to fix this serious flaw eventually. And since my cool proposed feature was not urgently needed, I could afford to delay it until after I'd fixed *ScriptClass* and saved myself the cost of a refactor made even more painful by the feature's avalanche of new code.
* *ScriptClass* itself is now easier to modify because the rewrite dramatically simplified the (originally hastily written) code. Fixing bugs in it or adding features was extremely risky both due to the original "spaghetti" code and the fact that the module's state was not private meant there were likely many unknown dependencies.
* Back to the first point: all of my projects were completely optional -- there were no deadlines, and no customers other than me! This gave me the luxury of doing the "right" thing from an engineering standpoint. If others had been depending on my work, I'd have had to take their needs into consideration around the timing of investing in a refactor. Of course, in that case I probably would be doing this on weekdays instead of weekends, and the overall time would be much less.

## Now on to the real fun

At the end, I felt great about finishing what looked like a daunting set of changes. I can sit back and breathe a sigh of relief, play with the projects, admire the code, and get back to scheming about adding bringing life to Graph types in PowerShell.

This time, let's just hope it takes less than 6 months.
