---
title: "What's Driving RDMA's Reinvention for AI Networks"
date: 2026-08-16
last_modified_at: 2026-09-08
tags: [rdma, networking, ai-infrastructure, falcon, uet, mrc, metaroce]
toc: true
---

RoCEv2 is the dominant RDMA transport for AI clusters today, but it's showing its age and limitations at scale.
The next generation of hyperscale AI networks is being built on enhanced RDMA protocols: Falcon (Google), UET (Ultra Ethernet Consortium), MRC (OCP), and MetaRoCE (Meta), each targeting the same set of fundamental limitations that RoCEv2 can't solve.
This post examines what those limitations are, how each protocol addresses them, and where they appear to be converging.

## RoCEv2 Issues At Scale

RoCEv2's RC (Reliable Connection) transport treats out-of-order arrival as loss.
This is OK on a single fixed path, but it is an issue the moment you want to spray traffic across multiple paths, e.g. for load balancing where reordering becomes the normal case, not the exception, and the transport can't tell the difference between a dropped packet and a late one.
PFC-based lossless Ethernet provides reliability at the cost of head-of-line blocking and pause-storm risk.
And DCQCN-style congestion control reacts on the timescale of RTTs, which is too slow for the microsecond-scale synchronized bursts that collective operations produce.
This causes problems at the scale of today's AI networks, where with hundreds of thousands of GPUs, a single congested link or failed switch can stall an entire training job while thousands of accelerators sit idle.

## The RDMA Extensions

Every new RDMA protocol converges on roughly the same set of extensions to fix these problems:

- **Decouple packet delivery from message semantics.** Track packet reception at a layer below RDMA operation processing, so reordering and loss recovery don't have to preserve application-visible ordering.
- **Multipath by default.** Spray packets or spray connections across the fabric rather than pinning a flow to one path, and treat path failure as a fast, local, sub-RTT event rather than a connection-ending one.
- **SACK/NACK-based loss recovery**, often paired with switch-assisted signaling (trimming, ECN, queueing-delay telemetry) so the sender gets a real congestion signal instead of inferring it from timeouts.
- **Hardware/NIC-offloaded, delay- or telemetry-driven congestion control** that reacts within microseconds, and not RTTs.
- **Programmability**: an adaptive engine or profile mechanism so congestion control (CC) and path selection can evolve without a hardware respin.

Falcon, UET, MRC, and MetaRoCE each instantiate this differently.

## Falcon (Google)

