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
    fullname: Nils Warnke
    organization: Deutsche Telekom
    email: nils.warnke@telekom.de
 -
    fullname: Martin Horneffer
    organization: Deutsche Telekom
    email: Martin.Horneffer@telekom.de
 -
    fullname: Nicklous Morris
    organization: VerizonWireless
    email: nicklous.morris@verizonwireless.com
 -
    fullname: Clayton Hassan
    organization: Bell Canada
    email: clayton.hassan@bell.ca
 -
   fullname: Daniel Voyer
   organization: Cisco Systems
   email: davoyer@cisco.com

normative:

informative:

...

--- abstract

   This document provides operational guidelines for designing Segment
   Routing over IPv6 (SRv6) addressing plans in Internet‑service‑provider
   backbones.  It specifies a compressed Segment Identifier (C‑SID)
   locator structure (F3216) that encodes Flex‑Algo, Region‑ID, Site‑ID
   and Node‑ID inside a fixed /48 locator while leaving space for Local
   and Global C‑SIDs. The aim is to accelerate brown‑field SRv6 migrations
   by offering deterministic field carving, summarization rules and
   actionable operational guidance.


--- middle

# Introduction

  IPv6 offers a vast 128‑bit space, allowing operators to build
   **hierarchical address plans** that align with their physical and
   operational topology.  When each byte has a clear meaning—Domain‑ID,
   Region, Flex‑Algo, Site‑Set, Node—summaries fall naturally at region
   borders and troubleshooting becomes visual.

   A well‑designed SRv6 addressing plan underpins summarization, fast
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
   *5f00::/16* for operators willing to deploy new strict at their border
   and edge ACLs ROA is also a viable option.

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
   * **Site-seT (ST)**:    256 C‑SIDs (one /40) sizing a site; even = GIB, odd = LIB
   * GIB:             Global‑ID Block — even‑indexed Sets carrying global C‑SIDs
   * LIB:             Local‑ID Block  — odd‑indexed Sets filtered at Region edge
   * SRv6 Site‑ID:    Byte‑sized identifier for a POP cluster


# Design Goals and Requirements

   The five goals below justify every bit choice in the locator.  They
   translate generic IPv6 abundance into a practical, byte‑aligned
   hierarchy that scales smoothly from small POPs to continental C‑SID
   domains.

   * G1: – Scalable Aggregation* ≤ 3 new routes in the core after any
   failure.
   * G2: – Per‑Node /48 Locator* Preserves /64 for services and /96 for
   locals while capping FIB size.
   * G3: – Flex‑Algo Plane Parity* Bit positions identical across planes.
   * G4: – Global vs Local Split* Even Sets = Global C‑SIDs, odd Sets =
   Local C‑SIDs.


# CSID Address‑Encoding Framework (F3216)

   This section converts theory into byte boundaries, showing exactly
   **how** IPv6 bit‑richness is partitioned into a multi‑level hierarchy
   (Global‑ID → Region → Flex‑Algo → Site‑seT → Node).

   The 128‑bit SRv6 locator is split into a 64‑bit *network* part and a
   64‑bit *host* part.  The network part encodes Domain‑ID, Region‑ID,
   Flex‑Algo, Set and Node‑ID in byte‑aligned fields.

## Locator Structure and Field Encoding (ULA-model)

   The diagram below shows, byte‑by‑byte, how an SRv6 locator encodes the
   hierarchy described in Section 4.  Reading the hexadecimal address
   reveals the Domain, Region, Flex‑Algo, Set and Node values without a
   lookup table.

Nibbles (4 bits)

    0             15             31             47          63
    |             |              |              |           |
    +-------+-------+-------+-------+-------+-------+----------+
    |  fd   |   D   |  Reg  |   FA  |   ST  |   NN  | host‑64  |
    +-------+-------+-------+-------+-------+-------+----------+
         <-----------  network / locator (48 bits) ---------->|

Field | Bits | Purpose
------|------|---------------------------------------------------------
fd    | 0‑7  | **ULA prefix keeps locators private**
D     | 8‑15 | **Domain‑ID - up to 256 autonomous domains** 
Reg   |16‑23 | **Region‑ID - groups Sites that share latency goals** 
FA    |24‑31 | **Flex‑Algo identifier**
ST    |32‑39 | **Site-seT - Site identifer per PoP**
NN    |40‑47 | **Node‑ID** - uniquely itentifies nodes(/48 locator)**

   *Smaller networks:* Operators that do not require multiple domains or
   regional partitions can leave **Domain‑ID** and/or **Region‑ID** set
   to `0x00`.  The byte‑aligned masks still summarise cleanly, and all
   downstream fields remain valid.

## Locator Structure for RFC 9602 (5f00::/16-model)

RFC 9602 reserves 5f00::/16 for SRv6 operator locators.  With the first
two bytes fixed (5f 00), the hierarchy shifts half a byte to the right:
Domain‑ID and Region‑ID now share the third byte, while the downstream
fields remain byte‑aligned.

    0              15              31              47         63
    |              |               |               |          |
    +--------+--------+--------+--------+--------+--------+----------+
    |  5F    |  00    |  D  R  |   FA   |   ST   |   NN   | host‑64  |
    +--------+--------+--------+--------+--------+--------+----------+
                            ^  ^                                   ^
                            |  |                                   |
                            |  +-- Region‑ID (4 bits)              |
                            +----- Domain‑ID (4 bits)              |
    <------------------------ network / locator (48 bits) ----->---+

