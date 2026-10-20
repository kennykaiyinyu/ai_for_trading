# AkiNext (ML Algo v2) — Architecture, Simulator & Execution Roadmap
**Author:** Kenny Kai Yin Yu  
**Stakeholders:** Ahmed Bhayat, Sukhbir, Trading Desk (Connor & Doug)  
**Epic References:** `JDOE-6638` (Risk Controls & Simulator) / `ML Algo v2` (`AkiNext`)  
**Status:** 2-Week Deep-Dive Proposal & Complete Code Study Draft  

---

## Executive Summary: Too Much Monolithic Code Doing Too Little

Our forensic code audit of all six registered Aki algorithm classes (`CrossingAlgo`, `MLAlgo`, `MLAggressiveAlgo`, `MLPassiveAlgo`, `MLTWAPAlgo`, and `TWAPAlgo`) reveals that the core multi-leg business logic is fundamentally straightforward: one parent order manages venue waves, cancels on a timer, reconciles `CumQty`, and repeats until filled, rejected, expired, or user-cancelled. 

However, legacy **Aki** buries this simple lifecycle inside thousands of lines of copy-pasted state machines, entangled market data calls, and unmockable venue dependencies. Rather than executing a risky blind rewrite from scratch or continuing high-risk in-place patching, we propose **AkiNext**: an extracted, modular Execution Management System (EMS) engine running on a **parallel GUI tab** alongside legacy Aki.

---

## Part 1: Forensic Code Audit of Legacy `Aki`

### 1.1 Critical Defects Identified Across the Legacy Algo Hierarchy

| Audit Finding | Code-Level Evidence in Legacy `Aki` | Production & Business Impact |
| :--- | :--- | :--- |
| **1. Copy-Pasted Lifecycle Loop** | `ProcessAlgo`, `Validate`, `CalcAlgo`, `UserCancel`, and `CanCxl` are duplicated across classes. The `CumQty` safety check (`slice + algo == total`, gated by `m_bCanUpdateCumQty`) is manually pasted **four times** inside each `ProcessAlgo`. | Every new risk control or trader improvement must be coded and tested multiple times, doubling delivery time and regression risk. |
| **2. Accidental State Divergence Bugs** | • After cancel-replace, `MLAlgo` stays in `CancelReplaceReady` while `MLAggressiveAlgo` overwrites it via the `CumQty` block (`Cxl` or `Completed`).<br>• No-fill idle counter (`m_numCXLForIdle`) resets to `0` in `MLAlgo` and `1` in `MLAggressiveAlgo` (so aggressive almost never accumulates the long idle wait).<br>• Only `MLAggressiveAlgo` enforces the 1800-second (30-min) hard stop timer. | Unpredictable desk behavior between passive, parameterized, and aggressive modes; silent order stalls and inconsistent cancel-replace handling. |
| **3. Broken Strategy Pattern & Memory Leaks** | `MLAlgo` indexes `vector<MLSplit*>` using `AlgoState` enum as a raw array key. It allocates via raw `new MLSplitNoMD()` with an empty destructor, and duplicates sizing across 3 classes (`MLAlgo`, `MLAggressiveAlgo`, `MLSplitNoMDParams`). | Memory leaks in long-running processes; adding any new state before the MD states corrupts the strategy lookup table. |
| **4. Abandoned Dead Code in Live Path** | • `MLSplitMD::Split` is registered but stubbed (returns `WavesReady` and builds nothing).<br>• `MLSplitMarketable` and `MLSplitNonMarketable` exist in the repo and are never called.<br>• `MLPassiveAlgo` contains `SetWavesQtyParam` and `EvaluateMaxWorkingQty` (including 70% double), but `Split` always calls `SetWavesNoQtyParam`—the quantity-parameter path is dead. | High cognitive load during debugging; impossible to safely enable book-ranked or quantity-parameterized slicing without untangling dead branches. |
| **5. Query Mutating State** | `CanCxl()` (a boolean query helper) secretly mutates member state by setting `m_isCancelRequestedByUser` (or returns always-false in `CrossingAlgo` with an empty `UserCancel()`). | Violates Command-Query Separation (CQS); makes deterministic unit testing and replay debugging hazardous. |

---

### 1.2 Architectural Comparison: Monolithic `Aki` vs. Layered `AkiNext`

