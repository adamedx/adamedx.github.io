---
title: "Juneteenth and the master branch overthrow"
date: 2020-06-19 23:00:00 -0700
categories: Juneteenth history freedom code engineering
---

Yesterday was [Juneteenth](https://www.theroot.com/what-is-juneteenth-1790896900), so I decided to finally get rid of the `master` branch in my most active `git` repositories. [Github](https://github.com) certainly made that easier as they [just announced they are eliminating `master` as the default branch name](https://www.theregister.com/2020/06/15/github_replaces_master_with_main/). In fact, they are adopting naming standard I had considered long ago, that of using the more descriptive term *main.*


**You can and should do the same thing in your repositories** -- it was pretty easy for me, and I'm [sharing those steps below](#How-to-rename-master-to-main).


## Why does it even matter?

Before getting to those renaming steps, you might ask why bother with such cosmetic changes regarding terminology when we have the [real and constant oppression caused by racism and white supremacy](https://www.raceforward.org/videos/systemic-racism) against the African diaspora and people of color generally? How does changing a name, particularly in a case like this one where there is no evidence that the use of `master`, which likely [originated in a non-American context of OSS software developers](https://en.wikipedia.org/wiki/Git#History) where English was not the first language, was informed by conscious our unconscious white supremacy?


Here are my answers:


* It's true renaming `master` to `main` (or `root` or `trunk` or `default` or `primary`, whatever) doesn't fix racism.
* Renaming terms and classifying terms like `master` as problematic sends a message that **racism requires active participation from everyone to stamp out**:
  * It acknowledges that racism exists
  * It acknowledges that it causes ongoing and terrible suffering
  * It acknowledges that racism even infects our shared languauge and terminology
  * It acknowledges that oppressed people are being constantly reminded of their oppression by this infected and we are somehow expected to be ok with that and treat it like it's no big deal
  * It acknowledges that racism isn't just some external force we're helpless against like the weather, it is ultimately driven by us individually in this racist system
  * It acknowledges that racism is so much *everyone's* problem, not just Black people's
  * It's enough of a problem that we will modify our language to fight back against our infected systems
* Sending this message is really a way to galvanize all communities to prioritize fixing injustice over the comfort and convenience of those in power
* It actually costs almost nothing to make changes that support and value oppressed people, so why not do it? A refusal here is really saying "not only are we unwilling to do the difficult and message work to pass anti-racist laws, dismantle racist institutions and systems, and create just replacements for those systems," we're not even willing to lift a small finger to change some terms that normalize your oppression.
  * Doing nothing when it's easy really means that Black people don't matter.
* It's not a choice between "superficial" vs. "real" change -- you can do both, especially when the cost of those "superficial" changes is minimal.
  * You're concerned that if we act on enough of these small changes, we'll just distract from the real work of changing the system? Here's an idea -- has that happened yet? Have we been willing to remove the [Aunt Jemimas](https://www.npr.org/sections/live-updates-protests-for-racial-justice/2020/06/17/879104818/acknowledging-racial-stereotype-aunt-jemima-will-change-brand-name-and-image) and the [Washington Redsk\*ns](https://en.wikipedia.org/wiki/Washington_Redskins_name_controversy) from acceptable communication? Let's try a little experiment and listen to the oppressed people rather than telling them what they should be ok with.

Ultimately if you respect and value someone, you will treat them that way. If you've hurt or damaged that person, you wouldn't continue throw that history in their face. To move forward, you have to acknowledge the truth, remember it, and **CHANGE**.

## How to rename master to main

Ok, the steps are pretty easy -- while there is no `rename` function for branches on Github, you can create a branch with your new name and tell Github to make that branch the default, then delete the `master` branch. Here it is step-by-step:

1. Push a new branch of the latest `master` with the desired name, e.g. `main`
2. Fix up any references (e.g. in URI's to the repository on Github) within the repository or outside of it to reference the new branch name.
3. Push any such changes required in your local `main` to the remote. Update other source repositories or stores as well.
4. Visit the *settings* section of your repository on Github, find the `branches` section, and then the *Default branch* setting and use that to (carefully) update the default branch name to the branch you just pushed, heeding any additional instructions or warnings it provides. Today you can also find that setting simply by viewing the branches in your repository on Github and choosing the current default branch of `master`.
5. Make a new test PR and verify that it defaults to merging to the new default branch rather than `master`.
6. If needed, re-release the repository's artifacts to production / package repositories in case they have functionality that depends on the name of the default branch. Obviously, make sure everything is working in production!
7. Check all of the external references as well and make sure that URI's resolve and any impacted functionality behaves as expected
8. Go to your repository and view the branches, then **delete the master branch.** Yes, I said it -- it's scary, but trust me it feels good! You can "undelete" it from the Github UX within a short time window if you catch any problems in time.
9. Recheck your tools and references from step 7 to make sure everything's working. If you think there's an issue, "undelete" master to see if that addresses it.

### Notes from my renaming

Here are the highlights from my renaming process for [AutoGraphPS](https://github.com/adamedx/autographps), [AutoGraphPS-SDK](https://github.com/adamedx/autographps-sdk), and [ScriptClass](https://github.com/adamedx/scriptclass). I definitely had to clean up obsolete references to `master` in each repository, and also cross-repository references. Even PowerShell Gallery had some references which I needed to (mostly) fix.

* I made sure the local `master` was up to date, and executed `git checkout -b main`. Then I pushed it with `git push origin main`. The new default was created! But it was not yet the default.
* The slightly tricky part began: I looked for all references *within* the repository to `master` and changed them to `main`
  * Do you have URIs like the following in any markdown files, build / test / CI scripts, or CI pipeline definitions: https://raw.githubusercontent.com/adamedx/poshgraph-sdk/main/assets/PoshGraphIcon.png
  * I hit that one because I was essentially using my remote Github repository as a CDN, and those URI's contain the branch name.
  * I also had a file that contained a [definition for Azure pipelines to trigger whenever the default branch was updated](https://github.com/adamedx/scriptclass/commit/82f8228597c9dbfa390dce73e4a922b19d805cd7), and it included the name of the default branch, i.e. `master`.
  * An easy way to find these in your local repository is a simple `git grep master` or a search of the remote branch in the Github UX.
* Now the really trick part started: Are there any references *outside* of the repository to its `master` branch? You'll need to replace those as well!
  * Perform a similar search and replace in other repositories that reference this repository's `master` branch to reference `main` instead. I was able to validate them just fine because I had pushed `main` earlier, so references to `main` would correctly resolve.
  * Are there references outside of known github repositories that you control, e.g. blog posts you've made, package repositories, or possibly even Github / Bitbucket repositories managed by others? Where you have access, you should update these also (through direct updates, pull requests, etc.)
  * In my case the icons for previously released packages that referenced their repository's master branch on [PowerShell Gallery](https://powershellgallery.com) did so because of a file contained in the package itself.
  * This meant that I **needed to release a new version** of the package with the [updated reference](https://github.com/adamedx/scriptclass/commit/9bbc506ca83abf74bfb4ef5eb5e4b9cdeab5d63d) just to get a version of the package that could show the correct icon. And once `master` was gone
  https://github.com/adamedx/autographps/blob/master/docs/WALKTHROUGH.md

This should give a good idea of what's involved. It's really easy if you don't employ the (somewhat questionable) practice of referencing branch names in URI's, particularly those outside of the repository.

One way to avoid reference problem in general is to create a tag for references to things like documentation or content assets like image files. The tag is essentially forever, where branches, even the default branch, can disappear. The downside of tags of course is that (as far as I know) they are not "dynamic" -- if you want to point people to your latest documentation in the repository, you'll need to create a new tag for it, and then update those sources. At least then you'll be aware of these dependencies (especially if you have to validate them whenever you make updates) intead of presumably forgetting about them completely until a branch rename is needed.

## Conclusion

That's it -- now I'm finally free of `master` (at least for these repositories), a fitting realization for Juneteenth.

This use of `master` has bothered me from day one of using `git`, and initially I filed it away as one of those things racist things about soceity I would work around eventually. After all, normal daily life for any Black person is determining which stupid aspects of racism require immediate attention and which are best tolerated for some time. In the case of the `master` branch, now that I'm seeing others pay attention to it, it was a great reminder for me that I could probably knock this one out on a small scale for my own repositories. These days I am far from the git n00b I was when `master` first annoyed me.

Looking back, one reason I was somewhat surprised to find this when I first started using `git` was simply that I had been using branching source control systems for years professionally even before `git` existed. I did this mostly at a gigantic software company and we actually used the term `main` for our default branch. We would have endless conversations about *"when are we going to merge to main?"* or *"we haven't reverse / forward integrated main in 6 weeks!"*. So to me, `main` was quite the natural term. [Subversion](http://subversion.apache.org/), with which I was only slightly familiar, used *[trunk](http://svnbook.red-bean.com/en/1.8/svn.tour.importing.html#svn.tour.importing.layout)*, which seemed perfectly fine as well. I was not familiar with [Mercurial](https://www.mercurial-scm.org/) at the time, but it uses the term [default](https://www.markheath.net/post/using-named-branches-in-mercurial).

So it seems people found it straightforward to avoid the `master` terminology for other source control systems.

I will say that my large software company is based in the USA, which I think increases the odds (and also the expectation!) that `master` would be added to the list of disallowed terms. That company also operates internationally in every market across the world, and as a result it had to develop a muscle around a certain level of cultural sensitivity. I'm guessing that even if engineers and product designers of the day had been sufficiently clueless as to think `master` was acceptable, the company's fairly extensive automation and review processes around cultural issues, which included signoffs on reviewed questionnaires around content, and code-scanning automation to identify sensitive words or phrases in source code, documentation, and the product artifacts themselves would have prevented `master` from seeing the light of day.

The adoption of OSS tools at this company and by the entire industry, however, provided a way for these issues to resurface; OSS project members and organizations did not have the admittedly amoral stick of a commercial orientation to look for even these surface forms of racism. The presence of `master` originating in the `git` OSS infrastructure later popularized by *Github* are examples of this re-emergence of offensive terminology.

Let's hope that beyond the short-sighted motive of profit, all of us can question what we've built and how we've built it, and act quickly on the racism that's under our control. Beyond sending a message on what's acceptable in society, we will need the habit of making uncomfortable change at the smallest levels if we're going to tackle the daunting challenges of racism, misogyny, homophobia, and inequality.

Happy Juneteenth.
