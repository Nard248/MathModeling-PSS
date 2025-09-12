# 📊 Stochastic Physics Simulations — Complete Reference

> **A one-stop, presentation-ready reference** you can lean on during demos and Q&A.  
> Connects **equations** ⇄ **intuition** ⇄ **code**, includes quick-derivations, validation tips, and fast answers.

---

## 📖 Notation & Symbols (Quick Lookup)

| Symbol | Description | Units/Notes |
|--------|-------------|-------------|
| **t** | Time | seconds |
| **Δt** (dt) | Time step in discrete simulations | seconds |
| **N(t)** | Number of undecayed nuclei at time t | count |
| **N₀** | Initial number of nuclei | count |
| **λ** | Decay constant (probability per unit time) | s⁻¹ |
| **A(t)** | Activity = λ·N(t) | Bq (decays/s) |
| **τ** | Mean lifetime = 1/λ | seconds |
| **t₁/₂** | Half-life = ln(2)/λ ≈ 0.693/λ | seconds |
| **β** | Dimensionless interaction parameter | altered decay |
| **xₙ** | State of logistic map at iteration n | [0,1] |
| **λ_log** | Control parameter of logistic map | [0,1] |
| **Λ** | Lyapunov exponent | bits/iteration |

📌 **Unit reminder:** Activity in SI is **Becquerel (Bq)** = 1 decay/s. Legacy: Curie (Ci).

---

## 🔬 PART I — Standard Nuclear Decay (Poisson Process)

### ⚡ Core Model

**Differential equation:**
```
dN/dt = −λN
```

**Solution:**
```
N(t) = N₀·e^(−λt)    and    A(t) = λ·N(t)
```

**Derived relations:**

| Quantity | Formula | Condition |
|----------|---------|-----------|
| Mean lifetime | τ = 1/λ | — |
| Half-life | t₁/₂ = ln(2)/λ ≈ 0.693/λ | — |
| Survival probability | S(t) = e^(−λt) | single nucleus |
| Decay probability | P(Δt) = 1 − e^(−λΔt) ≈ λΔt | λΔt ≪ 1 |

**Stochastic nature:**
- ✓ Decays form a **Poisson process** with rate λ per nucleus
- ✓ For many nuclei, counts per bin ~ **Poisson(μ = N·λ·Δt)**

### 🎯 Why Monte Carlo if we have the solution?

1. **Captures fluctuations** (what experiments actually measure)
2. **Enables realistic effects:** detector efficiency, dead time, background, branching chains
3. **Scales to complex models** without closed-form solutions

### 💻 Implementation Patterns

| Method | Description | Performance | Use Case |
|--------|-------------|-------------|----------|
| **Per-particle Bernoulli** | Loop over each nucleus | O(N) | Small N, teaching |
| **Aggregate Binomial** | `decays ~ Binomial(N, p)`<br>where `p = 1 − exp(−λΔt)` | O(1) | Large N, efficiency |
| **Event-driven (Gillespie)** | Sample waiting times<br>`T ~ Exp(rate = λ·N)` | Variable | Rare events |

**⚠️ Correctness constraints:**
- Choose Δt so that **λΔt ≲ 0.1** (prevents multiple-decay-per-step errors)
- Use appropriate integer types: `int32` for N₀ ≲ 10⁵ (safer than `int16`)

### 🔗 Micro-to-Macro Links

```
┌─────────────────────────────────────────┐
│ Expected decrement:  E[ΔN] = N·λ·Δt    │
│ Variance:           Var(ΔN) ≈ N·λ·Δt   │  
│ Relative noise:     σ/N ∼ 1/√N         │
└─────────────────────────────────────────┘
```

### 📈 Estimating λ from data

- **Event times** tᵢ (i=1..K): MLE λ̂ = K / Σtᵢ; for large K, λ̂ ≈ N(λ, λ²/K), so SE(λ̂) ≈ λ/√K
- **Binned N(t):** fit N(t) = N₀ e^(−λt) by nonlinear least squares or linear fit on ln N(t) (with Poisson-aware weights)