```text
LEGACY "AKI" (Entangled Monolith)                 PROPOSED "AKINEXT" (Modular Layered EMS)
=================================                 ========================================
[ GUI: Single Legacy Algo Tab ]                   [ GUI: Legacy Tab (Aki) ]  [ GUI: NEW Tab (AkiNext) ]
               |                                                                         |
               v                                                                         v
+---------------------------------------+         +----------------------------------------------------+
| Monolithic Algo Subclasses            |         | LAYER 1: OMS & Centralized Pre-Trade Risk          |
| (MLAlgo / MLAggressive / MLPassive /  |         | - Parent Order Lifecycle & Fat-Finger Checks       |
|  MLTWAP / TWAP / Crossing Clones)     |         | - Risk Controls Suite (JDOE-6010 to JDOE-6019)     |
|                                       |         +----------------------------------------------------+
| - 4x Pasted CumQty Accounting Guards  |                                    |
| - AlgoState Enum Used as Vector Index |                                    v
| - Timers Mixed with Slicing Logic     |         +----------------------------------------------------+
| - Raw Pointer Memory Leaks            |         | LAYER 2: AkiNext Unified Strategy Engine           |
| - Dead MLSplitMD / Passive Qty Paths  |         | - Single Event-Driven ProcessAlgo Lifecycle        |
+---------------------------------------+         | - Composable Schedule + SlicePolicy Interfaces:    |
               |                                  |   * EqualSplitPolicy (ex-MLSplitNoMD / TWAP)       |
               v                                  |   * FillRateTickWalkPolicy (MLWParams/Aggressive)  |
+---------------------------------------+         |   * SpreadPercentPassivePolicy (MLPassive)         |
| Tightly Coupled Venue / MD Calls      |         |   * DiscoveryTWAPSchedule (MLTWAP / TWAP)          |
| (No Interface Seam for Simulator)     |         +----------------------------------------------------+
+---------------------------------------+                    |                               |
                                                             v                               v
                                                  +---------------------+       +----------------------+
                                                  | LAYER 3: Abstract   |       | LAYER 4: Post-Trade  |
                                                  | IExecutionGateway & |       | TCA & Audit Logger   |
                                                  | IMarketData Seam    |       | - Slippage & Fill %  |
                                                  +---------------------+       | - Spread % & Idle    |
                                                     /        |        \        +----------------------+
                                                    /         |         \
                                          +----------+  +-----------+  +-------------------------------+
                                          | Path 4A: |  | Path 4B:  |  | Deterministic Exchange        |
                                          | Legacy   |  | MATRIX    |  | Simulator (JDOE-6647/51/52)   |
                                          | Aki FIX  |  | Line      |  | - Legging & Partial Fill Sim  |
                                          | Adapter  |  | Handler   |  | - Iteration & Cooldown Replay |
                                          +----------+  +-----------+  +-------------------------------+
```

### 1.3 Institutional Literature Supporting the `AkiNext` Architecture
* ***Algorithmic Trading and DMA* (Barry Johnson):** Establishes the canonical institutional EMS separation between *Parent Order Lifecycle & Pre-Trade Risk* (Layer 1), *Macro Scheduling vs. Micro Child Slice Policies* (Layer 2), and *Venue Line Handlers* (Layer 3).
* ***Building Low Latency Applications with C++* (Sourav Ghosh, Packt):** Demonstrates clean, zero-overhead C++ interface decoupling so identical strategy code runs against live exchange gateways or a deterministic **Exchange Simulator**.
* ***Working Effectively with Legacy Code* (Michael Feathers) & *Designing Data-Intensive Applications* (Martin Kleppmann):** Validates preserving battle-tested exchange invariants (`CumQty` guards, cancel-replace locks) via **Characterization Unit Tests** and deploying via a **Parallel Run GUI Tab (Strangler Fig Pattern)** to guarantee zero disruption to desk operations.

---

## Part 2: Strategic Options & Pros / Cons Decision Matrices

### 2.1 Decision Matrix A: Core Algo Engine Strategy
We evaluated four engineering approaches. Note that we explicitly reject a "blind from-scratch rewrite" because existing exchange safety invariants (`CanUpdateCumQty`, prior-wave finality checks, cancel-replace locks) must be preserved through extraction.

