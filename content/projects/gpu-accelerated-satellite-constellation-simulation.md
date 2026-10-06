---
title: "GPU-Accelerated Satellite Constellation Simulation"
description: "CUDA/C++17 satellite simulation with validated CPU/GPU consistency, reproducible benchmarking and up to 46.84× end-to-end acceleration for GPU-resident workloads."
---

# GPU-Accelerated Satellite Constellation Simulation

[← Back to projects](/#projects)

**Context:** Independent Performance Engineering Project · 2026  
**Focus:** CUDA · Parallel Computing · Benchmarking · Correctness Validation

I built a C++17/CUDA simulation for evaluating satellite motion and ground-point coverage on both CPU and GPU.

The project focuses on more than kernel acceleration. It compares serial CPU and CUDA execution under controlled workloads, separates kernel-level speedup from true end-to-end performance, validates CPU/GPU consistency, and measures how keeping simulation state resident on the GPU changes the cost of repeated timesteps.

[View source on GitHub →](https://github.com/LittleBigPluton/GPU-Accelerated-Satellite-Constellation-Simulation)

## Project snapshot

| Metric | Result |
|---|---|
| Implementation | C++17 · CUDA |
| Largest single-step workload | 1,000,000 satellites |
| CPU time, 1M satellites × 1 step | **93.0149 ms** |
| GPU kernel time, 1M satellites × 1 step | **1.9140 ms** |
| GPU end-to-end time, 1M satellites × 1 step | **11.0439 ms** |
| Single-step kernel speedup | **48.55×** |
| Single-step end-to-end speedup | **8.73×** |
| Largest resident workload | 1,000,000 satellites × 100 steps |
| Resident kernel speedup | **49.25×** |
| Resident end-to-end speedup | **46.84×** |
| Automated tests | **4** |
| Final benchmark validation | **All workloads passed** |

## The challenge

Satellite updates and coverage checks are naturally parallel across independent satellites, but raw kernel speed is only part of GPU performance.

A practical GPU implementation also has to account for:

- host-to-device transfers,
- device-to-host transfers,
- synchronization and runtime overhead,
- repeated use of the same simulation state,
- numerical consistency between CPU and GPU execution.

The project therefore focused on four questions:

- How naturally can the satellite workload be mapped to CUDA threads?
- How much acceleration comes from the GPU kernels themselves?
- How much of that speedup remains after transfer and runtime overhead?
- How should CPU and GPU correctness be validated while benchmarking performance?

## Simulation model

The simulator intentionally uses a simplified orbital model so that the project can isolate the computational structure of GPU acceleration.

Each satellite is represented by:

- orbital radius,
- angular velocity,
- current orbital angle.

The satellite follows a circular orbit in the Cartesian `x-y` plane:

```text
x = r cos(theta)
y = r sin(theta)
z = 0
```

The orbital angle evolves as:

```text
theta(t + dt) = theta(t) + omega * dt
```

and is wrapped into the interval `[0, 2π)`.

The goal is not high-fidelity astrodynamics. The simplified model provides a deterministic workload for comparing serial CPU execution with parallel CUDA execution.

## Ground-point coverage

Coverage is evaluated from the satellite-to-ground line-of-sight vector and the ground point's radial vector.

For satellite position `s` and ground position `g`:

```text
LOS = s - g
```

The elevation angle is computed from the line-of-sight direction relative to the local upward direction at the ground point.

A satellite is considered visible when its elevation exceeds the configured minimum elevation angle. The benchmark uses a **10° minimum elevation threshold**.

This same coverage logic is shared between the CPU and GPU paths so that performance comparisons can be paired with explicit correctness checks.

## CPU and GPU execution

The serial CPU implementation processes satellites one after another:

```text
Satellite 0 -> update -> coverage
Satellite 1 -> update -> coverage
Satellite 2 -> update -> coverage
...
```

The CUDA implementation maps independent satellites to GPU threads:

```text
Thread 0 -> Satellite 0
Thread 1 -> Satellite 1
Thread 2 -> Satellite 2
...
```

Two CUDA kernels perform the main workload:

```text
update_satellites_kernel
        |
        v
compute_coverage_kernel
```

This mapping is well suited to CUDA because position updates and coverage evaluations are independent across satellites.

## Benchmark design

I benchmarked two execution regimes:

1. **Single-step execution**
2. **GPU-resident multi-step execution**

The distinction is important because it separates a transfer-heavy use case from a compute-heavy workload where device memory residency can amortize transfer cost.

### Single-step benchmark

One single-step GPU execution includes:

```text
Host -> Device satellite transfer
            |
            v
      Position update
            |
            v
     Coverage evaluation
            |
            v
Device -> Host coverage transfer
```

Each workload size is executed **20 times** per benchmark execution, and the median is reported.

The end-to-end measurement includes:

- host-to-device satellite transfer,
- kernel execution,
- synchronization/runtime overhead,
- device-to-host coverage transfer.

GPU memory allocation is excluded from timing, and correctness validation is performed outside the timed region.

### Single-step results

Final values are the median across **three independent Release-mode benchmark executions**.

| Satellites | CPU compute | GPU kernels | GPU end-to-end | Kernel speedup | End-to-end speedup |
|---:|---:|---:|---:|---:|---:|
| 1,000 | 0.0826 ms | 0.0102 ms | 0.0417 ms | 8.06× | 1.98× |
| 10,000 | 0.8548 ms | 0.0288 ms | 0.1632 ms | 30.66× | 5.24× |
| 100,000 | 9.5530 ms | 0.2020 ms | 1.2694 ms | 47.41× | 7.55× |
| 1,000,000 | 93.0149 ms | **1.9140 ms** | **11.0439 ms** | **48.55×** | **8.73×** |

At the largest single-step workload, the CUDA kernels are almost 49× faster than the serial CPU compute path. The end-to-end gain is lower because transfers and synchronization remain a meaningful part of total runtime.

## GPU-resident multi-step execution

A simulation normally advances the same satellite state across many timesteps.

The resident benchmark therefore performs only one initial host-to-device transfer, keeps the constellation on the GPU for **100 timesteps**, and transfers the final result back only once:

```text
Host -> Device
      |
      v
+----------------------+
| Position update      |
| Coverage evaluation  | x 100 timesteps
+----------------------+
      |
      v
Device -> Host
```

Each resident workload is executed **5 times** per benchmark execution.

### Resident multi-step results

Final values are again the median across three independent benchmark executions.

| Workload | CPU total | GPU kernel loop | GPU end-to-end | Kernel speedup | End-to-end speedup |
|---|---:|---:|---:|---:|---:|
| 100,000 × 100 steps | 878.12 ms | 19.72 ms | 20.78 ms | 44.52× | 42.20× |
| 1,000,000 × 100 steps | 8740.33 ms | **177.36 ms** | **186.61 ms** | **49.25×** | **46.84×** |

For the largest resident workload, the simulation performs **100,000,000 satellite-step evaluations**.

Keeping the state on the GPU reduces the relative cost of data movement so that end-to-end acceleration approaches the raw kernel acceleration.

This is the main systems result of the project: **data residency and transfer strategy matter almost as much as kernel parallelization when optimizing GPU workloads.**

## Correctness validation

Performance measurements are only useful if the CPU and GPU implementations agree.

The project includes four automated tests:

| Test | Purpose |
|---|---|
| `satellite` | Initial position, updates and angle wrapping |
| `ground_point` | Ground-point construction and coordinate access |
| `coverage` | Covered and non-covered elevation-angle cases |
| `cpu_gpu_consistency` | CPU/GPU position and coverage agreement |

The CPU/GPU consistency test runs both implementations from the same initial constellation and simulation parameters.

Every benchmark workload also validates final CPU and GPU coverage results. The resident benchmark additionally checks sampled final satellite positions after repeated updates.

**All workloads passed validation in all three final benchmark executions.**

GitHub Actions verifies the CUDA build and CPU-side tests on the hosted runner. Device-dependent CPU/GPU consistency validation is performed locally when CUDA-capable hardware is available.

## Benchmark reproducibility

The benchmark uses deterministic satellite initialization based on a golden-angle phase distribution rather than random initialization.

The published measurements use:

```text
3 independent benchmark executions

Single-step:
    20 repetitions per workload
    median reported

Resident multi-step:
    5 repetitions per workload
    100 timesteps
    median reported
```

The final published values are the median of the three independent execution medians.

This setup reduces sensitivity to individual timing fluctuations while keeping the comparison between CPU and GPU workloads reproducible.

## Benchmark environment

The published results were measured on:

| Component | Configuration |
|---|---|
| CPU | Intel Core i5-8250U @ 1.60 GHz |
| CPU baseline | Single-threaded |
| GPU | NVIDIA GeForce MX150 |
| Compute capability | 6.1 |
| GPU memory | ~1994 MiB |
| CUDA Toolkit | 12.4 |
| NVCC | 12.4.131 |
| C++ compiler | G++ 13.4.0 |
| Build type | Release |
| CUDA architecture | `sm_61` |

The measured speedups are hardware- and system-dependent. They should be interpreted as results for this tested environment rather than universal CUDA performance claims.

## Engineering the implementation

The codebase separates reusable simulation logic from executables, tests and benchmarks.

Key components include:

- shared satellite and ground-point representations,
- shared host/device coverage logic,
- serial CPU simulation paths,
- CUDA simulation kernels,
- dedicated benchmark code,
- automated correctness tests,
- CMake-based builds,
- CTest test execution,
- GitHub Actions CI.

The project uses **C++17**, **CUDA**, **CMake** and **CTest**, with dedicated benchmark and validation paths rather than mixing measurement logic into the main simulation executable.

## Scope and limitations

This project is a **CUDA parallel-computing and performance-engineering exercise**, not a production astrodynamics package.

The current model intentionally excludes:

- orbital perturbations,
- inclination changes,
- Earth rotation,
- atmospheric drag,
- multi-body dynamics,
- high-fidelity orbital propagation,
- a multi-threaded CPU comparison baseline.

The CPU reference is single-threaded, so the reported speedups measure CUDA acceleration relative to that serial baseline.

The benchmark therefore demonstrates acceleration of the implemented computational workload, not the performance of a complete operational satellite-constellation simulator.

## What I contributed

- Built CPU and CUDA simulation paths for satellite position updates and ground-point coverage evaluation.
- Structured satellite updates and coverage checks for independent thread-level GPU execution.
- Implemented shared host/device logic to support direct CPU/GPU correctness comparison.
- Designed separate single-step and GPU-resident multi-step benchmarks.
- Measured kernel-level and end-to-end performance independently to expose data-transfer overhead.
- Added deterministic benchmark initialization and repeated median-based measurement.
- Validated CPU/GPU coverage outputs and sampled final satellite positions across benchmark workloads.
- Added automated tests, CTest integration, CMake configuration and GitHub Actions CI.
- Documented benchmark hardware, methodology, limitations and reproducibility requirements.

## What this project demonstrates

This project demonstrates how I approach performance engineering as a combination of **parallelization, measurement, correctness and systems-level reasoning**.

The main result is not simply that CUDA kernels execute faster than serial CPU code. The project shows how host-device data movement can limit application-level acceleration, and how retaining repeatedly used state on the GPU can convert a transfer-dominated workload into a compute-dominated one.

It also demonstrates experience with low-level performance measurement, reproducible benchmarking, CPU/GPU consistency testing and maintainable C++/CUDA project structure.

**Core technologies:** C++17 · CUDA · GPU Computing · Parallel Computing · Performance Optimization · Benchmarking · CMake · CTest · GitHub Actions · Linux

[View source on GitHub →](https://github.com/LittleBigPluton/GPU-Accelerated-Satellite-Constellation-Simulation)

[← Back to projects](/#projects)
