# SRE Capstone Roadmap and Outline

Practical Site Reliability Engineering for the AI-Augmented Era

This document is the working plan for the capstone. It covers the goals, the key decisions, the architecture, the day-by-day roadmap, the demo script, and the checklist that tells us when we are done. The guiding idea is that we are graded on how well an existing application is brought under a code-driven operating model. New application features earn nothing, so every task below serves that goal.

---

## 1. Goals and success criteria

The platform has to show all of the following.

- SLOs and error budgets that are documented, tied to user-facing symptoms, and wired to burn-rate alerts instead of static thresholds.
- Infrastructure and configuration held in a known good state through Terraform, Ansible, and Helm, with reruns that are idempotent and safe.
- A CI/CD pipeline that is the only path to production and enforces pull-request review before promotion.
- Monitoring and alerting that would catch the demo failure before a customer files a ticket.
- A visible, auditable trail of where AI assistants were used and how the output was verified.
- A blameless postmortem, drafted with AI and corrected by the team, that ties remediation back to the error budget.
- A demo that can be reproduced from a clean clone of the repository.

The hard rules from the brief are worth repeating here.

- No manual production-style changes. If it cannot be reproduced from a clean clone, it does not count.
- Direct cluster or host access is for diagnosis only and never for the fix.
- Every AI-generated artifact is reviewed by a teammate before it is committed or executed, and the review must be visible.
- AI assistants get no production credentials, no customer data, and no access outside the lab.
- No plaintext credentials in the repository.

---

## 2. Decisions to lock on day 0

| Decision | Default recommendation | Notes |
|---|---|---|
| Load balancer | HAProxy on a VM if Terraform VMs are available, otherwise Traefik through Helm | See section 3 |
| Cluster | Whatever the lab provides, kind as the local rehearsal fallback | Confirm baseline namespaces |
| CI/CD | GitHub Actions unless the lab mandates Jenkins | Use Environments for promotion |
| Secrets | Ansible Vault for host config and the CI secret store for pipeline values | Add gitleaks to lint |
| Metrics and dashboards | Prometheus, Alertmanager, Grafana | Provision dashboards as code |
| Logs | Loki with structured JSON logs | Needed for AI log analysis |
| SLO tooling | Sloth or Pyrra to generate rules from a spec | Keeps SLOs reviewable |
| Load testing | k6 with a baseline profile and a failure profile | Scripted so anyone can rerun it |
| Failure scenario | Memory limit too low or a bad timeout value | See section 13 |

Also confirm with the instructors that the sample app exposes a status-labeled request counter and a latency histogram, that its logs are structured, and how the load generator triggers degraded mode.

---

## 3. Load balancer selection

Avoid the community ingress-nginx controller. Its maintenance ended in March 2026, so it receives no further releases or security fixes and is a poor choice for a new build.

Recommended options, in order of preference for this project.

**HAProxy on a VM.** Terraform provisions the VM and Ansible installs and configures HAProxy from a template, with secrets in Ansible Vault. Traffic is forwarded to the cluster through a NodePort or ingress. This gives Ansible a real and idempotent job, and the load balancer becomes the best place to measure the user-facing SLI. HAProxy exposes a built-in Prometheus endpoint.

**Traefik through Helm.** Best when only the cluster is available. It deploys quickly, supports Gateway API, and exposes Prometheus metrics with latency histograms. Timeouts, retries, and weighted routing are simple to configure.

**Envoy Gateway.** A strong Gateway API native choice with rich outlier detection, at the cost of more setup time.

**MetalLB or cloud-provider-kind.** Needed only if a kind cluster requires LoadBalancer addresses.

Load balancer features worth showing in the demo.

- Active health checks, plus a scripted backend removal.
- Explicit timeouts and a bounded retry policy.
- Weighted routing for a canary rollout.
- Optional rate limiting.
- JSON access logs shipped to Loki.
- Edge latency histograms as the source for the latency SLI.

A misconfigured load balancer timeout or an over-aggressive retry setting also makes a realistic and educational failure.

---

## 4. Target architecture

```
Load generator (k6)
        |
Open source load balancer (HAProxy or Traefik)
        |
Kubernetes application (Helm, probes, limits, PDB)
        |
Downstream dependency

Prometheus scrapes the load balancer and the application
Prometheus -> Alertmanager -> on-call webhook
Loki collects logs -> Grafana
Alert enrichment service -> AI triage summary attached to the page
GitHub Actions -> Terraform, Ansible, Helm -> everything above
```

---

## 5. Repository layout

