---
layout: default
title: SRE - Learnings and Notes
nav_order: 9
has_children: true
permalink: docs/SRE
---

# Site Reliability Engineering
{: .no_toc }

## SRE Resources:
* [Building Secure and Reliable Systems](https://static.googleusercontent.com/media/sre.google/en//static/pdf/building_secure_and_reliable_systems.pdf)
* [SRE on Google](https://sre.google/resources/)
* [Google's Three Free SRE Books](https://sre.google/books/)
* [Consolidated list of SRE resources on Github](https://github.com/dastergon/awesome-sre#readme)

### The difference between DevOps and SRE?
> "If you think of DevOps as a philosophy and an approach to working, you can argue that SRE implements some of the philosophy that DevOps describes, and is somewhat closer to a concrete definition of a job or role than, say, “DevOps engineer.”

On the subject of SRE job roles - here's some JDs of SRE roles at [Dropbox](https://dropbox.github.io/dbx-career-framework/ic1_reliability_engineer.html)

### Challenging the traditional operations as a cost centre perspective:
> "The enterprise world, for example, often treats operations as a cost center,2 which makes meaningful improvements in outcomes difficult if not impossible. The tremendous short-sightedness of this approach is not yet widely understood."

_Chapter 1, The Site Reliability Workbook_

---
### The Guiding Principles

Breaking down silos:
> "extreme siloization of knowledge, incentives for purely local optimization, and lack of collaboration have in many cases been actively bad for business"


Accidents and Mistakes are normal, instead focus of safeguards and faster recovery:
> "The second key idea is that accidents are not just a result of the isolated actions of an individual, but rather result from missing safeguards for when things inevitably go wrong."

Embrace frequent, small changes and automated testing ("Continuous Integration, Continuous Deployment"):
> "Change is risky, true, but the correct response is to split up your changes into smaller subcomponents where possible. Then you build a steady pipeline of low-risk change out of regular output from product, design, and infrastructure changes.6 This strategy, coupled with automatic testing of smaller changes and reliable rollback of bad changes, leads to approaches to change management like continuous integration (CI) and continuous delivery or deployment (CD)."

Culture over tools:
> "proponents of DevOps strongly emphasize organizational culture—rather than tooling—as the key to success in adopting a new way of working. A good culture can work around broken tooling, but the opposite rarely holds true. As the saying goes, culture eats strategy for breakfast. Like operations, change itself is hard."

Measurement is crucial and provides an objective view of reality:  
> "...establish the reality of what’s happening by means of objective measurement, verify that you’re changing the situation as you expect"

Automate everything ("minimise Toil"):
> If you can automate a repetitive operation or admin task, you probably should so you can spend that time on project work, value and service creation activities.

> Google has a hard limit of 50% of time spent on toil, which encourages an engineering approach to problems.

---





## Notes on SRE:

### Overview and Culture
> Moving traditional organisations into the future by encouraging and embracing risk.

> The Service Level Objective is the unified onjective between Ops and devs.  When a service is above target, prioritise feature development, when it's below target, prioritise reliability.  This common goal removes the conflicting priorities between operations and development teams.

> CALMS - "Culture, Automation, Lean Continuous Improvement, Metrics, Sharing"

{: .fs-6 .fw-300 }
