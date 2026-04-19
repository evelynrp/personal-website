---
title: "Open Source Has a Sustainability Problem"
date: 2026-04-19
---
Open source has a sustainability problem, and it isn't primarily a funding problem or a marketing problem. It's a people problem: specifically, a problem with how projects treat the people who show up wanting to help.

The pipeline is broken at both ends. Not enough new contributors come in, and too many of the ones who do don't stay. The result is a chronic concentration of burden on a small group of maintainers who are already stretched thin, which makes the contributor experience worse, which drives retention lower, resulting in a slow, self-reinforcing collapse.

The good news is that most of the fixes don't require more maintainer time. They require better use of the time maintainers already spend, and a willingness to invest once in infrastructure that pays back indefinitely.

## Why People Don't Contribute (And Why They Leave)

Before proposing solutions, it's worth being honest about the problem. It's common to point fingers at bad marketing, a general ignorance about or indifference toward open source, and other PR issues, but these issues don't get at the root of the problem.

**It's not that people don't care.** There is goodwill toward open source software among developers. Many people want to give back to tools they use every day. We don't need to expand the global Linux marketshare; we need to empower the people we already have.

**It's that the onboarding experience sucks.** The typical first-contribution journey looks something like this: find a project you use, look for a way to help, read a CONTRIBUTING.md that was written for people who already understand the codebase, struggle to get a dev environment running, find a "good first issue" that is either trivially cosmetic or actually requires deep architectural knowledge, submit a PR, and then wait. Sometimes weeks. Sometimes indefinitely. The silence reads as rejection, and most people don't come back.

**Retention fails at the transition.** Even contributors who have a good first experience often disappear after one or two patches. The reason is almost always the same: there's no path forward. They finished the issue they came in for, and nothing tells them what to do next or signals that they're becoming part of something. The project didn't give them a reason to stay.

**The culture problem is not interpersonal; it's structural.** High-profile projects, particularly in the Linux ecosystem, have historically had review cultures that would be considered unprofessional in any workplace context. This has improved with the widespread adoption of Codes of Conduct, but it remains uneven. More importantly, individual behavior is only part of it. The structural environment, i.e. complex Git workflows, mailing list culture, and the implicit expectation that you already understand everything, is itself a form of hostility, regardless of whether any individual is rude.

## The Core Principle: Systems Over Interactions

The key insight for resource-constrained projects is this: **individual interactions don't scale, but infrastructure does.**

A maintainer who personally mentors every new contributor is doing heroic work that evaporates when they burn out. A project with a well-written onboarding guide, a real triage process, and a fast acknowledgment system gets compounding returns on those investments for years.

Every intervention below is evaluated by the same question: does this require ongoing maintainer presence, or does it work while maintainers are sleeping?

### 1. Onboarding as a first-class artifact

Most contributing guides are either too sparse ("fork, branch, PR") or too detailed in the wrong ways (commit message formats, license headers). Few answer the questions a new contributor actually has:

- What does this project do, at a high level, and why?
- Where does the relevant code live?
- How do I get a working environment in under 15 minutes?
- What does a good first contribution look like?
- What happens after I submit a PR?

A contributing guide that answers these questions is a one-time investment that reduces the load on maintainers every single time someone new shows up. The return compounds. Projects that have excellent onboarding documentation see measurably better first-PR conversion rates. Projects that don't are filtering for people who are already experienced enough not to need it, which is the opposite of growing your contributor base.

This document should be written by someone who recently went through the onboarding process themselves, not by the person who has been in the codebase for five years and forgot what was confusing.

### 2. "Good First Issue" as a real contract

The "good first issue" label is one of the most valuable and most abused conventions in open source. When it works, it's a curated entry point that gives newcomers a real, bounded problem with enough context to succeed. When it doesn't, it's a graveyard of vague tasks that haven't been touched in two years.

Making "good first issue" mean something requires a small, recurring investment: roughly 30 minutes a month from one maintainer to tag, scope, and describe approachable issues[^1]. A good entry-point issue description includes:

- What the problem is and why it matters
- What a solution would look like (not the implementation, but the shape of the answer)
- Where in the codebase to start looking
- What to ask if you get stuck

This is not hand-holding. It's the difference between a job posting that says "engineer wanted" and one that tells you what you'll actually be doing. The latter gets better applicants.

