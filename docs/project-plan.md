# Project 2 — Fault-Tolerant Multi-Clock Avionics Telemetry and Control Gateway

**Platform:** Zybo Z7-20 (Zynq-7000)  
**Duration:** six consecutive months (approximately 26 weeks) of project development, then one dedicated month of paper writing and conference submission  
**Development approach:** DO-254 / FAA AC 20-152A–informed, **DAL B representative exercise**  
**Publication aim:** submit an original, evidence-backed IEEE-format paper to a suitable peer-reviewed conference whose proceedings are intended for IEEE Xplore  
**Status:** concept baseline, 27 September 2026; requirements and numerical limits remain to be validated

## 1. Mission and assurance boundary

Build a bench-top data concentrator representing one aircraft subsystem. Two independent sources supply redundant measurements; an ARINC 429–inspired **digital word interface** supplies navigation/status telemetry. The programmable logic (PL) checks integrity and freshness, transfers frames across independent clocks, chooses a valid source, records faults, and presents validated telemetry and status to the Zynq processing system (PS). The PS configures and observes the gateway and issues a small set of **non-actuating demonstration commands**. No flight hardware or aircraft connection is contemplated.

The second deliverable is an auditable development record: safety assumptions, requirements, design, verification, traceability, configuration baselines, and recorded discrepancies. The third is a research paper developed from measured results. DO-254 is development-assurance guidance, not a circuit-level recipe for fault tolerance. An actual hardware DAL is allocated in a real system safety assessment; this exercise *assumes a hazardous/severe-major failure condition as a learning scenario* and applies representative DAL B practices. **Official DO-254 certification is not a project objective.** This exercise does not establish a certified DAL B implementation or certify a Zybo board. Independent assurance, approved installation context, applicant/authority coordination, environmental qualification, and assessment of the full hardware and software system lie outside this project. See [FAA AC 20-152A](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_20-152A.pdf) and [FAA AC 00-72](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_00-72.pdf).

### Example safety premise to validate before architecture freeze

| Item | Working statement |
|---|---|
| Intended function | Deliver fresh, valid telemetry from a selected source; reject invalid commands and expose health status. |
| Representative failure condition | Undetected erroneous telemetry is presented as valid, or a command is misrouted. |
| Proposed safe response | Mark data invalid and inhibit downstream acceptance; record and report the cause. Maintain valid output through failover only when the backup is independently fresh and healthy. |
| Allocation assumption | DAL B representative *process target*, subject to a fictional functional hazard assessment (FHA), interface definition, and explicit environmental assumptions. |

**Safety boundary:** the PL makes the telemetry validity and selection decision. The PS can request configuration and read results, but cannot override a PL invalid indication or force an unvalidated frame into the valid-output path. For this exercise, PS software is a demonstration/monitoring component; any claim that its commands control a real safety function would require separate system and software assurance work. Zynq PS, clock/reset infrastructure, and vendor IP remain explicit dependencies in the assurance case, not implicitly trusted elements.

## 2. V1 architecture and interfaces

| Interface or domain | V1 choice | Purpose / constraint |
|---|---|---|
| Redundant source A | External SPI source/emulator, independent SCLK A; framed sample + source ID + sequence + CRC | Validate in source domain; transfer complete records into gateway domain. |
| Redundant source B | Second independent SPI source/emulator, independent SCLK B | Same format, separate physical and logical channel. Shared board and power are documented common causes. |
| Gateway/PS AXI clock | PL system clock, nominal 100 MHz | Health, arbitration, event log, AXI registers, and PS access. Exact clocks and reset tree frozen at architecture review. |
| ARINC-like input | 32-bit word, label filter, parity, timing model | Digital-protocol demonstrator sampled in a defined PL clock domain. A real ARINC 429 electrical interface requires an appropriate transceiver, electrical compliance, and access to the specification. |
| PS control/output | AXI4-Lite status/configuration; interrupt; initially bounded register or FIFO readout | Bare-metal C monitor, event retrieval, and fault campaign control. DMA/Ethernet is a stretch feature. |
| Commands | Test/configure channel, request reset, set permissible threshold within constrained limits | Reject malformed, out-of-range, and misaddressed requests; unsafe actuation is outside V1. |

