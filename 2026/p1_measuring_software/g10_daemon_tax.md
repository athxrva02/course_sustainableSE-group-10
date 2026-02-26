---
author: Atharva Dagaonkar, Kasper van Maasdam, Ignas Vasiliauskas, Stefan Bud
group_number: 10
title: "The ‘Daemon’ Tax: An Energy Analysis of Docker and Podman across RESTful and Computational Workloads."
image: "img/gX_template/Docker vs Podman.jpg"
date: 12/02/2026
summary: |-
  This project evaluates the energy efficiency of containerization by comparing Docker’s daemon-based architecture against Podman’s daemonless and "pod-centric" model. By utilizing EnergiBridge to monitor CPU, DRAM, and GPU power draw, the study investigates whether Podman’s lower idle footprint is offset by potential overhead during high-intensity workloads, such as REST API networking, CPU-heavy π calculations, and GPU-accelerated matrix multiplication. Furthermore, the experiment expands into orchestration efficiency, comparing the energy consumption of Docker Compose against native Podman Pods across different operating systems to determine the most sustainable infrastructure choice for both microservices and intensive computational tasks.
identifier: p1_measuring_software_2026 # Do not change this
all_projects_page: "../p1_measuring_software" # Do not change this
---

## Introduction

Containerisation has become a cornerstone of modern software infrastructure. Whether deploying a simple web API or orchestrating a fleet of microservices, developers overwhelmingly reach for container runtimes to package, isolate, and ship their applications. Docker is the de-facto standard, used in practice by millions of developers worldwide. Podman has emerged as a notable alternative, promising a lighter footprint through its daemonless, rootless architecture.

Yet the energy implications of this architectural choice are rarely discussed. Data centres account for roughly 1–2% of global electricity consumption[^iea], and as container workloads make up an ever-growing proportion of that load, even small per-container overheads aggregate into meaningful energy expenditure at scale. Docker’s background daemon — a persistent process that mediates all container lifecycle operations — is a plausible source of such overhead, but it has not been quantified rigorously across diverse workload types.

This study fills that gap. We measure the total system energy consumed when running two representative workloads — a multi-service RESTful web application and a CPU-bound chess engine search — under Docker and Podman respectively. Our aim is to answer a simple but practically important question: **does Docker’s daemon architecture impose a measurable energy penalty, and if so, does it depend on the nature of the workload?**

---

## Background

### Docker: the daemon-based model

Docker follows a client–server architecture. A long-running background process — `dockerd` — is always present on the host, even when no containers are running. Every Docker CLI command (and every API call from Docker Compose) is a request sent to this daemon over a Unix socket. The daemon in turn delegates low-level container management to `containerd` and, ultimately, to the OCI runtime `runc`.

This indirection provides convenience: the daemon manages image caching, networking, volumes, and log streaming from a single, centralised process. The cost is a non-trivial idle energy baseline — the daemon consumes CPU cycles and DRAM even when it is simply waiting for instructions.

### Podman: the daemonless model

Podman (Pod Manager) takes a fundamentally different approach: there is no long-running daemon. Each `podman` invocation is a self-contained process that directly calls into the OCI runtime (`crun` by default). Because there is no mediating daemon, Podman can run containers without root privileges by default, and its idle energy footprint on the host is zero when no containers are active.

Podman also introduces the concept of **pods** — groups of containers that share a network namespace, mirroring the Kubernetes Pod model. This makes Podman a natural fit for Kubernetes-oriented workflows, while `podman compose` provides a compatible interface for Docker Compose files.

### EnergiBridge

Energy measurements in this study are collected with **EnergiBridge**[^energibridge], a cross-platform power sampling tool that reads hardware energy counters (RAPL on Intel/AMD, equivalent interfaces on ARM) and writes per-sample CSV output. EnergiBridge instruments the *entire experiment process*: the total joules reported reflect everything the system consumes during the measurement window, including the container runtime overhead — precisely what we want to capture.

---

## Methodology

### Research Questions

1. Does Docker’s daemon architecture consume measurably more energy than Podman’s daemonless model for a RESTful, network-bound workload?
2. Does the energy difference change when the workload is CPU-intensive rather than I/O-bound?
3. Are the differences, if any, statistically significant and practically relevant at the scale of a typical developer workstation?

### Workload 1 — RESTful multi-service application

The network workload consists of a three-service stack mimicking a realistic web application:

- **Frontend** — a Go HTTP server that serves three static "pages" of increasing payload sizes: 50 MB (`/page1`), 100 MB (`/page2`), and 150 MB (`/page3`). The large payloads ensure that network I/O rather than computation dominates the workload.
- **Backend** — a Go REST API backed by a MySQL database modelling a simple CPU webshop (`cpus`, `suppliers`, `orders` tables). It exposes stress endpoints:
  - `/stress/sql?intensity=N` — runs a heavy cross-join aggregate query N times.
  - `/stress/mem?size_mb=M&seconds=S` — allocates M MB, holds it for S seconds, then streams it over the network.
  - `/stress/cpu?threads=T&seconds=S` — runs SHA-256 hashing on T goroutines for S seconds.
  - `/seed?count=N` — populates the database with N CPUs, linked suppliers, and orders.
- **Database** — MySQL 8, seeded before the load test begins.

The three services are orchestrated with a single `compose.yaml` file. For Docker, this is managed by `docker compose`; for Podman, by `podman compose` using the same file.

The load test (`loadtest.sh`) fires **50 HTTP requests** at concurrency **10** across the eight distinct endpoints in round-robin order using `xargs` and `curl`. The database is seeded with two batch calls before the load test begins, so the query load is realistic.