| Strategy Option | Architectural Approach | Pros | Cons & Risks | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Option 1: In-Place Patching of Legacy `Aki`** | Continue adding Risk Controls (`JDOE-6010–6019`) and trader tweaks directly inside `MLAlgo`, `MLAggressiveAlgo`, `MLPassiveAlgo`, etc. | • Zero upfront setup.<br>• Uses existing GUI tab immediately. | • **Severe Regression Risk:** Editing 4x pasted blocks across 6 classes invites live bugs.<br>• Every feature coded multiple times.<br>• Cannot attach a clean Simulator. | **REJECT** |
| **Option 2: Blind From-Scratch Rewrite** | Discard all legacy code and write a brand-new state machine from a blank file without extracting existing guards. | • Clean greenfield C++ code without legacy naming. | • **Loses Battle-Tested Guards:** High risk of reintroducing `CumQty` overfill or cancel-replace race bugs in production. | **REJECT** |
| **Option 3: In-Place Extraction Inside `Aki`** | Extract `SlicePolicy` inside the current `Aki` binary without creating a parallel engine boundary or new GUI tab. | • Eliminates copy-pasted `ProcessAlgo` loops.<br>• Fixes memory leaks. | • Traders have no instant fallback if a refactored build behaves differently.<br>• Tied to legacy `Aki` plumbing. | **FALLBACK** |
| **Option 4: `AkiNext` Engine + Parallel GUI Tab (Modify by Extraction)** | Extract proven lifecycle guards into a unified `AkiNext` engine with `unique_ptr<SlicePolicy>`, Simulator, Unit Tests & TCA, exposed via a **new GUI Tab** alongside legacy `Aki`. | • **Zero Desk Disruption:** Legacy `Aki` tab stays live as an instant 1-click fallback.<br>• **Preserves Invariants:** Keeps proven exchange safety guards.<br>• **3x Faster Velocity** for new algos. | • Requires a thin connectivity adapter in Phase 1 (solved via Path 4A or 4B below). | **RECOMMENDED WINNER** |

### 2.2 Decision Matrix B: Phase 1 Connectivity (Starting Small & Fast)
To deliver rapid business value in Phase 1 without boiling the ocean, `AkiNext` places an abstract `IExecutionGateway` seam below the strategy engine. This enables two low-risk Phase 1 connectivity options:

| Phase 1 Connectivity Path | Advantages (Pros) | Trade-offs (Cons) & Best Fit |
| :--- | :--- | :--- |
| **Path 4A: Reuse Legacy `Aki` FIX Connectors** *(Fastest Go-Live)* | • **Zero Exchange Certification Delay:** Reuses existing production FIX sessions, drop copies, and venue entitlements.<br>• Strictly isolates Phase 1 validation to the new `AkiNext` Algo & Risk Engine. | • Requires a lightweight C++ adapter bridging `AkiNext` slice events to legacy `Aki` FIX objects.<br>• *Best when:* Desk wants the new GUI tab live in production within weeks. |
| **Path 4B: Use `MATRIX` as Phase 1 Line Handler** *(Strategic Stack)* | • **Modern Normalized Protocol:** Directly leverages our proven `MATRIX` line handler work.<br>• Eliminates legacy `Aki` FIX translation quirks from Day 1. | • Requires verifying multi-leg complex order tags across target venues in `MATRIX` during Phase 1.<br>• *Best when:* Management wants immediate consolidation onto `MATRIX`. |

---

## Part 3: Core Engine Refactoring Blueprint & Quality Pillars

### 3.1 Bounded Extraction Blueprint: Single Lifecycle + Composable Policies

```text
+-------------------------------------------------------------------------+
| AlgoBase / Unified ProcessAlgo Lifecycle                                |
| - Pre-Send Guard: All prior waves final + Wave CumQty == m_iAlgoCumQty  |
| - User Cancel Check immediately before SendWaves                        |
| - Parameterized Timers (IterationDuration, IdleTime, 1800s Stop Cap)    |
| - Pure Query CanCxl() (Side-effect free)                                |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| <<interface>> SlicePolicy (Owned via std::unique_ptr<SlicePolicy>)      |
| Selected by AlgoType Factory (Deletes brittle AlgoState enum vector)    |
+-------------------------------------------------------------------------+
       |                        |                        |
       v                        v                        v
+--------------------+  +-----------------------+  +----------------------+
| EqualSplitPolicy   |  | FillRateTickWalkPolicy|  | SpreadPassivePolicy  |
| (ex-MLSplitNoMD &  |  | (Unifies MLWParams,   |  | (ex-MLPassiveAlgo &  |
|  TWAP slice split) |  |  MLAggressive &       |  |  Discovery TWAP      |
|                    |  |  MLTWAP 70% double)   |  |  NBBO width quoting) |
+--------------------+  +-----------------------+  +----------------------+
```