```
repo/
  terraform/        namespaces, quotas, load balancer VM, monitoring releases
  ansible/          load balancer role, host hardening, vault-encrypted vars
  helm/app/         chart plus values-dev, values-staging, values-prod
  observability/    SLO specs, generated rules, promtool tests, dashboards, alertmanager
  policy/           conftest or OPA rules
  loadtest/         k6 baseline and failure profiles
  .github/          workflows, PR template, CODEOWNERS
  docs/
    slo.md
    runbooks/
    postmortem/
    ai-log.md
    incident-fixtures/
  README.md
```

The README should explain how to bootstrap from a clean clone, how to run the pipeline, how to run the load profiles, and where each deliverable lives.

---

## 6. SLIs, SLOs, error budgets, and burn-rate alerts

**SLIs.** Use two ratio-based indicators measured at the load balancer where possible.

- Availability is the count of non-5xx responses divided by total responses.
- Latency is the count of responses faster than the chosen threshold divided by total responses, computed from histogram buckets. Make sure a bucket exists at or near the threshold.

**SLO targets.** A 99.5 percent or 99.9 percent target over 30 days is defensible. Write down why the number fits the service and what the resulting budget is in minutes of full outage.

**Burn-rate alerts.** Use multiwindow, multi-burn-rate alerts.

| Severity | Burn rate | Long window | Short window | Budget consumed if sustained |
|---|---|---|---|---|
| Page | 14.4x | 1h | 5m | 2 percent |
| Page | 6x | 6h | 30m | 5 percent |
| Ticket | 1x | 3d | 6h | 10 percent |

Each row is a justification in itself. A 14.4x burn over one hour spends 2 percent of a 30-day budget, and the short window makes the alert reset quickly once the problem is fixed.

Example expression for the fast page on a 99.9 percent availability SLO. Metric names are placeholders and must match the app.

```
(
  sum(rate(http_requests_total{code=~"5.."}[1h]))
  / sum(rate(http_requests_total[1h])) > (14.4 * 0.001)
)
and
(
  sum(rate(http_requests_total{code=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) > (14.4 * 0.001)
)
```

**Demo window profile.** A 30-day window cannot fill during a 3 to 4 day project. Keep the production windows in the documentation and add a demo profile in the Helm values that compresses the windows, for example 5m and 1m for the fast page. The burn multiples stay the same, but the fraction of budget each window represents changes. State this openly in the deck.

**Error budget policy.** Document what happens as the budget drains. For example, below 50 percent remaining, risky changes need extra review, and at zero the pipeline blocks promotion to production. Implement the gate by querying Prometheus for remaining budget during the deploy stage.

**Alert testing.** Write `promtool test rules` cases with synthetic series proving each alert fires when it should and stays quiet when it should not. Run them in CI.

---

## 7. Infrastructure as code plan

- **Terraform** owns cluster-adjacent resources such as namespaces, resource quotas, the load balancer VM, and the monitoring stack releases. Use the remote state backend the lab provides.
- **Ansible** owns host configuration, the load balancer role, and runner setup. Encrypt sensitive variables with Ansible Vault.
- **Helm** owns the application and its per-environment values.

Idempotency proof. In CI, run `terraform plan` and `ansible-playbook --check --diff` and show that a second run reports no changes. Add a scheduled job that runs `terraform plan -detailed-exitcode` and raises an alert when drift is detected.

---

## 8. Kubernetes deployment standards

- Readiness, liveness, and startup probes with sensible thresholds.
- CPU and memory requests and limits on every container.
- A PodDisruptionBudget and a rolling update strategy with `maxUnavailable` set to zero.
- Optional HPA if the load profile justifies it.
- Read-only RBAC for humans and a service account that only the pipeline uses for writes.
- Stretch goal is Argo Rollouts with an analysis step that queries the same SLI and rolls back automatically. The demo fix must still be a new commit, so present this as a safety net only.

---

## 9. CI/CD pipeline design

Pull request stages.

1. **Lint.** yamllint, helm lint, kubeconform, terraform fmt and validate, tflint, ansible-lint, gitleaks.
2. **Policy.** conftest or OPA rules for required limits, no latest image tags, and a minimum memory floor. Checkov or tfsec for Terraform.
3. **Test.** promtool rule tests, chart template rendering, and any application unit tests.
4. **Plan.** Terraform plan and Ansible check mode output posted to the pull request.

Merge and promotion stages.

1. Deploy to dev and run a smoke test.
2. Promote to staging and run a longer smoke test.
3. Require a reviewer approval through a GitHub Environment before production.
4. Deploy to production, annotate Grafana with the deploy, and verify SLIs after rollout.

Enforcement. Branch protection, CODEOWNERS, required reviews, and a rule that only the pipeline holds write credentials to the cluster.

---

## 10. Observability and AIOps

