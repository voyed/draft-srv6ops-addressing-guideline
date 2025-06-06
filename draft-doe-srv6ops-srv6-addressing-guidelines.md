---
title: "SRv6 Addressing Guidelines"
abbrev: "SRv6 Addressing Partitions"
category: info

docname: draft-doe-srv6ops-srv6-addressing-guidelines-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "SRv6 Operations"
keyword:
 - SRv6 Addressing
 - IPv6 Partition
 - Guidelines
 - SRv6 Best practice Deployment
venue:
  group: "SRv6 Operations"
  type: "Working Group"
  mail: "srv6ops@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/srv6ops/"
  github: "voyed/draft-srv6ops-addressing-guideline"
  latest: "https://voyed.github.io/draft-srv6ops-addressing-guideline/draft-doe-srv6ops-srv6-addressing-guidelines.html"

author:
 -
    fullname: John Doe
    organization: Example
    email: john.doe@example.com

normative:

informative:

...

--- abstract

   This document provides operational guidelines for designing Segment
   Routing over IPv6 (SRv6) addressing plans in Internet‑service‑provider
   backbones of up to 50 000 nodes.  It specifies a compressed Segment
   Identifier (C‑SID) locator structure (F3216) that encodes Flex‑Algo,
   Region‑ID, Site‑ID and Node‑ID inside a fixed /48 locator while
   leaving space for Local and Global C‑SIDs.  An “ultra‑scale” profile
   (~300 000 nodes) using End.LBS locator‑block swap is also defined.
   The aim is to accelerate brown‑field SRv6 migrations by offering
   deterministic field carving, summarisation rules and actionable
   operational guidance.


--- middle

# Introduction

  IPv6 offers a vast 128‑bit space, allowing operators to build
   **hierarchical address plans** that align with their physical and
   operational topology.  When each byte has a clear meaning—Domain‑ID,
   Region, Flex‑Algo, Site‑Set, Node—summaries fall naturally at region
   borders and troubleshooting becomes visual.

   A well‑designed SRv6 addressing plan underpins summarisation, fast
   convergence, traffic‑engineering flexibility and security filtering.
   For operators migrating from legacy MPLS or flat IPv6 (*brown‑field*
   environments), renumbering entire domains is rarely feasible.  The
   scheme described here overlays gracefully on top of existing
   allocations: individual domain and regions or sites can migrate incrementally
   without a network‑wide flag‑day.

   Using **Unique Local Address (ULA)** space keeps infrastructure SIDs
   off the public Internet, limiting the blast‑radius of route leaks or
   spoofing.  A Global‑Unicast scheme is possible but demands airtight
   RPKI and strict ingress filters; it is therefore *NOT RECOMMENDED*
   unless strong business drivers outweigh the risk. RFC9602 specifies
   *5F00::/16* for operators willing to deploy new strict at their border and edge ACLs ROA is also a viable option.

   The **C‑SID** paradigm compresses 16‑bit Segment Identifiers inside a
   /48 locator—balancing header overhead with FIB scale while allowing
   deterministic summarisation.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

   This glossary maps each field or acronym to its role in the
   hierarchical IPv6 structure so readers can follow subsequent
   sections without ambiguity.


   * C‑SID:           16‑bit Compressed Segment Identifier
   * Locator‑Block:   Uppermost bits that identify an SRv6 domain
   * Flex‑Algo:       IGP Flexible Algorithm ID
   * Region‑ID:       8‑bit value (bits 17‑24) grouping sites
   * **Set (SS)**:    256 C‑SIDs (one /40) sizing a site; even = GIB, odd = LIB
   * GIB:             Global‑ID Block — even‑indexed Sets carrying global C‑SIDs
   * LIB:             Local‑ID Block  — odd‑indexed Sets filtered at Region edge
   * SRv6 Site‑ID:    Byte‑sized identifier for a POP cluster
   * End.LBS:         Endpoint behaviour performing Locator‑Block Swap

# Design Goals and Requirements

   The five goals below justify every bit choice in the locator.  They
   translate generic IPv6 abundance into a practical, byte‑aligned
   hierarchy that scales smoothly from small POPs to continental C‑SID
   domains.

   * *G1: – Scalable Aggregation* ≤ 3 new routes in the core after any
   failure.
   * *G2: – Per‑Node /48 Locator* Preserves /64 for services and /96 for
   locals while capping FIB size.
   * *G3: – Flex‑Algo Plane Parity* Bit positions identical across planes.
   * *G4: – Global vs Local Split* Even Sets = Global C‑SIDs, odd Sets =
   Local C‑SIDs.
   * *G5: – Seamless Domain Stitching* End.LBS swaps blocks without SRH
   growth.

# CSID Address‑Encoding Framework (F3216)

   This section converts theory into byte boundaries, showing exactly
   **how** IPv6 bit‑richness is partitioned into a multi‑level hierarchy
   (Global‑ID → Region → Flex‑Algo → Site‑Set → Node).

   The 128‑bit SRv6 locator is split into a 64‑bit *network* part and a
   64‑bit *host* part.  The network part encodes Domain‑ID, Region‑ID,
   Flex‑Algo, Set and Node‑ID in byte‑aligned fields.

## Locator Structure and Field Encoding (ULA-model)

   The diagram below shows, byte‑by‑byte, how an SRv6 locator encodes the
   hierarchy described in Section 4.  Reading the hexadecimal address
   reveals the Domain, Region, Flex‑Algo, Set and Node values without a
   lookup table.

```
Byte  0    1    2    3    4    5       6‑15
Bits 0‑7 8‑15 16‑23 24‑31 32‑39 40‑47   (host part)
     +---+---+----+----+----+----+----------------+
     |fd | D |Reg | FA | SS | NN |    host64      |
     +---+---+----+----+----+----+----------------+
```

   Field | Bits | Purpose
   ------|------|---------------------------------------------
   fd    | 0‑7  | ULA prefix keeps locators private
   D     | 8‑15 | **Domain‑ID** 
   Reg   |16‑23 | **Region‑ID** 
   FA    |24‑31 | **Flex‑Algo**
   SS    |32‑39 | **Set (SS)**
   NN    |40‑47 | **Node‑ID** (/48 locator)

   *Smaller networks:* Operators that do not require multiple domains or
   regional partitions can leave **Domain‑ID** and/or **Region‑ID** set
   to `0x00`.  The byte‑aligned masks still summarise cleanly, and all
   downstream fields remain valid.

## Locator Structure for RFC 9602 (5F00::/16-model)

   Under *5F00::/16*, the first 16 bits are fixed.  The locator therefore
   pushes the hierarchy one half‑byte to the right:

```
Nibbles (4 bits)                      host part
 0   1   2     3       4      5      6‑15
+----+----+-----+------+-------+----+----------------+
| 5F | 00 |  D  |  R  |  FA   | SS |  NN  |  host64  |
+----+----+-----+------+-------+----+----------------+
Bits 0‑15 fixed   16‑19 20‑23   24‑31 32‑39 40‑47
```

   * D  (Domain‑ID) – 4 bits → up to **15 domains** (0x1‑0xF)
   * R  (Region‑ID) – 4 bits → **15 regions** per domain (0x1‑0xF)
   * FA (Flex‑Algo) – full byte (24‑31)
   * SS / NN – identical meaning as in Section 4.1

   Because the first two bytes are locked (*5F00*), Domain and Region
   live in the third byte.  Flex‑Algo retains a full byte so existing
   deployments need no change.  ISPs can subdivide further—for example
   using the lower nibble of Flex‑Algo as a sub‑region key—while keeping
   summaries on nibble boundaries.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