[^1]: For large projects like Fedora, which has multiple SIGs and working groups working on a wide variety of projects, the overhead for maintaining "good first issues" becomes more expensive. There have been excellent efforts to scale onboarding, including the Join SIG and [I Make Fedora](https://imakefedora.me). Rather than introduce new complexity to an already-complicated system, detailed "good first issue" templates can be applied to similar issues; comprehensive newcomer documentation (not the "these are our principles" type, but the "this is why/how we do this thing" type) can be linked; "good first issues" can be curated at the SIG, not Project, level; automation can flag stale issues and ping maintainers so the "good first issues" tag isn't full of only old issues.

### 3. Fast first response, not fast full review

The single most discouraging experience a first-time contributor can have is submitting a PR and hearing nothing for two weeks. It doesn't matter that the maintainer is busy or that the review requires careful thought. The silence reads as "your contribution doesn't matter," and most people won't submit another one.

The solution is to decouple acknowledgment from review. A message that says "Thanks for this! We've seen it and it's in the queue, we'll have feedback by [rough timeframe]" takes 30 seconds and prevents the feeling of rejection. This can be partially automated with a bot that posts a templated response and pings the relevant maintainer, or it can be handled by a rotation among trusted contributors. What it cannot be is nothing.

### 4. Automate the mechanical feedback

A significant portion of review comments on first contributions are mechanical: formatting, linting, commit message conventions, test failures. When maintainers give this negative feedback manually, it feels personal, even when it isn't. When a CI system catches it automatically before any human looks at the PR, it's just a checklist.

Setting up automated formatting checks, linters, and test runners is a one-time cost that saves reviewer time on every subsequent PR and makes the feedback loop feel less adversarial. New contributors learn the project's conventions from the machine, not from a terse comment from a stranger.

### 5. Delegated review authority

Most projects concentrate merge rights in their original authors long past the point where that's necessary or efficient. The result is a bottleneck: everything waits for one or two people, who are already overwhelmed, which slows the review cycle, which frustrates contributors, which reduces retention.

A tiered trust model distributes this load and creates a legible growth path for contributors:

- **New contributors** submit PRs, receive review
- **Trusted contributors** can review and approve (but not merge) PRs in their area of the codebase
- **Maintainers** handle final merges, architecture decisions, and conflict resolution

The upfront cost is identifying who belongs in the middle tier and communicating the trust explicitly. The long-term payoff is that maintainers spend their time on things that actually require their judgment, and mid-level contributors have a meaningful role that gives them a reason to stay engaged.

### 6. Write down decisions

An enormous invisible tax on maintainers is re-explaining settled decisions every time a new contributor suggests revisiting them. Why does the project use this build system? Why is the API designed this way? Why isn't feature X supported?

The answer to each of these questions exists in someone's head. It should exist in a document.

Architecture Decision Records (ADRs) are a lightweight format for this: a short document per decision that captures what was decided, why, and what alternatives were considered. A maintainer who writes one ADR is writing it once and then never having to have that conversation again. The link replaces the explanation.

This is also valuable for new contributors trying to understand the project's reasoning, which is part of what makes a codebase feel navigable versus opaque.

## The "People Person" Problem

Open source has a shortage of community-oriented contributors relative to technically-oriented ones. The usual response is to hope that technically excellent people also turn out to be good communicators and community builders. This rarely works at scale.

The better response is to create roles that don't require that combination.

A **contributor experience role**, community manager, triage team member, or docs maintainer, is a valuable contribution pathway for people who are motivated but not yet writing production code. Their responsibilities might include:

- Welcoming new contributors and helping them find appropriate issues
- Following up on stalled PRs to check whether the contributor needs help or has moved on
- Maintaining and improving the onboarding documentation
- Triaging new issues and applying appropriate labels
- Running a first-response rotation for new PRs

None of this requires deep codebase knowledge. All of it is work that otherwise falls on maintainers or doesn't get done at all. Making it an explicit, valued role with its own section in the contributing guide and its own recognition in release notes gives projects access to a large pool of motivated contributors who currently have nowhere to go.

## On Retention

Retention fails at a predictable moment: right after a contributor's first successful contribution, when they have no idea what to do next.

Projects that retain contributors give them a **narrative arc**: a sense that there's a path from first patch to regular contributor to trusted reviewer, with legible steps at each transition. This doesn't require hand-holding. It requires that the path exist and be written down somewhere they can find it.

Recognition also matters more than open source culture typically acknowledges. A name in the changelog, a mention in the release notes, or a genuine thank-you from a maintainer cost almost nothing and signal that the person's work was noticed and mattered. The absence isn't neutral. It reads as indifference, and indifference is a good reason to quit.

## A Prioritized Starting Point

For projects with limited resources, the order of operations matters. Here is a rough prioritization by impact-per-effort:

**Do these first**: one-time investments with compounding returns:
- Write an effective onboarding guide, authored by someone who recently went through it
- Document existing architectural decisions in a shared, linkable format
- Set up automated linting, formatting, and CI checks

**Do these on a light recurring cadence**: low cost, high top-of-funnel impact:
- Audit and refresh "good first issue" tags monthly
- Maintain a first-response rotation for new PRs (can be partially automated)

**Do these when you have a few trusted contributors**: medium setup cost, high long-term payoff:
- Delegate review authority to a tier of trusted contributors
- Create and publicize a contributor experience role

Everything else, including contributor spotlights, dedicated community calls, and detailed mentorship programs, is valuable but secondary. The compounding effect of the basics, done well, outperforms elaborate programs built on top of broken core infrastructure.

## Why Any of This Matters

Open source infrastructure powers most of the modern internet. It is sustained by volunteers contributing labor with no guarantee of compensation or recognition, and they are getting tired. The response from foundations and companies has mostly been financial: grants, paid maintainers, and bounties. These help, but they don't fix the experience that determines whether new contributors show up and stay.

The contributor experience is largely an infrastructural problem, and infrastructural problems have infrastructural solutions. Most of the interventions described here don't require money. They require up-front investment, a willingness to delegate, and a recognition that treating contributors well is the mechanism by which open source actually sustains itself.

Projects that invest in this will have more contributors, more resilient maintainer teams, and a better chance of being around in ten years. Projects that don't will continue to concentrate burden on fewer and fewer people until someone burns out and the project quietly stops.

The fixes are not complicated, but they are consistently deprioritized in favor of more visible, less thankless, work, like developing the next new feature. That's a choice, and it's one that open source communities can make differently.

## Revision History

- 2026-04-19: Original posting.