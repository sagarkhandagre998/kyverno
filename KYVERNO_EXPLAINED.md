# Kyverno Explained — From Zero

A beginner-friendly guide. No prior knowledge assumed. We build up slowly.

---

## 1. First, what problem are we even solving?

Imagine you run a big building (an office). Lots of people walk in every day
carrying boxes, equipment, food, furniture — anything. You want some **rules**:

- "Every box must have a label."
- "No one can bring in dangerous chemicals."
- "If someone forgets to bring a fire extinguisher, give them one automatically."
- "I just want a report of who broke rules, but don't stop them yet."

Now you need a **security guard at the door** who checks every single thing
coming in against those rules — automatically, every time, without getting tired.

**Kyverno is that security guard. The "building" is Kubernetes.**

---

## 2. Quick background: what is Kubernetes? (super short)

You can skip this if you already know.

- **Kubernetes (K8s)** is a system that runs your applications inside small
  packages called **containers**.
- You don't start containers by hand. Instead, you write a **YAML file** that
  *describes what you want* ("run 3 copies of my web app"), and you send it to
  Kubernetes. Kubernetes makes it happen.
- These YAML descriptions are called **resources** or **manifests**. Examples:
  `Pod`, `Deployment`, `Service`, `ConfigMap`, etc.

So in Kubernetes, **everything is created by submitting a YAML file.** That YAML
file is the "box coming through the door." Now the security-guard idea makes sense.

---

## 3. So what is Kyverno?

> **Kyverno is a policy engine for Kubernetes.**

Breaking that down:

- **Policy** = a rule you want enforced (like the building rules above).
- **Engine** = the thing that automatically checks and applies those rules.

Its tagline is **"Cloud Native Policy Management — no new language required."**

That last part is the big selling point. Read the next section to understand why
it matters so much.

---

## 4. The killer feature: policies are just YAML

Most older policy tools (like a thing called **OPA/Gatekeeper**) make you learn a
**special programming language** called *Rego* to write rules. That's a steep
learning curve.

**Kyverno does NOT make you learn a new language.** You write Kyverno policies in
**plain YAML** — the *same* format you already use for everything else in
Kubernetes.

If you can read a Kubernetes YAML file, you can read a Kyverno policy. That's the
whole point.

---

## 5. The 4 things Kyverno can do

Kyverno policies fall into a few categories. These are the core ideas:

### a) **Validate** — "check and allow/block"
> "Block any Pod that doesn't have a `team` label."

The guard inspects the incoming YAML. If it breaks the rule, the guard **rejects
it** (or just warns, your choice). Nothing bad gets created.

### b) **Mutate** — "fix it automatically"
> "If a Pod has no resource limits, add default ones for them."

The guard doesn't just reject — it **edits the YAML on the way in** to fix it.
The user doesn't have to do anything. This is super convenient.

### c) **Generate** — "create related things automatically"
> "Whenever a new Namespace is created, automatically create a default
> NetworkPolicy and a ConfigMap inside it."

The guard **creates extra resources** for you so you don't forget.

### d) **VerifyImages** — "check that container images are trustworthy"
> "Only allow container images that are cryptographically signed by our company."

This protects against running tampered or fake images (supply-chain security).

> 🧠 **Mental model:** Validate = inspect, Mutate = fix, Generate = add,
> VerifyImages = authenticate.

---

## 6. How does Kyverno actually intercept things? (the magic)

Kubernetes has a built-in feature called an **Admission Webhook**. Think of it as
an official "hook" where Kubernetes promises:

> "Before I create ANY resource, I'll first send it to a registered helper and ask
> 'is this okay? do you want to change it?'"

Kyverno registers itself as that helper. So the flow is:

```
   You (kubectl apply pod.yaml)
            │
            ▼
   Kubernetes API Server
            │   "Hey Kyverno, someone wants to create this Pod. OK?"
            ▼
        Kyverno  ──checks your policies──►  Validate / Mutate / VerifyImages
            │
            ▼
   "OK, allow it"  OR  "No, reject it"  OR  "Allow, but I changed it a bit"
            │
            ▼
   Resource is created (or blocked)
```

This all happens in a fraction of a second, automatically, for every request.

There's also a **background scanner**: Kyverno can periodically re-check things
that *already exist* in the cluster, not just new ones. That produces **reports**
telling you what's compliant and what isn't.

---

## 7. A real policy, explained line by line

Here's a complete Kyverno policy. Goal: **require every Pod to have a `team`
label.**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy          # "I am a Kyverno policy, applies cluster-wide"
metadata:
  name: require-team-label