- **Dashboards as code.** Provision Grafana dashboards from the repository. Include the SLI panels, budget remaining, burn rate, load balancer health, pod restarts, and resource usage against limits.
- **Deploy annotations.** Mark every deploy on the dashboards so triage can see what changed right before a burn.
- **Alertmanager.** Group related alerts, route by severity, and use inhibition so that a latency alert is suppressed while an availability alert is already firing.
- **Paging path.** A webhook to Slack or email that behaves like a page, with the runbook linked in the alert annotations.
- **AI enrichment service.** A small service that receives the alert, gathers recent deploys, the top error log lines, and pod events, asks an approved assistant for a triage summary, and attaches it to the page. Use lab data only.
- **Anomaly detection.** Keep it simple with z-score or `predict_linear` recording rules, and compare paging volume with and without them.
- **Noise reduction metric.** Replay the same load run with and without grouping and inhibition and count the pages. Put the result in the deck.

---

## 11. AI usage and verification trail

The verification step has to be visible and not merely asserted. Build it into the process.

- A pull request template with an AI-assisted section.
- An `ai-assisted` label on every relevant pull request and a commit trailer naming the tool.
- A required reviewer who is not the author.
- A running `docs/ai-log.md` that records each AI use, what it produced, what the human changed, and how it was verified.
- Record the mistakes too. Examples include invented Terraform arguments, wrong burn-rate windows, or a suggestion to restart a pod instead of fixing a limit. Documented misses are the strongest proof that verification is real.

Pull request template section.

```
## AI assistance
- Tool used
- What it produced
- What a human changed
- How it was verified (plan output, promtool test, log or metric check)
- Reviewer
```

---

## 12. Test data and load generation

There is no static dataset in the brief. The data is generated by running load against the app.

- **Baseline profile.** A k6 script run steadily for 30 to 60 minutes before the demo so dashboards and alert windows have data.
- **Failure profile.** A second scripted profile started by a make target or pipeline job.
- **Synthetic series.** Hand-written series for promtool alert tests.
- **Optional backfill.** `promtool tsdb create-blocks-from openmetrics` can create weeks of fake history so budget graphs look realistic.
- **Fixture bundle.** After a rehearsal, save the logs, key query results, and Kubernetes events into `docs/incident-fixtures/`. Use it to test the triage prompt repeatedly and as a fallback if the live failure misbehaves.
- **Rehearsal apps.** OpenTelemetry Demo, Online Boutique, or podinfo work well in a private kind cluster if the sample app is too limited.
- **Log practice.** Public log collections such as Loghub can tune log summarization prompts offline. Never feed real work systems or data to an assistant.

---

## 13. Demo scenario

The story is that a developer ships a change, the platform catches a regression through SLO burn, and the team fixes it through code.

**Failure choices.**

- A memory limit set too low, which causes OOM kills and an availability burn.
- A bad configuration value such as a tiny connection pool or a wrong timeout, which causes a latency burn.
- A dependency timeout injected with toxiproxy or the load generator degraded mode.

Pick a value that passes lint and the policy floor but is still too low under load. That way the pipeline works as designed and the runtime signal is what catches the problem, which makes a good honest lesson for the deck.

**Run of show for 15 to 20 minutes plus Q&A.**

1. Frame the system, the SLOs, and the budget policy. Two minutes.
2. Open a pull request with an ordinary change and walk it through review and the pipeline to production. Three minutes.
3. Start baseline load and show healthy dashboards. One minute.
4. Merge the change that introduces the failure. Two minutes.
5. The burn-rate alert fires and pages. Explain what it measured and why it fired when it did. Two minutes.
6. Use the AI assistant for triage, then verify its suspected root cause against logs and metrics. Three minutes.
7. Push the fix as a new commit and watch it flow through the pipeline. Two minutes.
8. Confirm recovery on the same dashboards with no one logging into a pod. One minute.
9. Show the AI trail, the postmortem, and what we would do differently. Three minutes.

Record a backup video of a clean run.

---

## 14. Runbooks and postmortem

**Runbooks.** Write at least two.

- High burn rate from crash loops or OOM kills.
- Dependency timeout or elevated latency.

Additional candidates are a bad rollout rollback, error budget exhaustion, and certificate expiry. Draft with AI, then mark clearly which sections a human corrected or verified. Link each runbook from the matching alert annotation.

**Postmortem.** Blameless and specific.

- Summary and user impact.
- Timeline built from Git history, Alertmanager timestamps, and deploy annotations.
- Detection, including time to detect and why the alert fired when it did.
- Contributing factors framed as system gaps and never as people.
- Error budget consumed, expressed in percent and in minutes.
- What went well and what did not.
- Remediation items as tracked issues with owners, including at least one that would catch this class of problem at pull request time.
- A short section noting what the AI draft got wrong and how the team corrected it.

