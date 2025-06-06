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



# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
