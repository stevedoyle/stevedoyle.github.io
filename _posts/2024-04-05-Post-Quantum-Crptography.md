# Post Quantum Cryptography

There is a lot of hype in the industry around post quantum cryptography (PQC) and the threat that quantum computers bring to classical cryptography, in particular for cryptography based on public and private keys, including digital signatures and key exchanges used in most networking protocols including TLS, QUIC, IPsec and DNSSEC.

The [Cloudflare Blog](https://blog.cloudflare.com/) post on [The state of the post-quantum Internet](https://blog.cloudflare.com/pq-2024) provides and excellent snapshot of the state of PQC in the Internet in 2024, including the history of PQC and the challenges that faces widespread adoption of PQC.

In the very near future, as PQC support is rolled out in network equipment, a hybrid of traditional and PQ cryptography will be used. This caters for hte immaturity of the PQC algorithm standards and ensures that communications are secure as long as one of either the traditional algorithm or the PQ algorithm remains secure.

As in the case of TLS 1.3 adoption, one big challenge for the widespread adoption of PQC is the interoperability with legacy networking equipment which does not support PQC and makes certain hardcoded assumptions around the protocol message formats. This ossification needs to be factored into the updates made to existing network security protocols as they are updated to accomodate the use of hybrid key exchanges and hybrid signatures and to accomodate the increased size of the PQC algorithm keys.
