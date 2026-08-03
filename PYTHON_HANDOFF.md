# Handoff: Python EKS Parallelization — Context, Confirmed Root Cause, and the IRSA Fix

> Purpose: give a fresh Claude Code instance (or the future you) everything needed to work on the
> Python EKS parallel E2E tests. Updated **2026-08-03** after root-causing the "every push to main
> breaks EKS validation" failure. Verify live state before acting — details below may have moved.
>
> ⚠️ **This revision CORRECTS several claims in the previous version of this file.** See
> [§ Corrections to earlier assumptions](#corrections-to-earlier-assumptions). If you are re-reading
> remembered context from a prior session, prefer this file.

---

## Repos & branches

Two repos under `/Users/ehrican/ADOT/`:

1. **Test framework** — `aws-application-signals-test-framework`
   - Reusable per-language workflows: `.github/workflows/python-eks-test.yml`
   - Terraform: `terraform/python/eks/main.tf`, `variables.tf`
   - Remotes: `origin` = `haneric00/...` (user fork), `upstream` = `aws-observability/...`
   - Original proven-working Python parallelization: branch **`EKS-test-fix`** (now merged upstream).
   - **IRSA fix (2026-08-03, drafted, unpushed): branch `fix-eks-irsa-setup-race`, cut from
     `upstream/main`.**
   - ⚠️ The working tree is often on **`java-eks-parallel-wip`** (Java work). Run
     `git branch --show-current` first. **Do not read that branch's workflows as if they were CI's** —
     see Correction #1.

2. **Caller repo** — `aws-otel-python-instrumentation`
   - Orchestrator: `.github/workflows/application-signals-e2e-test.yml`, invoked by `main-build.yml`
   - **IRSA fix branch: `fix-eks-irsa-setup-race`, cut from `upstream/main`.**
   - Remotes: `origin` = `haneric00/...`, `upstream` = `aws-observability/...`

**Read the code CI actually runs, not the working tree.** The caller pins
`uses: aws-observability/aws-application-signals-test-framework/...@main`, so CI executes
**upstream/main**. Use `git show upstream/main:<path>` when reasoning about a real failure.

---

## The parallelization model (proven)

5 Python version jobs (py310–py314) run **concurrently on ONE shared long-lived EKS cluster**
`e2e-python-adot-test` (account `654654176582`, `us-east-1`, nodegroup `eks-default-nodegroup-v2`
= 2× t3.medium, maxSize 2). Isolation via:

- **Per-version namespace slug** (DNS-1123, no dots):
  `PY_VERSION_SLUG="py$(echo $PYTHON_VERSION | tr -d '.')"` →
  `SAMPLE_APP_NAMESPACE="ns-${slug}-${run_id}-${run_number}"`.
- **`TESTING_ID`** carries the slug, so every resource name + `OTEL_SERVICE_NAME` is unique.
- **NodePort collision avoidance** — terraform `locals.version_offset` map (`3.10`=0 … `3.14`=4):
  `main_node_port=30100+offset*2`, `remote_node_port=30101+offset*2`. NodePort is cluster-wide, so
  fixed ports across parallel jobs would collide ("port already allocated").

Verified end-to-end: parallel run **#468** vs sequential **#440** showed per-version isolation, no
cross-talk, overlapping timestamps. Writeups: `PARALLEL_TESTING_PROOF.md`,
`EKS_TEST_ENVIRONMENT_MAP.md`.

---

# The sequential→parallel refactor: why two lifecycle layers are required

**This is the core mental model. Most breakage traces back to violating it.**

The test framework was written for **sequential, exclusive** cluster use. Under exclusivity, "set up
App Signals" and "tear down App Signals" could live *inside each test*, because only one test ever
touched the cluster at a time. Idempotence and ordering didn't matter — exclusivity supplied both.

Parallelizing removes exclusivity, which **retroactively turns those in-test setup/teardown calls
into race conditions.** So yes — your understanding is correct, and the precise formulation is:

> **Resources must be lifecycle-managed at the scope at which they are shared.**
> Cluster-wide state gets **pipeline-scoped bookends** (exactly once, before and after the fan-out).
> Per-test state stays **job-scoped**. And per-job steps must treat shared state as **read-only**.

That's three rules, not two — the third is the one that's easy to miss.

### Scope table

| Resource | Scope | Owner | Rule |
|---|---|---|---|
| `amazon-cloudwatch-observability` addon | **cluster-wide** | `eks-setup` / `eks-cleanup` | Never create/delete from a test job |
| `cloudwatch-agent` IRSA service account | **cluster-wide** | `eks-setup` / `eks-cleanup` | Never recreate from a test job |
| `cloudwatch-agent` DaemonSet | **cluster-wide** | `eks-setup` only | Never restart from a test job (drops siblings' in-flight telemetry) |
| Operator `--auto-instrumentation-python-image` arg | **cluster-wide, single value** | test jobs (idempotent) | All jobs write the *same* value; patch only if it differs, retry on conflict |
| `ns-<slug>-<run>` sample app namespace | **per job** | test job | Delete only your own, never `ns-*` in bulk |
| `service-account-<TESTING_ID>` (app's AWS access) | **per job** | test job | Named by `TESTING_ID`, safe |
| Terraform state / NodePorts | **per job** | test job | Isolated via slug + `version_offset` |

### The three layers in practice

1. **`eks-setup-app-signals.yml`** (NEW, pipeline-scoped, runs FIRST) — installs addon + IRSA SA,
   waits for `ACTIVE`, **asserts the agent actually has credentials**, restarts the DaemonSet if not
   (safe *only here*, because no test is running yet).
2. **`python-eks-test.yml` / `python-eks-service-events-test.yml`** (job-scoped, run in parallel) —
   **assert-only** on shared state; fail fast with a clear message. Mutate only per-job resources.
3. **`eks-cleanup-app-signals.yml`** (pipeline-scoped, runs LAST) — deletes addon + IRSA SA, gated on
   every EKS job + a guard that refuses to run while any `ns-*` namespace is alive.

**Why "assert-only" is a distinct rule:** the obvious fix for a missing-credentials agent is
"restart the DaemonSet." In a per-job step that is *itself* a new race — job A's restart drops job
B's in-flight telemetry. Self-healing belongs exclusively in the pipeline-scoped layer.

---

# What went wrong: the IRSA injection race (CONFIRMED 2026-08-03)

### Symptom

Every push to main → `python-eks-test.yml` fails at `log-validation`:

```
> Task :validator:run
FAILURE: Build failed with an exception.
Execution failed for task ':validator:run'.
> Process 'command '.../bin/java'' finished with non-zero exit value 1
BUILD FAILED in 6m 49s
```

Misleading, because the validator is healthy — it's correctly reporting **absent telemetry**.
`CWLogValidator` retries `RetryHelper.retry(40, ...)` with `throwExceptionInTheEnd=true`, so missing
data = hard red after ~6–7 min. There is no soft-pass.

### Evidence chain

1. CloudWatch Insights on `/aws/application-signals/data`: **0 records** for the run's namespace,
   **0 records with `PlatformType = "AWS::EKS"` in the entire preceding 24h.** Only `AWS::EC2`,
   `AWS::Lambda`, `AWS::ECS` were flowing. Not a per-run flake — the whole EKS path was dark.
2. Addon reported `ACTIVE` / healthy, `serviceAccountRoleArn: null`, **no pod-identity associations**.
3. Node role `adot-python-e2e-release-t-e2epythonadottesteksdefau-ngcVDNcc0nbz` has only
   `AmazonSSMManagedInstanceCore`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`,
   `AmazonEKSWorkerNodePolicy` — **no `logs:PutLogEvents`**.
4. **Agent logs, the smoking gun:**
   ```
   AccessDeniedException: User: arn:aws:sts::654654176582:assumed-role/
     adot-python-e2e-release-t-e2epythonadottesteksdefau-ngcVDNcc0nbz/i-07c73d37a7238197d
     is not authorized to perform: logs:PutLogEvents on resource:
     .../log-group:/aws/application-signals/data:log-stream:otel-stream-...
     because no identity-based policy allows the logs:PutLogEvents action
   ```
   The agent was authenticating as the **EC2 node role**, not IRSA.
5. Agent pods had **no `AWS_ROLE_ARN` env var and no `aws-iam-token` volume** — IRSA was never
   injected. The IRSA role itself (`eksctl-e2e-python-adot-test-addon-iamservicea-Role1-...`) was
   correct all along: `CloudWatchAgentServerPolicy` + `AWSXrayWriteOnlyAccess`.

### Mechanism

- **IRSA credentials are injected by a mutating admission webhook at pod-admission time ONLY.** A pod
  admitted while its service account lacks the `eks.amazonaws.com/role-arn` annotation runs
  credential-less **for its entire lifetime** and never self-heals.
- `enable-app-signals.sh` runs
  **`eksctl create iamserviceaccount --name cloudwatch-agent ... --override-existing-serviceaccounts`
  UNCONDITIONALLY** (script lines ~40–49). That step is *not* guarded by an "already exists" check —
  **only the addon install is** (`if [[ "${result}" == *"No addon: "* ]]`).
- So with N parallel jobs each running the script, this interleaving is reachable:
  1. Job A creates the SA, starts `create-addon`, addon reconciles, **DaemonSet pods get admitted**.
  2. Job B runs the same script seconds later → `--override-existing-serviceaccounts` **deletes and
     recreates the SA underneath A's already-admitted pods.**
  3. Those pods keep running with stale/absent projected credentials → node-role fallback → **403 on
     every export, forever** → all jobs' telemetry dropped → every validator fails on missing data.

### Why it broke on main but passed twice on the feature branch

**The race is nondeterministic — a coin flip, not a deterministic failure.** A cold cluster doesn't
guarantee failure; it guarantees you *roll the dice*. Whoever wins the interleaving decides the run.

The asymmetry is **how many times you roll**:

| | Cold starts | Race exposure |
|---|---|---|
| Feature branch | at most once (`eks-cleanup` is wired only into the main-build caller graph) | one roll, then the cluster stays **warm** → green run 2, 3, 4… |
| **main** | **every run** — `eks-cleanup` tears the addon + IRSA SA down at the end of every pipeline | **a fresh roll on every push** |

So `eks-cleanup` (itself a correct fix) guarantees main re-enters the race every time, while a
feature branch rolled once and then coasted on warm, already-credentialed agent pods. "It built twice
in a row on my branch" is *exactly* what a warm cluster looks like — which is why this felt fixed
when it wasn't. **Consecutive green runs on a warm shared cluster are not evidence that cold-start
setup is correct.**

### Fast diagnostic (do this FIRST for any "validator says no telemetry" failure)

```bash
aws eks update-kubeconfig --name e2e-python-adot-test --region us-east-1

# Is IRSA actually projected into the agent pods?
kubectl get pods -n amazon-cloudwatch -l app.kubernetes.io/name=cloudwatch-agent \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[0].env[?(@.name=="AWS_ROLE_ARN")].value}{"\n"}{end}'

# Empty value == lost the race == guaranteed 403s. Confirm:
kubectl logs -n amazon-cloudwatch -l app.kubernetes.io/name=cloudwatch-agent --tail=200 \
  | grep -i accessdenied
```

Manual unblock (safe — only bounces the telemetry agent, touches no test data):
```bash
kubectl rollout restart daemonset/cloudwatch-agent -n amazon-cloudwatch
kubectl rollout status  daemonset/cloudwatch-agent -n amazon-cloudwatch --timeout=300s
```
This was performed on 2026-08-03 and verified: both new pods received `AWS_ROLE_ARN` + the
`aws-iam-token` volume, and `AccessDenied` count dropped to 0.

Also useful, no kubectl required — if this returns 0, the EKS path is dark regardless of any one run:
```
fields Service | filter PlatformType = "AWS::EKS" | stats count(*) as c
```

---

## The drafted fix (2026-08-03, unpushed)

Branch **`fix-eks-irsa-setup-race`** in **both** repos, cut from `upstream/main`.

**Framework:**
- **NEW `.github/workflows/eks-setup-app-signals.yml`** — pipeline-scoped setup: runs
  `enable-app-signals.sh` once, waits for addon `ACTIVE`, then **asserts every agent pod has
  `AWS_ROLE_ARN`** and restarts the DaemonSet (up to 6×) if not. Mirror image of `eks-cleanup`,
  including the "warn if `ns-*` namespaces are live" guard.
- **`python-eks-test.yml`** — removed the per-job `enable-app-signals.sh` invocation. Now
  **assert-only**: addon `ACTIVE` + agent IRSA present, else fail fast with an explicit error instead
  of timing out in a validator 30 minutes later.
- **`python-eks-service-events-test.yml`** — same de-racing, **plus** replaced the bulk
  `kubectl get namespace | awk '/^ns-[0-9]+-[0-9]+/' | xargs kubectl delete namespace` with a scoped
  `kubectl delete namespace "$SAMPLE_APP_NAMESPACE" --ignore-not-found`.

**Caller (`application-signals-e2e-test.yml`):**
- Added `eks-setup` job; gated all 5 `eks-py*`, `service-events-eks`, and `eks-cleanup` on it.
- Registered the new file in `exclusions:` so `validate-e2e-tests-are-accounted-for` still passes.

Verified: all 4 files parse as valid YAML; job graph has no dangling `needs`; zero remaining
`enable-app-signals.sh` invocations or bulk `ns-*` deletes in the EKS test workflows.

### ⚠️ Merge ordering — will break CI if ignored

The caller references `eks-setup-app-signals.yml@main`. **That file must exist on upstream framework
main BEFORE the caller change merges**, or every EKS job fails instantly on an unresolvable workflow
ref. Either merge framework → then caller, or temporarily pin the caller's `uses:` to the branch for
a test run (and revert before merge).

### Not yet verified

- The fix has **not** been exercised against a live cold-started cluster. The `jsonpath`+`awk` IRSA
  assertion is syntactically valid but unproven at runtime.
- The `enable-app-signals.sh` download + `chmod` steps were intentionally left in both test
  workflows (harmless: downloaded, never invoked). Removing them risks breaking the `sed` on
  `clean-app-signals.sh` and the `execute_and_retry` wiring — deliberately kept the diff minimal.
- **Cluster state as of 2026-08-03:** addon **absent**, `amazon-cloudwatch` namespace **empty**, IRSA
  SA **deleted**, no `ns-*` namespaces. This is normal post-`eks-cleanup` state, but it means a run
  on *current* main will still cold-start into the race until this fix lands.

---

## Corrections to earlier assumptions

Recorded because each cost real debugging time.

1. **`python-eks-service-events-test.yml` does NOT invoke `clean-app-signals.sh` on upstream main.**
   An earlier session claimed SE tore down the shared addon per-job and was the trigger. That came
   from reading the **local `java-eks-parallel-wip` working tree**, not `upstream/main`. On
   upstream/main, SE only *downloads* the script. **Always `git show upstream/main:<path>`.**
   (SE *did* still bulk-delete `ns-*` namespaces — a real, separate bug, now fixed.)

2. **The per-job teardown fix IS merged upstream.** `python-eks-test.yml@main` does not invoke
   `clean-app-signals.sh`; teardown correctly lives only in `eks-cleanup-app-signals.yml`. That fix
   was not the regression. Don't re-chase it.

3. **"Branch ref mismatch is most likely" was a dead end here.** The previous version of this file
   led with it. The refs resolved correctly; the caller legitimately pulls `@main` and the teardown
   fix was present there. Still worth a 30-second check, but **check the live cluster's IRSA state
   first** — it's faster and was decisive.

4. **`enable-app-signals.sh` does not "no-op" the part that matters.** The SA creation is
   unconditional on every invocation; only the *addon install* is conditional. "Just force it to run"
   is therefore not a fix — it already runs in full, and forcing it harder re-triggers the race.

5. **Addon `ACTIVE` does not imply the agent has credentials.** `python-eks-test.yml` already waited
   for `ACTIVE` (30×20s) and still failed. `ACTIVE` means the addon reconciled; it says nothing about
   whether the DaemonSet pods won the injection race. This gap is what the new assertion closes.

---

## Other known gotchas (still valid)

1. **CloudWatch Logs quirks.** Dotted keys like `$.Telemetry.Source` / `$.K8s.Namespace` parse as
   nested and fail — filter on non-dotted keys (`$.Service`). `$.Namespace` is the METRIC namespace
   (`ApplicationSignals`); the K8s namespace lives in `$.Environment`. In Insights, backtick dotted
   fields: `` `K8s.Namespace` ``.
2. **Confirm WHICH COMMIT ran.** `main-build` triggers only on push to `main`/`release/v*`, not
   feature branches. Pushing to `origin` but not `upstream` has caused "why didn't my change take"
   confusion more than once.
3. **Unrelated pre-existing flaky failure (NOT parallelization):**
   `terraform/python/k8s/cleanup/main.tf:50-51` — the **k8s / EC2** suite, not eks — runs
   `aws ssm delete-parameter`, which throws `ParameterNotFound`; as the last command in the block its
   exit 254 defeats `set +e` and fails the provisioner.
4. **Cluster access.** These clusters use `CONFIG_MAP` auth (`aws-auth`). An IAM role that isn't
   mapped gets `kubectl: must be logged in to the server` despite valid AWS creds. As of 2026-08-03
   the `Admin` role (`arn:aws:sts::654654176582:assumed-role/Admin/ehrican-Isengard`) **does** have
   working kubectl access. Use `ada credentials update` to refresh expired creds.

---

## Security constraints (persist)

Prefer ReadOnly / least-privilege creds. Never delete or modify production/shared resources without
explicit user confirmation; assume production when uncertain. **Don't commit or push — the user has
repeatedly asked that git stay with them for review.**

---

## Related Java context (cross-reference only)

The same parallelization was ported to Java (`java-eks-test.yml`, `terraform/java/eks`, caller
`parallelize-EKS-and-lambda-cache` in `aws-otel-java-instrumentation`). **The Java EKS path shares
`enable-app-signals.sh` and therefore has the same IRSA race** — it needs the same
`eks-setup`/assert-only treatment (cluster `e2e-adot-test`). Two Java-only issues also surfaced:
(a) a 63-char pod-name truncation broke trace validation for all-but-first version — fixed by
shortening `sample-r-app-deployment-` → `java-remote-` in `terraform/java/eks/main.tf`;
(b) a `traffic-generator` readiness timeout on 2-of-5 jobs, suspected node capacity (JVMs heavier
than Python interpreters; only mysql declares resource requests) — unconfirmed on-cluster.
