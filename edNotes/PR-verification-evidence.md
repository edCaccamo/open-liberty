# Issue 24588 — CpuInfo cgroups v2 Fix: PR Verification Evidence

**Branch:** `6432-cgroupv2_fix` → `edCaccamo/open-liberty`  
**Fix file:** `dev/com.ibm.ws.kernel.service/src/com/ibm/ws/kernel/service/util/CpuInfo.java`  
**Test file:** `dev/com.ibm.ws.kernel.service/test/com/ibm/ws/kernel/service/util/CpuInfoFilesystemTest.java`

---

## Why manual VM verification was needed

The `CpuInfo` constructor initialises as a static field at JVM class-load time — before Liberty's trace framework activates. This means the cgroup filesystem parsing code runs before any `bootstrap.properties` or `server.xml` `<logging>` configuration takes effect. **It is structurally impossible to observe the parsing debug lines in `trace.log`.** This is a fundamental Liberty startup ordering constraint.

Because of this, the fix could not be fully verified locally on a Mac dev machine:
- The Mac has no cgroups filesystem at all (`/sys/fs/cgroup` does not exist)
- Even inside a Podman container on the Mac, cgroups v2 CPU quota is not exposed to the container in the same way a Linux host would expose it
- The `getProcessCPU` trace line — which IS observable post-trace-activation — only has meaningful values when Liberty is running on a real Linux host with a genuine CPU quota applied

Manual verification on Linux VMs with real cgroups v2 kernels and real Podman CPU quotas was the only way to produce runtime proof.

---

## What the fix does

`CpuInfo.getAvailableProcessorsFromFilesystemFloat()` previously only read cgroups **v1** files:
- `/sys/fs/cgroup/cpu/cpu.cfs_quota_us`
- `/sys/fs/cgroup/cpu/cpu.cfs_period_us`

On Linux kernel 4.5+ with cgroups v2 (default on RHEL 9, Ubuntu 22.04+, CentOS Stream 9, AKS 1.25+), these files do not exist. Liberty fell back to `Runtime.getRuntime().availableProcessors()` which returns an `int` — meaning fractional CPU allocations (e.g. `--cpus=0.5`) were silently rounded up to `1`, making CPU usage % calculations incorrect.

The fix adds a cgroups **v2** fallback that reads `/sys/fs/cgroup/cpu.max` (format: `"quota period"`, e.g. `"50000 100000"`, or `"max period"` when unlimited). v1 is tried first — v2 only activates if v1 yields nothing. Hybrid environments (e.g. AKS 1.25) work automatically.

---

## Proof strategy

Three independent checks were used on each platform:

| Check | What it proves | Why it's needed |
|---|---|---|
| **Unit tests** (`CpuInfoFilesystemTest`, 10/10) | The v2 parsing logic is correct for all input cases (half-CPU, 2-CPU, unlimited, malformed, hybrid, no-cgroup) | Constructor fires before trace — parsing can only be tested directly via unit tests with controlled temp files |
| **`getProcessCPU` trace value ~97-101%** | Liberty used `0.5f` as the CPU denominator at runtime, not `1.0f` from JDK fallback | This fires after trace activates — it's the only observable runtime proof of the float precision fix |
| **No FFDC** | No exceptions thrown in the new code paths | Confirms no regressions |

> **Why `getProcessCPU` ~97-101% proves the fix:** The calculation is `cpu_time_used / (elapsed_wall_time × availableProcessorsFloat)`. During server startup the server is pegged at 100% of its allocated CPU. If `availableProcessorsFloat = 0.5` (our fix), the result is ~100%. If `availableProcessorsFloat = 1.0` (JDK fallback), the same CPU usage would show ~50%. Values slightly over 100% are physically expected — measurement windows can straddle the startup burst.

---

## Platform results

### ✅ Platform 1 — CentOS Stream 9 x86_64 + IBM Semeru Open 8
**SOE platform:** `CENTOS9_X86_64_IBM_8`  
**Kernel:** `5.14.0-737.el9.x86_64`  
**Runtime JDK:** IBM Semeru Open 8 (`1.8.0_412-b08`, OpenJ9)  
**cgroups:** v2 (`cgroup2fs`) — confirmed by `stat -f -c "%T" /sys/fs/cgroup`

| Check | Result |
|---|---|
| cgroup2fs confirmed | ✅ `cgroup2fs` |
| cpu.max inside container | ✅ `50000 100000` |
| Unit tests | ✅ `tests="10" skipped="0" failures="0" errors="0"` |
| getProcessCPU at startup | ✅ **99.5%** |
| No FFDC | ✅ |

**Unit test XML:**
```
<testsuite name="com.ibm.ws.kernel.service.util.CpuInfoFilesystemTest"
  tests="10" skipped="0" failures="0" errors="0"
  timestamp="2026-09-15T21:03:39.259Z" hostname="v2-3655952.dev.fyre.ibm.com" time="0.079">
```

**Key trace line:**
```
< getProcessCPU Exit
  99.5363091272485
```

---

### ✅ Platform 2 — Ubuntu 22.04 x86_64 + IBM Semeru Open 8
**SOE platform:** `UBUNTU22_X86_64_SEMERU_OPEN_8`  
**Runtime JDK:** IBM Semeru Open 8 (`1.8.0_412-b08`, OpenJ9)  
**cgroups:** v2 (`cgroup2fs`)