### 3.2 Four Concrete Extraction Rules (Zero-Regression Refactoring)
1. **Unify & Parameterize `AlgoBase`:** Move shared `Validate`, cancel handling, and the single pre-send `CumQty` guard (`every prior wave is final && total wave CumQty == m_iAlgoCumQty`) into `AlgoBase`. Parameterize the only real differences between modes: *no-fill idle counter reset (`0` vs `1`)*, *30-minute (`1800s`) stop timer*, and *leg ratio > 3 rejection*.
2. **Replace Enum-Indexed Vector with `unique_ptr<SlicePolicy>`:** Delete the brittle `vector<MLSplit*>` indexed by `AlgoState` and fix the raw-pointer memory leak (`new MLSplitNoMD()`). Instantiate policies cleanly via a single factory.
3. **Consolidate the 70%-Fill / Tick-Walk Logic into One Policy:** `MLSplitNoMDParams`, `MLAggressiveAlgo`, and `MLTWAPAlgo` all share the exact same family of rules (70% fill doubles working quantity; below 70% steps limit by tick size). Point them at one tested `FillRateTickWalkPolicy` implementation.
4. **Delete or Quarantine Dead Code:** Remove uncalled `MLSplitMarketable` / `MLSplitNonMarketable`, stubbed `MLSplitMD`, and the dead quantity-parameter branch inside `MLPassiveAlgo`.

### 3.3 The Three Engineering Quality Pillars

| Quality Pillar | Technical Implementation in `AkiNext` | Direct Value to Desk & Management |
| :--- | :--- | :--- |
| **1. Deterministic Exchange Simulator** (`JDOE-6647, 6651, 6652`) | Implements `IExecutionGateway` & `IMarketData` to replay composite NBBO books and deterministically inject edge cases: **Leg 1 partial fill + Leg 2 reject ("legging risk")**, **Iteration/Idle timer traps**, **Cancel-Replace races**, and **NBBO timeouts (`MDFailed`)**. | Validates complex multi-leg recovery and risk controls offline before a single order touches production venues. |
| **2. Characterization & Regression Unit Tests** | GoogleTest harness expanding `MLAlgoTests.cpp` to lock down exchange invariants: never send after user cancel, never advance `CumQty` twice, reject (`m_bAlgoCxl`) when wave qty != `m_iAlgoCumQty`, and block cancel-replace while a wave is live. | Gives engineers 100% confidence to refactor legacy classes and add new algos without breaking live desk workflows. |
| **3. Post-Trade TCA Telemetry** | Generalizes `MLTWAPAlgo`'s internal stats recording (execution time & fill's percent of NBBO width) across **all** algos: capturing **Arrival vs. Fill Slippage**, **Venue Fill-Rate Hit Ratio**, and **Spread Capture %**. | Provides Connor, Doug, and traders with objective quantitative proof of execution quality on the new `AkiNext` tab. |

---

## Part 4: Phase 1 MVP Roadmap, Jira Plan & Trader Sit-Down

### 4.1 Jira Ownership & Epic Restructuring Plan (`JDOE-6638` & `ML Algo v2`)