Field |  Bits | Purpose
----- | -----  ---------------------------------------------------
5F00  | 0‑15  | Fixed block reserved by RFC9602 for SRv6 locators
D     | 16‑19 | Domain‑ID — up to 15 SRv6 domains (0x1…0xF)
R     | 20‑23 | Region‑ID — up to 15 regions per domain (0x1…0xF)
FA    | 24‑31 | IGP Flexible‑Algorithm identifier
ST    | 32‑39 | Site‑seT; even = GIB, odd = LIB
NN    | 40‑47 | Node‑ID — uniquely identifies one router (/48)

Planning tip: single‑domain or flat‑region deployments may set D and/or
R to 0x0; masks still summarise cleanly. ISPs can subdivide further—for
example using the lower nibble of Flex‑Algo as a sub‑region key—while
keeping summaries on nibble boundaries.

# Deployment Profiles

   The Site-seT (ST) byte lets planners scale by powers of 256: **one even Set
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
   • Sets 0x0A00–0x0a10 → fd00:0500:0a00::/39
   • **Summary injected into L2:** fd00:0500:0a00::/39

###  Large Site – 890 routers
   • Sets 0x1000–0x1030 → fd00:0500:1000::/38
   • **Summary injected into L2:** fd00:0500:1000::/38

# Summarisation Best Practices

   IPv6 gives ample room to carve hierarchical blocks, but the hierarchy
   is only useful if *each layer advertises exactly one summary into the
   layer above*.  The guiding rule is **summarize at every 8‑bit field
   boundary** so the mask aligns to a full byte.

Topology Scope | Field ask | Prefix
---------------|-----------|--------
Core AS	      |  /32	   | Reg+FA
Region POPs    |  /38-/40  | Set range
Single node.   |  /48      | Node (NN)

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
   use the *medium‑site* case from Section 5.1:

     • Region‑ID 0x05, Flex‑Algo 0
     • Set 0x0a10, Node 0x13b
     • locator0 → **fd00:0500:0a13::/48**
     • L2 summary → **fd00:0500:0a00::/39**

## Loopback Address and NET-ID/Router-ID Derivation

   An IPv6-only backbone still requires legacy 32-bit identifiers for IS-IS
   NET-ID and BGP router-ID. Deriving these directly from the hierarchical
   locator eliminates manual spreadsheets and guarantees consistency.

 * Example (Medium Site from Section 5.1.2):
     * Region-ID = 0x05, Flex-Algo = 0x00
     * ST = 0x0a, Node-ID = 0x13a (hex)
     * locator0 = fd00:0500:0a13::/48
     * L1 summary = fd00:0500:0a00::/39

* Loopback:
     * Rule: Loopback = locator0 + ::1/128 from locator0.
     * Example: fd00:0500:0a13::1/128
     * Because this resides inside the /48 advertised by IS-IS, no additional
       loopback prefix is needed (reduces FIB noise)

* IS-IS NET-ID:
     * Format: 49.Area-ID.System-ID.00
     * Area-ID: Operator-defined (e.g., area 10)
     * System-ID: Last three bytes of locator0, nibble-grouped:
       locator0 (0a13:0B00:0000) → System-ID = 0a13.0B00.00
     * NET-ID = 49.10.0a13.0B00.00

* Router-ID (BGP & IS-IS):
     * Method: Use lower 32 bits of loopback, rendered in dotted-decimal.
     * Loopback low-32 bits = 0x00000a13 → 0.0.10.19
     * BGP Router-ID = 0.0.10.19
     * IS-IS System-ID (48-bit alt) = 0000.0a13.0000

 Note: Mismatches between locator0, loopback, and router-ID indicate
   configuration typos and aid in audits.


# Security Considerations

The hierarchical IP addressing plan proposed here offers natural ACL
boundaries—and operators should leverage them to minimize security risk:

 * Byte-Aligned Summaries as ACL Anchors

      Because locators and summaries align on full-byte boundaries (e.g.,
      /32 for Region, /48 for Site), ACLs can match on these exact prefixes.
      For example, an ACL permitting fd00:0500::/32 automatically allows all
      Site- and Node-level prefixes under Region 05 but denies anything
      outside. Maintaining ACLs at each Region border is therefore
      straightforward and less error-prone than arbitrary masks.

 * Locator->Loopback Consistency Checks

      Loopbacks derive directly from the /48 locator, e.g., fd00:0500:0a13::1.
      If an operator misassings a locator, a simple script can detect mismatches
      between the advertised /48 and the loopback's low-order 32-bit value. This
      check helps prevent accidental duplicate identifiers that could be exploited
      for spoofing.

 * Logging and Periodic Audits

      Log locator assignments and summary changes at each aggregation level.
      Periodically audit ACLs to ensure they align with the current hierarchy (e.g.,
      confirm that all allowed /48s fall under their parent /32). This catch-all
      approach helps identify any unauthorized or stale entries before they
      compromise security.

By combining byte-aligned summarization with simple ACL rules at Region or
Site boundaries, operators can enforce strict filtering with minimal complexity,
leveraging the hierarchical plan to make ACL configuration both robust and
scalable.


# IANA Considerations

TBD

--- back

# Acknowledgments
TBD


