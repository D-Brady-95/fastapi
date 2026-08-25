# Log summariser fork debate — sync HTTP vs async queue

## Fork (one sentence)
Should log summarisation be handled synchronously over HTTP (compute inline), or asynchronously via a queue + worker (enqueue then process)?

## Decision criteria (agreed up front)
1) p95 create-summary response <= 300ms for <=200 lines (excluding model call time if external).
2) Handle 100 req/min/tenant without timeouts; degrade with 429.
3) Summariser outage: user-visible errors within 2s; retries max 3 with backoff.
4) Ops constraint: no new paid infrastructure; minimal on-call surface.
5) Reversibility: can switch strategies within 1 sprint without data migration.

---

## Option A — Sync HTTP only: strongest case (Round 1)
- Claim: request parsing/validation/routing for <=200 lines fits comfortably under 300ms p95 when model time is excluded. [assumption]
  - Notes: cites general benchmarks (TechEmpower, API Gateway/Lambda guidance) but not a benchmark of *our* stack under our payload shapes.
- Claim: 100 req/min/tenant (1.67 RPS) is trivial for a stateless HTTP tier; rate limiting can degrade with 429 without queue complexity. [assumption]
- Claim: outage behavior is deterministic: fail within 2s with bounded retries (max 3 with exponential backoff) rather than waiting behind a queue. [assumption]
- Claim: minimal ops surface: no broker, no worker fleet, no DLQ/runbooks beyond existing API platform. [assumption]
- Claim: reversibility is strong: adding a queue later can be internal and requires no data migration. [assumption]

## Option B — Async queue + worker: strongest case (Round 1)
- Claim: enqueue admission keeps create-summary latency reliably <300ms even under burst load; worker concurrency absorbs model-time variance. [assumption]
- Claim: queue-based load leveling is recommended practice (AWS Well-Architected / Azure Architecture Center) for protecting interactive APIs from downstream latency/outages. [evidence-backed]
- Claim: deterministic tenant admission control (429) is easier when decoupled from execution capacity. [assumption]
- Claim: workers can do bounded retries (max 3 with backoff) without tying up HTTP connections. [assumption]
- Claim: can use open-source queue tech (e.g., Redis-backed) to meet “no paid infra” while isolating failure domains. [assumption]
- Claim: reversibility preserved: API contract can remain unchanged while switching execution path. [assumption]

---

## Cross-pollinated dissent (Round 2: strongest objections)

### Option A’s strongest objection to Option B
- Objection: Option B cannot actually guarantee criterion #3 (“user-visible failure within 2s”) without additional state + status delivery; queue wait/backlog makes end-to-end failure latency unbounded during worker/queue incidents. [evidence-backed]
  - Failure mode: backlog amplification — admission is fast but completion/failure is delayed; requires job-state persistence + polling/notification to make failure visible.
  - Operational cost: adds always-on surfaces (queue health, worker health, retry/DLQ, job-state storage, status propagation) which may violate “minimal on-call surface”. [assumption]

### Option B’s strongest objection to Option A
- Objection: sync-only design risks retry-induced cascading failure under summariser latency/outage; 3 retries can turn 100 req/min into up to 400 execution attempts/min, exhausting concurrency and causing timeouts before rate limiting stabilizes. [evidence-backed]
  - Failure mode: feedback loop (latency -> more in-flight -> retries -> more load -> timeouts).
  - Quantified cost: retry multiplier under incident conditions increases load by up to 4x per tenant, multiplied across tenants. [assumption] (depends on exact retry policy placement and whether retries are client/server)

---

## Decision
Decision: **Defer pending evidence.** Driver: criteria #2 and #3 conflict without measurement (degrade-without-timeouts vs fail-within-2s). We will not commit to sync-only or queue-first until we measure burst behavior and outage visibility.

## Evidence to collect (to unblock decision)
1) Load test: 100 req/min/tenant across N tenants with injected summariser latency (e.g., 200ms -> 2s) and injected summariser outage; measure:
   - p95/p99 response times
   - timeout rate
   - 429 rate correctness
   - retry amplification (attempts/min)
2) If evaluating async: measure end-to-end time-to-failure visibility under worker backlog (queue delay scenarios) and confirm whether “user-visible failure within 2s” is feasible without violating ops constraints.

## Reversal / revisit criteria (falsifiable)
Revisit within 2 weeks if any of the following are observed:
- If sync prototype shows >0.1% timeouts at 100 req/min/tenant during summariser latency spikes, switch to async-first (queue) to prevent cascade.
- If async prototype cannot surface terminal failure state to users within 2s under worker backlog in test, reject async-first (or relax criterion #3 explicitly).
- If ops overhead for queue+worker (runbooks/alerts) exceeds “minimal on-call surface” threshold agreed by the team, prefer sync-first.