**Data path:** source framing/integrity → per-source dual-clock FIFO → freshness/sequence/health checks → redundant-source selector → validity-gated telemetry output → PS-visible interface. The event logger separately records detection and selection changes with a gateway-domain timestamp. **Control path:** PS requests → checked AXI registers → command validator → bounded configuration or local demonstration action → acknowledgment/status. Define timestamp origin, rollover, overflow policy, and resets explicitly.

**Clocking rule:** do not infer a free-running “protocol clock” from ARINC data. Use a documented reference clock and bit-timing logic. For SPI, verify that the chosen external SCLKs, pin assignments, I/O voltage, setup/hold constraints, and reset behavior work on the actual board. Inventory every crossing including reset, status, interrupts, and multi-bit configuration; use a suitable handshake or atomic transfer for multi-bit controls. For a FIFO, analyze full/empty behavior on each side and reset sequencing. Treat CDC reports as structural evidence and investigate warnings; simulation cannot demonstrate metastability absence. [AMD UG906 CDC report](https://docs.amd.com/r/en-US/ug906-vivado-design-analysis/Report-Clock-Domain-Crossings) and [UG903 constraints](https://docs.amd.com/r/en-US/ug903-vivado-using-constraints/Constraining-Asynchronous-Signals) inform implementation.

**IP decision record:** compare a transparent custom FIFO with AMD `XPM_FIFO_ASYNC` as a reference implementation. Safety-relevant behavior needs requirements, configuration control, and verification regardless of whether the RTL is custom or vendor supplied. Do not presume custom code is automatically safer than a tested primitive. Use PS7/AXI infrastructure pragmatically, record versions/configuration, and assess their effect at the defined boundary. See [AMD XPM FIFO documentation](https://docs.amd.com/r/en-US/ug953-vivado-7series-libraries/XPM_FIFO_ASYNC) and AC 20-152A's COTS IP guidance.

## 3. Preliminary fault model and measurable behavior

The following limits are **initial engineering targets**, not safety-derived requirements. Freeze or revise them after defining source periods, maximum latency, acceptable data age, clock tolerances, and output consumer behavior.

| Injected condition | Required V1 response | Evidence |
|---|---|---|
| CRC/parity error | Discard the affected record; increment a saturating counter and log it; never assert its valid output. | Bad-frame directed test and assertion. |
| Duplicate/out-of-order sequence | Reject or explicitly flag per defined wraparound rule; prevent replay from appearing fresh. | Sequence boundary tests. |
| Missing frames / stale primary | Detect after an example **20 ms** freshness deadline; invalidate immediately upon detection. | Parameterized timeout test, counter boundary, timestamped trace. |
| Primary fails, backup fresh | Switch to backup within an example **1 ms after fault declaration**; emit only a complete backup record; log one selection transition. | Fault injection and measured detection-to-selection latency. End-to-end latency also includes the timeout. |
| Both sources invalid / contradictory | Set output invalid, log the condition, assert PS-visible fault; disagreement policy must be specified before automatic selection is allowed. | Simultaneous fault tests; no fabricated continuity. |
| FIFO overflow/underflow | Set sticky diagnostic; prevent partial or stale record presentation; define recovery and event logging. | Bursty/asymmetric-clock stress. |
| Fault in PS communications | PL validity gating remains enforced; mark PS status stale where observable. | PS pause/reset tests under explicit power/clock assumptions. |
| Clock stops / reset anomaly | Specify what an **independently clocked** observer can detect; on loss of the only monitoring clock, no detection-time guarantee is claimed. | Clock interruption and reset-recovery tests; residual-risk entry. |

Coverage of injected faults is constrained by the fault model and injector location: a fault inserted *before* CRC exercises a different failure mechanism from corruption inside a FIFO or selector. Maintain a fault-to-observation matrix. Two sensors sharing the board, power, firmware image, or upstream signal do not provide independence against those common-cause failures.

### Example requirement chain (draft only)

| ID | Statement / link |
|---|---|
| SYS-SAF-001 | A telemetry record with failed integrity shall not be presented as valid to the consumer. |
| FPGA-HLR-014 | The PL shall suppress validity for an input record when its CRC check fails. Parent: SYS-SAF-001. |
| FPGA-LLR-014A | After the final CRC bit is evaluated, the source-channel record shall be rejected before enqueueing; the diagnostic counter shall update on the defined source-clock edge. Parent: FPGA-HLR-014. |
| Design | `source_frame_checker`, FIFO write-enable gate, CRC polynomial/bit-order specification. |
| Verification | `TC-INT-014`, `SVA-INT-014`, coverage item `COV-INT-014`, hardware replay `HW-INT-014`; record versions and actual results. |

Review requirements in **both directions**: each safety/system need has implementation and verification, and each HDL element/test has an intended requirement or documented rationale. A passing regression does not validate a bad requirement. Keep injected-fault controls physically or logically isolated from normal operation and make their disabled state observable.

## 4. Lifecycle evidence and repository structure

| Evidence family | Deliverables |
|---|---|
| Safety and scope | Concept of operations, assumed aircraft context, interface/control boundary, illustrative FHA, fault-tree/FMEA notes, assumptions and residual risks. |
| Planning and standards | PHAC-*style* exercise plan, requirements standard, RTL/CDC/reset/design standard, verification and validation plans, configuration management and process assurance plans. Label all as project artifacts, not submitted certification material. |
| Development | Baselined FPGA high-level and detailed requirements, architecture and state diagrams, interface definitions, clock/reset and CDC inventories, design reviews, controlled RTL/constraints/scripts. |
| Verification | Requirement-based test procedures/results, SVA, functional and structural coverage analysis, CDC and timing reports, fault-injection matrix, hardware test logs, independent review findings and dispositions. Analyze exclusions and uncovered elements. |
| Reproducibility | Hardware Configuration Index (HCI)-style inventory: RTL, constraints, IP settings, bitstream/hash, build scripts, board revision; Hardware Environment Configuration Index (HECI)-style tool/OS versions. |
| Closure | Bidirectional trace matrix, discrepancy log, change-impact records, release checklist, Hardware Accomplishment Summary–style account of completed work and limitations. |

Suggested repository folders: `assurance/`, `requirements/`, `architecture/`, `rtl/`, `verification/`, `constraints/`, `software/`, `hardware_tests/`, `releases/`. Keep a tagged baseline at each review gate. A classmate/colleague can independently review representative requirements and tests; record review identity, independence limitations, comments, and dispositions. One person's self-review must be described honestly.

**Coverage goal:** every released FPGA requirement has an accepted verification method and an actual result; review structural HDL coverage (statements/branches/conditions/FSMs as available) for gaps and explain exclusions. Percent coverage alone does not prove completeness. Archive reproducible tool output and a bench-log subset of critical behaviors.

## 5. Six-month development roadmap and exit gates

Months 1–6 are one continuous engineering effort. The weeks below are approximate; keep the release gate within six calendar months, allowing the final two weeks for integration contingency and paper-ready data. Conference scanning and experimental design run alongside engineering, while the dedicated manuscript-writing month starts after the hardware release.

| Weeks | Work | Gate and reviewable exit evidence |
|---|---|---|
| 1–3 | ConOps, FHA exercise, safety boundary, source and fault model, interface/clock inventory; write PHAC-style plan and standards; draft requirements and verification strategy. | **G0 — concept baseline:** reviewed safety assumptions, testable requirements, assurance scope, and change control. No safety RTL depends on an unrecorded assumption. |
| 4–6 | Board and source-emulator bring-up; SPI framing, CRC, reset strategy, CDC FIFO prototype; testbench and build automation. | **G1 — acquisition:** two truly asynchronous source domains transfer complete frames with no silent corruption in tested conditions; CDC and constraints inventory reviewed. |
| 7–9 | Digital ARINC-like word receiver/formatter, parity/label filtering and bit-timing tests; clarify exact standard access and physical-interface boundary. | **G2 — protocols:** verified word vectors and interface definitions; no claim of electrical ARINC compliance. |
| 10–13 | Health states, timeouts, sequence/freshness logic, source-selection policy, validity gating, sticky events and safe reset behavior. | **G3 — FDIR:** fault-to-response table demonstrated, both-source-invalid case inhibited, bounded latencies measured from defined reference events. |
| 14–17 | AXI register map, interrupts, constrained command decoder and bare-metal C monitor; PS/PL reset and handshake cases. | **G4 — system integration:** PS cannot bypass validity gate; representative controls and telemetry operate on Zybo Z7-20. |
| 18–21 | Requirements-driven regression, SVA, review of coverage holes, CDC/timing closure, fault campaign, hardware replays and independent review. | **G5 — verification:** each baselined requirement has disposition, recorded result, trace links, and open discrepancies. |
| 22–24 | Close findings; measure resource use, timing, latency, fault coverage within the stated fault model, and recovery behavior. Freeze tables/figures and conference shortlist; finalize HCI/HECI, instructions, demonstration and limitations. | **G6 — evidence freeze:** reproducible experimental dataset and draft paper figures tied to a tagged hardware baseline. |
| 25–26 | Resolve integration contingency, rerun only affected evidence, package design/verification release, and hold project review. Preserve the data and scripts required to regenerate every figure. | **G7 — six-month project release:** working Zybo demonstration, released evidence package, results audit, and a defensible paper claim. |

**Minimum bench setup:** Zybo Z7-20, two independently clocked SPI-capable source emulators or generators, a host for logs/fault scripts, USB/UART connection, and safe level-compatible wiring. Confirm external clock and I/O feasibility before ordering anything. A real ARINC 429 transceiver is optional and outside V1. [Zybo Z7 reference manual](https://digilent.com/reference/_media/reference/programmable-logic/zybo-z7/zybo-z7_rm.pdf) is the board authority. Additional independent clock hardware may be needed to demonstrate genuine clock-domain separation; generated clocks from one root illustrate CDC techniques but share a common clock-failure mode.

## 6. Month 7 — research paper and conference submission

**Paper objective:** report a verified multi-clock telemetry gateway with explicit fault detection, safe invalidation, bounded failover under stated assumptions, and measured hardware costs. The DO-254-informed workflow is a development method and an evidence trail, **not** a claim of certification. The paper needs a real technical question and evaluation, such as the latency/resource tradeoff of redundancy and FDIR; a catalog of RTL blocks or assurance documents alone is insufficient.

| Timing | Paper work | Deliverable |
|---|---|---|
| During project Months 4–6 | Review closely related FPGA avionics gateways/FDIR studies, identify the specific contribution, preselect venues, define fair baselines and measurement scripts, and capture raw data as experiments run. | Related-work matrix, target-venue shortlist, reproducible experiment plan and results by G7. |
| Month 7, week 1 | Choose a conference with a deadline **after** the paper is ready; check topic fit, IEEE Conference Publications Program/Xplore intentions, page limit, peer review, presentation format, fees, and travel/visa practicality against its current author instructions. Build figures and paper outline. | Venue decision, author template, complete figure/table inventory and outline. |
| Month 7, week 2 | Draft architecture, methods, safety assumptions, verification method, results, related work, and limitations. Explain exactly how fault coverage and timing were measured. | Complete first manuscript, with all measurements linked to released runs. |
| Month 7, week 3 | Get technical review from 2–3 colleagues if available; check novelty, evidence, prose, citations, comparison fairness, and unsupported safety claims. Revise and run the conference's format/plagiarism checks where available. | Reviewed manuscript and recorded responses to substantial comments. |
| Month 7, week 4 | Final proof, coauthor sign-off where relevant, submission portal checks and **conference submission before its deadline**. Archive submitted PDF, source, figures, and submission confirmation. | Submitted paper; project and manuscript source remain reproducible. |

**Publication lifecycle after Month 7:** await peer review and decision; if accepted, address reviews, prepare camera-ready files, register and present in the conference's permitted format, then verify proceedings submission and eventual IEEE Xplore appearance. These steps can extend well beyond the writing month and are controlled partly by the conference and IEEE. IEEE says the conference committee sends accepted papers to IEEE, and IEEE reviews the final proceedings for scope and quality before posting; conference selection or acceptance alone does not guarantee Xplore publication. [IEEE Author Center: View Your Paper in IEEE Xplore](https://conferences.ieeeauthorcenter.ieee.org/get-published/view-your-paper-in-ieee-xplore/) and [IEEE guidance on conference selection/presentation](https://conferences.ieeeauthorcenter.ieee.org/become-an-ieee-conference-author/find-an-ieee-conference/).

**Paper evidence checklist:** architecture and clock/reset diagram; traceable representative safety requirements; fault model/injector placement; timing of detection, isolation and selection with distributions or bounds; measurement setup and clock assumptions; clean and faulted telemetry results; LUT/FF/BRAM/DSP use and achieved clock; fair comparison to relevant literature; limitations and reproducibility details. Distinguish safety behavior demonstrated on a development board from system-level safety assurance. Use the VCL writing style: concise claims, exact comparative language, properly placed figures/tables, and explicit evidence for every quantitative assertion.

## 7. Precursor reading plan (publicly accessible)

Read in this order, making a one-page note with **what applies to the demonstrator**, **what requires a real certification context**, and **one question to resolve** after each item.

| Priority | Source | Read for | Immediate exercise |
|---|---|---|---|
| 1 | [FAA AC 20-152A — Development Assurance for Airborne Electronic Hardware](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_20-152A.pdf) (free PDF) | Applicability, custom-device objectives, validation/verification, and COTS IP treatment. It recognizes DO-254; it does not reproduce its entire text. | List five objectives or practices that change how you would develop the gateway. |
| 2 | [FAA AC 00-72 — Best Practices for AEH Design Assurance](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_00-72.pdf) (free PDF) | Readable examples for complex devices, reviews, coverage and configuration information; complementary guidance. | Draft your HCI/HECI inventory. |
| 3 | [NASA Systems Engineering Handbook](https://www.nasa.gov/reference/systems-engineering-handbook/) and its [requirements verification matrix appendix](https://www.nasa.gov/reference/appendix-d-requirements-verification-matrix/) (free) | Need → requirement → design → verification, and distinction between validation and verification. NASA process is useful practice, not a substitute for DO-254. | Write five atomic FPGA requirements and a trace row. |
| 4 | [AMD UG906 — Report Clock Domain Crossings](https://docs.amd.com/r/en-US/ug906-vivado-design-analysis/Report-Clock-Domain-Crossings), [UG903 — Constraining Asynchronous Signals](https://docs.amd.com/r/en-US/ug903-vivado-using-constraints/Constraining-Asynchronous-Signals), and [UG953 — XPM_FIFO_ASYNC](https://docs.amd.com/r/en-US/ug953-vivado-7series-libraries/XPM_FIFO_ASYNC) (free) | CDC structures, report limitations, reset/FIFO behavior, and timing exceptions. Match the docs to your installed Vivado release before implementing. | Draw A→gateway and B→gateway CDCs, including status and resets. |
| 5 | [NASA: Fault Management in Manned Spacecraft](https://ntrs.nasa.gov/citations/20090024819) (free presentation) | Fault detection, isolation, recovery, and operational tradeoffs. Spacecraft context differs from aircraft certification. | Specify the response when both sensors disagree. |
| 6 | [Digilent Zybo Z7 board reference manual](https://digilent.com/reference/_media/reference/programmable-logic/zybo-z7/zybo-z7_rm.pdf) (free PDF) | Clock sources, connectors, voltage limits, reset/boot and PS/PL integration. | Draw the actual bench wiring and voltage/clock assumptions. |

**Standards to obtain through a legitimate library/employer/SAE/RTCA channel if available:** RTCA **DO-254 / EUROCAE ED-80** for the normative development-assurance guidance and **ARINC 429 Part 1** for exact word, timing, and electrical requirements. [FAA AC 20-152A](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_20-152A.pdf) is freely accessible, whereas the full DO-254 and [ARINC 429 specification](https://www.sae.org/standards/arinc429p1-19-429p1-19-digital-information-transfer-system-dits-part-1-functional-description-electrical-interfaces-label-assignments-word-formats) are standards publications that may require purchase/access. Until the ARINC specification is available, label the V1 block **ARINC-like digital word demonstrator** and document every simplified assumption.

## 8. First design session checklist

1. Choose the fictional monitored subsystem and explicitly define the downstream consumer's valid/invalid behavior.
2. Decide source update period, maximum acceptable data age, sample disagreement tolerance, and the conditions for failover versus inhibit.
3. Define clock ownership and reset hierarchy, including what detects loss of each domain.
4. Approve the source frame fields, CRC parameters, sequence-wrap behavior, timestamp origin, and output atomicity.
5. Write the first ten system/FPGA requirements and their verification methods before implementing safety-relevant RTL.

### Source basis

This plan adapts the attached FPGA learning roadmap and AMR phase outline for the user's second domain-expansion project, preserving sensor interfacing, real-time integration, hardware demonstration, and PS/PL work while making avionics assurance the new focus. The user's earlier many-core paper is useful context for independently clocked communication, but does not provide DO-254 compliance evidence for this gateway.
