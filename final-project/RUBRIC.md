# CSS4005 Final Project — Grading Rubric (40 points)

| Part | Points |
|---|---|
| **Report** (graded with the table below) | **20** |
| **Presentation & demo** | **8** |
| **Oral defense** | **12** |
| **Total** | **40** |

Each report area is scored at three levels:

- **Full**: everything in the README checklist is done **and** shown with
  evidence, and your variant values are applied.
- **Partial** (about half): built but evidence is missing or weak, some
  checklist items are missing, or variant values are not applied.
- **None** (0): missing, not working, or only described with no evidence.

---

## 1. Report — 20 points

| # | Area | README § | Points | "Full" means |
|---|---|---|---|---|
| R1 | Variant applied + architecture (diagram, shared responsibility, ≥ 4 ADRs) | 4, 5.8 | 2 | Variant table complete with evidence; diagram matches what was built; ADRs give real trade-offs |
| R2 | Networking | 5.1 | 2 | 3 tiers × 2 zones from your CIDR, SG/firewall chaining by reference, second layer, DB unreachable test |
| R3 | Compute, Linux hardening | 5.2 | 1 | Templates + startup scripts for nodes and DB, non-root, no public SSH, no keys in user data |
| R4 | **Kubernetes cluster** | 5.4 | 3 | Cluster per your `cluster` variant over 2 zones; Deployment with requests/limits + both probes; zone spreading; Service + Ingress behind the cloud LB; HPA at `hpa_cpu_target`; rolling update + rollback; node/zone drill with pods rescheduled |
| R5 | Load balancing & auto scaling | 5.5 | 2 | LB in 2 zones with `/readyz` checks; pod + node scaling fit to `traffic_pattern`; load-test graphs; self-healing; HTTPS |
| R6 | Storage | 5.3 | 1 | Versioning, lifecycle, BPA, encryption, signed URLs, separate DB disk, scheduled snapshots with variant retention |
| R7 | **Self-hosted database** | 5.6 | 3 | Primary + replica in 2 zones (+ ClickHouse if required); 3 roles with denied actions; password policy rejecting weak passwords; backups meeting RPO; restore test and failover drill timed vs RTO; managed-vs-self-hosted comparison |
| R8 | IAM & secrets | 5.7 | 2 | Roles/service accounts least privilege (or documented ideal policy on Learner Lab); all secrets in the cloud secret manager, fetched at boot/deploy; nothing secret in git/images/state outputs; rotation plan |
| R9 | Containers & CI/CD | 5.9 | 1 | Lint/test/build/scan/push with immutable tags; deploy to the cluster gated by readiness; rollback demonstrated |
| R10 | Security checks | 5.10 | 1 | gitleaks, image scan, IaC scan, password tests run in CI; findings table complete with status |
| R11 | Infrastructure as Code | 5.11 | 1 | Everything in Terraform modules (+ manifests/Helm); protected state; fmt/validate in CI; timed destroy → apply rebuild |
| R12 | Monitoring, logging, HA drills & cost | 5.12, 5.13 | 1 | Dashboards incl. pods/DB; ≥ 4 alerts with variant thresholds, one firing → resolving; `request_id` log query; zone drill; cost estimate vs actual, budget alert at cap, ≥ 3 optimizations; teardown proof |
| | **Total** | | **20** | |

---

## 2. Presentation & demo — 8 points

| Criterion | Points |
|---|---|
| Clear architecture walk-through tied to **your** variant | 2 |
| Live demo (or demo video) of the working platform through the LB, incl. the cluster (`kubectl get pods -o wide`, HPA) | 3 |
| One drill shown live or on video (node/zone failure, DB failover or rollback) | 2 |
| Time kept (~7 min), readable slides/diagram | 1 |

## 3. Oral defense — 12 points

Four questions, 3 points each (3 = correct and explained in your own
words with reference to your build, 2 = mostly correct, 1 = vague, 0 =
can't answer). At least one question is about a file in **your**
repository you must explain line by line, and at least one is about the
Kubernetes cluster or the self-hosted database.

---

## 4. Penalties and caps

| Situation | Effect |
|---|---|
| A secret (password, key, token) committed to the repo or baked into an image | −4 points and must be rotated before grading |
| Database reachable from the internet | Report capped at **10 / 20** |
| Managed DB (RDS / Cloud SQL) used instead of a self-hosted engine | R7 = 0 and report capped at **12 / 20** |
| Database placed inside the Kubernetes cluster | R7 capped at partial (1.5 / 3) |
| Variant ignored (wrong CIDR, prefix, engine, cluster type, thresholds) | Report capped at **12 / 20** |
| Cannot explain own code at the defense | Defense capped at **4 / 12** |
| Plagiarism (shared code/report, another student's platform) | 0 for the whole project, reported per university policy |
| Resources left running after grading with no teardown proof | −2 points |
| Kubernetes cluster shown **only on a local kind / k3d cluster** because cloud credits ran out (see FREE_TIER_GUIDE.md §7) | R4 capped at **half (1.5 / 3)**; the cloud LB → cluster path in R5 counts as partial. Every other area still needs **cloud** evidence (screenshots, outputs) or the demo video recorded while it ran in the cloud |

Documented **Learner Lab limitations** (no IAM role creation, blocked
services, session expiry) are **not** penalised when the report explains
the limitation and the real-account design.
