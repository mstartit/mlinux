# Inside Secure / SafeXcel driver reverse-engineering notes (for netdev-oriented re-use)

## 1) What exists today in `drivers/crypto/inside-secure/`

### 1.1 High-level split

The subtree is split into two unrelated hardware generations:

- `drivers/crypto/inside-secure/safexcel*` implements EIP97/EIP197-class engines.
- `drivers/crypto/inside-secure/eip93/*` implements an older EIP93 block with its own descriptor format and flow.

In short: **there is no common backend between `eip93` and `safexcel`** beyond both registering Linux crypto algorithms.

### 1.2 SafeXcel data-path model (EIP97/EIP197)

The SafeXcel driver is organized around two HW rings per software ring context:

- **CDR**: command descriptor ring (host -> accelerator)
- **RDR**: result descriptor ring (accelerator -> host)

The SW allocates DMA-coherent descriptor memory for both, plus shadow/token areas.
The token pointer is pre-wired into CDR descriptors once at init.

Request handling is crypto-API centric:

1. A crypto request is enqueued on a per-ring software queue.
2. Workqueue context dequeues and builds CDR+RDR descriptors.
3. Doorbells/threshold registers are updated so HW fetches descriptors.
4. IRQ top-half acknowledges ring interrupts; threaded handler drains RDR.
5. Result dispatch uses an RDR index->`crypto_async_request *` lookup table.

Notable architectural detail: on EIP197 the driver can load IFPP/IPUE firmware and initializes/probes the transform-record cache RAMs before enabling data-plane use.

### 1.3 Important objects and responsibilities

- `struct safexcel_crypto_priv`: global device state (MMIO base, ring array, firmware/hw caps, algorithm masks, pools).
- `struct safexcel_ring`: per-ring runtime state (software queue, locks, workqueue item, CDR/RDR pointers).
- `struct safexcel_desc_ring`: generic producer/consumer cursor state used by both CDR and RDR.
- Cipher/hash frontends (`safexcel_cipher.c`, `safexcel_hash.c`) convert crypto API requests into descriptors/tokens.
- `safexcel.c` contains probe, hw setup, ring interrupt plumbing, firmware loading, algorithm registration, and the central dequeue/completion machinery.

### 1.4 Why it matters for re-use

The code is **not** structured as a generic DMA engine; it is a crypto offload driver with DMA descriptor mechanics embedded inside crypto request lifecycles.

The reusable parts are mostly:

- ring allocation/cursor management,
- descriptor writeback/result harvesting,
- IRQ-threaded completion path.

The non-reusable parts without refactor:

- token generation and context-record management,
- crypto algorithm registration and per-alg request glue,
- cache invalidation commands used for crypto context coherency.

## 2) Reverse-engineered design doc (control flow)

### 2.1 Bring-up sequence (SafeXcel)

1. Probe allocates `safexcel_crypto_priv`, maps registers, configures clocks/IRQs.
2. Per-ring descriptor memory is allocated and initialized.
3. Hardware init programs descriptor-fetch/result-fetch engines.
4. For EIP197 variants, cache probing and firmware load/start are performed.
5. Supported crypto algorithms are registered in the kernel crypto API.

### 2.2 Fast path enqueue -> complete

1. Frontend chooses ring (`safexcel_select_ring`) and queues async request.
2. Worker builds one or more CDR entries and matching RDR entries.
3. RDR-slot metadata stores backpointer to original async request.
4. HW processes descriptors and writes result tokens/error bits.
5. IRQ thread drains ready RDR descriptors, checks errors, resolves request pointer, and invokes algorithm-specific completion.
6. Driver advances ring read pointers and may push more queued work.

### 2.3 Ring semantics that are useful for netdev-like DMA usage

- Producer wraps at `base_end`, consumer wraps similarly.
- Full ring detection is simple “next write equals read” logic.
- A rollback helper exists to unwind partially constructed descriptor bursts.
- Separate command/result rings naturally model TX submission and RX completion semantics.

## 3) Applying this to an EIP97/EIP197-derived netdev DMA path

Your desired packet paths:

- Ingress: external FIFO -> EIP97 bypass/null-crypto -> host ring (RDR)
- Egress: Linux net stack -> netdev -> EIP97 bypass/null-crypto -> external FIFO

### 3.1 Practical architecture for upstreamable evolution

A good direction is **split-core + frontends**:

- Create a shared low-level “safexcel ring core” (descriptor/ring/irq primitives only).
- Keep existing crypto driver as one frontend.
- Add a new netdev (or auxiliary) frontend for bypass/null-crypto datapath.

This avoids forcing netdev abstractions into crypto core and avoids crypto-community objections that crypto driver is becoming a transport DMA framework.

### 3.2 Why not wire netdev directly into current crypto request path

Current flow assumes:

- crypto async request objects,
- algorithm-specific token/context templates,
- completion callbacks tied to crypto transforms.

For pure packet DMA/bypass these layers are overhead and type-mismatch.
A clean separation lowers lock contention and makes NAPI integration straightforward.

### 3.3 Ingress mapping sketch

- Program engine into bypass/null-crypto transform profile.
- Use redirect tweak so result lands into host-visible descriptor ring.
- In threaded IRQ/NAPI poll, consume result descriptors, build skb/page_pool-backed buffers, and submit to GRO/netif_receive_skb_list.
- Carry metadata sideband for errors/length/status from result token into skb->cb or descriptor-private data.

### 3.4 Egress mapping sketch

- ndo_start_xmit builds command descriptors pointing to skb fragments.
- Map completion model to BQL + `netdev_tx_completed_queue`.
- Reclaim in NAPI poll (preferably) instead of hardirq to match modern netdev practice.

### 3.5 DMA API and memory model notes

- Keep descriptor rings DMA-coherent.
- Data buffers should remain streaming-mapped (page_pool for RX, skb frags for TX).
- Preserve strict ownership protocol between HW and host for descriptor ready bits.

## 4) Bypass/null-crypto as a DMA provider: upstream concerns

### 4.1 DMAengine subsystem fit

Using dmaengine for networking datapath is usually awkward:

- netdev drivers typically own descriptor format directly,
- dmaengine is best for generic memcpy/slave transfers, not high-rate skb lifecycle.

Recommendation: implement as a **native netdev (or netdev offload) driver**, not as dmaengine provider, unless you have non-network consumers needing the exact same channel API.

### 4.2 Crypto community concerns

- Avoid exposing bypass mode through crypto algorithms if no crypto semantics exist.
- Keep crypto API registrations only for true crypto transforms.
- If shared code is needed, move only neutral ring/MMIO helpers into common library code under driver maintainers’ agreement.

### 4.3 Netdev community concerns

Expect requests for:

- NAPI-based RX/TX completion,
- page_pool/XDP readiness (or explicit rationale if unsupported),
- proper queue-stop/wake and BQL behavior,
- full ethtool stats/debuggability,
- clear error accounting and recovery (watchdog/reset paths),
- documented devicetree bindings for bypass/redirect resources.

### 4.4 Suggested incremental upstream plan

1. Refactor: isolate ring helper layer from crypto-specific logic (no behavior change).
2. Add internal selftests/kunit for ring cursor and rollback behavior.
3. Add new netdev driver using bypass mode on EIP97/EIP197 variant.
4. Optionally add shared mailbox/auxiliary-bus glue if crypto+net coexist on one block.

## 5) What exists under `drivers/net/ethernet/` today w.r.t SafeXcel

A repository-wide search in this tree shows:

- No direct `safexcel` references in `drivers/net/ethernet/`.
- No direct `inside-secure`, `eip97`, `eip197`, or `eip93` references in `drivers/net/ethernet/`.

Implication: in this kernel snapshot, there is no in-tree netdev driver reusing this SafeXcel crypto driver as a packet DMA engine.

## 6) Pointers for “learn next”

1. Study descriptor and token structs in `safexcel.h` first; they define what is truly reusable.
2. Trace one AEAD and one hash request through enqueue/build/complete to understand synchronization boundaries.
3. Examine IRQ threading and queue push/pull in `safexcel.c` to identify where a NAPI poll loop could be substituted.
4. For your EIP97/EIP197 variant, document the exact bypass/redirect register programming and whether firmware participation is required.
5. Prototype with a minimal single-queue netdev first; add multiqueue only after proving completion ordering and backpressure.

## 7) Extra note on EIP93 folder relevance

`eip93/` is a separate driver lineage and not a template for EIP97/EIP197 bypass integration:

- different register map,
- different descriptor/control words,
- different algorithm capability probing.

Use it only as a conceptual example of “crypto-driver-embedded DMA rings”, not as shared code candidate.
