# Distributed Entropy Sourcing

## Abstract

This document describes a means of sourcing high-entropy data from multiple remote sources, akin to [Entropy-as-a-Service](https://csrc.nist.gov/Projects/Entropy-as-a-Service).  The protocol described herein assumes the majority of sources "queried" can generate high-entropy random data, but carries protections against the scenario that one or more do not in reality.  Potential applications for this are networked compute resources where there is no reliable means of local generation of sufficiently random data.  "Trust" in the degree of entropy is via statistical probability, the more discrete sources of entropy queried (and redundantly so), the higher the likelihood of "good-quality randomness".

A proof-of-concept Python implementation is included in this repository.

## Overview

The "entropy client" contacts multiple remote "entropy servers", requesting random data.  These servers will respond with random data which will be combined to form a single (and overall smaller) random sample.

Network communications are not secured by this protocol (TLS etc. should be used in real-world applications).  Protection against low-entropy (or compromised) hosts is provided by munging the received data.

## A worked example

Alice desires 2048 bits of high-entropy data.

Alice requests 256 bit "entropy chunks" from 5 remote servers: Bob, George, Dan, Tim, and Jon (2048 + 256 bits, or `N+1` in "entropy chunk" terms).

Bob, George, Dan, Tim, and Jon all return Alice an entropy chunk (256 bits of random data) each.

Alice secretly salts and hashes every unique permutation of `N-1` responses to reduce the dataset to her desired 4-chunk length, without allowing any one entropy chunk to unfairly sway the result:

```
  d1 = SHA256(secretSalt + Bob  + Grge + Dan + Tim)
  d2 = SHA256(secretSalt + Bob  + Grge + Dan + Jon)
  d3 = SHA256(secretSalt + Bob  + Grge + Tim + Jon)
  d4 = SHA256(secretSalt + Bob  + Dan  + Tim + Jon)
  d5 = SHA256(secretSalt + Grge + Dan  + Tim + Jon)

  c1 = SHA256(d1[0..255]   + d2[0..63])
  c2 = SHA256(d2[64..255]  + d3[0..127])
  c3 = SHA256(d3[128..255] + d4[0..191])
  c4 = SHA256(d4[192..255] + d5[0.255])

  output = c1 + c2 + c3 + c4
```

The secret salt ensures that even if every remote host was unexpectedly low entropy or otherwise compromised, the resulting data secretly derived by Alice would remain unknown to them given a very small sample of derived data.  Provided the issue was promptly addressed, no ill effect should occur (but Alice should be informed to generate a new secret salt).

## Entropy server protections

The servers generating random data should salt and hash their responses to protect against any client collecting enough example responses to detect patterns if entropy is lacking:

```
  entropyChunk = SHA256(salt + min256BitLongRandomData)
```

The salt may simply be the IP address and port number of the client.

## Increasing "entropic redundancy"

In the example above, Alice used an `N+1` entropy chunk redundancy.  This can be increased, using more entropy servers (e.g.: `2N`, called `e1` through `e8`):

```
  d1 = SHA256(secretSalt + e1 + e2 + e3 + e4 + e5 + e6 + e7)
  d2 = SHA256(secretSalt + e1 + e2 + e3 + e4 + e5 + e6 + e8)
  d3 = SHA256(secretSalt + e1 + e2 + e3 + e4 + e5 + e7 + e8)
  d4 = SHA256(secretSalt + e1 + e2 + e3 + e4 + e6 + e7 + e8)
  d5 = SHA256(secretSalt + e1 + e2 + e3 + e5 + e6 + e7 + e8)
  d6 = SHA256(secretSalt + e1 + e2 + e4 + e5 + e6 + e7 + e8)
  d7 = SHA256(secretSalt + e1 + e3 + e4 + e5 + e6 + e7 + e8)
  d8 = SHA256(secretSalt + e2 + e3 + e4 + e5 + e6 + e7 + e8)

  c1 = SHA256(d1 + d2)
  c2 = SHA256(d3 + d4)
  c3 = SHA256(d5 + d6)
  c4 = SHA256(d7 + d8)

  output = c1 + c2 + c3 + c4
```
