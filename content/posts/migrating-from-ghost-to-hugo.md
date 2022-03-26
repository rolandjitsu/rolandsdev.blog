---
title: "Migrating from Ghost to Hugo"
description: "First impressions of the Hugo static site generator."
date: 2022-03-26T13:26:57+08:00
lastmod: 2022-03-26T13:26:57+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["thoughts"]
categories: []
series: []

toc:
  enable: no
---

It's been some time since I have posted anything (I recently became a father, so I spend most of my free time with the :baby:) and for the past few months (about 5 months), my blog has been down.

I've noticed some time in November that my blog was throwing a 500 and I just spent maybe 5 minutes to check the logs on the VM where it was running (GCP) to see why. Everything seemed ok (nginx was up and [ghost](https://ghost.org/) showed no issues either).

So I thought to myself: "*well, it's probably gonna take some time to investigate wtf is going on and I don't have time right now, I'm gonna do it maybe next week*." So I just stopped the instance so it doesn't incur cost.  And here we are, 5 months later and I still didn't take the time to investigate the issue :laughing:

During that time I also found [Hugo](https://gohugo.io/) - I was looking for an easy way to write some docs for some work related stuff - and I said to myself: "*if I can run this for free and I cannot find out why ghost doesn't work, I'll switch to this*".

I know, it just sounds cheap and lazy. But cheap is good in some cases, especially when it comes to running things on the web (or in the :cloud:). And so is lazy, if some tool can do things faster and easier, I will almost always go for that option.

So first I just wanted to see if I can restore my current blog. The first thing I tried was bumping the resources, since [ghost requires at least 1GB of memory](https://ghost.org/docs/hosting/#self-hosting) and I was using a [E2 instance](https://cloud.google.com/compute/vm-instance-pricing#e2_sharedcore_machine_types) - `e2-micro` which had 1GB mem, but only 0.25 cores - because it was cheaper to run. And after bumping to a N2 with 4GB of mem, ghost was back online! So that was about all the investigation I had to do :laughing:

This was great. But the cost for running that also would have increased from $30 USD/month to about $40 USD/month (that includes storage, networking, VM, etc). And while the sum is small, I definitely didn't want to pay for it if there was an alternative which I could run for free! It also didn't make sense that a small blog would need so many resources to run :exploding_head:

So I decided to try out [Hugo](https://gohugo.io/), with the [DoIt](https://hugodoit.pages.dev/) theme, to see how it works and if I like it. So I spent about half a day just testing it out and setting up a blog with a couple of posts. It was easy! Everything from config, theming, running and adding new content was just straightforward. Then I spent the rest of the day porting all the content from the ghost blog (manually going through each post) to the new platform, hugo. I was trivial, but boring :laughing: . But still worth it!

For hosting, I went with [netlify](https://www.netlify.com/), because there's `0` cost to running a static website (at least at the traffic I'm getting). You can even add a custom domain (with SSL - they use [letsencrypt](https://letsencrypt.org/)) at no cost. And the content is on [Github](https://github.com/rolandjitsu/rolandsdev.blog) - also for free. So I ended up with 100% reduction in cost to run my blog :tada:

So my thoughts so far on [Hugo](https://gohugo.io/) are:

1. It's easy to get started with their [quick start guide](https://gohugo.io/getting-started/quick-start/)
2. The [docs](https://gohugo.io/documentation/) are comprehensive
3. The community is large, so it's easy to get help (though, I had no issues at all, yet)
4. There's a decent amount of options for [deployments](https://gohugo.io/hosting-and-deployment/) and they provide docs for those as well; it's a static site, so deployments are also easy
5. The resulted site is fast to load in browsers
6. [Build times are fast](https://forestry.io/blog/hugo-vs-jekyll-benchmark/)
7. It's extensible via [go modules](https://github.com/golang/go/wiki/Modules)
8. There's tons of [themes](https://themes.gohugo.io/) to pick from

But don't take my word for it. Go ahead and try it out yourself!
