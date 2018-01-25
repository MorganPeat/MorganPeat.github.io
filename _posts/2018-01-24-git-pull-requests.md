---
layout: post
title:  "Best practices for git Pull Requests"
date:   2018-01-24 15:00:00
categories: [misc]
tags: [misc]
draft: false
---
On my project I have noticed that the quality of pull requests has gradually been getting worse. This isn't entirely a bad thing because we are a close-knit team with regular scrums, design reviews, etc. and everyone knows what is going on. This means there are few surprises when it comes to code review time.  
But there are things we can do to make code reviewing easier which will give faster turnaround and increase team velocity. My opinion is that if code is easy to review and understand then all the developers on the team will be more likely to look at the pull requests of others. That way
* Everyone gets a better "handle" on the code base
* We are more likely to catch subtle bugs due to more pairs of eyes on the code
* We will spot more opportunities for code sharing / reuse

So I have been scouting around for what constitutes "best practice" when writing a Pull Request. In no particular order, it seems the following things are important.

## Make it small
> if the Pull Request is big, I'll need to schedule time to review it: I can't review a big chunk of code using five minutes here and five minutes there; I need contiguous time to do that. This already introduces a delay into the review process.

http://blog.ploeh.dk/2015/01/15/10-tips-for-better-pull-requests/  

> Making smaller pull requests is the #1 way to speed up your review time. Because they are so small, it’s easier for the reviewers to understand the context and reason with the logic.

https://www.atlassian.com/blog/git/written-unwritten-guide-pull-requests  


## Do only one thing
> The more concerns you address in a single Pull Request, the bigger the risk that at least one of them will block acceptance of your contribution. Do only one thing per Pull Request. It also helps you make each Pull Request smaller.

http://blog.ploeh.dk/2015/01/15/10-tips-for-better-pull-requests/  


## Write useful descriptions and titles  
> A useful summary title (instead of just the issue key) makes it clear to the reviewer what’s being solved at a high level.

https://www.atlassian.com/blog/git/written-unwritten-guide-pull-requests

> Writing a useful description in the “details” section of your pull request can almost be as effective as making a smaller pull request! If I do make a large pull request, I’ll make sure to spend a lot of time making a really helpful description. The most helpful descriptions guide the reviewer through the code as much as possible, highlighting related files and grouping them into concepts or problems that are being solved.

https://www.atlassian.com/blog/git/written-unwritten-guide-pull-requests


## Merged up to HEAD
> Good pull requests tend to have a few things in common:
> * They pass the existing tests
> * Merge conflicts have been resolved

http://www.zagaja.com/2016/08/anatomy-of-a-good-pull-request/

> Before creating a pull request, you should compare your outgoing requests to the destination repository. It is good practice to make sure that there are no incoming changes before you make your pull request.

https://confluence.atlassian.com/bitbucket/work-with-pull-requests-223220593.html

If a pull request does not target HEAD of the target branch there may be merge conflicts and test failures when the request is merged.
