---
layout: post
title: "The Cost of Complex Configuration"
description: "Why Mike Hadlow's Configuration Complexity Clock still matters"
---

Some old software posts age badly. Mike Hadlow's short 2012 post, [The Configuration Complexity Clock](https://mikehadlow.blogspot.com/2012/05/configuration-complexity-clock.html?m=1), did not.

The idea is simple. Moving a few values out of code and into configuration is often a good step. But if a team keeps pushing more behavior into configuration, the configuration stops being simple settings. It becomes rules, exceptions, and logic. At that point, it is really code in another form.

That is what makes the post stick. It does not say configuration is bad. It says configuration has a cost. Simple settings are fine. But when configuration starts carrying business rules, the team is often building a second programming system with weaker tooling and harder debugging.

## Why teams do this

Teams usually end up here for good reasons.

They want speed. They want flexibility. They do not want to rebuild and redeploy the whole application for every small business change. That goal makes sense.

But each extra layer adds a new cost. Someone has to learn it. Someone has to test it. Someone has to debug it. Someone has to teach it to new team members. And when only a small group can change it safely, the system becomes less flexible in practice.

## Values in config, logic in code

One of the strongest points in the post is that plain code is sometimes the simpler answer.

That sounds strange at first, because many of us were taught never to hard-code anything. But Hadlow's point is more balanced than that. Once configuration becomes too complex, writing the rule in a normal programming language may be the cleaner choice.

Normal code already comes with the things complex logic needs:
- version control
- review
- tests
- debugging tools
- a release process the team already understands

If a rule has many branches, exceptions, and edge cases, code may be the better home for it.

## The real lesson

This does not mean configuration is bad.

Some configuration clearly belongs outside the codebase: environment-specific values, endpoints, feature flags, thresholds, and deployment settings. The problem is not configuration itself. The problem is pretending logic is configuration just because it lives in a file, table, or UI.

Complexity does not go away when we move it out of the codebase. It only moves. We can hide it in files, admin screens, rule editors, or special tools. But if the thing we are changing contains real logic, it still needs the same care as code.

What I like most about the post is that it is not extreme. Hadlow does not say rules engines or DSLs are always wrong. He is saying they have real costs, and those costs are easy to underestimate when a team is trying to move fast.

That is why the post still matters. We do not make a system simpler by giving its logic a different name. We only decide where it lives, and what tools we will have to maintain it with.

## Source

- [Mike Hadlow, "The Configuration Complexity Clock"](https://mikehadlow.blogspot.com/2012/05/configuration-complexity-clock.html?m=1)