| Jira Reference | Workstream / Deliverable | Action / Owner | Role in `AkiNext` Phase 1 MVP |
| :--- | :--- | :--- | :--- |
| **`JDOE-6010` to `JDOE-6019`** (9 Tickets) | Pre-Trade & In-Flight Risk Controls Suite | **Reassigned: Kenny** | Implemented once inside `AkiNext` Layer 1 (OMS/Risk Gate) instead of duplicating across 6 legacy algo classes. |
| **`JDOE-6647, 6651, 6652`** (3 Tickets) | Multi-Leg Exchange Simulator & Test Harness | **Reassigned: Kenny** | Builds the deterministic simulation market behind `IExecutionGateway` to test slicing, timers, and partial-fill legging. |
| **`NEW-01`** (Under `ML Algo v2`) | `AkiNext` Core Lifecycle & `SlicePolicy` Extraction | **Create: Kenny** | Unifies `ProcessAlgo`, fixes memory leaks, and implements `EqualSplitPolicy` & `FillRateTickWalkPolicy` (Phase 1 MVP). |
| **`NEW-02`** (Under `ML Algo v2`) | Phase 1 Line Handler Adapter (`Aki` FIX vs. `MATRIX`) | **Create: Kenny** | Implements concrete execution binding per management choice (Path 4A Aki FIX shim or Path 4B MATRIX connector). |
| **`NEW-03`** (Under `ML Algo v2`) | Parallel GUI Tab (`AkiNext`) & TCA Telemetry Log | **Create: Kenny** | Exposes `AkiNext` in a dedicated parallel GUI tab alongside legacy `Aki` and outputs slice-level TCA metrics. |

### 4.2 Key Questions for the Upcoming Trader Sit-Down
1. **Idle Counter & Stop Timer Consistency:** `MLAggressiveAlgo` resets the no-fill idle counter to `1` (almost never triggering long idle wait) and enforces a 30-minute (`1800s`) hard stop, whereas `MLAlgo` resets to `0` and has no 30-minute stop. What should the unified rule be in `AkiNext`?
2. **Leg Ratio Cap & Fallback:** `MLAggressiveAlgo` rejects any order with a leg ratio above 3 and has no all-ratio-venue fallback. Should `AkiNext` make max leg ratio configurable via Risk Controls (`JDOE-6010–6019`)?
3. **Passive Algo Quantity Parameters:** `MLPassiveAlgo` has code for 70%-fill quantity doubling, but it is currently dead/bypassed (always capping initial slices at `min(5, qty/venues)`). Do traders want quantity sizing enabled in Passive mode?
4. **Market-Data Book Ranking (`MLSplitMD`):** Since `MLSplitMD` is currently a non-functional stub, can we defer book-ranked slicing to Phase 2 while prioritizing `MLAlgo` / `MLAggressiveAlgo` unification and Simulator validation in Phase 1?

---
---

# Appendix A: Complete Legacy `Aki` Code Study Reference

This appendix documents the exact runtime behavior, selection mechanics, and state-machine flows of all six active algorithm classes in legacy `Aki`, serving as the baseline specification for `AkiNext` characterization unit tests.

## A.1 Algo Selection & Factory Mechanics

* **Six algo classes** are registered and can run. The `AlgoType` enum has eight labels, but two of those (`MAAlgo` and `MLWParams`) are modes of `MLAlgo`, not separate classes. `MLSplit*` types only build waves; they are not algos.
* `InboundOrderManager` creates the class from the algo-data string:

| Class | How It Is Selected (`InboundOrderManager` / `AlgoHelper`) |
| :--- | :--- |
| **`CrossingAlgo`** | `NewOrderCross` or `NewOrderCrossML`. `CrossingPassThrough` uses this same class. |
| **`MLAlgo`** | Default multileg. Also used for `MAAlgo` and `MLWParams`. |
| **`MLTWAPAlgo`** | Algo data contains `MLTWAP`. |
| **`TWAPAlgo`** | Algo data contains `TWAP` and not `MLTWAP`. |
| **`MLAggressiveAlgo`** | Algo data contains `Aggressive`. |
| **`MLPassiveAlgo`** | Algo data contains `Passive`. |

* **Factory Quirks:**
  * `AlgoType` in `AlgoHelper` is: `CrossingPassThrough`, `MLAlgo`, `MAAlgo`, `MLWParams`, `MLTWAP`, `MLAggressive`, `MLPassive`, `TWAP`.
  * `GetAlgoType` never returns `MAAlgo` or `MLWParams`; those strings still construct `MLAlgo`. `ParseAlgoData` then sets `MAAlgo` from the name, or `MLWParams` if `StartPx` or `StartQty` is present. `MAAlgo` only changes which splitter `MLAlgo` uses (`MLSplitNoMDParams` instead of `MLSplitNoMD`) and which exchange list is loaded.
* **Splitter Status Under `MLAlgo`:**
  * `MLSplitNoMD` is the live default.
  * `MLSplitNoMDParams` is the live `MLWParams` path.
  * `MLSplitMD` is registered but stubbed (not on the live path).
  * `MLSplitMarketable` and `MLSplitNonMarketable` exist in the codebase and are never called.