---

## 🧬 PART II — Altered Nuclear Decay (Density-Dependent Rate)

### 📐 Model & Motivation

**Mean-field correction to rate:**
```
λ_eff(N) = λ·(1 − β·N/N₀)    where 0 ≤ β·N/N₀ < 1
```

**ODE:**
```
dN/dt = −λN + (λβ/N₀)·N²
```

This is a logistic-type decay (nonlinear). For small β, it's a perturbation of exponential decay.

### 📊 Exact Solution & Closed Form

**Solution with N(0) = N₀:**
```
        N₀·e^(−λt)
N(t) = ─────────────────────
       1 + β·N₀·(1 − e^(−λt))
```

**Checks:**
- β → 0 ⇒ N(t) → N₀ e^(−λt)
- Early time: N(t) ≈ N₀ − λN₀t + (λβ − λ²/2)N₀t² + …

**Fixed points & stability:**
- f(N) = −λN + (λβ/N₀)N² ⇒ f(N*)=0 at N*=0 (stable) and N*=N₀/β (outside physical range if βN₀<1)

**Physical interpretations (β>0):**
- Screening / Pauli blocking: effective rate is smaller at high density
- Initial decay slower than exponential; later approaches exponential as N drops
- β<0 would imply positive feedback (generally unphysical for nuclear decay)

### 🔧 Simulation Mapping

- Update per step with `p_decay = 1 − exp[−λ(1 − βN/N₀)Δt]`
- Small-Δt approximation `p ≈ λΔt(1 − βN/N₀)` is OK when λΔt ≪ 1 and βN/N₀ not close to 1

**Validation signals:**
- For β>0, N(t) lies above the exponential at early times (slower loss), then converges later
- Compare ensemble mean to the closed-form curve

---

## 🌀 PART III — Logistic Map & Chaos

### 🔄 Map Definition

**Logistic map:**
```
x_{n+1} = f(xₙ; λ_log) = 4·λ_log·xₙ·(1 − xₙ)
```
where `x ∈ [0,1]` and `λ_log ∈ [0,1]`

📌 **Connection:** Standard form has `r = 4·λ_log`, so `r ∈ [0,4]`

### 🎭 Fixed Points & Stability

- **x* = 0:** Stable if |f′(0)| = |4λ_log| < 1 ⇒ λ_log < 0.25
- **x* = 1 − 1/(4λ_log):** (exists if λ_log > 0.25). Stability requires |f′(x*)| = |2 − 4λ_log| < 1 ⇒ 0.25 < λ_log < 0.75

### 🌊 Bifurcations and Chaos

| Event | λ_log | r | Description |
|-------|-------|---|-------------|
| **First period-doubling** | 0.75 | 3.0 | Stable → 2-cycle |
| **Chaos onset** | ≈0.892 | ≈3.5699 | Accumulation point |
| **Full chaos** | 1.0 | 4.0 | Ergodic on [0,1] |

**Universal constants:**
- Feigenbaum δ ≈ 4.6692016…
- Feigenbaum α ≈ 2.5029…

### 📏 Lyapunov Exponent

```
Λ = lim_{n→∞} (1/n) Σ_{i=0}^{n−1} ln|f′(xᵢ)|
```
where `f′(x) = 4λ_log(1 − 2x)`

- **Λ > 0** → chaos
- **Λ < 0** → periodic orbit/fixed point
- **Λ = 0** → bifurcation

**Computation:** Discard transients, sum over subsequent iterates, use double precision

**Invariant density (r=4):** ρ(x) = 1/[π√(x(1−x))] on (0,1) → U-shaped occupancy

---

## 🔧 Code ⇄ Theory Mapping (Notebook Guidance)

