---
title: "PQC News - November 2024"
tags:
  - PQC
  - Cryptography
---

- NIST published [NIST IR 8547](https://csrc.nist.gov/pubs/ir/8547/ipd) "Transition to Post-Quantum Cryptography Standards", a report that outlines the proposed transition to PQC. It outlines a ten year timeline to migrate the world's cryptographic subsystems.
- Many of the [IETF 121](https://www.ietf.org/meeting/121/) working groups had agenda items related to PQC. It is good to see some of the [documents](https://datatracker.ietf.org/wg/pquip/documents/) from the Post Quantum Use In Protocols (pquip) working group getting close to publication, including [Terminology for Post-Quantum Traditional Hybrid Schemes](https://datatracker.ietf.org/doc/draft-ietf-pquip-pqt-hybrid-terminology/), [Post-Quantum Cryptography for Engineers](https://datatracker.ietf.org/doc/draft-ietf-pquip-pqc-engineers/) and [Hybrid Signature Spectrums](https://datatracker.ietf.org/doc/draft-ietf-pquip-hybrid-signature-spectrums/).
- NIST held a public meeting to outline [plans](https://csrc.nist.gov/Projects/stateful-hash-based-signatures/presentations) for updates to [SP 800-208](https://csrc.nist.gov/pubs/sp/800/208/final) "Recommendation for Stateful Hash-Based Signature Schemes" to address industry feedback on the restriction against private key export.
- Cloudflare published a blog post [looking at the latest post-quantum signature standardization candidates](https://blog.cloudflare.com/another-look-at-pq-signatures/) which included an [analysis of the impact of ML-DSA on TLS](https://blog.cloudflare.com/another-look-at-pq-signatures/#how-many-added-bytes-are-too-many-for-tls). Their analysis concluded that "For the majority of QUIC connections, using ML-DSA as a drop-in replacement for classical signatures would more than double the number of transmitted bytes over the lifetime of the connection." and again suggested an alternative path forward to use a KEM [instead of a signature](https://kemtls.org/) for handshake authentication.