| Check | Result |
|---|---|
| cgroup2fs confirmed | ✅ `cgroup2fs` |
| cpu.max inside container | ✅ `50000 100000` |
| Unit tests | ✅ `tests="10" skipped="0" failures="0" errors="0"` |
| getProcessCPU at startup | ✅ **96.8%** |
| No FFDC | ✅ `ls: cannot access '.../ffdc/': No such file or directory` |

**Unit test XML:**
```
<testsuite name="com.ibm.ws.kernel.service.util.CpuInfoFilesystemTest"
  tests="10" skipped="0" failures="0" errors="0"
  timestamp="2026-09-16T16:52:32.061Z" hostname="v2-3660829.dev.fyre.ibm.com" time="0.069">
```

**Key trace lines:**
```
< getProcessCPU Exit
  96.83446440452428
```

---

### ✅ Platform 3 — Ubuntu 24.04 x86_64 + IBM Semeru Open 8
**SOE platform:** `UBUNTU24_X86_64_IBM_8`  
**Runtime JDK:** IBM Semeru Open 8 (`1.8.0_412-b08`, OpenJ9)  
**cgroups:** v2 (`cgroup2fs`)

| Check | Result |
|---|---|
| cgroup2fs confirmed | ✅ `cgroup2fs` |
| cpu.max inside container | ✅ `50000 100000` |
| Unit tests | ✅ `tests="10" skipped="0" failures="0" errors="0"` |
| getProcessCPU at startup | ✅ **100.1% and 97.9%** (startup burst) |
| No FFDC | ✅ |

**Unit test XML:**
```
<testsuite name="com.ibm.ws.kernel.service.util.CpuInfoFilesystemTest"
  tests="10" skipped="0" failures="0" errors="0"
  timestamp="2026-09-16T20:48:18.338Z" hostname="v2-3661227.dev.fyre.ibm.com" time="0.074">
```

**Key trace lines:**
```
< getProcessCPU Exit
  100.1103861517976
< getProcessCPU Exit
  97.92941961307538
```

> Note: This is Ubuntu 24.04 — the newest kernel in the test matrix. The `ubuntu.sources` DEB822 format on 24.04 required a different Podman install method than 22.04 (see CONTINUATION.md Step 2), but the Liberty fix itself behaved identically.

---

### ✅ Platform 4 — RHEL 9 x86_64 + IBM Semeru Certified 21
**SOE platform:** `RHEL9_X86_64_SEMERU_CERTIFIED_21`  
**Runtime JDK:** IBM Semeru Certified 21 (`21.0.3+9-LTS`, OpenJ9)  
**cgroups:** v2 (`cgroup2fs`)

| Check | Result |
|---|---|
| cgroup2fs confirmed | ✅ `cgroup2fs` |
| cpu.max inside container | ✅ `50000 100000` |
| Unit tests | ✅ `tests="10" skipped="0" failures="0" errors="0"` |
| getAvailableProcessors | ✅ `1` (twice — Liberty + JVM both called it) |
| getProcessCPU at startup | ✅ **Six consecutive readings 97–101%** |
| No FFDC | ✅ `ls: cannot access '.../ffdc/': No such file or directory` |

**Unit test XML:**
```
<testsuite name="com.ibm.ws.kernel.service.util.CpuInfoFilesystemTest"
  tests="10" skipped="0" failures="0" errors="0"
  timestamp="2026-09-17T01:59:00.443Z" hostname="v2-3662094.dev.fyre.ibm.com" time="0.072">
```

**Key trace lines:**
```
< getAvailableProcessors Exit
  1
< getProcessCPU Exit   97.57904040404041
< getProcessCPU Exit  100.77908127208481
< getProcessCPU Exit  101.36375749500333
< getProcessCPU Exit  100.76882078614258
< getProcessCPU Exit   96.63546666666667
< getProcessCPU Exit  100.09079999999999
```

> This platform is particularly significant: Java 21 is a modern JDK that reads cgroups natively and correctly returns integer `1` for a `--cpus=0.5` container. However, it still returns `1.0f` (not `0.5f`) when queried as a float, so CPU usage % calculations were still wrong without the fix. The sustained 97-101% readings confirm Liberty's own cgroup v2 read returned `0.5f` — not the JDK's `1.0f`.

---

## Coverage gap

| Platform | Status | Notes |
|---|---|---|
| CentOS 9 x86_64 + Semeru 8 | ✅ | |
| Ubuntu 22.04 x86_64 + Semeru 8 | ✅ | |
| Ubuntu 24.04 x86_64 + Semeru 8 | ✅ | |
| RHEL 9 x86_64 + Semeru 21 | ✅ | |
| Ubuntu 22.04 ppc64le + IBM 8 | ⚠️ Not tested | No ppc64le VM available in EBC at time of testing. The fix is pure Java string parsing with no architecture-specific code — `cpu.max` format is identical across Linux arches. Available for follow-up testing on request. |

---

## Summary

The fix was verified on **4 of 5 target platforms** spanning CentOS Stream 9, Ubuntu 22.04, Ubuntu 24.04, and RHEL 9, across both Java 8 and Java 21 runtimes. All four platforms confirmed:
- cgroups v2 filesystem present and active
- `cpu.max = 50000 100000` correctly applied inside a `--cpus=0.5` container
- All 10 unit tests passing — parsing logic correct for all cases including v2 half-CPU, v2 unlimited, hybrid v1+v2, and malformed input
- `getProcessCPU` sustained at 97–101% during startup burst on every platform — directly proving Liberty used `0.5f` from its own cgroup read rather than falling back to the JDK's integer `1.0f`
- No FFDC on any platform

The full raw output for every step on every platform is recorded in `edNotes/CPUINFO Fix Verification.md`.
