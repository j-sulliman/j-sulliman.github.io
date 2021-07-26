---
layout: default
title: Infrastructure as Code
nav_order: 7
has_children: true
permalink: docs/infrastructure-as-code
---

# Overview
{: .no_toc }
Custom scripting and automation works well in project delivery where there's usually one or two delivery engineers implementing the solution.  

The further we progress from delivery - design, implementation to operational hand-over, the more important the choice and supportability of automation tools becomes. Particularly in Enterprises where infrastructure and operations teams do not have easy access to developers to provide the review and QA of the automation tools that infrastructure engineers have created.

Here's some food for thought around the supportability of the automation approach and tool sets you use.

* Does the problem you are solving through automation need to be supported and maintained by a wider team on an ongoing basis? Or by one or two delivery engineers?
* Is DevOps practices, capability and adoption a current reality or a future aspiration?  
* Does the code need to be supported on an ongoing basis?
* Who is going to support it?
* Is the team equipped to maintain often poorly documented python code? Is that code free of bugs, fully tested? Secure?
* Or are you better off using a well documented and supported off-the-shelf tool?

This is where configuration automation or "Infrastructure as Code" tools such as Ansible have greater relevance, automation of environments at scale, where supportability and re-use by a wider team of engineers, each likely with varying levels of programming and automation skills.... Those organisations, where DevOps is still immature, or an aspirational goal.

Here we explore tools such as Ansible and Terraform to automate the deployment and configuration of network and data center infrastructure.
{: .fs-6 .fw-300 }
