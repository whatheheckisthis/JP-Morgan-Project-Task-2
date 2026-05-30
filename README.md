# Intent-to-Auditable-Trust-Object (IATO)

**Structural abstraction theory of observation-induced equivalence over heterogeneous state spaces.**

Grounded in structural operational semantics (SOS), labelled transition systems (LTS), and coalgebraic behavioural structure — reinterpreted through an observation-first perspective.

Derived as an emblematic representation of OS internals. The framework models behaviour invariant across cache, database, and timing side-channel characteristics. Spectre and Meltdown disclosure primitives emerge as direct by-products: they are admissible transformations.


## Motivation

Classical behavioural equivalence assumes a fixed, uniform state space and derives equivalence from transition structure. This is inadequate for OS-level execution environments where:

- Execution carriers are architecture-indexed, vector-length-indexed, or VM-level structured spaces
- No canonical state space exists
- Timing, cache, and microarchitectural state are not first-class in the semantic model but leak through observation

This framework inverts the dependency. **Observation is primary.** Equivalence is induced, not assumed. Transformations are constrained by preservation of the observational quotient — not by semantic proximity or metric approximation.

The implication for side-channel analysis is that a transformation (context switch, speculative execution, cache eviction, VM migration) is **admissible** iff it preserves `O_μ(T(s)) = O_λ(s)`. Spectre/Meltdown violate this constraint by construction — they permit a distinguishing observation over states, abstraction treats as equivalent.


## Repo Structure

```
obs-equiv/
├── README.md
├── theory/
│   ├── primitives.md          # Core structural primitives
│   ├── equivalence.md         # Observation-induced quotient semantics
│   ├── transformations.md     # Admissibility constraints on T_λμ
│   └── refinement.md          # Observational discrimination ordering
├── os-model/
│   ├── carrier-index.md       # Heterogeneous execution carrier taxonomy
│   ├── cache-channel.md       # Cache side-channel as inadmissible transform
│   ├── timing-channel.md      # Timing channel as observational distinguisher
│   └── spectre-meltdown.md    # Spectre/Meltdown as quotient-violation primitives
└── formal/
    ├── obs-map.v              # Coq/Lean sketch — observation map family
    ├── quotient.v             # Quotient construction
    └── admissibility.v        # Admissibility predicate for transformations
```



## Core Framework

### Key Primitives

| Primitive | Definition |
|---|---|
| Execution carrier | `S_λ` — architecture/context-indexed state space |
| Observation map | `O_λ : S_λ → Obs` |
| Induced equivalence | `s ~ t ⟺ O(s) = O(t)` |
| Quotient | `Q = S / ~` |
| Admissible transform | `T_λμ : S_λ → S_μ` such that `O_μ(T(s)) = O_λ(s)` |

No metric. No optimisation. No implementation grounding.


### (A) Observational Structure is Primary

A family of observation maps over non-isomorphic carriers:

```
O_λ : S_λ → Obs
```

Domains `S_λ` need not be identical. `Obs` is a shared observational codomain — the only structure that is uniform across execution contexts.

In OS terms: `Obs` captures what is architecturally visible to an unprivileged observer — register file snapshots, memory-mapped I/O reads, syscall return values. It does **not** include microarchitectural state (cache occupancy, TLB residency, branch predictor state) — until those become observable through a side channel.



### (B) Equivalence is Induced, Not Assumed

No fixed bisimulation relation is assumed a priori.

```
s ~ t  ⟺  O(s) = O(t)
```

Equivalence is defined **after** structural heterogeneity is introduced — not before. The quotient `Q = S / ~` is the primary semantic object.

```
-- Haskell sketch
type Obs = ...                          -- observational domain
type S λ = ...                          -- carrier indexed by λ

obs :: S λ -> Obs

equiv :: S λ -> S λ -> Bool
equiv s t = obs s == obs t
```



### (C) Transformations as the Real Object of Study

```
T_λμ : S_λ → S_μ
```

Admissibility constraint:

```
O_μ(T_λμ(s)) = O_λ(s)   ∀s ∈ S_λ
```

This is not a bisimulation condition on transition structure. It is a **preservation condition on the observational quotient**. A transformation that alters which states are observationally distinguishable is inadmissible.

```python
def admissible(T, obs_lambda, obs_mu, S_lambda):
    """
    T          : S_lambda -> S_mu
    obs_lambda : S_lambda -> Obs
    obs_mu     : S_mu     -> Obs
    """
    return all(
        obs_mu(T(s)) == obs_lambda(s)
        for s in S_lambda
    )
```



### (D) Refinement as Observational Discrimination Ordering

Not a fixed-point construction.

```
S_λ ≤ S_μ  ⟺  (s ~_μ t  ⟹  s ~_λ t)
```

A finer carrier distinguishes more states. A coarser carrier collapses more states into equivalence classes. Refinement is the preorder on carriers induced by the relative distinguishing power of their observation maps.