Falcon is a connection-oriented, request-response hardware transport carrying both RDMA and NVMe ULPs.
It is structured as four layers: a ULP mapping layer, a Transaction Layer for on-NIC resource management and ordering, a Packet Delivery Layer (PDL) for delay-based congestion control and SACK-based loss recovery, and the Falcon Adaptive Engine (FAE) that implements CC and multipath load balancing programmably.
Falcon connections can be explicitly **ordered or unordered**, so applications that don't need strict ordering aren't paying its tax.
It also supports inline per-connection encryption.
Falcon started as Google-internal and is now under OCP.
Going forward, [Google's stated intent](https://cloud.google.com/blog/products/compute/ai-infrastructure-at-next26) for its next-gen infrastructure is to fold Falcon concepts directly into NIC silicon rather than keep it as an overlay.
Intel's E2000 series adapters are a notable early hardware implementation, bringing the protocol into merchant silicon and signaling that Falcon is no longer solely a Google-internal stack.

## UET (Ultra Ethernet Consortium)

UET is a ~200-member consortium effort to redesign Ethernet transport for AI from first principles rather than extend RoCEv2.
Its architecture is structured as distinct transport, semantic, and security layers.
The key abstraction is the **Fabric Endpoint**: a logical endpoint decoupled from the physical adapter, so multiple endpoints can share a single NIC and each can be addressed, managed, and migrated independently.
Above that, a semantic layer maps ULPs (e.g. RDMA verbs) to the transport without inheriting RC's ordering constraints.
Its defining transport features:

- **Packet spraying via entropy** (e.g., UDP source port variation) so every packet can take an independent path, which requires the receiver to support reordering.
- **Trim-and-NACK loss signaling**: instead of silently dropping under congestion, switches trim the packet and let the receiver NACK the missing sequence for fast, targeted retransmission rather than a full window replay.
- **NSCC (multi-signal congestion control)**, combining ECN marking with queueing-delay measurement for faster, fairer convergence than DCQCN. Notably, NSCC originated as AMD's contribution via MRC and has since been folded into the UEC Congestion Control spec, a sign of real convergence between these efforts rather than pure competition.

UET is the most architecturally ambitious of the four: a clean-slate redesign rather than an extension of RC semantics.
The UEC 1.0 specification was published in early 2026; no production hardware deployments have been announced at the time of writing.

## MRC (Multipath Reliable Connection, OCP)

MRC takes the opposite path: extend InfiniBand RC semantics minimally, rather than replace them, so a single RDMA connection can spray across multiple paths via ECMP or SRv6.
It adds a packet delivery sublayer decoupled from RDMA semantic processing, SACK with a cumulative offset plus an out-of-order bitmask (so the requester can distinguish real loss from transient reordering), and NACKs driven by deterministic events like trimmed-packet arrival.

Two things stand out for anyone doing driver or verbs-layer work: the application API is deliberately modeled on libibverbs so existing RDMA software (NCCL, RCCL) needs minimal change, and the protocol currently restricts itself to **Write and Write IMM only** (no READ, SEND, or ATOMIC) which suggests that it's optimized narrowly for the collective/parameter-exchange traffic pattern that dominates training, not general-purpose RDMA.
Developed jointly by NVIDIA, AMD, Broadcom, Intel, and Microsoft and published through OCP alongside Falcon, MRC is already in production at OpenAI and Microsoft, running on ConnectX-8, AMD Pollara/Vulcano, and Broadcom Thor Ultra NICs, with SRv6 switch support from NVIDIA Spectrum-4/5 and Broadcom Tomahawk 5.

## MetaRoCE (Meta)

MetaRoCE is Meta's RDMA transport protocol purpose-built for AI workloads on commodity Ethernet.
Its guiding principle, "the fabric sees packets, but the NIC sees intent", places control in the endpoint NIC rather than the network, enabling autonomous per-path decisions without switch-level intelligence.

Like UET, MetaRoCE treats out-of-order delivery as the normal case and dispenses with PFC entirely.
Each connection maintains multiple first-class paths, each with a distinct UDP source port, and the NIC shifts traffic dynamically away from congested or failing links without stalling the connection.
Loss recovery uses 256-bit selective acknowledgment bitvectors per path, triggering targeted retransmission of only the missing packets on the affected path.
Congestion control combines sender-driven ECN-based AIMD with receiver-driven fair-share rate hints, giving faster convergence during incast than a purely sender-side scheme.

MetaRoCE maintains compatibility with existing RDMA Verbs APIs, so existing software stacks (NCCL, RCCL) work without modification; advanced features like multiplane support are exposed through extensions.
The initial implementation runs on AMD Pensando programmable NICs; testing on a 64-node AMD GPU cluster showed ~86% throughput at 1% packet loss, outperforming RoCEv2 under the same conditions.
Meta plans to release the specification, a DPDK-optimized software reference implementation (libsoftmetaroce), and a compliance framework through OCP at the 2026 OCP Global Summit.

## The Shape of Convergence

These aren't really four competing bets so much as four points on a spectrum of how much you're willing to break from RC semantics:

| | Falcon | UET | MRC | MetaRoCE |
| --- | --- | --- | --- | --- |
| Base model | New HW transport, RC-inspired | Ground-up redesign | Minimal RC extension | Clean-sheet, NIC-driven |
| Ordering | Explicit ordered/unordered | Reordering-tolerant by design | RC-compatible, reordering via SACK bitmap | Reordering-tolerant by design |
| Loss + CC | SACK + NACK, delay-based CC | Trim + NACK, NSCC | SACK + NACK, NSCC | SACK bitvectors, ECN AIMD + receiver hints |
| API surface | IB Verbs-compatible | New semantic layer | libibverbs-modeled, Write/Write-IMM only | Verbs-compatible, extensions for multiplane |
| Status | OCP, Google, Intel E2000 series | UEC spec | OCP, in production (OpenAI, Microsoft) | OCP (planned, 2026 Summit); AMD Pensando |

Several convergence signals stand out across the four protocols.
NSCC-style congestion control appears in both UET and MRC.
MetaRoCE and UET independently converge on the same multipath primitive — per-path UDP source port entropy — without switch coordination.
And Falcon, MRC, and MetaRoCE all maintain Verbs-compatible API surfaces, suggesting the industry is unwilling to break existing RDMA software regardless of how much the transport changes underneath.
UET remains the outlier on API surface (its new semantic layer is the furthest break from existing RDMA software), but its adoption of NSCC aligns it with MRC on congestion control, which is the more fundamental convergence signal.
For anyone architecting IPU or NIC support today, the practical implication is that congestion control and multipath handling are becoming pluggable engine concerns rather than fixed-function silicon, a direction all four protocols point toward.

*Sources: [OCP Falcon Transport Protocol v1.1](https://www.opencompute.org/documents/ocp-specification-falcon-transport-protocol-v1-1-0-pdf), [RDMA over Falcon v1.1](https://www.opencompute.org/documents/rdma-over-falcon-spec-v1-1-pdf), [OCP MRC v1.0](https://www.opencompute.org/documents/ocp-mrc-1-0-pdf), [Google Cloud Falcon announcement](https://cloud.google.com/blog/topics/systems/introducing-falcon-a-reliable-low-latency-hardware-transport), [Google Next '26 AI infrastructure update](https://cloud.google.com/blog/products/compute/ai-infrastructure-at-next26), [UEC Specification 1.0](https://ultraethernet.org/ultra-ethernet-consortium-uec-launches-specification-1-0-transforming-ethernet-for-ai-and-hpc-at-scale/), [The Multipath Reliable Connection (MRC) Transport (arXiv, 2026)](https://arxiv.org/abs/2606.18170), and [MetaRoCE: RDMA Transport for AI Ethernet (Meta Engineering, 2026)](https://engineering.fb.com/2026/08/24/networking-traffic/metaroce-rdma-transport-ai-ethernet/).*
