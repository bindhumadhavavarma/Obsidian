
# What the Xen Credit scheduler does (in a nutshell)

* **Goal:** fair-share CPU scheduling across VCPUs on SMP hosts with optional caps.
* **Key knobs per VCPU:**

  * **weight** (relative share; default often 256).
  * **cap** (0 = uncapped; otherwise a % of one PCPU).
* **Currency:** **credits**. VCPUs spend credits while running; credits are periodically refilled based on weight.

# Core objects

* **PCPU** = physical CPU; each has a **runqueue** (RQ) of runnable VCPUs.
* **VCPU** = virtual CPU (belongs to a VM/domain).
* **States / priorities:**

  * **BOOST** (just became runnable or was recently blocked on I/O) → short-term priority bump to reduce latency.
  * **UNDER** (has credits ≥ 0) → normal priority.
  * **OVER** (credits < 0) → deprioritized but not starved (still runs if no UNDER/BOOST ready).

# Time accounting & credits

* **Timeslice / tick:** Time is chopped into ticks; on each tick, the running VCPU is charged credits.
* **Refill period:** Every period (e.g., 30ms–300ms depending on build), each VCPU’s credits are refilled ≈ `weight * constant`. This gives proportional share: VCPU with double weight earns double credits per period.
* **Spending rate:** Running consumes credits at ≈ real time rate (scaled to cap).
* **Negative credits:** If a VCPU keeps running past its fair share, its credits go negative → it drops to OVER.

# Picking who runs

1. **Favor BOOST**, then **UNDER**, then **OVER**.
2. Within each band, pick by **highest credits** (ties can be FIFO/round-robin).
3. **Preemption:** If a higher-priority/better-credited VCPU appears, current one may be preempted at next tick.
4. **Load balancing:** VCPUs can **migrate** between PCPUs to balance runnable load (try to keep cache affinity but avoid imbalance).

# Weight & Cap semantics (common confusion)

* **Weight:** Relative across all runnable VCPUs. If VCPU A has weight 512 and B has 256, A targets \~2× CPU share **when both are runnable**.
* **Cap:** Hard upper bound of CPU a VCPU can get **per PCPU**.

  * **cap = 0** → uncapped: can temporarily exceed its nominal share if others are idle.
  * **cap = 50** → at most 50% of one PCPU, even if the machine is idle.
  * With **multiple PCPUs**, cap is per VCPU instance, not per domain unless you enforce it domain-wide.

# BOOST logic (latency vs fairness)

* A VCPU that wakes from I/O gets **BOOST** so it can promptly run and keep pipelines full; BOOST is **transient**. After it uses some time (or a tick), it falls back to UNDER and resumes normal credit competition.

# Overcommit / starvation behavior

* **OVER** VCPUs are not starved: they run **only when no UNDER/BOOST** VCPUs are ready. This preserves fairness while keeping CPUs busy.

# Runqueue & migration

* Each PCPU keeps separate queues per band (BOOST/UNDER/OVER).
* Periodic **load balance**: if one PCPU has many runnable VCPUs and a sibling is idle/light, a VCPU migrates (usually the one with the most credits/lowest affinity cost). Affinity/pinning can restrict this.

# Typical pitfalls (and answers)

* **“Does negative credit mean blocked?”** No. It means **deprioritized**; it may still run if nothing better is ready.
* **“Do weights matter when others are idle?”** Not really; if others are idle and you’re **uncapped**, you can use spare cycles.
* **“What if a domain has multiple VCPUs?”** Shares apply **per VCPU**. A domain with more runnable VCPUs can consume more total CPU unless you enforce domain-level policy.
* **“Cap vs weight precedence?”** Cap is a **hard ceiling**; weight governs **distribution below the ceiling** (and when contending).
* **“Why BOOST?”** To cut interactive latency and avoid head-of-line blocking after I/O wakeups.




