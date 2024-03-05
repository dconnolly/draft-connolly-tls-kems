---
title: "ML-KEM Post-Quantum Key Agreement for TLS 1.3"
abbrev: connolly-tls-mlkem-key-agreement
category: info

docname: draft-connolly-tls-mlkem-key-agreement-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
keyword:
 - mlkem
 - tls
 - post-quantum

area: "Security"
workgroup: "Transport Layer Security"
venue:
  group: "Transport Layer Security"
  type: "Working Group"
  mail: "tls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tls/"
  github: "dconnolly/draft-tls-mlkem-key-agreement"

author:
 -
    fullname: Deirdre Connolly
    organization: SandboxAQ
    email: durumcrustulum@gmail.com

normative:
  RFC9180:
  FIPS203: DOI.10.6028/NIST.FIPS.203

informative:
  CDM23:
    title: "Keeping Up with the KEMs: Stronger Security Notions for KEMs and automated analysis of KEM-based protocols"
    target: https://eprint.iacr.org/2023/1933.pdf
    date: 2023
    author:
      -
        ins: C. Cremers
        name: Cas Cremers
        org: CISPA Helmholtz Center for Information Security
      -
        ins: A. Dax
        name: Alexander Dax
        org: CISPA Helmholtz Center for Information Security
      -
        ins: N. Medinger
        name: Niklas Medinger
        org: CISPA Helmholtz Center for Information Security

  DOWLING: DOI.10.1007/s00145-021-09384-1
  FO: DOI.10.1007/s00145-011-9114-1
  HHK: DOI.10.1007/978-3-319-70500-2_12
  HPKE: RFC9180
  hybrid: I-D.ietf-tls-hybrid-design
  tlsiana: I-D.ietf-tls-rfc8447bis

--- abstract

This memo defines ML-KEM-768 and ML-KEM-1024 as a standalone
`NamedGroup` for use in TLS 1.3 to achieve post-quantum key agreement.

--- middle

# Introduction

## Motivation