* **Universal Pre-Send Guard:** Every multileg algo except `CrossingAlgo` shares the same guard before a send: **every prior wave must be final, and the total wave `CumQty` must equal `m_iAlgoCumQty`**. A mismatch sets `m_bAlgoCxl`, and the state becomes `Rejected`. A user cancel is checked again immediately before `SendWaves`.

---

## A.2 `CrossingAlgo`

* **Summary:** One-shot cross, not a slicer. `GetAlgoType` only returns `CrossingPassThrough` for that string; bulk cross message types construct this class. `UserCancel()` is empty and `CanCxl()` is always `false`.
* **Initialization & Routing:** `InitAlgo` parses a `NewCrossRequest` and requires the preferred exchange to be in the cross-exchange list for that leg count. `DoIteration` builds one `MLOrder` (`CrossSL` or `CrossML` if there are legs) and sends it to the preferred exchange. On a later pass, it sends to the next exchange that has not already been tried. A customer-to-customer cross is limited to `BOX`, `CBOE`, and `EDGX`.
* **Lifecycle:** `ProcessAlgo` does not cancel on a timer. A fill or partial fill completes the algo immediately. If every tried venue rejected or cancelled, the state is `Rejected`. If some venues are still untried, the state is `Cxl`, and the engine calls `DoIteration` again for the next venue.

```text
[A: New cross] 
      |
      v
[B: Validate preferred exchange] 
      |
      v
[C: Send cross to that venue] 
      |
      v
[D: Wave state]
  |---> (Filled or PartiallyFilled) ---------> [E: Completed]
  |---> (New or PendingNew) -----------------> [F: Ack, keep waiting]
  |---> (Rejected or Cxl, venues left) ------> [G: Cxl, try next venue] ---> (Loops to C)
  +---> (Every allowed venue failed) --------> [H: Rejected]
```

---

## A.3 `MLAlgo` (Default Multi-Leg, `MAAlgo`, `MLWParams`)

* **Summary:** Default multileg algo (`MLAlgo.cpp`). It does not size the slice itself; the constructor installs splitters into a vector indexed by `AlgoState`:

| `AlgoState` | Who Builds the Waves |
| :--- | :--- |
| `MDNotRequired`, `MDFailed` | `MLSplitNoMD` |
| `MDNotRequiredParams` | `MLSplitNoMDParams` |
| `MDWaiting`, `MDReady` | `MLSplitMD` (stub, not on the live path) |

* **Slicing & Iteration Rules:**
  * `InitAlgo` parses the request, validates it, and calls `DoIteration`. If `StartPx` or `StartQty` was present, `ParseAlgoData` already set `AlgoType::MLWParams`, so the state becomes `MDNotRequiredParams`. Otherwise it is `MDNotRequired`. `MAAlgo` only changes which exchange list is loaded.
  * `MLSplitNoMD` splits the remaining quantity equally across venues that filled the last slice, or across every active venue if none filled. The limit is the order price. No walking.
  * `MLSplitNoMDParams` is the same family as aggressive: a start quantity, a start price, a tick step, and a double-size after a 70% fill. Distance from the order price to `m_dStartPx` stretches the iteration timer by `2` or `3`. A venue with `NoFillsIterationsPrc() == 4` stretches it by `10`.
* **Execution Loop (`ProcessAlgo`):**
  * Waits in `Ack` until `IterationDuration()` elapses, then cancels.
  * After `MLAlgoNumIterations()` no-fill cancels, it waits `IdleTime()` before the next cancel.
  * When every wave is final, it adds the slice `CumQty` once. Filled means `Completed`; otherwise the state is `Cxl` and the engine starts the next slice. A quantity mismatch becomes `Expired`.
  * Cancel-replace cancels the live waves, then returns `CancelReplaceReady` without always overwriting it.

```text
[A: InitAlgo] ---> [B: MLWParams?]
                     |-- (no) --> [D: MLSplitNoMD] -------+
                     +-- (yes) -> [C: MLSplitNoMDParams] -+
                                                          |
                                                          v
                  +-------------------------------> [E: SendWaves]
                  |                                       |
                  |                                       v
                  |                          [F: Ack until iteration timer]
                  |                                       |
                  |                                       v
            (remainder)                            [G: CancelWaves]
                  |                                       |
                  |                                       v
                  +------------------------- [H: CumQty matches & order filled?]
                                                |-- (filled) ----> [I: Completed]
                                                +-- (mismatch) --> [J: Expired]
```

