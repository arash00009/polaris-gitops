# platform/

Placeholder for platform-level Argo CD Applications — an ingress controller, cert-manager,
Prometheus/Grafana, and similar cluster-wide components that later phases will add here as their
own Applications, following the same app-of-apps pattern already used by `../apps/`.

Nothing lives here yet. Phase 8 only wires up the `ai-platform` application, one per environment
(`../apps/dev-app.yaml`, `../apps/staging-app.yaml`, `../apps/prod-app.yaml`). This directory
exists now because `polaris-ai-platform/docs/architecture.md`'s Phase 8 design already names "one
root Application for platform components" as part of the target structure — recorded here so a
future phase has an obvious place to add it, not left as something to invent later.