**Standard decay (Part I):**
- Theory: Bernoulli trials with p ≈ λΔt per nucleus per step (Poisson limit)
- Optimization: replace per-particle loop with binomial: `decays ~ Binomial(N, 1 − exp(−λΔt))`
- Theoretical overlay: N₀ e^(−λt) matches ensemble mean; ±σ band visualizes fluctuations

**Altered decay (Part II):**
- Theory: N(t) = N₀ e^(−λt) / [1 + βN₀(1 − e^(−λt))]
- Higher accuracy per step: `p = 1 − exp(−λ(1 − βN/N₀)Δt)`

**Logistic map (Part III):**
- Practice: discard transient M≈500; plot N≈300 points per λ to draw attractor
- Use finer λ-grid for richer detail

---

## 📏 Numerical & Statistical Hygiene

**Time-step & probability:**
- Prefer `p = 1 − exp(−rate·Δt)` to avoid p>1 and reduce bias
- Keep λΔt ≲ 0.1; validate by halving Δt and checking convergence

**Types & ranges:**
- `int16` limits ±32768; safe for N₀ ≤ 10,000
- For larger ensembles or sums, use `int32/64`

**RNG & reproducibility:**
- Set seeds for demos
- Prefer vectorized draws over Python loops in hot paths

**Complexity:**
- Per-particle method: O(M · T · N_avg)
- Binomial method: O(M · T)

**Validation checklist:**
✓ Ensemble average matches theory within statistical error  
✓ χ² or KS test for histogram vs Poisson/exponential  
✓ Convergence in Δt and M (1/√M error law)  
✓ Ensure N(t) never becomes negative; clamp at 0 as guard

---

## 📝 Ready-to-Use Derivations (Compact)

**(A) Exponential law from constant hazard**  
If P(survive t+dt | survive t) = 1 − λdt, then S(t+dt) = S(t)(1 − λdt) ⇒ dS/dt = −λS ⇒ S(t)=e^(−λt)

**(B) Mean & variance per step (Poisson limit)**  
Let X ~ Binomial(N, p) with p = 1 − e^(−λΔt) ≈ λΔt. Then E[X]=Np, Var[X]=Np(1−p) ≈ NλΔt

**(C) Altered decay closed form**  
Integrate dN/[N(1 − βN/N₀)] = −λdt and use N(0)=N₀ to obtain N(t) above

**(D) Logistic map stability**  
Fixed point x* satisfies x* = 4λ_log x*(1 − x*). Stability from |f′(x*)|<1 with f′(x)=4λ_log(1−2x)

**(E) Lyapunov computation recipe**  
1. Choose λ_log and x₀∈(0,1)
2. Iterate M_transient times (discard)
3. Accumulate S = Σ ln|4λ_log(1−2xᵢ)| over next N iterates
4. Λ ≈ S/N; Λ>0 indicates chaos

---

## ⚡ Common Pitfalls → Fast Fixes

| Problem | Solution |
|---------|----------|
| **p > 1** | Use `p = 1 − exp(−rate·dt)` or reduce dt |
| **Multiple decays missed** | Same fix; or switch to event-driven |
| **Negative N** | Clamp to 0; verify integer underflow not occurring |
| **Thick/fuzzy bifurcation** | Increase λ-resolution and points per λ; reduce marker size |
| **Slow runs** | Vectorize; use Binomial; preallocate arrays |

---

## 🚀 Advanced Extensions (Talking Points)

- **Branching chains:** A→B→C with distinct λ's (Bateman equations)
- **Spatial decay-diffusion:** ∂ρ/∂t = D∇²ρ − λρ
- **Detector realism:** efficiency ε, background b, dead time τ_d (non-/paralyzable models)
- **Non-Markov/fractional kinetics:** power-law waiting times; Mittag–Leffler survival
- **Stochastic resonance:** weak signal enhanced by noise
- **Coupled maps / lattices:** synchronization, spatiotemporal chaos

---

## ❓ Q&A Bank (Concise)

