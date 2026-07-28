# E404 R15 BPF and Android 16 memory cgroups

This device uses E404 R15 BPF on LineageOS 23.2 / Android 16.  Android 16's
`init` tries to activate the cgroup v2 `memory` controller for per-UID process
groups.  The shipped R15 Alioth configuration does not provide that controller,
so the system log contains repeated messages such as:

```
Activation of cgroup controller +memory failed ... Invalid argument
```

## What was verified

- The running kernel is E404 R15 BPF (`4.19.404R`, built 2026-05-16).
- The current working boot and vendor_boot images were backed up before any
  investigation; no test image was flashed.
- The R15 `main-bpf` source at commit `fed88a6024cd4d46c52eda607a95d97d7371ba81`
  contains the matching KernelSU submodule and the Alioth defconfig.
- `arch/arm64/configs/vendor/alioth_defconfig` sets `CONFIG_MEMCG` off.
- Enabling the memory-cgroup options while keeping all KernelSU options on
  causes the kernel to fail to compile in `mm/memcontrol.c`.  This source tree
  has incomplete/incompatible memory-controller code for that configuration.

## Decision

Do not flash a locally modified E404 kernel just to suppress this log line.
Doing so would require porting and testing a substantial memory-cgroup change,
and an untested kernel could break boot, radio, camera, or KernelSU.

The safe state is to keep the verified R15 BPF kernel with KernelSU intact.
The log is a compatibility limitation between this legacy 4.19 kernel design
and Android 16's cgroup-v2 expectations, not a Leica Camera fault.
