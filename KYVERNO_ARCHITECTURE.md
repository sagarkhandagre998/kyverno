# Kyverno Architecture & Internal Working — In Depth

A design-perspective deep dive. This builds on [KYVERNO_EXPLAINED.md](KYVERNO_EXPLAINED.md)
(read that first if "what is Kyverno" is still fuzzy). Here we open the hood:
**which components exist, what each one does, and how a request flows through the
system end to end.** File/folder references point to this actual repository.

---

## Part 1 — The 10,000-foot view

Kyverno is **not one program**. It is a set of cooperating processes (controllers),
each a separate binary, all extending Kubernetes. They share one big idea:

> **The Kubernetes API server is the center of the universe. Kyverno plugs into it
> in two ways: (1) it intercepts requests live via webhooks, and (2) it watches and
> reacts to resources via controllers.**

These two modes map to two fundamental patterns in Kubernetes:

| Pattern | Trigger | Used by Kyverno for |
|---------|---------|---------------------|
| **Admission control** (webhook) | A request is *in flight* (someone is creating/updating a resource) | Validate, Mutate, VerifyImages — **synchronous**, can block |
| **Controllers** (watch + reconcile) | A resource *already changed* in etcd | Generate, background scans, reports, cleanup — **asynchronous** |

Everything below is an elaboration of this table.

---

## Part 2 — The binaries (the `cmd/` folder)

Kyverno ships as **multiple independent deployments**, not a monolith. Each has its
own entrypoint under [`cmd/`](cmd/). This separation exists so that heavy or
failure-prone work can't take down the critical admission path.

```
cmd/
├── kyverno/               → the "admission controller" — runs the WEBHOOK server
├── background-controller/ → handles Generate + Mutate-existing (async)
├── reports-controller/    → builds PolicyReports (compliance scanning)
├── cleanup-controller/    → runs CleanupPolicies (delete resources on a schedule)
├── kyverno-init/          → init container: setup/migration before main starts
├── readiness-checker/     → health/readiness probe helper
└── cli/                   → the `kyverno` CLI (apply/test on a laptop, no cluster)
```

### Why split into separate controllers? (key design decision)

1. **Blast-radius isolation.** The admission webhook is on the **critical path** —
   if it's slow or crashes, it can block *every* resource creation in the cluster.
   Report generation and background scanning are CPU/memory heavy. Keeping them in
   separate pods means a runaway report job can't freeze admission.
2. **Independent scaling.** You can run the admission controller as a highly
   available replicated set, while the reports controller runs as a single replica.
3. **Independent failure policy.** Each can be restarted/upgraded without disturbing
   the others.

> 🧠 **Design takeaway:** Kyverno deliberately splits the *synchronous, must-be-fast*
> work (admission) from the *asynchronous, can-be-slow* work (generate, reports,
> cleanup).

---

## Part 3 — The four "powers" and where each one runs

Recall the four capabilities. Here's the crucial design detail — **they don't all
run in the same place:**

| Power | Runs in | Mode | Why |
|-------|---------|------|-----|
| **Validate** | `kyverno` (webhook) | Synchronous | Must block bad resources before they exist |
| **Mutate** (on new resources) | `kyverno` (webhook) | Synchronous | Must edit the resource *as it's admitted* |
| **VerifyImages** | `kyverno` (webhook) | Synchronous | Must block untrusted images before admission |
| **Generate** | `background-controller` | Asynchronous | The *trigger* is admitted instantly; the generated resources are created just after, by a controller |
| **Mutate existing** | `background-controller` | Asynchronous | Edits resources that *already exist*, not in-flight ones |

This is why Generate "feels" different from Validate: validation happens *during*
the request; generation happens *just after*, driven by a separate controller that
reacts to the admitted resource.

---

## Part 4 — The components (the `pkg/` folder), grouped by role

The real logic lives in [`pkg/`](pkg/). Here are the components that matter, grouped
by what job they do.

### 4.1 The entry point: Webhook server
- [`pkg/webhooks/server.go`](pkg/webhooks/server.go) — the HTTPS server that
  Kubernetes calls. It registers HTTP routes for each handler type.
