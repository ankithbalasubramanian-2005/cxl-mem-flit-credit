# CXL.mem Transaction-Layer Design Specification

**Project:** M2S/S2M Flit Packing and Credit-Based Flow Control
**Source:** CXL Specification, Revision 2.0, Version 1.0 (Oct 26, 2020) — all section/table/figure numbers below refer to this document.

---

## 1. Flit Structure (§4.2.2)

The spec defines the CXL.cache/CXL.mem flit as a **fixed 528-bit (66-byte)** unit:

```
66 bytes = 2 bytes CRC + 4 slots × 16 bytes
```

| Slot | Role |
|---|---|
| Slot 0 | **Header Slot** — carries the flit header (routing/credit info) plus one protocol message, using a "H" format |
| Slots 1–3 | **Generic Slots** — each carries one or more messages or a 16B data chunk, using a "G" format |

**Note on "68B":** the industry-common name "68-byte flit" (used in your project brief and secondary sources) refers to the **wire transfer unit**: a 16-bit (2-byte) Protocol ID precedes each 528-bit flit at the physical layer (§4.2.9). **66B (flit) + 2B (Protocol ID) = 68B.** The flit's internal structure itself is 66 bytes; this spec uses "flit" to mean the 66-byte unit and calls out the Protocol ID separately.

### 1.1 Flit Header (Table 45)

Present in **every** flit (protocol or control), total 32 bits, packed into Slot 0:

| Field | Bits | Purpose |
|---|---|---|
| Type | 1 | 0 = Protocol Flit, 1 = Control Flit (Table 46) |
| Ak | 1 | Acknowledges 8 successfully received flits (link-layer retry) |
| BE | 1 | Byte-Enable-present flag for the data message in this flit |
| Sz | 1 | Data size: 1 = full 64B cacheline (4 chunks), 0 = 32B half (2 chunks) |
| ReqCrd | 4 | Request-channel credit return (exponential encoding) |
| DataCrd | 4 | Data-channel credit return |
| RspCrd | 4 | Response-channel credit return |
| Slot0–Slot3 | 3 each | Format-type selector for each slot (Table 50) |
| RSVD | 4 | Reserved |

### 1.2 Credit Return Encoding (Table 48)

`ReqCrd` / `DataCrd` / `RspCrd` are each 4 bits: **1 protocol-select bit + 3 magnitude bits**, magnitude encoded **exponentially**:

| Encoding | Credits |
|---|---|
| 000 | 0 |
| 001 | 1 |
| 010 | 2 |
| 011 | 4 |
| 100 | 8 |
| 101 | 16 |
| 110 | 32 |
| 111 | 64 |

Channel mapping (Table 49, CXL.mem downstream/upstream relevant rows):

| Field | Direction | Channel |
|---|---|---|
| ReqCrd | Downstream (host→device) | M2S Request |
| DataCrd | Downstream | M2S Request with Data (RwD) |
| DataCrd | Upstream (device→host) | S2M Response with Data (DRS) |
| RspCrd | Upstream | S2M No-Data Response (NDR) |

**Key point:** credit return is **continuous and piggybacked** on the header of every flit already being transmitted — it is not a separate message under normal traffic.

**Important — direction of the grant vs. direction of the channel:** Table 49's "Link Direction" column labels each channel's own *native* data-flow direction (e.g. M2S Request is inherently a "Downstream" channel). It does **not** indicate which physical direction the *credit-return* field travels. Per §4.2.7: *"It is the responsibility of the Rx to transmit credits to the sender using standard credit return mechanisms."* The receiver of a channel is the one who returns credit for it, on flits **it** transmits — the opposite physical direction from the data itself:

| Channel | Data flows | Credit returned by | Credit-return flit direction |
|---|---|---|---|
| M2S Request | Downstream (Host→Device) | Device | Upstream |
| M2S RwD | Downstream | Device | Upstream |
| S2M NDR | Upstream (Device→Host) | Host | Downstream |
| S2M DRS | Upstream | Host | Downstream |

---

### 1.3 Data Chunk Rollover Rules (§4.2.5)

**Rollover** is defined as any time a data transfer needs more than one flit to complete. A 128-bit (16B) data chunk (format G0) can only be scheduled in Slots 1, 2, or 3 — never Slot 0, since Slot 0 has only 96 bits available after the 32-bit flit header. These exact rules govern how rollover chunks are packed into the *next* flit:

| Rollover chunks remaining | Rule |
|---|---|
| > 3 | The next flit **must** be an all-data flit (all 4 slots are data chunks). |
| = 3 | Slots 1, 2, and 3 **must** contain the 3 rollover chunks. Slot 0 is packed independently (may hold the Data Header for the *next* transfer). |
| = 2 | Slots 1 and 2 **must** contain the 2 rollover chunks. Slot 0 and Slot 3 are packed independently. |
| = 1 | Slot 1 **must** contain the rollover chunk. Slots 0, 2, and 3 are packed independently. |
| 0 | Each of the 4 slots is packed independently — no constraint. |

