# The H.A.C.K. Loop

**An Evidence-First Investigation Loop for Diagnosing Application Issues and Outages Through Event-Funnel Analysis, inspired by the [USE Method](https://www.brendangregg.com/usemethod.html) and [C4 Model](https://c4model.com/).**

Modern applications depend on internal modules, services, databases, queues, SDKs, and third-party APIs. During an incident, an engineer may not know every component, its implementation details, or where the failure originated.

The problem is not simply: **"How do I find the error?"**

It is: **"How do I understand the application's expected health, identify where its behavior changed, and follow the evidence to the boundary where the change occurred?"**

H.A.C.K. provides a repeatable way to do that.

---

## 1. The Health Funnel

H.A.C.K. starts with a **Health Funnel**: a continuously maintained view of the application's expected event and data flow.

The goal is to know:
- what enters the system
- what should happen next
- what should eventually exit
- which boundaries are involved
- what healthy behavior looks like

Instead of asking only *"Are the components healthy?"*, the Health Funnel also asks:

> **"Did the expected events happen, in the expected order, at the expected volume, within the expected time, and reach an expected terminal state?"**

An anomaly is a measurable deviation from that baseline.

---

## 2. Health Is Context-Specific

Health is not one universal metric. A CPU, a matchmaking API, and an authentication flow do not become healthy in the same way.

For infrastructure resources, the [USE Method](https://www.brendangregg.com/usemethod.html) defines three useful dimensions: **Utilization · Saturation · Errors**. The same discipline applies at any application boundary by defining what those dimensions mean for that specific resource.

| Resource / Boundary | Example utilization signal |
|---|---|
| CPU | CPU busy percentage |
| Matchmaking API | Successfully matched players / players requiring matches |
| Payment service | Successful payments / payment attempts |
| Authentication service | Successful authentications / authentication attempts |
| Queue | Messages successfully processed / messages received |
| SDK boundary | Successful terminal operations / operations initiated |

Define the resource. Define what healthy utilization means. Define what saturation looks like. Define what constitutes an error. Then observe using those signals.

---

## 3. The H.A.C.K. Loop

The Health Funnel answers **"What is different?"** H.A.C.K. answers **"Why?"**

**H — Health**
Establish the expected baseline and continuously observe important application boundaries — both resource health and application event-flow health.

**A — Anomaly**
Identify where observed behavior diverges from baseline. An anomaly may appear as missing events, unexpected volume, latency divergence, cohort-specific collapse, queue accumulation, missing terminal states, or an unexplained accounting gap. The objective is to narrow the failure domain before making a change.

**C — Candidate**
Form a testable candidate explanation or intervention. A candidate is *not necessarily a fix* — it may be a hypothesis, a diagnostic probe, additional instrumentation, a mitigation, a rollback, or a code change. The question is: *what candidate explanation could account for this anomaly, and what evidence would confirm or refute it?*

**K — Knowledge**
Capture what the investigation established: confirmed causes, refuted hypotheses, discovered dependency behaviors, missing instrumentation points, regression tests, monitoring rules, updated runbooks. The objective is to make the next investigation less dependent on guesswork.

```text
┌───────────────────────┐
│      H — HEALTH       │
│ Expected baseline     │
│ Resource + event flow │
└───────────┬───────────┘
            │ Detect deviation
            ▼
┌───────────────────────┐
│     A — ANOMALY       │
│ Where did the expected│
│ flow diverge?         │
└───────────┬───────────┘
            │ Form candidate
            ▼
┌───────────────────────┐
│    C — CANDIDATE      │
│ What explanation or   │
│ probe can be tested?  │
└───────────┬───────────┘
            │ Evidence
            ▼
┌───────────────────────┐
│     K — KNOWLEDGE     │
│ What was confirmed,   │
│ refuted, or learned?  │
└───────────┬───────────┘
            │ Update baseline,
            │ telemetry & runbooks
            └──────────────► H
```

The loop is intentionally continuous. Knowledge changes what should be observed during the next investigation.

---

## 4. Application Event Accounting

At any defined application boundary — an SDK call, authentication flow, ad request, payment operation, or service-to-service call — we should be able to account for what entered and what happened afterward.

> **Inputs = Outputs + ΔBuffer + Sinks + Unaccounted**

**Inputs** — Total operations entering the boundary  
**Outputs** — Operations that successfully reach completion  
**ΔBuffer** — Operations legitimately in flight or queued  
**Sinks** — Operations reaching an explicit terminal state (error, timeout, cancellation, rejection)  
**Unaccounted** — Operations with no observable terminal state

```text
              INPUTS
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
OUTPUTS       SINKS      ΔBUFFER
successful    explicit    bounded
completion    terminal    in-flight
               state
    │           │           │
    └───────────┼───────────┘
                │
                ▼
         Anything left?
                │
                ▼
          UNACCOUNTED
                │
                ▼
            INVESTIGATE
```

The unaccounted remainder is itself an investigation signal. Many failures do not produce an explicit error. An asynchronous operation that never settles produces no exception, no timeout, no failure metric — it simply disappears from the observable journey. Infrastructure may remain healthy. Error counters may remain flat. Yet the user journey is broken.

This gives three distinct layers:

> **Infrastructure Health ≠ Application Health ≠ User Journey Health**

Application Event Accounting exists to make that missing layer observable.

---

## 5. USE and Event Accounting Are Complementary

H.A.C.K. does not replace the USE Method. They answer different questions.

|  | USE Method | H.A.C.K. / Event Accounting |
|---|---|---|
| Primary focus | Resource health | Application journey health |
| Core question | Is this resource healthy? | Did the expected flow make it through? |
| Typical signals | Utilization · Saturation · Errors | Inputs · Outputs · ΔBuffer · Sinks · Unaccounted |
| Useful for | Resource constraints and failures | Missing, incomplete, or divergent journeys |

A green resource-health dashboard is useful evidence. It is not proof that the user journey is healthy. When resource health looks normal but expected events are disappearing, follow the event flow.

---

## 6. Case Study: P0 Ad Funnel Collapse

*The following is a reconstructed investigation. Details are representative.*

Gameplay engagement was normal. Infrastructure was within nominal thresholds. No binary release had gone out. Yet interstitial ad impressions on iOS collapsed — component health was normal while an application path had stopped completing.

**H — Health**

The expected journey: `ad_request → ad_load (success | fail)`. The health property was not *"Are ad services returning errors?"* but: *"Does every initiated operation eventually reach a terminal state?"*

**A — Anomaly**

Event accounting exposed the gap:

```
REQUESTS
ad_request success            100%  ████████████████████████████████████

RESOLVED
ad_load success / fail         62%  ████████████████████

UNACCOUNTED
no terminal state              38%  ██████████
```

Not an increase in explicit failures — an absence of terminal states. The accounting gap narrowed the investigation to operations that had started but never settled.

**C — Candidate**

Cohort isolation narrowed the anomaly to iOS, specific ad-unit types, and a specific third-party SDK version. A diagnostic timeout was introduced at the interop boundary:

```javascript
const adLoadWithTimeout = Promise.race([
    sdk.loadAd(adUnitId),
    new Promise((_, reject) =>
        setTimeout(
            () => reject(new Error('PLATFORM_SDK_TIMEOUT | Platform: iOS')),
            20000
        )
    )
]);
```

This transformed an invisible state into an observable terminal state — diagnostic as well as protective.

**K — Knowledge**

*"Under specific conditions, the third-party SDK could leave an async operation unresolved. Absence of terminal accounting exposed the boundary; the timeout converted the silent failure into observable evidence."*

This drove vendor remediation, explicit async timeouts, regression tests, monitoring for unresolved operations, and updated runbooks. The incident produced a stronger system than the one that existed before it.

---

## 7. Operational Anti-Patterns

**Traffic-Light Anti-Pattern** — *"Everything is green, so the application is healthy."*  
Dashboards measure what they were designed to measure. Pair resource health with event accounting. If resources are healthy but events are disappearing, follow the journey.

**Streetlight Anti-Pattern** — *"Search where dashboards already have visibility."*  
The failure may be precisely where observability is weakest. If existing evidence doesn't explain the anomaly, identify the dark boundary and instrument the smallest missing part.

**Cross-Boundary Deflection** — *"The problem must be on the other side."*  
Account for what entered, crossed, returned, failed, and remained unaccounted. This transforms *"your system is broken"* into *"these operations crossed successfully, these failed, these remained unresolved"* — actionable evidence.

**Random Change Anti-Pattern** — *"Change something until symptoms disappear."*  
Recovery doesn't prove causation. Before declaring resolution: *what changed in the execution path, and why should that explain the recovery?*

---

## 8. Three Diagnostic Questions

At every investigation boundary, Application Event Accounting reduces to three questions:

**Where?** — Which boundary, runtime layer, dependency, or code path stopped accounting for events?

**Which?** — Which cohort isolates the anomaly? (OS, app version, geography, SDK version, feature flag, user segment)

**When?** — When did the accounting last balance? That timestamp defines the investigation window.

> **Count what entered. Count what exited. Investigate the difference.**

---

## 9. The Principle

The most important idea in H.A.C.K. is simple:

> **A system can be healthy at the resource level while unhealthy at the journey level.**

That is why application event flow deserves to be treated as a first-class health signal.

When an application behaves unexpectedly: establish what healthy looks like, count what entered, count what exited, account for legitimate in-flight work, identify explicit terminal states, find what remains unexplained, narrow by boundary and cohort, form a candidate explanation, test it with evidence, and capture the resulting knowledge.

The unexplained remainder is not merely missing data.

> **It is where the investigation should begin.**

H.A.C.K. is a repeatable way of thinking:

> **Measure → Investigate → Validate → Learn → Repeat**

The engineering asset created by an incident is not only the code change that restored service. It is the knowledge of what failed, where, how the failure became observable, which candidates were confirmed or rejected, and how the system can detect or prevent the same failure class next time.

---

**Health → Anomaly → Candidate → Knowledge**

*Count what entered. Count what exited. Investigate the difference.*

---

## Author

**Jatinderpal Singh**

**Domain:** Application Health Framework · Business & System Observability  
**Status:** Published Specification  
**Canonical URI:** [erjsingh2012.github.io/HACK-Loop/](https://erjsingh2012.github.io/HACK-Loop/)  
**Medium Article:** [The H.A.C.K. Loop](https://medium.com/@singhjatinderpal/the-h-a-c-k-loop-2b54104ecd64)

**Inspired from:**

* [USE Method — Brendan Gregg](https://www.brendangregg.com/usemethod.html)
* [C4 Model — Simon Brown](https://c4model.com/)

Application Event Accounting · Health Funnel · H.A.C.K. Investigation Loop