---

## A.4 `MLAggressiveAlgo`

* **Summary:** Same parent loop as `MLAlgo`, with the splitter inlined and no market data.
* **Timers & Guards:** `InitAlgo` starts a **1800-second (30-min) timer**. `DoIteration` sets `m_bStopAlgoFlag` when that elapses, and `ProcessAlgo` rejects the algo. A **leg ratio above 3 is always a reject**. There is no all-ratio-venue fallback.
* **First Slice vs. Later Slices:**
  * Split on the first pass divides `m_iStartQty` by the venue count when it differs from the order quantity; otherwise it splits the order evenly. The limit is `m_dStartPx` if that is below the order price, otherwise the order price.
  * **Later slices:**
    * A venue that filled at least **70%** of its working quantity **doubles that quantity**, capped by what is left.
    * Below 70%, the size stays and `SetNewLmt` steps the working limit up by `m_dTickSize`, never through the order price. If the next tick would cross, it pulls back toward the midpoint of the start price and the current limit.
    * If the last slice had fills and quantity parameters are off, the remainder goes only to the venues that filled, with no price step.
    * One no-fill iteration after the algo already has `CumQty` calls `ReSetQty` and resends the unfilled venues smaller, without a price step.
* **Divergence from `MLAlgo`:** The idle counter is reset to `1` after every standard cancel, so the long idle wait almost never accumulates. After cancel-replace, `CancelReplaceReady` is overwritten by the `CumQty` block (`MLAlgo` can leave it in place).

```text
[A: First slice at start price]
               |
               v
[B: Work until iteration timer] <------------------------------------+
               |                                                     |
               v                                                     |
[C: Cancel and read fill rate]                                       |
               |                                                     |
               v                                                     |
      [D: >= 70% filled?]                                            |
        |-- (yes) -> [E: Double working qty, same limit] -+          |
        +-- (no) --> [F: Same qty, step limit by tick] ---+          |
                                                          |          |
                                                          v          |
                                            [G: Order filled or 30 min?]
                                              |-- (no) --------------+
                                              |-- (filled) --> [H: Completed]
                                              +-- (30 min) --> [I: Rejected]
```

---

## A.5 `MLPassiveAlgo`

* **Summary:** Same cancel-and-reslice loop, but the price comes from the **composite NBBO**, and it pauses between slices.
* **Market Data & Initial Slice:** `InitAlgo` subscribes each leg's NBBO (`ReloadNBBO`). `ProcessAlgo` will not send until `CheckMD` returns `MDReady`. A market-data timeout becomes `MDFailed`. The first slice is small: `min(5, qty / venues)`, at `m_dStartPx` if that is below the order price (`m_iStartQty` is read and then ignored).
* **30-Second Pause & Spread-Percent Repricing:**
  * After a slice is final, if there was a fill or at least one no-fill iteration, it does not immediately resend. It sets `m_bWaitRequired`, sets `m_iExtraOff = 1`, and returns `KeepWaiting` for `m_iWaitTime` (30 seconds). When the wait ends, it calls `CalcAlgo` again.
  * On the first fill, `SetLmtPct` records where that limit sat in the spread: `100%` if it was the ask, otherwise `100 - (bid - limit) / width`. Later `SetNewLmt` tries to reprint that same percent of the current width, minus `m_iExtraOff`. If the ask and width have not changed, it returns to the first-fill limit. The result is capped by the ask and by the order price. Before any fill, the limit simply steps by `m_dTickSize`.
* **Sizing & Dead Code:** If the last slice did not fill, each venue gets at most `5`. If it did fill, the remainder is split across those venues, capped at twice the largest working quantity. `SetWavesQtyParam` and `EvaluateMaxWorkingQty` exist (including the 70% double), **but `Split` always calls `SetWavesNoQtyParam`—the quantity-parameter path is dead**.

```text
[A: Wait for NBBO]
        |
        v
[B: Send small slice at start price] <-------------------+
        |                                                |
        v                                                |
[C: Ack, then cancel on timer]                           |
        |                                                |
        v                                                |
[D: Record fill percent of the spread]                   |
        |                                                |
        v                                                |
[E: Wait 30 seconds]                                     |
        |                                                |
        v                                                |
[F: Reprice to that percent of the new width] -----------+
```

