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
"Photos queue in a local encrypted database and upload via a background sync service" is design from the PRD's point of view.
At the system level it's a perfectly good requirement, because the system boundary is where that decision becomes visible.
Each statement belongs at the level where it is a "what".

## The Same Feature at Both Levels

The running example is a mobile photo app that keeps working when the phone loses network connectivity: photos taken offline are queued locally and uploaded automatically once the connection returns.

### PRD

```
PRD-UC-003 [Must]: As a user, I need photos taken while offline to be saved and
  uploaded automatically once I reconnect, so that I never lose a photo because
  of a dropped connection.

PRD-CAP-005 [Must]: Photos taken while offline are queued on the device and
  uploaded automatically when connectivity returns.
  Parent: PRD-UC-003 · Acceptance: take photos in airplane mode, reconnect, and
  confirm every photo appears in the cloud library

PRD-PERF-004 [Must]: Once connectivity returns, a queued photo finishes
  uploading within [TBD-8] minutes.
  Parent: PRD-UC-003 · Acceptance: measure time from reconnect to the photo
  appearing in the cloud library

PRD-SEC-002 [Must]: A queued photo is readable only by this app on this device
  until it uploads; no other app or user can access it.
```

Notice what's missing: no SQLite schema, no retry-library name, no encryption algorithm.
A customer can test every one of these without knowing how the product is built.

### System Requirements

```
SYS-FR-008: Where local storage has space, the app SHALL queue a captured photo
  for upload immediately on capture, independent of network state.
  Parent: PRD-CAP-005 · Verify: Test (capture in airplane mode, confirm photo
  appears in the local queue)

SYS-FR-009: When connectivity is restored, the app SHALL begin uploading queued
  photos within [TBD-21] s, oldest first.
  Parent: PRD-UC-003, PRD-PERF-004 · Verify: Test (reconnect, measure start
  latency)

SYS-FR-010: If an upload attempt fails, then the app SHALL retry with
  exponential backoff, up to [TBD-22] attempts, before marking the photo
  failed and notifying the user.
  Parent: PRD-PERF-004 · Verify: Test (simulate server errors during upload)
  Rationale: unbounded retries drain battery and hide a real failure; the user
  needs to know when to intervene.

SYS-SEC-002: Queued photos SHALL be stored in the app's private, encrypted
  storage sandbox, inaccessible to other apps on the device.
  Parent: PRD-SEC-002 · Verify: Test (attempt cross-app file access)
```

Now the storage mechanism (local queue, retry policy), the system boundary (app vs OS vs cloud service), and the failure modes appear.
`SYS-FR-010` is the interesting one: the PRD never asked "what happens when the upload keeps failing?", but the system level has to answer it.
Writing the "If ... then" form is what flushed it out.

The system spec then adds an **allocation** table that assigns each SYS requirement to components and splits numeric budgets across them, e.g. the upload-start latency across the connectivity monitor, the queue manager, and the network client.
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
| **Write use cases, including out-of-scope ones** | ✅ | | "The user can't manually manage the upload queue" is what forces automatic retry onto the app. |
| **Acceptance a customer could run** | ✅ | | If only a developer could test it, it's probably SYS or lower. |
| **Use EARS patterns** | | ✅ | *When / While / If-then / Where* make triggers, states, and error handling explicit. |
| **Record forced "hows" as design constraints** | CON | DC | A mechanism mandated by a customer, regulation, or upstream policy, with its source named. No source means it's a preference, not a constraint. |
| **Allocate to components and split budgets** | | ✅ | Unallocated requirements and budgets that don't sum are the most valuable SYS findings. |
| **Trace to the level above** | MRD | PRD | Parents with no children are coverage gaps; children with no parent are scope creep. |

## The Takeaways

- Write the PRD so a customer could sign it off and the SYS spec so a test engineer could execute it.
- When a review stalls on "that's too detailed" or "that's too vague", check the level before debating the wording.
- Don't skip a level. Going straight from PRD to component requirements hides the system decisions in between, and those are exactly the ones that cause integration surprises.
