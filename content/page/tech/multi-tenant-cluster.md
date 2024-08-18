---
title: "Multi tenant clusters"
date: 2024-18-08T12:32:12+11:00
draft: true
topic: tech
---

# Multi tenant clusters


What are the benefits?

A well designed platform can offer a lot to teams, particularly if there are "features".
- guardrails and patterns for best practices




## Protect people from themselves (and others)
Don't let devs hurt themselves (or others, or you!).
- resourceLimits
- monitor resource hogging.
- namespace isolation. 
- limit what people can see or touch (ie if you let special resources be provisioned through self service, like DBs, don't let them be able to delete them)

- resourceQuotas and limitRanges


# DR DR DR DR