This matters directly for the RTL flit packer: a naive implementation that treats "3 remaining" the same as ">3 remaining" will incorrectly force an all-data flit when it shouldn't, and one that doesn't special-case Slot 0/Slot 3 availability during a 2-chunk or 1-chunk rollover will either drop data or violate slot alignment. Cover all five cases above as **directed test cases** in the reference testbench (Phase 2), not just as randomized coverage.

---

## 2. Slot Formats In Scope (§4.2.3)

| Direction | Format | Content | Size (bits) |
|---|---|---|---|
| Downstream (M2S) | **H5** | M2S Req only | 87 |
| Downstream (M2S) | **H4** | M2S RwD Header | 87 |
| Upstream (S2M) | **H3** | S2M DRS Header + S2M NDR | 70 |
| Upstream (S2M) | **H4** | 2× S2M NDR | 60 |
| Upstream (S2M) | **H5** | 2× S2M DRS Header | 80 |

### 2.1 M2S Request (Req) — Table 29 / Figure 54 (H5)

| Field | Bits | Notes |
|---|---|---|
| Valid | 1 | |
| MemOpcode | 4 | See §3.1 below (scope-restricted) |
| MetaField | 2 | Fixed to No-Op (`11`) — simplification, see §5 |
| MetaValue | 2 | Don't-care when MetaField = No-Op |
| SnpType | 3 | Fixed to No-Op (`000`) — simplification, see §5 |
| Address[51:5] | 47 | Host physical address |
| Tag | 16 | Transaction ID, reflected back in the response |
| TC | 2 | Traffic Class (reserved for future use, pack as 0) |
| LD-ID | 4 | Fixed to 0 — simplification, see §5 |
| RSVD | 6 | |
| **Total** | **87** | |

### 2.2 M2S Request with Data (RwD) — Table 35 / Figure 53 (H4)

Same fields as M2S Req, plus:
| Field | Bits | Notes |
|---|---|---|
| Address[51:6] | 46 | One bit narrower than Req — RwD is always cacheline-aligned |
| Poison | 1 | Data-error indicator |

### 2.3 S2M No-Data Response (NDR) — Table 38

| Field | Bits | Notes |
|---|---|---|
| Valid | 1 | |
| Opcode | 3 | `Cmp` / `Cmp-S` / `Cmp-E` (Table 39) — see §5 correction |
| MetaField | 2 | Fixed No-Op |
| MetaValue | 2 | |
| Tag | 16 | Reflects the request's Tag |
| LD-ID | 4 | Fixed to 0 |
| DevLoad | 2 | Device load indicator (Table 40) — pack as `00` (Light Load) |
| **Total** | **30** | |

### 2.4 S2M Data Response (DRS) — Table 41

| Field | Bits | Notes |
|---|---|---|
| Valid | 1 | |
| Opcode | 3 | `MemData` (`000`) is the only opcode in scope (Table 42) |
| MetaField | 2 | Fixed No-Op |
| MetaValue | 2 | |
| Tag | 16 | |
| Poison | 1 | |
| LD-ID | 4 | Fixed to 0 |
| DevLoad | 2 | |
| RSVD | 9 | |
| **Total** | **40** | |

---

## 3. Opcodes In Scope

### 3.1 M2S Req Memory Opcodes (Table 30)

| Opcode | Encoding | Behavior (simplified, single-VC model) |
|---|---|---|
| `MemRd` | `0001` | Normal read |
| `MemRdData` | `0010` | Read, no Meta/Snoop side-effects in our simplified model |

### 3.2 M2S RwD Memory Opcodes (Table 36)

| Opcode | Encoding | Behavior |
|---|---|---|
| `MemWr` | `0001` | Full-line write |
| `MemWrPtl` | `0010` | Partial write — spec defines 64 byte-enable bits, one per byte |

### 3.3 S2M NDR Opcodes (Table 39)

| Opcode | Encoding | Behavior |
|---|---|---|
| `Cmp` | `000` | Generic completion — **use this for write completions.** |

> **Correction to the original project brief:** the brief lists `MemWrCmp` as a core opcode. **This opcode does not exist anywhere in the CXL 2.0 spec.** The correct term for a write completion is the generic `Cmp` opcode (Table 39). `Cmp-S`/`Cmp-E` exist only for read-ownership semantics tied to the coherence model, which is out of scope here. This spec uses `Cmp` for all NDR completions.

### 3.4 S2M DRS Opcodes (Table 42)

| Opcode | Encoding | Behavior |
|---|---|---|
| `MemData` | `000` | Read data response |

---

## 4. Credit-Based Flow Control (§4.2.6–4.2.8)

### 4.1 Primary mechanism: piggybacked credit return

Every flit header carries `ReqCrd`/`DataCrd`/`RspCrd`. Whenever the link layer has credits to return, it packs the exponentially-encoded value into the next flit it sends anyway — no dedicated message needed. A sender decrements its available-credit counter by 1 per message sent on a channel; it cannot send if its counter is 0.

