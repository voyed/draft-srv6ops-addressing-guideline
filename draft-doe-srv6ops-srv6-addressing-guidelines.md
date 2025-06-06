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
   fd    | 0‑7  | **ULA prefix keeps locators private**
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

# Deployment Profiles

   The Set (SS) byte lets planners scale by powers of 256: **one even Set
   supports up to 256 node locators**.  Sizing logic:

     • Small POP (≤256 nodes): 1 even Set  → summary /40
     • Medium POP (257‑512 nodes): 2 even Sets → summary /39
     • Large POP (513‑1024 nodes): 4 even Sets → summary /38

   The examples below show the Sets allocated and the summary route that
   each Region injects into IS‑IS Level 2.

##  Worked Examples (Region‑ID 0x05, Flex‑Algo 0)

###  Small Site – 110 routers
   • Set 0x0600 → fd00:0500:0600::/40
   • **Summary injected into L2:** fd00:0500:0600::/40

###  Medium Site – 315 routers
   • Sets 0x0A00–0x0A10 → fd00:0500:0a00::/39
   • **Summary injected into L2:** fd00:0500:0a00::/39

###  Large Site – 890 routers
   • Sets 0x1000–0x1030 → fd00:0500:1000::/38
   • **Summary injected into L2:** fd00:0500:1000::/38

# Ultra-Scale ISP Deployment with End.LBS

   When the node count is beyond 50k, a single Global‑ID Block (GIB)
   may no longer be enough.  ultra-scale Operators with 300k nodes can
   allocate **one /32 Locator‑Block per Region**, then use the End.LBS
   behaviour to swap blocks at the border router.
   The SRH carries only *one* extra SID (the End.LBS C‑SID) regardless of hop‑count.

   The following topology provides an example (hex abbreviated):

```
  West Region (fd20::/32)                East Region (fd21::/32)

   fd20:..:0600::       fd20:..:0601::                fd21:..:0a00::
        +----+              +----+                       +----+
        |WP1|-------------->|WP2|------------------------>|EP1|
        +----+  C-SID_W1    +----+  C-SID_W2             +----+
             \                                    /
              \                                  /
               \                                /
                +------------------------------+
                |  End.LBS Border Router (BR)  |
                +------------------------------+
                           C-SID fd20:..:00ff::
```
Legend:  WP1, WP2 = West‑region provider routers EP1 = East‑region provider router


   **Segment List example** (SRH left→right):

     `[C‑SID_W1, C‑SID_W2, fd20:..:00ff:: (End.LBS), C‑SID_E3]`

   * At hop 3 the border router executes **End.LBS**: it rewrites the
     destination address from `fd20:xxxx:yyzz::/32` to the equivalent
     `fd21:xxxx:yyzz::/32` while leaving the C‑SID suffix unchanged.
   * No additional SID is needed per hop; header growth is constant.
   * Each Region advertises only its own /32 into the core, keeping FIB
     size flat even at continental scale.

# Summarisation Best Practices

   IPv6 gives ample room to carve hierarchical blocks, but the hierarchy
   is only useful if *each layer advertises exactly one summary into the
   layer above*.  The guiding rule is **summarise at every 8‑bit field
   boundary** so the mask aligns to a full byte.

   ┌─────────────┬────────────┬────────────┐
   │ Topology    │ Field mask │ Prefix     │
   ├─────────────┼────────────┼────────────┤
   │ Core AS     │  /32       │ Reg+FA     │
   │ Region POPs │  /38–/40   │ Set range  │
   │ Single node │  /48       │ Node (NN)  │
   └─────────────┴────────────┴────────────┘

   **How the summarization flow**

   ## **Region → Core (IS‑IS L2):** Every Region advertises one /32 per
      Flex‑Algo.  The prefix is derived by masking after the FA byte so
      all downstream Sets collapse naturally.  A 300 k‑node continental
      backbone therefore injects at most *3 × Regions* routes into the
      core.

   ## **Site → Region (IS‑IS L1):** Each Site advertises a summary that
      stops at the Set byte:  /40 if it needs 1 Set (Small), /39 if it
      needs 2 Sets (Medium), /38 if it needs 4 Sets (Large).  These
      prefixes never leave the Region.

   ## **Core → Region (downward):** The core leaks only the /32s back
      into all Regions.  Routers discard any packet whose destination is
      outside those blocks—protecting against accidental leaks.

   **Provider‑wide Aggregate.**  ASBRs SHOULD also originate a single
   prefix that covers the *entire* ISP address block (e.g., fd00::/20 or
   the equivalent GUA).  Advertising this grand summary into the core
   and to upstream transit providers adds a final safety net—any stray
   Internet route leak that re‑enters the network will be suppressed at
   Region borders because it can never be more specific than the /32s
   already authorised inside the domain.

   **TE exception.**  Operators may temporarily leak a specific /39 into
   the core for traffic‑engineering.  Automation MUST withdraw the
   prefix after use to prevent FIB bloat.

   **Error containment.**  Because the summary masks are byte‑aligned, a
   single mis‑originated /48 cannot cross a Region boundary—either the
   packet is caught by a default‑drop rule or it is outside the /32
   admitted by the core.

# Other Operational Considerations

   In an IPv6‑only backbone the control‑plane still needs several legacy
   32‑bit identifiers—IS‑IS NET‑ID, BGP router‑ID, IS‑IS System‑ID—that
   were traditionally filled with an IPv4 loopback.  Deriving these
   values **directly from the hierarchical locator0** eliminates manual
   spreadsheets and guarantees internal consistency.  The examples below
   use the *medium‑site* case from Section 5.1.2:

     • Region‑ID 0x05, Flex‑Algo 0
     • Set 0x0A10, Node 0x13B
     • locator0 → **fd00:0500:0a13::/48**
     • L2 summary → **fd00:0500:0a00::/39**

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
