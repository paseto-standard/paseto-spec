# Rationale for V5/V6

This document aims to capture the rationale for specifying new modes
(v5 to succeed v3, v6 to succeed v4) for PASETO.

## Primary Motivations for New Versions

The primary motivation for Version 5 and Version 6 is the threat of 
Cryptography-Relevant Quantum Computers (CRQCs), which can break the Elliptic
Curve Discrete Logarithm Problem, which completely breaks the security of RSA,
ECDSA, and EdDSA.

Therefore, we propose new PASETO versions for Post-Quantum Cryptography.

### v5.public

For v5.public, we opt for ML-DSA-87 (one of the [FIPS-204](https://csrc.nist.gov/pubs/fips/204/final)
parameter sets). This provides the most compact signature that targets the 256-bit
security level, and is compatible with [CNSA 2.0's parameter recommendations](https://media.defense.gov/2022/Sep/07/2003071834/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS_.PDF).

>  Use Level V parameters for all classification levels.

The main reason for this parameter selection is to choose a compact FIPS-compatible
signature algorithm that achieves post-quantum security. We choose ML-DSA because
it has an excellent security margin.

### v6.public

For v6.public, we opt for SLH-DSA-SHA256-128s.

The even-numbered protocol versions have historically built on non-NIST 
cryptography (e.g., Curve25519 and ChaCha20). Keeping with tradition (despite
all the available post-quantum algorithms being FIPS approved NIST standards),
we opt for SLH-DSA for version 6, which was developed by the same 
cryptographers that built Curve25519 and BLAKE2.

The primary rationale is to have a signature mode that isn't based on lattices
for the sake of diversity. If ML-DSA is desired, v5.public should be selected.

We opt for SHA256 for better performance in software and 128s for the most 
compact signatures permitted by the standard.

### v5.local / v6.local

No specific changes were needed from (v3.local, v4.local) respectively.
