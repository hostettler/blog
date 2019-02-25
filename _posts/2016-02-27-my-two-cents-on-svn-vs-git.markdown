---
layout: post
title: "My two cents on SVN vs GIT"
date: 2016-02-27 11:13:50 +0100
comments: true
categories: 
---

{% include warning.html content="This post is 3 years old and a lot has changed in the mean time. I would now recommend to upgrade to Git. 
Especially because the Cloud ecosystem is extremely dynamic. That being said, do it in a managed way starting by a low profile project." %}

_________________
**Executive summary** : Yes, Git is functionally richer and has several quality attributes that are really better than SVN but let's face it : it comes with an extra layer of complexity. This complexity may (and did in several instances I am aware of) result in time to market issues, tensions during stressful phases, loss of commits, dreadful merges and major impacts on the release management process. Henceforth, the question is :

- Are the extra functionnalities worth the investment and the risks?

In my opinion, Git is worth the investment if your devellopers are meant to work off site / off line.
If not (e.g., standard banking and financial industries) and it you already have an up and running SVN based software factory then do not bother 
to migrate to Git for existing project. Try a proof of concept on a new **low profile project** and do not hesitate to invest in training **and**  to re-engineer you existing release management processes. Once your teams have some experience with it, you can decide to migrate the existing codebase. 
That's beeing said, if you have no SCM then start with Git directly.

__________________

### Introduction

In many companies, SVN is still the SCM of choice (and yes some of them just finished up migrating from CVS). In the open-source world, the situation is very different. Indeed, Git is the de facto winner. As devellopers tend to stay up to date with the latest technologies, they want to migrate to the coolest new thing. I keep having discussions about GIT and whether or not a company or a department should migrate to it. In this blog post, I explain my position and argument on whether or not a company or department should move to Git. 


### Git vs SVN
There are plenty of very detailled comparison between Git and SVN out there. Among them let me cite the following articles [1] [2] [3].

Here a selection of the most important functionnal and non-functional Git features.

#### Dencentrilization
Let me start with the obvious, Git is dencentrilized by design. This is great for highly collaborative and disseminated teams. Actually, this is most important feature. Basically, Git has been designed to answer that problematic.
This has also nice side effects :
- Working offline
- Cloning a repository allows to quickly fork a project, make a couple of tests and submit a new version (by mean of pull requests) even if you do not have the commit rights.

This is great but it is probably not the most useful features for teams working in closed business such as banking systems and other plateforms with very few mobility (for security reasons). Moreover, in these cases having conmit rights is definitely not an issue.

**Disseminated teams are a very compelling argument to migrate from SVN to Git. Of course, the contraposition is also true**

#### Branches
Branches are feared in SVN and first class citizens in Git. This is definitly a game changer as it allows much better release management procedures.

#### Performances
In both performance (chekout, checkin) and space GIT is the winner. For instance, using GIT the Mozillia projects gained a factor 30x is terms of space.

#### Useful features

GIT comes with some nice tools that really improve the productivity of experienced developers. Here is a non-exhaustive list:

* ```git bisect``` is great for investigating regressions and discover when that a given bug has been introduced.


* ```git stash``` takes all of the staged changes and stores them away somewhere. This is useful if you want to break apart a number of changes into several commits, or have changes that you don’t want to get rid of (i.e. “git reset”) but also don’t want to commit. 
```git stash``` puts staged changes onto the stash and ```git stash pop``` applies the changes to the current working copy. 
It operates as a FILO stack (e.g. “First In, Last Out”) stack in the default operation.

* Branches are lightweight and merging is easy, and I mean really easy. It's distributed, basically every repository is a branch. It's much easier to develop concurrently and collaboratively than with Subversion, in my opinion. It also makes offline development possible.

#### Some important differences to know
* SVN has predicticable and simple version numbers, Git relies on UUID.

* GIT tracks contents rather than files

### Conclusion
Yes, Git is functionally richer and has several quality attributes that are really better than SVN but let's face it : it comes with an extra layer of complexity. This complexity may (and did in several instances I am aware of) result in time to market issues, tensions during stressful phases, loss of commits, dreadful merges and major impacts on the release management process.
In my opinion, Git is worth the investment if your devellopers are meant to work off site / off line.
If not (e.g., standard banking and financial industries) and it you already have an up and running SVN based software factory then do not bother 
to migrate to Git for existing project. Try a proof of concept on a new **low profile project** and do not hesitate to invest in training **and**  to re-engineer you existing release management processes. Once your teams have some experience with it, you can decide to migrate the existing codebase. 
That's beeing said, if you have no SCM then start with Git directly.


#### Bibliography
[1] http://stackoverflow.com/questions/871/why-is-git-better-than-subversion

[2] https://git.wiki.kernel.org/index.php/GitSvnComparison

[3] http://www.codeforest.net/git-vs-svn

[4] http://nvie.com/posts/a-successful-git-branching-model/

[5] https://github.com/nvie/gitflow

[6] https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow