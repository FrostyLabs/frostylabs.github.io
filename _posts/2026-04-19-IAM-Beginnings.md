---
title: "IAM: Simple Until it Isn't"
date: "2026-04-19"
author: frosty
categories:
  - "blog"
tags:
  - "iam"
  - "identity"
  - "security"
---

Ask most people in security what Identity and Access Management is and they'll give you a reasonable answer. Something about users, permissions, and making sure the right people have access to the right things. And that's basically correct. The problem is that summary makes it sound far more straightforward than it is in practice.

IAM is one of those domains where the theory is clean and the reality is messy. The gap between the two is where most organizations quietly accumulate risk.

## What IAM Actually Is

At its core, IAM is the discipline of managing who can access what, under what conditions. That covers authentication (proving who you are), authorization (deciding what you're allowed to do), and the lifecycle management that keeps both accurate over time. In practice it spans directories, identity providers, SSO platforms, privileged access systems, SaaS applications, cloud IAM policies, and more.

The reason it matters so much from a security perspective is simple: access control is ultimately the thing standing between an attacker and your data. Firewalls, EDR, SIEM, these are all important, but if someone can just log in as a legitimate user with legitimate permissions, a lot of that becomes noise.

## The World Has Changed, IAM Has Struggled to Keep Up

A decade ago, IAM was conceptually simpler. Most organizations ran Active Directory on-premises, employees worked from office networks, and applications lived in the data center. The perimeter was a physical thing. You knew where your users were, you knew where your systems were, and you managed access inside that boundary.

That world is largely gone. Today a typical organization might have its identity stored in Entra-ID (ex. Azure AD), its workloads split across AWS, Azure, and on-prem, its business applications spread across Salesforce, GitHub, Jira, and its employees connecting from home, from airports, from personal devices. Each of those environments has its own identity model, its own access controls, and its own idea of what a permission even means.

The result is that identity is no longer in one place. It gets federated across systems, translated between protocols (SAML, OIDC, SCIM.. maybe even CSV), and stitched together in ways that nobody fully mapped out intentionally. That is the environment IAM teams are working in today.

## Where Things Actually Break

The principles behind IAM, least privilege, role-based access control, regular access reviews, are well understood and largely uncontroversial. What's hard is operationalizing them in a real organization, over years, across teams that have competing priorities.

Joiner-Mover-Leaver processes are a good place to start. When someone joins, they need access provisioned quickly so they can do their job. When they move to a new team, old access should be removed and new access granted. When they leave, everything should be revoked. Simple enough in theory. In practice, provisioning is often manual or partially automated, dependent on HR data that arrives late, and nobody has a complete picture of everywhere an account has access. People accumulate permissions over time. Someone joins the engineering team and gets access to AWS. They move to product management and the AWS access sticks around because nobody noticed or nobody filed the right ticket. Two years later they leave the company, and there is a 48-hour gap before their accounts are fully disabled because the offboarding process hit a snag. None of that is malicious. It is just entropy.

Role explosion is another one. RBAC looks elegant on a whiteboard: define roles, assign users to roles, done. But organizations evolve. Teams create custom roles for specific use cases. Exceptions get made. One-off permissions get granted and never revisited. Over time you end up with hundreds of roles, significant overlap between them, and no real understanding of what a user actually has access to at any given moment. Access reviews become a checkbox exercise because nobody has the context to meaningfully evaluate what they are approving.

Non-human identities are increasingly where things go wrong at scale. Service accounts, API keys, machine identities, CI/CD pipeline credentials, these do not follow the same lifecycle as human users. They rarely get rotated. They often carry more privilege than they need because it was easier to give them broad access than to figure out exactly what they required. And unlike a human account, they do not show up on an HR offboarding list. I have seen environments where the number of service accounts exceeds the number of human users by a significant margin, and the documentation for most of them is thin or nonexistent.

Visibility is the thread running through all of this. You cannot manage what you cannot see. Most organizations do not have a clean, unified view of who has access to what across all their systems. Cloud IAM policies, SaaS app assignments, AD group memberships, PAM vaults, these are all separate data sources with no easy way to correlate them. Answering a basic question like "what does this user actually have access to?" can take serious investigation.

## Why This Creates Real Risk

Overprivileged accounts give attackers more to work with once they are in. Stale accounts from departed employees are a persistent attack vector, often still valid credentials with nobody monitoring them. Service account compromise is a common lateral movement path because those accounts frequently have broad, persistent access and generate less suspicious-looking traffic than a human doing the same thing.

There is a compliance angle worth mentioning, though it is worth separating compliance from actual security. Many organizations put significant effort into access reviews, segregation of duties controls, and audit trails because a regulator requires it. That work has value, but it can also create a false sense of security. A quarterly access review that nobody takes seriously is a compliance artifact, not a security control.

The organizational side of this is underappreciated too. IAM sits at the intersection of HR, IT, and security, three teams that typically do not share the same priorities, use different tooling, and sometimes do not communicate well. The consequence is that policies that look solid on paper do not always survive contact with how the business actually operates.

## A Foundation Worth Getting Right

IAM is not glamorous. It does not have the same visibility as detection engineering or offensive security. But it is foundational in a way that is hard to overstate. A mature IAM program reduces the blast radius of a compromise, limits lateral movement, and gives you the visibility to actually understand who has access to your critical systems.

Most organizations are somewhere in the middle: functional enough to pass an audit, not mature enough to confidently answer hard questions. The interesting work is in closing that gap.

In future posts I will get into specific areas in more depth, how to think about access governance, where automation helps and where it creates its own problems, and how IAM looks in cloud environments where the model is fundamentally different from on-premises.
