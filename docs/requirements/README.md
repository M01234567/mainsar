# Requirements Baseline – MainSAR

| Field            | Value                                              |
|------------------|----------------------------------------------------|
| Document set     | Requirements baseline, version 1.0         |
| Date             | 2026-09-27                                         |
| Status           | **Approved** by product owner (2026-09-27)          |
| Owner            | Moritz Schäfer                                     |
| Next milestone   | Approval → spikes S-1, S-5, S-6                     |

## Purpose

This folder holds the requirements for an application that measures ground deformation
with **multi-temporal InSAR (MT-InSAR)** from Sentinel-1 data. It is modelled on
EZ-InSAR: the user picks an area and a time range, sets the processing parameters, and gets
a Persistent Scatterer (PS) displacement result (velocity map + time series).

These documents are the input for design and implementation. **Baseline approved on 2026-09-27.** Changes go through the decision log. The decisions are listed in the decision log below.

## Document map

| #  | Document | Content |
|----|----------|---------|
| 01 | [Vision & Scope](01_vision_and_scope.md) | Problem, goals, scope, non-goals, success criteria |
| 02 | [Domain & Method Analysis](02_domain_and_method_analysis.md) | InSAR/PS primer, comparison of processing approaches, **recommended approach** |
| 03 | [Stakeholders & Use Cases](03_stakeholders_and_use_cases.md) | Personas, use cases, user workflow |
| 04 | [Functional Requirements](04_functional_requirements.md) | All functional requirements with priority and acceptance criteria |
| 05 | [Non-Functional Requirements](05_non_functional_requirements.md) | Performance, resources, reliability, usability, security, maintainability |
| 06 | [Data & Interface Requirements](06_data_and_interface_requirements.md) | External data sources, credentials, input/output formats, project data model |
| 07 | [Processing Parameters](07_processing_parameters.md) | Parameter catalogue with defaults, ranges, and the stage each one belongs to |
| 08 | [System Architecture](08_system_architecture.md) | Proposed architecture, components, technology stack, architecture decisions (ADRs) |
| 09 | [Constraints, Assumptions & Risks](09_constraints_assumptions_risks.md) | Constraints, assumptions, risk register |
| 10 | [Verification & Validation](10_verification_and_validation.md) | Test strategy, reference datasets, acceptance tests |
| 11 | [Roadmap](11_roadmap.md) | Technical spikes, phases, MVP definition |
| 12 | [Open Questions](12_open_questions.md) | **Decisions needed from the product owner** |
| 13 | [Glossary](13_glossary.md) | Terms and abbreviations |
| 14 | [Installation, Hardware & Operations](14_installation_and_operations.md) | Recommended production hardware, installer requirements (incl. SNAPHU, StaMPS, MATLAB), operations |

## Decision log

| Date | Decision | Source |
|------|----------|--------|
| 2026-09-27 | Dolphin (PS/DS) default engine, StaMPS-PS second engine | Q-01 |
| 2026-09-27 | Users: university students and researchers with basic InSAR knowledge | Q-02/03 |
| 2026-09-27 | AOIs mostly in Europe, ≤ 100 km², time spans ≤ 5 years | Q-04 |
| 2026-09-27 | One dedicated production machine; the laptop is for development only | Q-05 |
| 2026-09-27 | One user at a time, no login; localhost + SSH tunnel | Q-05b |
| 2026-09-27 | Browser UI; application code in TypeScript; Python only in processing workers | Q-06, Q-17 |
| 2026-09-27 | MATLAB with individual, machine-bound licence → native (non-container) installation | Q-07 |
| 2026-09-27 | Public git repository, licence GPL-3.0, mainly internal use | Q-11, Q-18 |
| 2026-09-27 | The installer installs all dependencies; MATLAB is an optional, licensed prerequisite | PO |
| 2026-09-27 | Sentinel-1 only, including S1B archive data (all units A/B/C/D as one constellation) | Q-08 |
| 2026-09-27 | British English everywhere | Q-13 |
| 2026-09-27 | Team: Claude implements, product owner reviews | Q-14 |
| 2026-09-27 | Tool name "MainSAR" (`mainsar`), replacing "MinSAR" (name clash with an existing InSAR project) | Q-16 |
| 2026-09-27 | StaMPS on MATLAB only for now; Octave deferred | Q-20 |
| 2026-09-27 | Reference sites: Bologna (DS-REF), Lorca / Alto Guadalentín (DS-STRESS) | Q-10 |
| 2026-09-27 | Documented formats accepted; KMZ for Google Earth is a Must; own viewer in-app plus standalone export | Q-09 |
| 2026-09-27 | Mostly one-off studies; reprocessing must be easy (retained stacks, partial re-runs) | Q-15 |
| 2026-09-27 | Production machine: Ryzen 7 3700X, 32 GB RAM, 1 TB SSD, RTX 3080 Ti | Q-19 |
| 2026-09-27 | **PSInSAR and SBAS** with open-source software: SBAS engine (MintPy) added; MATLAB never mandatory | Q-21 |
| 2026-09-27 | Spike S-1 results: own verified download layer, per-date streaming coregistration, SBAS interferograms from the cropped stack, command-by-command execution of ISCE2 run files | S-1 |
| 2026-09-27 | Spikes S-2, S-3 (pre-MATLAB), S-5, S-6, S-7 done; findings applied (FR-AOI-06 → Must, SBAS defaults, split viewer data, StaMPS patch set, installer design). Open: S-3 MATLAB run, S-2 GPU, S-4 on the production machine | S-2…S-7 |

## Conventions

### Requirement identifiers

| Prefix     | Meaning                                  | Example        |
|------------|------------------------------------------|----------------|
| `FR-xxx-nn`  | Functional requirement, grouped by area  | `FR-AOI-01`    |
| `NFR-xxx-nn` | Non-functional requirement               | `NFR-PERF-02`  |
| `DR-xx`      | Data / interface requirement             | `DR-07`        |
| `C-nn`       | Constraint                               | `C-03`         |
| `A-nn`       | Assumption                               | `A-05`         |
| `R-nn`       | Risk                                     | `R-04`         |
| `ADR-nnn`    | Architecture decision record             | `ADR-003`      |
| `S-n`        | Technical spike (time-boxed experiment)  | `S-2`          |
| `Q-nn`       | Open question                            | `Q-01`         |

IDs are stable. Deleted requirements keep their ID and are marked *withdrawn*, never reused.

### Priorities (MoSCoW)

- **M – Must**: MVP cannot ship without it.
- **S – Should**: Important. Planned for the first release after the MVP, or the MVP if cheap.
- **C – Could**: Useful. Implement when capacity allows.
- **W – Won't (this time)**: Explicitly out of scope for now. Recorded so nobody has to ask again.

### Status markers

- `[TBC]` – *to be confirmed*: depends on an open question or a spike result.
- `[VERIFY]` – a fact about a third-party tool or data source that must be re-checked during the
  spikes (versions, APIs, and services change).

## How to review

1. Read **01** and **02** first. They set the direction; everything else follows from them.
2. Answer the questions in **12** (inline answers in the file are fine).
3. Go through **04** and adjust the priorities. Anything marked *Must* that you don't need
   makes the MVP bigger.
4. Mark anything wrong or missing directly in the files (or as comments), and the baseline will be revised.