FIPS 203 standard (ML-KEM) is a new FIPS / CNSA 2.0 standard for
post-quantum key agreement via lattice-based key establishment mechanism
(KEM). Having a fully post-quantum (not hybrid) FIPS-compliant key
agreement option for TLS 1.3 is necessary for eventual movement beyond
hybrids and for users that need to be fully post-quantum sooner than later.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Key encapsulation mechanisms {#kems}

This document models key agreement as key encapsulation mechanisms
(KEMs), which consist of three algorithms:

- `KeyGen() -> (pk, sk)`: A probabilistic key generation algorithm,
  which generates a public encapsulation key `pk` and a secret
  decapsulation key `sk`.
- `Encaps(pk) -> (ct, shared_secret)`: A probabilistic encapsulation
  algorithm, which takes as input a public encapsulation key `pk` and
  outputs a ciphertext `ct` and shared secret `shared_secret`.
- `Decaps(sk, ct) -> shared_secret`: A decapsulation algorithm, which takes as
  input a secret decapsulation key `sk` and ciphertext `ct` and outputs
  a shared secret `shared_secret`.

ML-KEM-768 and ML-KEM-1024 conform to this API:

- ML-KEM-768 has encapsulation keys of size 1184 bytes, decapsulation
  keys of 2400 bytes, ciphertext size of 1088 bytes, and shared secrets
  of size 32 bytes

- ML-KEM-1024 has encapsulation keys of size 1568 bytes, decapsulation
  keys of 3168 bytes, ciphertext size of 1568 bytes, and shared secrets
  of size 32 bytes

# Construction {#construction}

We align with {{hybrid}} except that instead of joining ECDH options
with a KEM, we just have the KEM as a `NamedGroup`.

## Negotiation {#negotiation}

Each method is its own solely post-quantum key agreement method, which
are assigned their own identifiers, registered by IANA in the TLS
Supported Groups registry:

~~~
    enum {

         ...,

          /* ML-KEM Key Agreement Methods */
          mlkem768(0x0768),
          mlkem1024(0x1024)

         ...,

    } NamedGroup;
~~~

## Transmitting encapsulation keys and ciphertexts {#construction-transmitting}

The encapsulation key and ciphertext values are directly encoded with
fixed lengths as in {{FIPS203}}; the representation and length of
elements MUST be fixed once the algorithm is fixed.

In TLS 1.3 a KEM encapsulation key or KEM ciphertext is
represented as a `KeyShareEntry`:

~~~
    struct {
        NamedGroup group;
        opaque key_exchange<1..2^16-1>;
    } KeyShareEntry;
~~~

These are transmitted in the `extension_data` fields of
`KeyShareClientHello` and `KeyShareServerHello` extensions:

~~~
    struct {
        KeyShareEntry client_shares<0..2^16-1>;
    } KeyShareClientHello;

    struct {
        KeyShareEntry server_share;
    } KeyShareServerHello;
~~~

The client's shares are listed in descending order of client preference;
the server selects one algorithm and sends its corresponding share.

For the client's share, the `key_exchange` value contains the `pk`
output of the corresponding ML-KEM `NamedGroup`'s `KeyGen` algorithm.

For the server's share, the `key_exchange` value contains the `ct`
output of the corresponding ML-KEM `NamedGroup`'s `Encaps` algorithm.


## Shared secret calculation {#construction-shared-secret}

The shared secret output from the ML-KEM `Encaps` and `Decaps`
algorithms over the appropriate keypair and ciphertext results in the
same shared secret `shared_secret`, which is inserted into the TLS 1.3
key schedule in place of the (EC)DHE shared secret, as shown in
{{fig-key-schedule}}.

~~~~
                                    0
                                    |
                                    v
                      PSK ->  HKDF-Extract = Early Secret
                                    |
                                    +-----> Derive-Secret(...)
                                    +-----> Derive-Secret(...)
                                    +-----> Derive-Secret(...)
                                    |
                                    v
                              Derive-Secret(., "derived", "")
                                    |
                                    v
             shared_secret -> HKDF-Extract = Handshake Secret
             ^^^^^^^^^^^^^          |
                                    +-----> Derive-Secret(...)
                                    +-----> Derive-Secret(...)
                                    |
                                    v
                              Derive-Secret(., "derived", "")
                                    |
                                    v
                         0 -> HKDF-Extract = Master Secret
                                    |
                                    +-----> Derive-Secret(...)
                                    +-----> Derive-Secret(...)
                                    +-----> Derive-Secret(...)
                                    +-----> Derive-Secret(...)
~~~~
{: #fig-key-schedule title="Key schedule for key agreement"}

# Discussion {#discussion}

**Larger encapsulation keys and/or ciphertexts**  The `HybridKeyExchange`
struct in {{construction-transmitting}} limits public keys and
ciphertexts to 2^16-1 bytes; this is bounded by the same (2^16-1)-byte
limit on the `key_exchange` field in the `KeyShareEntry` struct. All
defined parameter sets for ML-KEM have encapsulation keys and
ciphertexts that fall within the TLS constraints.

**Failures**  Some post-quantum key exchange algorithms, including
ML-KEM, have non-zero probability of failure, meaning two honest parties
may derive different shared secrets.  This would cause a handshake
failure. ML-KEM has a cryptographically small failure rate; implementers
should be aware of the potential of handshake failure. Clients can retry
if a failure is encountered.

# Security Considerations

**IND-CCA** The main security property for KEMs is indistinguishability under
adaptive chosen ciphertext attack (IND-CCA2), which means that shared
secret values should be indistinguishable from random strings even given
the ability to have other arbitrary ciphertexts decapsulated.  IND-CCA2
corresponds to security against an active attacker, and the public key /
secret key pair can be treated as a long-term key or reused.  A common
design pattern for obtaining security under key reuse is to apply the
Fujisaki-Okamoto (FO) transform {{FO}} or a variant thereof {{HHK}}.

Key exchange in TLS 1.3 is phrased in terms of Diffie-Hellman key
exchange in a group.  DH key exchange can be modeled as a KEM, with
`KeyGen` corresponding to selecting an exponent `x` as the secret key
and computing the public key `g^x`; encapsulation corresponding to
selecting an exponent `y`, computing the ciphertext `g^y` and the shared
secret `g^(xy)`, and decapsulation as computing the shared secret
`g^(xy)`. See {{HPKE}} for more details of such Diffie-Hellman-based key
encapsulation mechanisms. Diffie-Hellman key exchange, when viewed as a
KEM, does not formally satisfy IND-CCA2 security, but is still safe to
use for ephemeral key exchange in TLS 1.3, see e.g. {{DOWLING}}.

TLS 1.3 does not require that ephemeral public keys be used only in a
single key exchange session; some implementations may reuse them, at the
cost of limited forward secrecy.  As a result, any KEM used in the
manner described in this document MUST explicitly be designed to be
secure in the event that the public key is reused.  Finite-field and
elliptic-curve Diffie-Hellman key exchange methods used in TLS 1.3
satisfy this criteria.  For generic KEMs, this means satisfying IND-CCA2
security or having a transform like the Fujisaki-Okamoto transform
{{FO}} {{HHK}} applied.  While it is recommended that implementations
avoid reuse of KEM public keys, implementations that do reuse KEM public
keys MUST ensure that the number of reuses of a KEM public key abides by
any bounds in the specification of the KEM or subsequent security
analyses.  Implementations MUST NOT reuse randomness in the generation
of KEM ciphertexts.

**Binding properties** TLS 1.3's key schedule commits to the the ML-KEM
encapsulation key and the encapsulated shared secret ciphertext as the
`key_exchange` field as part of the `key_share` extension are populated
with those values are included as part of the handshake messages,
providing resilience against re-encapsulation attacks against KEMs used
for key agreement.

ML-KEM is MAL-BIND-K-PK-secure but only LEAK-BIND-K-CT and
LEAK-BIND-K,PK-CT-secure, but because of the inclusion of the ML-KEM
ciphertext in the TLS 1.3 key schedule there is no concern of malicious
tampering (MAL) adversaries, not just honestly-generated but leaked key
pairs (LEAK adversaries). The same is true of other KEMs with weaker
binding properties, even if they were to have more constraints for
secure use in contexts outside of TLS 1.3 handshake key agreement.These
computational binding properties for KEMs were formalized in {{CDM23}}.

# IANA Considerations

This document requests/registers two new entries to the TLS Named Group
(or Supported Group) registry, according to the procedures in {{Section
6 of tlsiana}}.

 Value:
 : 0x0768 (please)

 Description:
 : MLKEM768

 DTLS-OK:
 : Y

 Recommended:
 : N

 Reference:
 : This document

 Comment:
 : FIPS 203 version of ML-KEM-768


 Value:
 : 0x1024 (please)

 Description:
 : MLKEM1024

 DTLS-OK:
 : Y

 Recommended:
 : N

 Reference:
 : This document

 Comment:
 : FIPS 203 version of ML-KEM-1024


--- back

# Acknowledgments
{:numbered="false"}

Thanks to Douglas Stebila for consultation on the
draft-ietf-tls-hybrid-design design.