### 4.2 Fallback mechanism: LLCRD control flit (§4.2.6, Table 53–54)

`LLCRD` is a **Control flit type** (`Type=1`, `LLCTRL=0000`, Figure 78) whose payload carries **only** credit-return and Ack information — no protocol content. It exists for the case where the link is otherwise idle and there is no protocol flit available to piggyback the credit return on.

Per §4.2.8.2 ("LLCRD Forcing"), an LLCRD flit is force-injected when either:
- a programmable number of pending Acknowledgments has accumulated (default threshold recommendation: 16), or
- a timer (which resets whenever any Ack/credit-carrying flit is sent) expires while credits or Acks are still pending.

Control flits are exempt from normal flow-control rules — they can be sent even with zero credits, since crediting them would create a circular dependency.

**Simplified model for this project:** we model two credit-return paths — (1) piggybacked on protocol flits (the common case), and (2) a distinct LLCRD control flit sent when no protocol flit is available within a bounded number of cycles (the fallback case) — rather than implementing the full programmable-threshold/timer logic.

---

## 5. Simplifications (documented, with justification)

| # | Simplification | Justification |
|---|---|---|
| 1 | Single virtual channel / single message class | Real CXL supports multiple; multi-VC arbitration is a scheduling problem orthogonal to flit packing/credit correctness, which is this project's focus. |
| 2 | LD-ID always 0 | Multi-Logical-Device (MLD) support is a separate feature (§9); not required to demonstrate flit/credit correctness. |
| 3 | MetaField/MetaValue fixed to No-Op | Full bias-based coherency (Meta0-State tracking) is CXL.cache-adjacent machinery outside the CXL.mem-only scope. |
| 4 | SnpType fixed to No-Op | Snoop semantics depend on the coherence engine (DCOH), out of scope per the project's "CXL.mem only" restriction. |
| 5 | DevLoad fixed to Light Load (`00`) | QoS throttling behavior is a performance feature, not a correctness requirement for flit packing. |
| 6 | LLCRD modeled without full programmable threshold/timer | The forcing *logic* (§4.2.8.2) is a tuning mechanism; this project verifies that credits are returned correctly and promptly, not the exact hardware timer/threshold values. |
| 7 | "68B flit" used informally to mean the 66B flit + 2B Protocol ID | Matches common industry usage (also used in the assigned project brief and secondary literature); the spec itself calls it a "528-bit flit." Documented here to avoid ambiguity in the RTL/testbench naming. |
| 8 | `MemWrCmp` opcode replaced with `Cmp` | `MemWrCmp` does not exist in the CXL 2.0 spec (verified via full-text search of all 628 pages). `Cmp` (Table 39) is the correct completion opcode for writes. |

---

## 6. Worked Example — Single `MemRd` Transaction

1. **Host → Device (M2S Req, format H5, Slot 0):**
   `Valid=1, MemOpcode=0001 (MemRd), MetaField=11 (No-Op), SnpType=000 (No-Op), Address[51:5]=<target addr>, Tag=<TID>, LD-ID=0000`
2. Host's own send-credit counter for the **M2S Request** channel decrements by 1 (the host is the sender of this channel, so it tracks its own remaining allowance locally).
3. **Device → Host (S2M DRS, format H3 or H5, in a later flit):**
   `Valid=1, Opcode=000 (MemData), Tag=<TID> (reflected)`, plus 4×16B data chunks in Slots 1–3 / a following all-data flit (since a full 64B cacheline requires `Sz=1`, 4 data chunks — Table 47). Sending this also decrements the **device's own** send-credit counter for the S2M DRS channel.
4. Two independent credit returns now need to happen, each by the *receiver* of the channel (§4.2.7):
   - The **Device** (receiver of M2S Req) returns a `ReqCrd` credit to the Host, piggybacked in the header of an **upstream** flit it transmits — e.g. the same flit carrying the DRS response in step 3, or a later one. This replenishes the Host's ability to issue further M2S Req-class transactions.
   - The **Host** (receiver of S2M DRS) returns a `DataCrd` credit to the Device, piggybacked in the header of a **downstream** flit it transmits (or via a forced LLCRD if the link goes idle). This replenishes the Device's ability to send further DRS responses.

This trace is the reference your cocotb testbench (Phase 2) should reproduce byte-for-byte against the RTL's actual flit output.

---

## 7. Scope boundary (from project brief, retained)

**In scope:** CXL 2.0, single VC, M2S Req/RwD + S2M NDR/DRS, opcodes `MemRd`, `MemRdData`, `MemWr`, `MemWrPtl`, `Cmp`, `MemData`, LLCRD-style credit return.

**Out of scope:** multiple message classes, CXL 3.0 256B flit format, BISnp coherence flows, physical-layer link training/retry/FEC/CRC, multi-device pooling.
