# CIRCT Vulnerability Report - [Calyx] Segfault in SCFToCalyx when func.call exists inside loop body 

**Vulnerability ID:** CVE-PENDING  
**CVSS Score:** 4.7 (Medium)  
**Discovery Date:** 2026-01-18  
**Discoverer:** M2kar (@m2kar)  
**GitHub Issue:** https://github.com/llvm/circt/issues/9529   
**Fix PR:** https://github.com/llvm/circt/pull/9530

---

## Table of Contents

- [1. Vulnerability Overview](#1-vulnerability-overview)
- [2. Technical Details](#2-technical-details)
- [3. Reproduction](#3-reproduction)
- [4. Impact Analysis](#4-impact-analysis)
- [5. Remediation](#5-remediation)
- [6. CVE Classification](#6-cve-classification)
- [7. References](#7-references)

---

## 1. Vulnerability Overview

### 1.1 Description

`lower-scf-to-calyx` pass crashes with a segmentation fault when processing a function containing `func.call` operations inside a loop body. The crash occurs in `BuildControl::buildCFGControl` when constructing `mlir::SuccessorRange` with an invalid Block pointer.

When an `affine.for` loop containing `func.call` is lowered through `--lower-affine` and `--scf-for-to-while`, the resulting SCF while loop triggers a null pointer access during Calyx control flow construction.


### 1.2 Affected Scope

- **Affected Versions:** 
  - CIRCT firtool-1.139.0 and earlier
- **Affected Components:** CIRCT Calyx (SCFToCalyx)
- **Affected Scenarios:**

  - ❌`lower-scf-to-calyx` pass crashes with a segmentation fault when processing a function containing `func.call` operations inside a loop body.

---

## 2. Technical Details

### 2.1 Root Cause

 SCFToCalyx does not support `func.call` operations inside loop bodies

 `buildCFGControl` encounters `func.call` which introduces unexpected CFG edges or Block successor structure, leading to null pointer access in `SuccessorRange` constructor.
### 2.2 Error Signature

Compiler crashes in the lower-scf-to-calyx pass

More Details are in the error.log
### 2.3 Vulnerable Code Example

**Vulnerable Code (bug.mlir) - ❌ Crashes :**
```mlir

func.func private @ext()

func.func @f() {
  affine.for %i = 0 to 1 {
    func.call @ext() : () -> ()
  }
  return
}

```
---

## 3. Reproduction

### 3.1 Project Structure

```
circt-b2/
├── Dockerfile                 # Docker vulnerability reproduction environment
├── test.sh                    # Quick test script
├── bug.mlir                   # Vulnerable code
└── results/                   # Test output directory
  └──  error.log              # Vulnerable code error output
```

### 3.2 Quick Start

#### Method 1: Using Quick Script

```bash
# 1. Build image (first run)
./test.sh build

# 2. Run test
./test.sh run

# 3. Save output files to ./results/error.log
./test.sh save

```




### 3.3 Test Results

**Test Platform:** macOS (Apple M3 Pro) + Docker (linux/amd64)  
**Test Date:** 2026-01-21  
**CIRCT Version:** firtool-1.139.0

| Test | Expected | Actual | Status |
|------|----------|--------|--------|
| Vulnerable code (bug.mlir) | Complier Crash | Crash (Segfault) ❌ | ✅ PASS | |
| Error signature detection | lower-scf-to-calyx | Detected | ✅ PASS |

**Conclusion:** ✅ **Vulnerability Successfully Reproduced**

---

## 4. Impact Analysis

### 4.1 Functional Impact

| Category | Level | Description |
|----------|-------|-------------|
| Design Correctness |🔴 HIGH |Valid IR containing func.call inside loop bodies causes a segmentation fault during Calyx lowering, blocking correct control generation |
| Tool Interoperability | 🔴 HIGH | Breaks the pipeline, preventing integration with the Calyx backend. |
| Development Workflow | 🟡 MEDIUM | Requires manual refactoring to avoid func.call in loops before lowering. |
| Verification Coverage | 🟡 MEDIUM |Reduces backend robustness by preventing full coverage of valid SCF constructs. |

### 4.2 Security Implications

1. **Code Integrity Risk**
   - Required refactoring of loop bodies increases risk of unintended semantic changes
   - Manual control-flow restructuring may introduce new logic errors
2. **Supply Chain Vulnerability**
   - Automated hardware generation pipelines may produce standard-compliant code that fails compilation
   - Requires manual intervention, breaking automated workflows
   - Affects reproducibility of hardware builds

3. **Compiler Trust**
   - May mask or indicate presence of other undiscovered compilation issues
   - Reduces confidence in compiler's ability to correctly translate HDL specifications

### 4.3 Affected Use Cases

- **Calyx Backend Flows:** SCF-to-Calyx lowering pipelines involving loop-based control generation
- **Affine-to-SCF Pipelines:** Designs lowered via --lower-affine and --scf-for-to-while passes
- **Automated Synthesis Toolchains:** End-to-end MLIR → Calyx compilation flows in CI environments
- **Hardware Fuzzing and Robustness Testing::** Generated test cases containing func.call inside loop bodies

---

## 5. Remediation


Upgrade to CIRCT version containing PR #9530 fix:

```bash
git clone https://github.com/llvm/circt.git
cd circt
git checkout main  # Ensure PR #9530 is included
# Build according to official documentation
```

---

## 6. CVE Classification

### 6.1 CVSS v3.1 Scoring

**Vector String:**  
`CVSS:3.1/AV:L/AC:H/PR:N/UI:R/S:U/C:N/I:N/A:H`

**Base Score:** 4.7 (MEDIUM)

| Metric | Value | Rationale |
|--------|-------|-----------|
| Attack Vector (AV) | Local | Requires local access to compilation environment |
| Attack Complexity (AC) | High | Require specific IR structure and executing a sequence of lowering passes |
| Privileges Required (PR) | None | Any compilation user can trigger |
| User Interaction (UI) | Required | User must invoke the compliation pass on crafted IR |
| Scope (S) | Unchanged | Impact limited to compilation pipeline |
| Confidentiality (C) | None | No sensitive information is disclosed |
| Integrity (I) | None | Does not modify external code or system; only causes pass crash |
| Availability (A) | High | Compilation process is interrupted, causing denial of service until IR or pass is fixed |

### 6.2 CWE Classification

- **CWE-754:** Improper Check for Unusual or Exceptional Conditions
- **CWE-248:** Uncaught Exception
- **CWE-476:** NULL Pointer Dereference
### 6.3 Risk Assessment

**Risk Level:** 🟡 MEDIUM (4.7 CVSS 3.1)

**Recommended Priority:** MEDIUM - Should deploy fix in next maintenance cycle

**Rationale:**
- The compiler crash interrupts automated build and hardware generation workflows, resulting in denial of service during compilation
- Triggered by structurally valid IR containing SCF loops with func.call, exposing insufficient robustness in pass handling
- Requires manual modification of IR or pass sequencing to avoid the crash, increasing risk of human error and maintenance overhead
- Impact is confined to the development/compilation phase and produces explicit crash behavior (not silent miscompilation)
- No confidentiality breach or privilege escalation

---

## 7. References

### 7.1 GitHub Resources

- **Issue:** https://github.com/llvm/circt/issues/9529
- **Fix PR:** https://github.com/llvm/circt/pull/9530

### 7.2 Official Documentation

- **CIRCT:** https://circt.llvm.org/
- **LLHD Dialect:** https://circt.llvm.org/docs/Dialects/LLHD/
- **MLIR:** https://mlir.llvm.org/

### 7.3 Contributors

- **Reporter:** M2kar (@m2kar) kaituo-crypto (@kaituo-crypto )
- **Analysis:** M2kar (@m2kar)
- **Maintainer:**  Chris Gyurgyik (@cgyurgyik)
- **Fix Implementation:** M2kar (@m2kar)

---

## 8. Appendix

### 8.1 Test Environment

- **Operating System:** Ubuntu 24.04 (x86_64) in Docker
- **Platform:** macOS (Apple M3 Pro) with linux/amd64 emulation
- **CIRCT Version:** firtool-1.139.0
- **LLVM Version:** 22.0.0git

### 8.2 Generated Files

```
results/
└── error.log                   Vulnerable code error output
```

### 8.3 Disclosure Policy

This report follows coordinated disclosure practices:

1. ✅ Used public issue tracker (GitHub Issues) - vendor's preferred channel
2. ✅ Maintainers engaged, fix in progress (PR #9530)
3. ✅ No active exploitation observed (compiler toolchain bug)
4. ⏳ Awaiting CVE assignment and official vendor advisory

**Status:** Public disclosure with active fix development

---

## 9. Contact

**Reporter:** M2kar  
**GitHub:** [@m2kar](https://github.com/m2kar)  
**Email:** zhiqing.rui@gmail.com  
**Issue Tracker:** https://github.com/llvm/circt/issues/9529

---

**Document Version:** 2.0  
**Last Updated:** 2026-03-04  
**Status:** Ready for CVE Submission  
**License:** MIT
