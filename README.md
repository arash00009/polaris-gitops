# polaris-gitops

Deployment configuration for the [Polaris AI Platform](https://github.com/arash00009/polaris-ai-platform) — the second half of Phase 8's repository split (see that repository's `docs/adr/README.md`, ADR-08 and ADR-26).

> **Honest scope.** This repository holds *what to deploy and where*: Argo CD `Application` manifests and one image-tag fragment per environment. It has no application source code, no Dockerfile, and no CI pipeline of its own. Everything about *how* the application is built, tested and containerised lives in `polaris-ai-platform`. This split is one specific, deliberate consequence of the GitOps principle "separate source from configuration" — not a general claim that every project needs two repositories.

## Why a separate repository

Argo CD's control plane runs inside the cluster and pulls from Git — it never receives push credentials or a CI token (`pull not push`, one of GitOps's core principles). Splitting deployment configuration out of the application repository means that pull can be scoped to *only* configuration: nothing in this repository can trigger a build, and nothing in `polaris-ai-platform`'s CI has (or needs) write access to a running cluster.

It also means the two things that actually change at different rates and for different reasons — application code, and "which image tag is live in which environment" — are versioned separately. A commit here is always a deployment decision; a commit in `polaris-ai-platform` is always a code change.

## Structure

```
.
├── bootstrap/
│   └── root-app.yaml       # the one manifest a human applies by hand (make gitops-bootstrap in
│                            # polaris-ai-platform); an "app of apps" whose own source is apps/
├── apps/
│   ├── dev-app.yaml         # ai-platform-dev:     automated sync (docs/architecture.md's "dev: automatic")
│   ├── staging-app.yaml     # ai-platform-staging: automated sync (gate = the PR into this repo)
│   └── prod-app.yaml        # ai-platform-prod:    NO automated sync (gate = an explicit manual sync)
├── environments/
│   ├── dev/image.yaml       # { image: { repository, tag } } — the one field these Applications
│   ├── staging/image.yaml   # add on top of polaris-ai-platform's own values-<env>.yaml, since
│   └── prod/image.yaml      # that chart deliberately ships no default image.tag (ADR-19)
└── platform/                # placeholder for future platform-wide Applications (ingress,
                              # cert-manager, observability, ...) — empty in Phase 8
```

Each `apps/<env>-app.yaml` is an Argo CD **multi-source Helm** `Application`: source 1 renders `polaris-ai-platform`'s `helm/ai-platform` chart with that environment's own `values-<env>.yaml`; source 2 is this repository itself (aliased `ref: values`), contributing only `environments/<env>/image.yaml` as an extra values file. So this repository never duplicates the chart's environment shape — replica count, resources, probes, ingress host — only the one thing that changes on every build: the image tag.

## Bootstrapping (once)

From a `polaris-ai-platform` checkout, with this repository cloned next to it (the default `GITOPS_DIR`, `../polaris-gitops`):

```bash
make argocd-install                 # installs the pinned Argo CD release (versions.env)
make gitops-bootstrap               # applies bootstrap/root-app.yaml — the only manual kubectl apply
make argocd-status                  # watch the three Applications appear and reconcile
```

From then on, every change to `apps/*.yaml` or `environments/*/image.yaml` in this repository reaches the cluster on its own — Argo CD polls this repository (default every 3 minutes) and reconciles.

## Promoting a new image

```bash
# in a polaris-ai-platform checkout, after make image-build && make image-push
make gitops-bump-dev ARGS=--push        # writes environments/dev/image.yaml, commits, pushes
```

`dev` and `staging` are `syncPolicy.automated`, so a push here is live within one poll interval. `prod` has no automated sync policy on purpose — a push updates its *desired* state (Argo CD marks it `OutOfSync`), but nothing is applied until an explicit sync (see `apps/prod-app.yaml`'s header comment for the exact `kubectl patch` command — no `argocd` CLI is installed in this project, by design; see `polaris-ai-platform`'s ADR-26).

## Verifying

No `argocd` CLI is used anywhere in this project (ADR-26). Verification is `kubectl` against the `Application` custom resource, plus `polaris-ai-platform`'s own Phase 7 verification for real application health:

```bash
kubectl --context k3d-polaris -n argocd get applications \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
# or: make argocd-status (from polaris-ai-platform)

make verify-dev    # Phase 7's rollout/health/readyz/AI-answer/logs check, unchanged by GitOps
```

## Self-heal

`dev` and `staging`'s `syncPolicy.automated.selfHeal: true` means a manual change to a live resource (`kubectl edit`, `kubectl scale`, ...) is reverted automatically the next time Argo CD reconciles — the demonstration `polaris-ai-platform/docs/architecture.md` explicitly calls for in Phase 8. See the Phase 8 guide (`polaris-ai-platform`'s Claude Project docs, or that repository's own Phase 8 write-up) for the exact drift-and-revert steps.

## What this repository is not

It is not a place for secrets (none exist yet — the mock backend has none; Sealed Secrets is the documented plan for when a real one is needed, ADR-10). It is not wired into `polaris-ai-platform`'s CI — `scripts/gitops/bump-image-tag.sh` there is a deliberately manual script, not a CI job, this phase (ADR-26 names CI integration as a named next step, not an oversight). And it does not install or configure Argo CD itself — that is `polaris-ai-platform`'s `scripts/bootstrap/argocd.sh` (`make argocd-install`), since the control plane is infrastructure the application repository's tooling already owns (`versions.env`, the same pinned-download pattern as every other tool in that project).
