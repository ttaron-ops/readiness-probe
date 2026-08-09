# Anatomy of a Container — Module 01b deliverable

A **publishable** teardown plus a reusable **diagnostic toolkit**, built across this
module's lessons. It doubles as one of your public-evidence artifacts (a blog post)
and feeds the GPU operator's observability story later.

> **The thesis to prove:** *there is no "container" object — a container is just
> namespaces + a cgroup + a rootfs, and here are the exact kernel files that make it
> one, each limit demonstrably enforced.*

## Part 1 — The teardown (written, ~2–3k words, publishable)

Take **one running pod/container** (a GPU-adjacent workload, or a
`docker run --cpus/--memory` CPU stand-in if you have no GPU) and reconstruct it
entirely from kernel primitives:

- **Namespaces** — enumerate all of it via `/proc/<pid>/ns/*` and `lsns`; show what
  each isolates (Lesson 02).
- **cgroup** — locate its cgroup under `/sys/fs/cgroup/...` and read every resource
  file (`cpu.max`, `cpu.weight`, `memory.max`, `memory.high`, `cpu.stat`), mapping each
  value back to the spec that produced it (Lesson 03).
- **Enforcement, demonstrated** — *throttle it* (pin CPU, read `nr_throttled`), *OOM it*
  (constrain `memory.max`, read the `dmesg` kill), *pressure it* (watch `*.pressure`
  climb) — with the kernel evidence for each (Lessons 03–05).

**Acceptance:** a `teardown.md` that reconstructs a real container from kernel files
alone, with a mapping table (spec → cgroup file → observed enforcement) and the OOM /
throttle / pressure evidence inline. Publishable as-is.

## Part 2 — The diagnostic toolkit (small, reusable)

4–6 snippets + a script that answer real fleet questions, gathered as you do the
lesson practice:

| Tool | Answers | From lesson |
|------|---------|-------------|
| `bpftrace` syscall-count / read-size / execve | "what is this process actually doing" | 08 |
| `bpftrace` off-CPU / page-fault | "why is it stalled / thrashing" | 08–09 |
| PSI + cgroup inspection script | "which cgroup is CPU-throttled / driving memory pressure" | 03–04 |
| conntrack headroom check | "are we about to drop packets" | 07 |
| flame graph capture | on-CPU profile of a hot process | 09 |

**Acceptance:** a `toolkit/` directory with each snippet/script, plus a one-line
"fleet question this answers" per tool, and one committed flame graph SVG.

## Suggested layout

```
anatomy-of-a-container/
├── teardown.md          # the publishable writeup
├── toolkit/
│   ├── syscalls.bt      # bpftrace snippets
│   ├── offcpu.bt
│   ├── psi-inspect.sh   # PSI + cgroup inspection
│   ├── conntrack-headroom.sh
│   └── flamegraph.svg
└── README.md            # how to run each, and the fleet question it answers
```

## Guardrails

- Do the labs on a throwaway VM/laptop/kind node — some (OOM, conntrack-fill) disrupt
  the box. **Never on a shared/production node.**
- Publishable-by-default, but scrub any real hostnames/cluster details before posting.
- No secrets or kubeconfigs in git (repo `.gitignore` guards these).