---

## 15. Day-by-day roadmap

**Day 0 recon and decisions, a few hours**

- Confirm app metrics and log format, cluster type, runners, and secrets store.
- Lock the load balancer choice.
- Create the repository, branch protection, CODEOWNERS, and the pull request template.
- Assign starting roles and plan the rotation.
- Exit criteria. Every decision in section 2 has an owner and an answer.

**Day 1 foundation**

- Helm chart with probes, limits, PDB, and safe rollout settings.
- Load balancer deployed and routing to the app.
- Prometheus scraping both, Loki collecting logs, baseline dashboards, draft SLO document, baseline k6 profile.
- Exit criteria. The app is reachable through the load balancer and dashboards show live traffic.

**Day 2 pipeline and infrastructure as code**

- Lint, policy, and test stages on pull requests, then dev to staging to production promotion with a required reviewer.
- Terraform and Ansible in the pipeline with plan and check output, secrets in Vault or the CI store, and a scheduled drift job.
- SLO rules generated from a spec, with promtool tests.
- Human RBAC locked to read-only.
- Exit criteria. A pull request travels from commit to production through the pipeline and a second run shows zero changes.

**Day 3 alerting, AI, and runbooks**

- Burn-rate alerts with the demo window profile, plus Alertmanager routing and inhibition.
- Paging webhook and the AI enrichment service.
- Two runbooks with corrections marked, and the AI log filled in as work happens.
- Script the failure scenario and run a full dress rehearsal.
- Exit criteria. The injected fault pages within the target time and the AI summary arrives with the page.

**Day 4 demo and documentation**

- Two more rehearsals and a recorded backup video.
- Postmortem, README, exported dashboards and alert rules, and the final deck.
- Final test. Clone the repository fresh and reproduce the demo.
- Exit criteria. The clean clone reproduces every step.

If the project is only 3 days, merge days 3 and 4 and cut stretch items first.

---

## 16. Team roles and pair rotation

Suggested lanes, rotated at least once so everyone can explain every part.

- **Infrastructure and load balancer.** Terraform, Ansible, HAProxy or Traefik, RBAC, secrets.
- **Pipeline and application.** Helm chart, CI/CD stages, policy checks, promotion.
- **Observability and AI.** SLOs, alert rules and tests, dashboards, enrichment service, runbooks, AI log.

Everyone contributes to the postmortem and the deck, and every person should be ready to narrate a part they did not build.

---

## 17. Deliverables map

| Deliverable | Location |
|---|---|
| Single repository with clear README | repository root |
| SLO and error budget documentation | docs/slo.md |
| Working CI/CD pipeline | .github/workflows |
| Monitoring and alerting configuration | observability/ plus exported screenshots |
| Two AI-assisted runbooks | docs/runbooks/ |
| Blameless postmortem | docs/postmortem/ |
| AI audit trail | PR template, labels, docs/ai-log.md |
| Final presentation deck | separate file |

---

## 18. Risks and cut lines

- **Scope creep.** One scenario done well beats three done halfway. Argo Rollouts, anomaly detection, and Gateway API are optional.
- **Manual fixes.** Anything done by hand does not count, so every fix goes through a commit.
- **Demo flakiness.** Keep fixtures and a backup video ready.
- **Compressed SLO windows.** Explain them early so reviewers do not read them as a mistake.
- **AI misuse.** Keep assistants away from anything outside the lab and keep the review evidence visible.

---

## 19. Definition of done

- [ ] Clean clone reproduces the full demo
- [ ] No plaintext secrets in the repository or history
- [ ] Pipeline is the only path to production and humans hold read-only access
- [ ] SLOs, burn-rate thresholds, and justifications documented
- [ ] Alert rules covered by promtool tests
- [ ] Second run of Terraform and Ansible shows no changes
- [ ] Load balancer health checks, timeouts, and metrics demonstrated
- [ ] Two runbooks with human corrections marked
- [ ] AI log and pull request evidence show verification for each use
- [ ] Postmortem ties remediation to error budget
- [ ] Dashboards and alert rules exported for review
- [ ] Deck rehearsed and backup video recorded

---

## 20. Presentation outline

1. The problem and our operating model
2. Architecture and the load balancer choice
3. SLOs, budgets, and why the alerts fire when they do
4. Commit to production walkthrough
5. Live or recorded incident cycle
6. How AI helped and how we verified it
7. Results, including time to detect, time to recover, budget consumed, and paging noise before and after
8. What worked, what did not, and what we would do differently
9. Questions