In OS terms: moving from architectural to microarchitectural observation is a refinement step — it separates states that were previously equivalent under the coarser map.



## OS Internals Grounding

This framework is derived as an emblematic representation of OS internals. The execution carriers directly correspond to observable levels in a real system:

```
Execution carrier S_λ         OS-level instantiation
─────────────────────────────────────────────────────
S_arch                         architectural ISA state
S_uarch                        microarchitectural state (cache, TLB, BPU)
S_vm                           hypervisor-level virtualised state
S_db                           persistent store / transactional state
S_rt                           real-time / scheduler-visible timing state
```

Observation maps at each level define what is **architecturally visible** vs what leaks through covert channels.


## Side-Channel Analysis: Spectre and Meltdown as Quotient Violations

Spectre-class and Meltdown-class vulnerabilities are a direct by-product of this model. They are inadmissible transformations — transformations that violate the observational quotient preservation constraint.

### Formal characterisation

Let:
- `S_arch` = architectural state space (ISA-visible)
- `S_uarch` = microarchitectural state space (cache, BPU, TLB)
- `O_arch : S_arch → Obs` = architectural observation map
- `O_uarch : S_uarch → Obs_μ` = microarchitectural observation map

Under normal operation, speculative execution is intended to be a transparent implementation detail:

```
O_arch(T_spec(s)) = O_arch(s)   -- intended invariant
```

Spectre/Meltdown break this. The speculative transform `T_spec` causes a microarchitectural side-effect (cache line load) that is observable via a timing channel:

```
O_uarch(T_spec(s)) ≠ O_uarch(s)
```

And because `O_uarch` can be probed from an unprivileged context (FLUSH+RELOAD, PRIME+PROBE), it becomes a distinguisher over states that `O_arch` treats as equivalent:

```
∃ s, t :  O_arch(s) = O_arch(t)   -- architecturally equivalent
          O_uarch(T_spec(s)) ≠ O_uarch(T_spec(t))   -- μarch-distinguishable
```

This is exactly a quotient violation. `T_spec` is inadmissible under the observational quotient of `S_arch`.

### Cache and timing channels — general form

```
                    ┌──────────────┐
                    │   S_arch     │   architectural carrier
                    │  O_arch(s)   │──────────────────────► Obs_arch
                    └──────┬───────┘
                           │ T_spec (speculative exec / context switch / syscall)
                           ▼
                    ┌──────────────┐
                    │   S_uarch    │   microarchitectural carrier
                    │  O_uarch(s)  │──────────────────────► Obs_uarch
                    └──────────────┘
                           │
                    timing probe / cache probe (FLUSH+RELOAD)
                           │
                           ▼
                    distinguishable  ──► quotient violated ──► inadmissible
```

The invariant that the framework enforces — stability of the observational quotient under admissible transformations — is precisely the property that hardware and OS mitigations (KPTI, Retpoline, microcode serialisation, speculation barriers) attempt to restore.


## Correctness Characterisation

> Correctness = stability of the observational quotient under admissible transformations.

A system is behaviourally correct iff for every admissible transformation `T_λμ` and every pair of states `s, t`:

```
s ~ t  ⟹  T_λμ(s) ~ T_λμ(t)
```

Equivalently: admissible transformations induce well-defined maps on the quotient `Q = S / ~`.

This characterisation is:
- independent of metrics
- independent of optimisation criteria
- independent of implementation-specific assumptions
- directly applicable to cache, DB transaction, and timing side-channel analysis

---

### References:

Milner, Robin. Communication and Concurrency. Prentice Hall, 1989.

Park, David. “Concurrency and Automata on Infinite Sequences.” Theoretical Computer Science, vol. 138, no. 2, 1982, pp. 167–183.

Larsen, Kim G., and Arne Skou. “Bisimulation through Probabilistic Testing.” Information and Computation, vol. 94, no. 1, 1991, pp. 1–28.

Milner, Robin. A Calculus of Communicating Systems. Springer, 1980.

Hennessy, Matthew, and Robin Milner. “Algebraic Laws for Nondeterminism and Concurrency.” Journal of the ACM, vol. 32, no. 1, 1985, pp. 137–161.

Winskel, Glynn. The Formal Semantics of Programming Languages: An Introduction. MIT Press, 1993.

Plotkin, Gordon D. “A Structural Approach to Operational Semantics.” Aarhus University, 1981.

Aspinall, David, and Lars Birkedal. “Type-Theoretic Foundations of Programming Languages.” In Handbook of Logic in Computer Science, vol. 5, Oxford University Press, 2000.

Rutten, J. (2000). Universal coalgebra: a theory of systems. *TCS 249(1).*

Kocher et al. (2019). Spectre attacks: exploiting speculative execution. *IEEE S&P.*

Lipp et al. (2018). Meltdown: reading kernel memory from user space. *USENIX Security.*

Ge et al. (2018). A survey of microarchitectural timing attacks. *J. Cryptographic Engineering.*
