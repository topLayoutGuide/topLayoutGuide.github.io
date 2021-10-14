---
layout: post
title:  "TIL: CircleCI Workflow Approvals"
date:   2019-01-19 12:00:00
color:  "QUIN"
---

#### You Shall Not Pass!... Maybe!

### Intro

In the beginning the Universe was created. This has made a lot of people very angry and been widely regarded as a bad move… just kidding!

It all started in 2019. On one Thursday evening at the office, I had some bandwidth available to tackle a long-standing thorn in my team’s side — enabling manual deployment using our CircleCI environment.

Up until then, we had had various ideas of using (see: building & maintaining) different tools, or utilising an entirely separate service, to achieve our end-goal. Ideally, we should have been able to tap into the CircleCI API, and trigger jobs this way. It didn't look possible with our current setup, and we were upset.

However, we had a simpler solution at hand, unbeknownst to us.

### Workflows

According to CircleCI’s documentation, in order to increase the speed of software development through faster feedback, shorter reruns, and more efficient use of resources, we should configure Workflows.

We already used these, and they served us well. However there was a config key which we weren’t aware of at the time — `type: approval`. 

The documentation surrounding it was confusingly worded and difficult to understand. But after a few re-reads, I got there.

### Approval

When you give a job under a CircleCI workflow the approval type, the “job” must be entirely unique within your `config.yml`. For example, if you have three main jobs defined:

- build
- test
- deploy

Each of which run inside your workflow/s, then the manual trigger must be named something other than any of the above. 

It also must be defined only within your workflows. Furthermore, in order for the manual trigger to actually work, another subsequent job must depend upon it.

I named my trigger `deploy-branch`. This will only be run when one opens the CircleCI Web UI (via Github in our case), and initiates it manually. 

In other words, when one gives approval.

### Our Results

Once I went ahead and tweaked a workflow to include a job with `type: approval`, we had a bona fide manual trigger available to us, finally. 

It didn't have a fancy interface, such as a Github label or a Slackbot command, however it did exactly what we needed it to do.

### Conclusion

As an iOS Developer by specialty, I well and truly stepped outside my comfort zone while fiddling with our CI environment.

However, I feel that this little adventure helped me deepen my skillset overall. Especially when it comes to the build environment enveloping the apps that I help create.

I can only hope that this helps other teams... If you already knew how to do this, then good for you! However I didn't, so I decided to blog about it. 😋

Thanks for reading!