- [`pkg/webhooks/resource/`](pkg/webhooks/resource/) — the handlers for **resource**
  admission (the user's Pods/Deployments/etc). This is where `Validate(...)` and
  `Mutate(...)` live — see [`handlers.go`](pkg/webhooks/resource/handlers.go).
- [`pkg/webhooks/policy/`](pkg/webhooks/policy/) — admission handlers for **Kyverno's
  own policies** (validating that a policy YAML is well-formed before accepting it).
- [`pkg/webhooks/handlers/`](pkg/webhooks/handlers/) — shared middleware: decoding
  the admission request, metrics, tracing, dumping, etc.

### 4.2 The brain: the Engine
- [`pkg/engine/engine.go`](pkg/engine/engine.go) — **the policy engine.** This is the
  heart of Kyverno. It exposes the core methods:
  - `Validate(...)` — run validation rules
  - `Mutate(...)` — run mutation rules
  - `Generate(...)` — compute what to generate
  - `VerifyAndPatchImages(...)` — image verification
  - `ApplyBackgroundChecks(...)` — re-evaluate for reports/scans
- [`pkg/engine/handlers/validation/`](pkg/engine/handlers/validation/) and
  [`pkg/engine/handlers/mutation/`](pkg/engine/handlers/mutation/) — the actual
  rule-type implementations the engine dispatches to.
- [`pkg/engine/api/`](pkg/engine/api/) — the data types: `PolicyContext`,
  `EngineResponse`, `RuleResponse` (the structured result of evaluating each rule).

> 🧠 **The engine is stateless and pure-ish.** You hand it a `PolicyContext` (the
> resource + the policies + context data) and it returns an `EngineResponse`
> (pass/fail per rule, plus any mutated resource). It doesn't talk to the API server
> to *apply* changes — it just *computes* results. The webhook/controller decides
> what to do with them. This separation makes the engine reusable by the **CLI**
> too (same engine, no cluster needed).

### 4.3 Pattern/expression evaluation (how rules are actually checked)
- [`pkg/engine/pattern/`](pkg/engine/pattern/) — the YAML "pattern matching"
  (`?*`, anchors, etc.).
- [`pkg/engine/anchor/`](pkg/engine/anchor/) — conditional anchors `()`, equality
  `=()`, existence `^()`, negation `X()`.
- [`pkg/engine/jmespath/`](pkg/engine/jmespath/) — JMESPath: the query/expression
  language Kyverno uses for variables like `{{ request.object.metadata.name }}`.
- [`pkg/cel/`](pkg/cel/) — **CEL** (Common Expression Language) support, the newer
  expression engine Kubernetes itself uses; Kyverno supports CEL-based policies too.
- [`pkg/engine/variables/`](pkg/engine/variables/) — variable substitution
  (replacing `{{ ... }}` placeholders before evaluation).

### 4.4 Context loading (where the data for variables comes from)
- [`pkg/engine/context/`](pkg/engine/context/) — the **rule context**: a bag of data
  a rule can reference. It can be enriched by:
  - **API calls** ([`pkg/engine/apicall/`](pkg/engine/apicall/)) — fetch other
    resources from the cluster.
  - **ConfigMap lookups**, **image registry data**
    ([`pkg/registryclient/`](pkg/registryclient/)),
  - **Global context** ([`pkg/globalcontext/`](pkg/globalcontext/)) — cached,
    shared data refreshed periodically so policies don't hammer the API server.

### 4.5 The controllers (the `pkg/controllers/` folder)
These are the **asynchronous reactors**. Each watches some resource and reconciles.
- [`pkg/controllers/webhook/`](pkg/controllers/webhook/) — **manages the webhook
  configurations themselves.** This is subtle and important: Kyverno dynamically
  *writes* the `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration`
  objects in Kubernetes based on which policies exist (so the API server only calls
  Kyverno for resource kinds that actually have policies). See `validating.go`.
- [`pkg/controllers/report/`](pkg/controllers/report/) — builds and updates
  PolicyReports.
- [`pkg/controllers/cleanup/`](pkg/controllers/cleanup/) and
  [`pkg/controllers/deleting/`](pkg/controllers/deleting/) — CleanupPolicy execution.
- [`pkg/controllers/certmanager/`](pkg/controllers/certmanager/) — generates and
  rotates the **TLS certificates** the webhook needs (the API server only talks TLS).
- [`pkg/controllers/policystatus/`](pkg/controllers/policystatus/) — keeps each
  policy's `.status` up to date.
- [`pkg/controllers/ttl/`](pkg/controllers/ttl/) — TTL-based resource cleanup.

### 4.6 Supporting infrastructure
- [`pkg/policycache/`](pkg/policycache/) — an **in-memory cache of all policies**,
  indexed so the webhook can instantly answer "which policies apply to this Pod?"
  without querying the API server on every request. Critical for latency.
- [`pkg/client/`](pkg/client/) & [`pkg/clients/`](pkg/clients/) — generated and
  wrapped Kubernetes API clients.
- [`pkg/informers/`](pkg/informers/) — Kubernetes *informers* (watch + local cache)
  feeding the policy cache and controllers.
- [`pkg/leaderelection/`](pkg/leaderelection/) — so only one replica does
  singleton work (e.g., writing webhook configs) even when many replicas run.
- [`pkg/tls/`](pkg/tls/) — certificate handling.
- [`pkg/metrics/`](pkg/metrics/), [`pkg/tracing/`](pkg/tracing/),
  [`pkg/event/`](pkg/event/) — observability (Prometheus metrics, OpenTelemetry
  traces, Kubernetes Events).
- [`pkg/exceptions/`](pkg/exceptions/) — PolicyException handling (a way to exempt
  specific resources from policies).
- [`pkg/breaker/`](pkg/breaker/) — circuit breaker / rate limiting to protect the
  cluster from overload.

### 4.7 The API types (the `api/` folder = the CRDs)
Kyverno teaches Kubernetes new object kinds via **CRDs**. Their Go definitions live
in [`api/kyverno/`](api/kyverno/), versioned: `v1`, `v2`, `v2beta1`, `v2alpha1`,
`v1beta1`.
- [`api/kyverno/v1/clusterpolicy_types.go`](api/kyverno/v1/clusterpolicy_types.go) —
  `ClusterPolicy`
- [`api/kyverno/v1/policy_types.go`](api/kyverno/v1/policy_types.go) — `Policy`
  (namespaced)
- [`api/kyverno/v1/rule_types.go`](api/kyverno/v1/rule_types.go) — a single `Rule`
  (match/exclude + validate/mutate/generate/verifyImages)
- [`api/kyverno/v1/spec_types.go`](api/kyverno/v1/spec_types.go) — the policy `spec`
- [`api/reports/`](api/reports/) & [`api/policyreport/`](api/policyreport/) — the
  report object types.

> 🧠 **Design takeaway:** A "policy" in Kyverno is just data (a CRD object). The
> engine is the *interpreter* of that data. CRD = the language; engine = the runtime.

---

## Part 5 — The admission flow, step by step (the synchronous path)

This is the most important flow to understand. Let's trace **`kubectl apply -f
pod.yaml`** all the way through.

```
 ┌──────────┐   1. apply Pod
 │  kubectl │ ───────────────────────►  ┌─────────────────────┐
 └──────────┘                           │  Kube API Server     │
                                        │  (authn, authz, ...) │
                                        └──────────┬───────────┘
                                                   │ 2. "Mutating webhooks?"
                                                   ▼
                                        ┌──────────────────────┐
                                        │  Kyverno webhook      │  ◄── MutatingWebhookConfiguration
                                        │  /mutate endpoint     │      (written by pkg/controllers/webhook)
                                        └──────────┬───────────┘
            3. engine.Mutate() runs mutation rules │  returns JSON patch
                                                   ▼
                                        ┌──────────────────────┐
                                        │  API Server applies   │
                                        │  the patch, re-checks │
                                        └──────────┬───────────┘
                                                   │ 4. "Validating webhooks?"
                                                   ▼
                                        ┌──────────────────────┐
                                        │  Kyverno webhook      │  ◄── ValidatingWebhookConfiguration
                                        │  /validate endpoint   │
                                        └──────────┬───────────┘
   5. engine.Validate() + VerifyImages; Enforce?  │  allowed=true/false
                                                   ▼
                                        ┌──────────────────────┐
                                        │  Persist to etcd      │  (only if allowed)
                                        └──────────┬───────────┘
                                                   │ 6. resource now EXISTS
                                                   ▼
                                  ┌────────────────────────────────┐
                                  │ background-controller notices  │ ──► Generate / Mutate-existing
                                  │ reports-controller notices     │ ──► PolicyReport
                                  └────────────────────────────────┘
```

### What happens *inside* each Kyverno webhook call

When the API server POSTs an `AdmissionReview` to Kyverno
([`pkg/webhooks/resource/handlers.go`](pkg/webhooks/resource/handlers.go)):

1. **Decode & middleware** — shared handlers in
   [`pkg/webhooks/handlers/`](pkg/webhooks/handlers/) decode the request, start a
   trace span, record metrics, and check the failure policy.
2. **Select policies** — `retrieveAndCategorizePolicies(...)` asks the
   **policy cache** ([`pkg/policycache/`](pkg/policycache/)) "which policies `match`
   this resource?" — instantly, from memory. Policies are split into **Audit** vs
   **Enforce** buckets (see `mergeEngineResponses`).
3. **Build the PolicyContext** — `buildPolicyContextFromAdmissionRequest(...)`
   packages the incoming resource, the old resource (on update), user info, and
   admission metadata into one object for the engine.
4. **Run the engine** — `engine.Mutate(...)` or `engine.Validate(...)`. For each
   matching rule the engine:
   - checks `match`/`exclude` precisely (`engine.matches`),
   - applies any `PolicyException`,
   - **loads context** (API calls, configmaps, image data) if the rule needs it,
   - **substitutes variables** (`{{ ... }}` → real values via JMESPath/CEL),
   - evaluates the rule (pattern match for validate; build a JSON patch for mutate),
   - produces a `RuleResponse` (pass/fail/skip/error + message).
5. **Decide the verdict:**
   - **Mutation:** collect all patches, return them as a JSON Patch to the API
     server (the resource gets edited).
   - **Validation:** if *any* **Enforce** rule failed → `allowed = false`, request
     **blocked** with the failure message. **Audit** failures don't block — they're
     recorded for reports instead.
6. **Emit side effects** — metrics, Events, and (for audit results) data that the
   reports controller will turn into a `PolicyReport`.

> 🧠 **Why mutation runs before validation:** Kubernetes always calls *mutating*
> webhooks first, then re-validates the (possibly mutated) object against *validating*
> webhooks. So a mutate rule can "fix" a resource that a validate rule would
> otherwise reject. The ordering is the API server's, and Kyverno relies on it.

---

## Part 6 — The asynchronous flows (the controller path)

### 6.1 Generate flow
1. User creates a *trigger* resource (e.g., a new `Namespace`). It's admitted
   instantly — generation does **not** block admission.
2. The webhook (or an update-request mechanism) records an **UpdateRequest** —
   "namespace X matched a generate policy, please reconcile."
   ([`pkg/webhooks/updaterequest/`](pkg/webhooks/updaterequest/))
3. The **background-controller** picks it up, runs `engine.Generate(...)` to compute
   the target resources (e.g., a default NetworkPolicy + ConfigMap), and **creates
   them** in the cluster.
4. It keeps them in sync: if `synchronize: true`, edits/deletes to the generated
   resource are reverted to match the policy (`clone`/`data` sources).

### 6.2 Background scanning & reports flow
1. The **reports-controller** periodically re-evaluates *existing* resources against
   all policies using `engine.ApplyBackgroundChecks(...)`.
2. Results are written as `PolicyReport` / `ClusterPolicyReport` objects
   ([`api/policyreport/`](api/policyreport/)). This is how you see compliance of
   things that were created *before* a policy existed, or that only run in **Audit**
   mode.

### 6.3 Cleanup flow
1. A `CleanupPolicy` says "delete resources matching X on schedule Y."
2. The **cleanup-controller** ([`pkg/controllers/cleanup/`](pkg/controllers/cleanup/))
   runs on a cron-like schedule and deletes matching resources.

---

## Part 7 — Two "self-management" mechanisms worth understanding

These are non-obvious but central to how Kyverno stays correct and fast.

### 7.1 Dynamic webhook configuration (the webhook controller)
Naively, Kyverno could tell Kubernetes "call me for *every* resource of *every*
kind." That would be catastrophic for cluster performance. Instead:

- [`pkg/controllers/webhook/`](pkg/controllers/webhook/) **watches the installed
  policies** and **writes the `ValidatingWebhookConfiguration` /
  `MutatingWebhookConfiguration` dynamically**, listing only the resource kinds that
  actually have policies.
- No Pod policies? The API server never calls Kyverno for Pods. This keeps the
  admission overhead proportional to how much you actually use Kyverno.

> 🧠 This is a controller that manages the *configuration of the webhook that the
> rest of Kyverno serves.* Kyverno configures the hook into Kubernetes that then
> calls Kyverno back. Mind-bending but elegant.

### 7.2 Certificate management
The API server will only call a webhook over **HTTPS with a trusted certificate**.
[`pkg/controllers/certmanager/`](pkg/controllers/certmanager/) +
[`pkg/tls/`](pkg/tls/) generate a CA + serving cert, inject the CA bundle into the
webhook configs, and rotate them before expiry — all automatically.

---

## Part 8 — Failure policy & safety (critical design concern)

Because the webhook sits on the critical path, **what happens if Kyverno is down?**
Kubernetes' webhook spec has a `failurePolicy`:

- **`Fail`** — if Kyverno can't be reached, **block** the request. Safer
  (no policy bypass) but risks freezing the cluster if Kyverno is unhealthy.
- **`Ignore`** — if Kyverno can't be reached, **allow** the request. Keeps the
  cluster usable but allows policy bypass during an outage.

Kyverno also uses:
- **`namespaceSelector` / `objectSelector`** to exclude its own namespace and system
  namespaces (so it never tries to police itself into a deadlock).
- **Timeouts** — each webhook call has a tight timeout; the engine must finish fast.
- **The circuit breaker** ([`pkg/breaker/`](pkg/breaker/)) — sheds load to protect
  the cluster under pressure.

> 🧠 **Design tension:** correctness (Fail) vs availability (Ignore). Production
> setups carefully choose per-policy, run Kyverno highly-available, and exclude
> critical system namespaces.

---

## Part 9 — The CLI: the engine without a cluster

[`cmd/cli/`](cmd/cli/) + [`pkg/cli/`](pkg/cli/) reuse the **exact same engine**
([`pkg/engine/`](pkg/engine/)) to evaluate policies against YAML files on your
laptop — no Kubernetes required.

- `kyverno apply` — run policies against resource manifests, see pass/fail.
- `kyverno test` — declarative test files asserting expected results (used in CI).

This is only possible *because* the engine is decoupled from the API server (Part
4.2). The same code that powers live admission also powers offline testing. Much of
the recent work in this repo (e.g., CLI CRD handling, CEL exceptions in `apply`)
lives here.

---

## Part 10 — Putting it all together (one mental diagram)

```
                         ┌───────────────────────────────────────┐
                         │            KUBE API SERVER             │
                         └───────┬───────────────────────┬───────┘
            admission (sync)     │                       │   watch (async)
                                 ▼                       ▼
   ┌──────────────────────────────────────┐   ┌─────────────────────────────┐
   │  kyverno (admission controller)       │   │  controllers (separate pods)│
   │  ┌────────────────────────────────┐   │   │  • background → Generate     │
   │  │ webhook server (pkg/webhooks)   │   │   │  • reports → PolicyReport    │
   │  │   ▼                             │   │   │  • cleanup → delete on cron  │
   │  │ ENGINE (pkg/engine)             │◄──┼───┤  • webhook → write webhook   │
   │  │   • match/exclude               │   │   │       configs dynamically    │
   │  │   • context load (apicall, etc) │   │   │  • certmanager → TLS certs   │
   │  │   • variables (jmespath/cel)    │   │   └─────────────────────────────┘
   │  │   • validate/mutate/verifyImg   │   │
   │  └────────────────────────────────┘   │   shared:
   │  policy cache · metrics · tracing      │   • policycache · informers
   └──────────────────────────────────────┘   • leaderelection · breaker

   Same ENGINE is also embedded in:  cmd/cli  (kyverno apply / test, no cluster)
```

### The one-paragraph design summary
Kyverno extends Kubernetes through two mechanisms: **admission webhooks** (a fast,
synchronous path that intercepts in-flight requests to **validate**, **mutate**, and
**verify images** — and can block them) and **controllers** (an asynchronous path
that reacts to already-stored resources to **generate** companion resources, run
**background compliance scans/reports**, perform **cleanup**, and even **manage
Kyverno's own webhook configs and TLS certs**). At the center sits a **stateless
engine** that takes a `PolicyContext` (resource + policies + loaded context data),
substitutes variables, evaluates each rule, and returns a structured
`EngineResponse` — the same engine reused by the live cluster and by the offline
CLI. The work is split across **separate binaries** so that slow, heavy jobs
(reports, generation) can never jeopardize the latency-critical admission path. That
split — sync vs async, engine vs API, and configuration that Kyverno writes about
itself — is the essence of Kyverno's design.

---

## Where to read next in the code (a suggested reading order)
1. [`api/kyverno/v1/rule_types.go`](api/kyverno/v1/rule_types.go) — understand the
   *data* a policy is.
2. [`pkg/webhooks/resource/handlers.go`](pkg/webhooks/resource/handlers.go) — the
   admission entry point (`Validate`, `Mutate`).
3. [`pkg/engine/engine.go`](pkg/engine/engine.go) — how the engine dispatches rules.
4. [`pkg/engine/handlers/validation/`](pkg/engine/handlers/validation/) — a concrete
   rule implementation.
5. [`pkg/controllers/webhook/validating.go`](pkg/controllers/webhook/) — how the
   webhook configs are written dynamically.
```