**Q1. Why is decay exponential?**  
A: Constant hazard λ ⇒ survival e^(−λt) ⇒ N(t)=N₀e^(−λt)

**Q2. When is λΔt too large?**  
A: Keep λΔt ≲ 0.1; halve Δt and check stability

**Q3. Monte Carlo mean doesn't match theory?**  
A: Increase M (1/√M), reduce Δt, use p=1−exp(−λΔt), check integer types/underflow

**Q4. Why Binomial sampling helps?**  
A: Aggregates N Bernoulli trials in O(1); identical statistics with correct p

**Q5. Altered decay early-time curve above exponential—why?**  
A: λ_eff < λ at high density; initial decay suppressed

**Q6. Does altered decay hit exactly 0?**  
A: ODE approaches 0 asymptotically; discrete Monte Carlo hits 0 when last particle decays

**Q7. Logistic map: why 4λ?**  
A: Re-parameterization so λ∈[0,1] maps to r∈[0,4]; full chaos at λ=1

**Q8. Fast Lyapunov in code?**  
A: After transients, accumulate ln|4λ(1−2x)| and divide by count

**Q9. How many runs M?**  
A: Error ∝ 1/√M. For ~1% relative error on means, M ≈ 10⁴ (rule of thumb)

**Q10. Fit λ from a single trajectory?**  
A: MLE on event times or Poisson-weighted fit of ln N(t) vs t

**Q11. Meaning of Feigenbaum δ?**  
A: Universal ratio of successive period-doubling intervals near chaos

**Q12. Chaos looks random but isn't—why?**  
A: Deterministic with Λ>0 → exponential separation of nearby initial conditions

**Q13. Bifurcation diagram shows blank stripes?**  
A: Periodic windows embedded in chaos (e.g., period-3)

**Q14. Is int16 safe?**  
A: For N₀=10,000 yes; int32 is safer when scaling up

**Q15. Event-driven vs time-stepped?**  
A: Event-driven is efficient for rare events; time-stepped is simpler and vectorizes well

---

## 💡 Annotated Snippets (Drop-in Upgrades)

### Accurate per-step probability and vectorized binomial

```python
# Accurate per-step probability with interaction
rate = lam * (1 - beta * N / N0)  # set beta=0 for standard decay
p = 1.0 - np.exp(-rate * dt)

# Vectorized binomial draw for decays
decays = np.random.binomial(N.astype(int), np.clip(p, 0.0, 1.0))
N = N - decays
```

### Lyapunov exponent (post-transient)

```python
x = x0
# burn-in
for _ in range(M):
    x = 4 * lam_log * x * (1 - x)

# measure
S = 0.0
for _ in range(N):
    x = 4 * lam_log * x * (1 - x)
    S += np.log(abs(4 * lam_log * (1 - 2 * x)))
Lambda = S / N
```

### Gillespie-style (event-driven) decay

```python
N = N0
t = 0.0
while N > 0 and t < T_max:
    rate_tot = lam * N
    t += np.random.exponential(1.0 / rate_tot)
    N -= 1
    # record (t, N) as needed
```

---

## ✅ Demo-Day Checklist (Presenter's Aid)

- [ ] Seed RNG for repeatability
- [ ] State assumptions (constant λ; independence; small Δt)
- [ ] Show ensemble vs theory; call out ±σ band
- [ ] Flip β on/off to illustrate interaction effects
- [ ] Sweep λ_log to reveal period-doubling → chaos
- [ ] Mention Feigenbaum universality
- [ ] Close with validation: Δt-halving test; M-scaling (1/√M)

---

## 📚 Further Reading

- **Poisson processes & exponential waiting times:** Gardiner; Van Kampen
- **Logistic map & Feigenbaum constants:** May (1976); Feigenbaum (1978); Strogatz (Nonlinear Dynamics)
- **Gillespie algorithm (1976)** for exact stochastic simulation of reaction kinetics

---

*End of reference sheet*