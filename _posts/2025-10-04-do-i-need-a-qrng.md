---
title: "Do I Need a QRNG?"
tags:
  - Security
  - Cryptography
  - PQC
toc: true
---

Organizations are rushing to invest in or mandate the use of quantum random number generators (QRNGs). India's [Telecommunication Engineering Centre](https://tec.gov.in/pdf/TR/final%20Technical%20Report%20on%20Q%20Secure%205G%20and%20beyond%205G%20core%2028-03-25.pdf) is integrating QRNGs into 5G infrastructure, and Samsung has released multiple QRNG-powered smartphones. However, as [Cloudflare explains](https://blog.cloudflare.com/you-dont-need-quantum-hardware/), this might be an unnecessary investment.

## The Key Insight

**Quantum computers do not enable any practical new attacks against classical TRNGs.** Both classical and quantum entropy sources can produce the unpredictable randomness needed for cryptography.

The critical requirement is a **cryptographically-secure pseudorandom number generator (CSPRNG)** with an unknown seed. Whether that seed comes from quantum or classical sources doesn't matter—what matters is the CSPRNG implementation and seed protection.

## The Practical Approach

Secure random number generation is straightforward:

1. Collect a small seed of true randomness (typically 256 bits)
2. Feed this seed into a CSPRNG
3. Generate unlimited pseudorandom output

This works equally well with classical entropy sources like thermal noise or chaotic physical systems.

## Where to Focus Instead

Rather than investing in quantum hardware for RNG, focus on:

- **Proper CSPRNG implementation** and algorithm selection
- **Seed protection** - keeping entropy sources uncompromised
- **Regular security audits** of your RNG implementation
- **Post-quantum cryptography migration** - where quantum threats are real

Classical TRNGs with proper CSPRNGs provide excellent security. Save your quantum budget for transitioning to quantum-resistant algorithms—that's where the actual quantum threat lies.
