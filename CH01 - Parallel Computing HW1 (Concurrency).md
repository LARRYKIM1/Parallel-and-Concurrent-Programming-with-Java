# 1 Parallel Computing HW

## Summary

- Sequential vs Parallel
- Flynn's Taxonomy
- Parallel Computing Architecture: SISD, MISD, SIMD, MIMD
- Parallel Programming Model: SPMD, MPMD
- Shared vs Distributed Memory
  - Shared
    - UMA, SMP(focus in lecture)
  - Distributed Memory
    - Scalable
    - Most supercomputer selects it

## QUIZ

- In most modern multi-core CPUs, cache coherency is usually handled by the `_____`
  - processor hardware
- A key advantage of distributed memory architectures is that they are `_____` than shared memory systems.
  - scalable
- Modern multi-core PCs fall into the `_____` classification of Flynn's Taxonomy.
  - MIMD
- The four classifications of Flynn's Taxonomy are based on the number of concurrent `_____` streams and `_____` streams available in the architecture.
  - instruction; data