---

## A.6 `MLTWAPAlgo`

* **Summary:** Time-sliced aggressive logic. The parent order is cut into slices that must finish before `TwapEndTime`. Inside a slice, it still cancels on `IterationDuration()` and resizes with the 70% rule.
* **NBBO Subscription & Discovery Schedule:** `InitAlgo` does not send. It subscribes each leg NBBO and the underlying spot, sums delta and gamma, and sits in `MDWaiting` until the book is ready. `CalcTWAP` turns the remaining time and quantity into a slice size and a wait. `CalcDiscoveryLimits` may spend about 10% of the time (capped at 120 seconds) walking limits across the market width before the working slices. Width above 7 cents is narrowed by 25% before that walk. The schedule is a list of `(numJumps, jumpSize)` pairs, coarser when the width is wide and the time is short.
* **In-Slice Execution:** A working slice starts at the composite bid, capped by the order limit. `SetNewLmt` steps toward the ask. A fill at a tick larger than 1 cent pulls the next limit back. The step is also capped by the order limit. The slice quantity is split across venues. A 70% fill doubles the working quantity *inside that slice only*.
* **Slice Completion & Embedded TCA:** `ProcessFills` ends the slice when `m_iSliceCumQty >= m_iSliceQty`, **records the execution time and the fill's percent of width**, then waits. If a slice's `CumQty` exceeds the slice size, the algo expires. When the order quantity is done, it completes. The next slice does not start just because the iteration timer fired—the slice clock has to elapse too.

```text
[A: Subscribe NBBO, wait until end time]
                   |
                   v
[B: Calc slice qty and discovery limits] <---------------+
                   |                                     |
                   v                                     |
    [C: Quote from bid toward ask] <----------+          |
                   |                          |          |
                   v                          | (no)     | (no)
    [D: Cancel and resize inside the slice]   |          |
                   |                          |          |
                   v                          |          |
        [E: Slice qty filled?] ---------------+          |
                   | (yes)                               |
                   v                                     |
        [F: Record stats, wait]                          |
                   |                                     |
                   v                                     |
        [G: Order done or end time?] --------------------+
                   | (done)
                   v
            [H: Completed]
```

---

## A.7 `TWAPAlgo`

* **Summary:** The simpler clock. It also waits for NBBO, but it does not discover the spread and it does not double size.
* **Slice Calculation:** `CalcTWAP` sets the slice quantity to the leaves divided by the number of iterations. The iteration count is the leaves divided by the venue count, unless the leaves are smaller than the remaining seconds, in which case the leaves become the iteration count. The wait is the remaining seconds divided by the iterations, and at least `60 / MaxTWAPOrdersMin()`. The first slice does not wait (`m_iSliceWaitTime` is set to `-1`).
* **Slicing & Repricing:** `Split` is `MLSplitNoMD` applied to the slice remainder, not the whole order. Venues that filled the last iteration are reused. If none filled, every non-rejected venue is used, and missing venues are added. The limit is `min(order limit, composite ask)`. If `m_bUseLmtOverAsk` is set and the order limit is above the ask, a no-fill iteration reprices to `ask + 3 cents`, still capped by the order limit.
* **In-Slice Loop:** Inside a slice, an unfinished slice calls `DoIteration` again immediately—it does not wait. The wait starts only after `ProcessFills` sees the slice quantity filled and returns `TWAPSliceCompleted`. `CalcTWAPWaitTime` then zeros the slice counters and arms `m_SliceTimer`. If the end time has already passed, the next slice is the entire remainder, sent at once. A slice `CumQty` above the slice size expires the algo.

```text
[A: NBBO ready]
       |
       v
[B: Calc slice qty and min interval] <------------------------------+
       |                                                            |
       v                                                            |
[C: Equal-split slice at min of limit and ask] <---------+          |
       |                                                 |          |
       v                                                 |          | (no)
[D: Slice filled?]                                       |          |
  |-- (no) --> [E: Cancel and send another iteration now]+          |
  +-- (yes) -> [F: Wait remaining time over iterations]             |
                               |                                    |
                               v                                    |
                      [G: End time passed?] ------------------------+
                               | (yes)
                               v
                      [H: Send all leaves as the last slice]
```