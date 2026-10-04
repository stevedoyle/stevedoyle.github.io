---
title: "PRD vs System Requirements: Same Feature, Different Questions"
date: 2026-10-04
tags: [requirements, prd, systems-engineering, writing]
toc: true
---

Product requirements (PRD) and system requirements (SYS) often get written by different people, at different times, in different documents, and then argued about in the same review.
Most of those arguments come from mixing the two levels: a PRD full of API names, or a system spec full of business goals.
This post sets out what separates them, shows the same feature written at both levels, and maps which writing tips apply to which level.

*Disclaimer: I used Claude AI to help draft this post and the examples, which I then edited.*

## Two Levels, Two Questions

| | PRD | System requirements |
|---|---|---|
| **Question** | What must the product do for the customer? | What must the engineered system do to deliver that? |
| **Viewpoint** | Product as a black box, in the customer's language | System as a black box, in engineering terms: interfaces, standards, budgets |
| **Audience** | PMs, customers, sales | Architects, component owners, test |
| **Verified by** | A customer acceptance test | A system or integration test |

The key idea is that **one level's "how" is the next level's "what"**.
"Hosts reach remote storage through NVMe emulation on the IPU" is design from the PRD's point of view.
At the system level it's a perfectly good requirement, because the system boundary is where that decision becomes visible.
Each statement belongs at the level where it is a "what".

## The Same Feature at Both Levels

The running example is remote NVMe-oF storage served through an IPU (infrastructure processing unit), so tenants get block storage without spending host CPU on storage protocols.

### PRD

```
PRD-UC-003 [Must]: As an infrastructure operator, I need tenant I/O to keep running
  when one storage network path or one storage server fails, so that a single fault
  does not cause an outage.

PRD-CAP-005 [Must]: Tenant I/O continues through the loss of one network path.
  Parent: PRD-UC-003 · Acceptance: pull one storage link under load; the
  application sees no I/O errors

PRD-PERF-004 [Must]: During a path or server failure, no tenant I/O stalls for
  longer than [TBD-8] s.
  Parent: PRD-UC-003 · Acceptance: maximum I/O completion gap during fault injection

PRD-SEC-002 [Must]: A tenant with full control of its OS cannot change its own
  storage configuration or reach the storage network.
```

Notice what's missing: no ANA (Asymmetric Namespace Access, the NVMe mechanism by which a target tells the host which paths to a namespace are optimized, usable, or down), no multipath driver, no NVMe/TCP.
A customer can test every one of these without knowing how the product is built.

### System Requirements

```
SYS-FR-008: Where more than one path to a namespace exists, the IPU SHALL continue
  host I/O on a surviving path within [TBD-21] s of a path failure.
  Parent: PRD-CAP-005, PRD-PERF-004 · Verify: Test (link pull under load)

SYS-FR-009: When a target reports an ANA state change, the IPU SHALL route
  subsequent I/O only to paths in the Optimized state, if one exists.
  Parent: PRD-UC-003 · Verify: Test (target failover with ANA)

SYS-FR-010: If no path to a namespace is available, then the IPU SHALL hold host
  I/O for up to [TBD-22] s before completing it with a path error.
  Parent: PRD-PERF-004 · Verify: Test (all paths down)
  Rationale: holding I/O rides out a short outage; failing it after a bound
  avoids a hung tenant.

SYS-SEC-002: The host SHALL have no network path to the storage fabric and no
  access to the IPU management plane.
  Parent: PRD-SEC-002 · Verify: Test (host-side scan and connect attempts fail)
```

Now the standards (ANA), the system boundary (host vs IPU), and the failure modes appear.
`SYS-FR-010` is the interesting one: the PRD never asked "what happens when *every* path is gone?", but the system level has to answer it.
Writing the "If ... then" form is what flushed it out.

The system spec then adds an **allocation** table that assigns each SYS requirement to components and splits numeric budgets across them, e.g. the failover time across the host front end, storage service, and transport.
If the shares don't add up to the parent, you've found a problem before anyone writes code.

## Which Tips Apply Where

Some practices apply at both levels; others belong to one.

| Tip | PRD | SYS | Notes |
|---|:-:|:-:|---|
| **Say what, not how** | ✅ | ✅ | The boundary moves. PRD: no APIs, protocols, or data structures. SYS: no component internals. Test: could two different implementations both pass? |
| **Never invent a number** | ✅ | ✅ | Use `[TBD-n]` or `[TBR-n: value]` and track it. A plausible made-up number gets copied into test plans and customer commitments. |
| **One behaviour per requirement** | ✅ | ✅ | Split on "and", "or", "with fallback to". A half-delivered requirement can't pass or fail. |
| **Ban weasel words** | ✅ | ✅ | *Fast, robust, seamless, support*: replace each with the behaviour or number it hides. |
| **ID, parent, and verification on every item** | ✅ | ✅ | IDs are permanent: never renumber, mark deleted items "withdrawn". |
| **Prioritize with MoSCoW** | ✅ | | Must / Should / Could / Won't (this release): makes the "what do we drop if we slip?" trade-off explicit. At SYS level the SHALL/SHOULD keyword carries the obligation; add a release tag only if delivery is phased. |
| **Write use cases, including out-of-scope ones** | ✅ | | "Tenants can't manage their own connections" is what forces multipath onto the IPU. |
| **Acceptance a customer could run** | ✅ | | If only a developer could test it, it's probably SYS or lower. |
| **Use EARS patterns** | | ✅ | *When / While / If-then / Where* make triggers, states, and error handling explicit. |
| **Record forced "hows" as design constraints** | CON | DC | A mechanism mandated by a customer, regulation, or upstream policy, with its source named. No source means it's a preference, not a constraint. |
| **Allocate to components and split budgets** | | ✅ | Unallocated requirements and budgets that don't sum are the most valuable SYS findings. |
| **Trace to the level above** | MRD | PRD | Parents with no children are coverage gaps; children with no parent are scope creep. |

## The Takeaways

- Write the PRD so a customer could sign it off and the SYS spec so a test engineer could execute it.
- When a review stalls on "that's too detailed" or "that's too vague", check the level before debating the wording.
- Don't skip a level. Going straight from PRD to component requirements hides the system decisions in between, and those are exactly the ones that cause integration surprises.
