# STATUS.md — Task Tracking

> This file is the **single source of truth** for the current task.
> All agents read from and write to designated sections only.
> Orchestrator maintains the top-level log and makes all routing decisions.

---

## Task

**Description:** Increase Pollard Kangaroo search interval from 125-bit to 256-bit, improve speed and correctness across GPU targets.

**Started:** 2026-02-26T12:14:49Z

**Branch:** copilot/increase-search-interval-to-256bit

---

## Active Agent Chain

_[Orchestrator fills this after routing rule evaluation]_

```
[ ] Orchestrator
[ ] Architect          — trigger: New feature / interval extension
[ ] Coder
[ ] QA                 — trigger: Behaviour change requires regression coverage
[ ] Security           — trigger: not applicable
[ ] Performance        — trigger: Hot path speed requirements
[ ] DX-CI              — trigger: not applicable
[ ] Docs               — trigger: not applicable
[ ] Refactor           — trigger: not applicable
[x] Auditor            — always last
```

---

## SCOPE

_[Filled by Architect or Orchestrator for Fast-path]_

**In scope:**
- 

**Out of scope:**
- 

**Affected modules/files:**
- 

---

## DESIGN

_[Filled by Architect]_

### Architecture
_[Description + diagrams]_

### Acceptance Criteria
- [ ] AC-01: 
- [ ] AC-02: 

### Run / Test Commands
```bash
# Build
# Test
```

### Design Status
```
STATUS: IN_PROGRESS
AGENT: architect
PHASE: design
TIMESTAMP: 
```

---

## TEST PLAN

_[Filled by QA agent if triggered]_

### Acceptance Criteria → Test Mapping
| AC | Test ID | Test Name | Type |
|---|---|---|---|
| | | | |

### Edge Cases
| Scenario | Expected Behaviour |
|---|---|
| | |

### Test Plan Status
```
STATUS: IN_PROGRESS
AGENT: qa
PHASE: test-plan
TIMESTAMP: 
```

---

## IMPLEMENTATION

_[Filled by Coder]_

### Changes Made
| File | Change Type | Description |
|---|---|---|
| | | |

### Test Results
```
(paste test output)
```

### Acceptance Criteria Status
- [ ] AC-01: 
- [ ] AC-02: 

### Implementation Status
```
STATUS: IN_PROGRESS
AGENT: coder
PHASE: implementation
TIMESTAMP: 
```

---

## SECURITY REVIEW

_[Filled by Security agent if triggered]_

### Findings
_None_

### Security Review Status
```
STATUS: IN_PROGRESS
AGENT: security
PHASE: security-review
TIMESTAMP: 
```

---

## PERF REVIEW

_[Filled by Performance agent if triggered]_

### Findings
_None_

### Perf Review Status
```
STATUS: IN_PROGRESS
AGENT: performance
PHASE: perf-review
TIMESTAMP: 
```

---

## BUILD/CI

_[Filled by DX-CI agent if triggered]_

### Changes Made
| File | Change | Reason |
|---|---|---|
| | | |

### Build/CI Status
```
STATUS: IN_PROGRESS
AGENT: dx-ci
PHASE: build-ci
TIMESTAMP: 
```

---

## DOCS

_[Filled by Docs agent if triggered]_

### Changes Made
| File | Change | Description |
|---|---|---|
| `README.md` | replaced | Full rewrite: comprehensive project documentation for Kangaroo v2.2 — algorithm explanation, requirements, CPU/GPU build instructions, input file format, CLI options table, usage examples, work-file management, distributed mode, Bitcoin puzzle context, performance notes, license |

### Public Interface Changes Documented
- [x] All new/changed CLI flags documented in README (full options table).
- [x] All new/changed env variables added to `.env.example` — N/A (no env variables used).
- [x] All breaking changes have a migration guide — N/A (no breaking changes; initial docs write).
- [x] CHANGELOG entry added — N/A (no CHANGELOG.md present in repository).

### Docs Status
```
STATUS: VERIFIED
AGENT: docs
PHASE: documentation
TIMESTAMP: 2025-01-30T00:00:00Z
DETAILS: README.md replaced with comprehensive Kangaroo v2.2 documentation (551 lines)
```

---

## REFACTOR

_[Filled by Refactor agent if triggered]_

### Changes Made
| File | Pattern Applied | Description |
|---|---|---|
| | | |

### Refactor Status
```
STATUS: IN_PROGRESS
AGENT: refactor
PHASE: refactor
TIMESTAMP: 
```

---

## AUDIT

_[Filled by Auditor — always last]_

### Summary
| Category | Result | Notes |
|---|---|---|
| Acceptance Criteria Coverage | IN_PROGRESS | |
| Test Quality | IN_PROGRESS | |
| Code Correctness | IN_PROGRESS | |
| Security Basics | IN_PROGRESS | |
| Build & Test Execution | IN_PROGRESS | |
| Write-Zone Compliance | IN_PROGRESS | |
| STATUS.md Integrity | IN_PROGRESS | |

### Defects
_None yet_

### Audit Status
```
STATUS: IN_PROGRESS
AGENT: auditor
PHASE: audit
TIMESTAMP: 
```

---

## Orchestrator Log

| Timestamp | Event |
|---|---|
| 2026-02-26T12:14:49Z | Task started; routed chain: Orchestrator → Architect → QA → Coder → Performance → Auditor |
