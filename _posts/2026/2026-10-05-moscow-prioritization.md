---
title: "MoSCoW: Prioritizing Requirements So Something Can Be Dropped"
date: 2026-10-05
tags: [requirements, prd, prioritization, moscow, dsdm]
toc: true
---

Every requirements list is longer than the schedule.
The point of prioritizing requirements is to decide, before the crunch, what gets dropped when time runs out.
MoSCoW is the simplest widely used technique for doing that, and most of its value comes from a few rules that are easy to skip.
This post covers what MoSCoW is, the practices that make it work, and where each one comes from.

*Disclaimer: I used Claude AI to help research and draft this post, which I then edited.*

## What MoSCoW Is

MoSCoW sorts each requirement into one of four buckets.
The lowercase o's are only there to make it pronounceable.

| Bucket | Meaning |
|---|---|
| **Must have** | The *Minimum Usable SubseT*: what the project guarantees to deliver. Without it there's no point delivering on the target date, or the result is illegal, unsafe, or not viable. |
| **Should have** | Important but not vital. Leaving it out may be painful and need a workaround, but the solution is still viable. |
| **Could have** | Wanted, but with less impact if left out than a Should. |
| **Won't have this time** | Agreed *not* to be delivered in this timeframe. |

Dai Clegg developed the technique in 1994 while at Oracle for rapid application development ([TechTarget](https://www.techtarget.com/it-infrastructure/definition/MoSCoW-method), [Wikipedia](https://en.wikipedia.org/wiki/MoSCoW_method)).
It became a core part of DSDM, the agile framework now maintained by the Agile Business Consortium.
Their guide, [What is MoSCoW Prioritization?](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/), is the authoritative reference and the best place to read more.

MoSCoW assumes the deadline and budget are fixed and the scope is what flexes.
If the date can move whenever something slips, the buckets don't mean anything.

## An Example

Here are some product requirements for remote NVMe-oF storage through an IPU, the running example from my [PRD vs system requirements](https://slog.stevedoyle.io/prd-vs-system-requirements/) post:

```
PRD-CAP-001 [Must]:   Tenants see remote volumes as NVMe block devices using
                      their OS's inbox driver.
PRD-CAP-005 [Must]:   Tenant I/O continues through the loss of one network path.
PRD-CAP-004 [Should]: The operator can grow a volume online, and the tenant sees
                      the new capacity without a reboot.
PRD-DEP-004 [Could]:  The operator can export per-volume latency histograms.
PRD-UC-005  [Won't]:  Tenants manage their own storage connections from inside
                      their OS.
```

Ask "what happens if we don't ship this?" for each one.
Without CAP-001 or CAP-005 there's no product.
Without online resize, operators have to schedule downtime: painful, but workable.
Histograms are nice to have.
And writing down the Won't stops the "can't the tenant just configure it?" debate coming back every review.

## Tips and BKMs

### 1. Use the cancellation test for Musts

Ask "what happens if this requirement isn't met?"
If the honest answer is "cancel the project, there's no point shipping without it", it's a Must.
If there's a workaround, even an ugly one, it's a Should at most.
*Source: [Agile Business Consortium](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/).*

### 2. Keep Musts to about 60% of the effort

DSDM recommends no more than 60% of the effort be Must Haves, with around 20% Could Haves as contingency.
Percentages are of **effort, not count**: ten small Musts and one huge Should can still break the budget.
If everything is a Must, nothing has been prioritized, and there's nothing to drop when an estimate turns out wrong.
*Source: [Agile Business Consortium](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/).*

### 3. A Must can only depend on other Musts

If a Must depends on a Should, the Must is only as safe as the Should.
Either promote the dependency or find another way to meet the Must.
*Source: [Agile Business Consortium](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/).*

### 4. Decompose before you prioritize

A high-level Must usually splits into sub-requirements with mixed priorities.
"Survive path failure" is a Must; "report path state in the management UI" may only be a Should.
Splitting lets you drop detail without dropping the requirement.
*Source: [Agile Business Consortium](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/).*

### 5. Agree the Should vs Could line before you start

Musts are usually clear; the argument is over Should vs Could.
Agree up front how you'll tell them apart, for example by the degree of pain if it's left out, measured in business value or the number of people affected.
*Source: [Agile Business Consortium](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/).*

### 6. Say "Won't have *this time*"

The most common criticism of MoSCoW is ambiguity over whether Won't means "not in this release" or "never" ([Wikipedia](https://en.wikipedia.org/wiki/MoSCoW_method)).
Always write the timeframe, keep Won'ts in the backlog, and revisit them at the next planning cycle.

### 7. Rank within each bucket

MoSCoW doesn't help choose between two Shoulds when only one fits ([Wikipedia](https://en.wikipedia.org/wiki/MoSCoW_method)).
Keep each bucket in a ranked order so the drop decision is already made.

### 8. Prioritize per timeframe and review regularly

A requirement can be a Must for the project but a Could for this sprint or increment.
Review priorities at the end of each timebox and increment, as business needs change.
Agree early who decides when a Must is at risk, and how decisions escalate.
*Source: [Agile Business Consortium](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/).*

### 9. Put technical work in the same list

Another known failure mode is a focus on new features at the expense of technical improvements such as refactoring ([Wikipedia](https://en.wikipedia.org/wiki/MoSCoW_method)).
If test infrastructure or a necessary refactor is in a separate list, it will lose every argument.
Prioritize it alongside the features it enables.

### 10. Use MoSCoW for product requirements, not system requirements

This one is from my own practice.
At PRD level MoSCoW makes the customer trade-off explicit.
At system level, the requirement keyword already carries the obligation (SHALL is mandatory, SHOULD is a goal), so adding MoSCoW says the same thing twice and the two can disagree.
Use a release tag at system level if delivery is phased.

## Takeaways

- MoSCoW works only if scope is the variable and the date isn't.
- The two rules that matter most are the cancellation test and the 60% cap on Must effort.
- Write the Won'ts down, with "this time" attached.

## Further Reading

- [What is MoSCoW Prioritization?](https://www.agilebusiness.org/resource/what-is-moscow-prioritization/), Agile Business Consortium (DSDM)
- [MoSCoW method](https://en.wikipedia.org/wiki/MoSCoW_method), Wikipedia
- [MoSCoW method](https://www.techtarget.com/it-infrastructure/definition/MoSCoW-method), TechTarget
