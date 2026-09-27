# 12 – Open Questions

## 1. Answered (2026-09-27)

| Q | Topic | Answer | Consequence |
|---|-------|--------|-------------|
| Q-01 | Engine order | Dolphin (PS/DS) first, StaMPS-PS second | ADR-004 accepted |
| Q-02/03 | Users & domain | University students and researchers with basic InSAR knowledge | Personas P1/P2 revised; strong in-app help and presets |
| Q-04 | AOI & time | Mostly Europe, ≤ 100 km², a few years, never > 5 years | AOI limit 100 km²; EGMS as validation reference; hardware sizing |
| Q-05 | Compute | One powerful dedicated machine; the laptop is for development only | HW-PROD spec in [14](14_installation_and_operations.md) §1; remote/distributed execution withdrawn |
| Q-05b | Usage mode | One user at a time | No auth; localhost binding + SSH tunnel (ADR-008) |
| Q-06 | UI | Browser application | ADR-002 accepted |
| Q-07 | MATLAB licence | Individual, machine-bound | Native installation, no processing containers (ADR-006) |
| Q-11 | Distribution | Mostly internal, public git repository | – |
| Q-17 | Backend language | TypeScript (Node.js); Python only in processing workers | ADR-001 accepted; worker contract (08 §2.2) |
| Q-18 | Repository licence | GPL-3.0 | NFR-LEG-01 |
| – | Installation | Installer provides all dependencies; MATLAB is an optional, licensed prerequisite | New doc [14](14_installation_and_operations.md), FR-INS-* |
| Q-08 | Sensors | Sentinel-1 only, but **S1B archive data must be included** (S1A/B/C/D as one constellation) | FR-DIS-08, 02 §1.1 |
| Q-12 | Accounts | NASA Earthdata account exists | S-1 can start |
| Q-13 | Language | British English everywhere | NFR-USE-04 |
| Q-14 | Team | Claude implements, product owner reviews | Roadmap: sequential, small reviewable increments |
| Q-16 | Name | "MainSAR" (for now) | CLI and packages `mainsar` |
| Q-09 | Outputs | Documented formats OK; must be viewable in **Google Earth** (KMZ → Must) and in MainSAR's **own viewer**, also as a standalone export; vertical/east-west decomposition raised to Should | FR-EXP-04, FR-EXP-08, FR-ANA-08 |
| Q-10 | Reference sites | Bologna (DS-REF) and Lorca / Alto Guadalentín (DS-STRESS) confirmed | 10 §4 |
| Q-15 | Usage pattern | Mostly one-off, but reprocessing must be easy | FR-PRJ-06, FR-ACQ-09, FR-PRE-07 → Must |
| Q-19 | Production machine | Ryzen 7 3700X, 32 GB, 1 TB SSD free, RTX 3080 Ti | 14 §1.2 assessment; NFR-PERF-01 relaxed |
| Q-21 | Techniques | PSInSAR **and SBAS** must be supported with open-source software | SBAS engine MintPy (FR-PRE-09, FR-ANA-12, FR-ANA-14), spike S-7 |
| Q-20 | Octave | Acceptable fallback, but out of scope for the first implementation; focus on MATLAB | FR-INS-13 (Could) |

## 2. Still open

None. New questions that come up during the spikes will be added here.