```
┌─────────────────────────────────────────────────┐
│                  Host machine                   │
│                                                 │
│  loadtest.sh  ──►  frontend :8080               │
│                        │                        │
│                    backend :8081  ──►  db :3306  │
│                                                 │
│  [all three services share a bridge network]    │
└─────────────────────────────────────────────────┘
```

### Workload 2 — CPU-intensive chess engine search

The CPU workload runs a Rust chess engine (**Sandy**, version 0.6.3) inside a single container. Sandy implements alpha-beta minimax with iterative deepening and communicates via the UCI (Universal Chess Interface) protocol over stdin/stdout. The workload drives the engine with:

```
position startpos
go movetime 10000
```

This instructs the engine to search the starting position for exactly 10 000 ms, using all available cores via Rayon’s parallel search. The result is a sustained, compute-bound load with no network I/O, making it a natural counterpart to the RESTful workload.

The engine binary is compiled inside a multi-stage Docker/Podman build (Rust `slim` → Alpine `3.20`) and embedded in the container image. The measurement window covers container startup, the 10-second search, and container shutdown.

```
┌──────────────────────────────────────────┐
│              Host machine                │
│                                          │
│  search.sh  ──stdin──►  chesseng        │
│               (UCI)      (inside ctr.)  │
└──────────────────────────────────────────┘
```

### Experimental procedure

Each trial follows this sequence:

1. Start the container(s) with the target runtime (`docker` or `podman`).
2. Wait for all health checks to pass (`container-ready.sh`).
3. Run the workload (load test or chess search).
4. Stop and remove the container(s).

EnergiBridge wraps the outermost measurement script so the energy counter captures the full lifecycle: startup, running, and teardown. Thirty independent trials are performed per runtime per workload (120 trials total), with a **60-second rest** between consecutive trials to allow thermal stabilisation and prevent tail energy from bleeding into the next measurement.

Trial order within each workload is randomised to avoid systematic order effects.

### Controlled environment (Zen Mode)

All experiments are conducted with the following constraints:

- All non-essential applications closed; notifications disabled.
- Screen brightness fixed at minimum; automatic brightness disabled.
- Wi-Fi and Bluetooth disabled; Ethernet used for any network traffic.
- No USB peripherals attached beyond keyboard and mouse.
- Room temperature held at a stable value throughout.
- A CPU warm-up phase (SHA-256 hashing loop) runs before the first trial to bring the system to thermal steady state.

### Hardware and software specifications

| Component | Specification |
|-----------|---------------|
| **TODO** | *(fill in machine spec)* |

| Software | Version |
|----------|---------|
| Docker | **TODO** |
| Podman | **TODO** |
| EnergiBridge | **TODO** |
| OS | **TODO** |

---

## Results

> **TODO**: Insert violin plots / box plots of energy consumption per runtime per workload.

> **TODO**: Insert summary statistics table (mean, median, std dev, min, max) for each condition.

> **TODO**: Report results of normality tests (Shapiro-Wilk) and choose appropriate significance test (Student’s t / Mann-Whitney U). Report p-value and effect size (Cohen’s d or rank-biserial r).

---

## Discussion

> **TODO**: Interpret results in light of the daemon architecture. If Docker consumes more energy for the RESTful workload, attribute this to the always-on `dockerd`/`containerd` processes mediating network and volume I/O. If the difference is smaller for the CPU workload, discuss why: the daemon overhead may be amortised over a longer, compute-saturated workload window.

> **TODO**: Discuss whether the measured difference is practically significant at developer-workstation scale vs. data-centre scale.

> **TODO**: Compare with findings from related work on container runtime energy consumption (e.g., the 2023 Container Runtimes paper from this course[^g10_2023]).

---

## Limitations and Future Work

**Single hardware platform.** All experiments ran on one machine. The RAPL energy counters and their relationship to actual power draw differ across CPU microarchitectures (Intel vs. AMD vs. ARM). Results should be replicated on diverse hardware before generalising.

**Podman Compose vs. native Pods.** This study uses `podman compose` to maintain parity with the Docker Compose workflow. Podman’s native Pod model (where containers share a network namespace without a compose layer) could exhibit different energy characteristics and merits its own study.

**Warm vs. cold image cache.** We pre-pull images before trials begin, so image-layer decompression is excluded from measurements. Including cold-start scenarios would capture the energy cost of first-time deployments.

**Limited workload diversity.** Two workloads cannot characterise the full spectrum of container use cases. Future work could extend to GPU-accelerated workloads (matrix multiplication), long-running idle services, and high fan-out microservice topologies.

**Daemon idle baseline.** We do not separately measure the steady-state idle cost of `dockerd`. Isolating this baseline would help decompose the total overhead into daemon-specific and workload-specific components.

---

## Conclusion

> **TODO**: Summarise the key finding — whether Docker’s daemon architecture imposes an energy penalty and under which workload types that penalty is most pronounced. Provide a practical recommendation for developers and operators who care about energy efficiency.

---

## References

[^iea]: International Energy Agency. *Data Centres and Data Transmission Networks*. [iea.org](https://www.iea.org/energy-system/buildings/data-centres-and-data-transmission-networks), 2023.

[^energibridge]: Durieux, T. *EnergiBridge: A Cross-Platform Energy Measurement Tool*. [github.com/tdurieux/EnergiBridge](https://github.com/tdurieux/EnergiBridge), 2024.

[^g10_2023]: Sustainable SE Course, Group 10. *Container Runtimes Energy Comparison*. [course_sustainableSE-group-10/2023](https://luiscruz.github.io/course_sustainableSE/2023/p1_measuring_software/g10_Container_Runtimes.html), 2023.