spec:
  validationFailureAction: Enforce   # Enforce = block. (Audit = just warn)
  rules:
    - name: check-team-label
      match:                         # WHICH resources does this rule apply to?
        any:
        - resources:
            kinds:
              - Pod                   # only Pods
      validate:                       # the actual check
        message: "Every Pod must have a 'team' label."   # shown if it fails
        pattern:                      # the shape the resource MUST match
          metadata:
            labels:
              team: "?*"              # "?*" means "must exist and not be empty"
```

Reading it in English:

> "For every **Pod** being created, **enforce** (block if broken) that it has a
> `metadata.labels.team` value that is not empty. If it doesn't, reject it with
> the message *'Every Pod must have a team label.'*"

Key beginner takeaways:

- `match` decides **what** the rule looks at.
- `validate` (or `mutate`/`generate`) decides **what** to do.
- `pattern` is just a **partial copy of the YAML** that the resource must match —
  very intuitive, no special language.
- `?*` is a wildcard: `?*` = "something must be here", `*` = "anything including
  empty".
- `Enforce` vs `Audit` is the single most important knob:
  - **Audit** = don't block, just record violations (safe, great for starting out).
  - **Enforce** = actually block bad resources.

---

## 8. A mutate example (auto-fixing)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
    - name: add-env-label
      match:
        any:
        - resources:
            kinds:
              - Pod
      mutate:
        patchStrategicMerge:          # "merge these fields into the resource"
          metadata:
            labels:
              environment: production # add this label if missing
```

> "Any Pod created without an `environment` label gets `environment: production`
> added automatically." The user never even notices — it just gets fixed.

---

## 9. Important vocabulary (cheat sheet)

| Term | Plain meaning |
|------|---------------|
| **Policy** | A set of rules you want enforced. |
| **ClusterPolicy** | A policy that applies to the whole cluster. |
| **Policy** (namespaced) | A policy that applies only inside one namespace. |
| **Rule** | One single check/action inside a policy. |
| **match / exclude** | Which resources a rule applies to (or skips). |
| **Validate** | Allow or block a resource. |
| **Mutate** | Automatically modify a resource. |
| **Generate** | Automatically create extra resources. |
| **VerifyImages** | Check container image signatures/trust. |
| **Audit** | "Just report violations, don't block." |
| **Enforce** | "Actually block violations." |
| **Admission Webhook** | Kubernetes' hook that lets Kyverno intercept requests. |
| **PolicyReport** | A report of what passed/failed the policies. |
| **CRD** | "Custom Resource Definition" — how Kyverno teaches Kubernetes about new object types like `ClusterPolicy`. |

---

## 10. Why do people use Kyverno? (real use cases)

- **Security guardrails:** block privileged containers, block `latest` image tags,
  require resource limits, disallow running as root.
- **Compliance:** enforce company standards automatically (labels, naming, etc.).
- **Best practices:** auto-add defaults so teams don't have to remember them.
- **Multi-tenancy:** auto-create NetworkPolicies/quotas for every new namespace.
- **Supply-chain security:** only run signed, verified container images.
- **Auditing:** generate reports of what's compliant without blocking anyone yet.

---

## 11. How you'd actually try it (high-level steps)

1. **Install Kyverno** into a cluster (usually via Helm):
   ```
   helm install kyverno kyverno/kyverno -n kyverno --create-namespace
   ```
2. **Apply a policy** like any other YAML:
   ```
   kubectl apply -f require-team-label.yaml
   ```
3. **Test it** by trying to create a Pod that breaks the rule — Kyverno blocks it
   (in Enforce mode) or logs it (in Audit mode).
4. **Check reports:**
   ```
   kubectl get policyreport -A
   ```

> 💡 There's also a **Kyverno CLI** (`kyverno apply`, `kyverno test`) that lets you
> test policies on your laptop *without* a cluster — great for learning and CI
> pipelines. This is what a lot of the code in *this* repository powers.

---

## 12. The one-paragraph summary

**Kyverno is a security guard for Kubernetes.** Every time someone tries to create
or change a resource, Kyverno checks it against rules you wrote in plain YAML. It
can **block** bad resources (validate), **auto-fix** them (mutate), **auto-create**
related resources (generate), and **verify** that container images are trustworthy.
Because the rules are ordinary YAML — no special language to learn — it's far
easier to adopt than older tools. You start in **Audit** mode (just reports), gain
confidence, then switch to **Enforce** mode (actually block). That's Kyverno.

---

## 13. Where to go next

- Official docs: https://kyverno.io
- Ready-made policy library: https://kyverno.io/policies/ (copy-paste real policies)
- This repo: the actual source code of Kyverno (written in Go).

When you're ready, a good next step is to read one real policy from the policy
library and map every line back to Section 7 above. Once that clicks, you
understand Kyverno.
```
