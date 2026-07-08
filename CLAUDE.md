# CLAUDE.md

This repo is Talos/Kubernetes GitOps config deployed via ArgoCD. Read `CONTEXT.md`
before making changes — it maps every workload, where Pangolin routing is defined,
how secrets are encrypted, and known gotchas (e.g. duplicate Termix deployments).

## Keep CONTEXT.md current

`CONTEXT.md` is a living document, not a one-time snapshot. Whenever a change in
this repo would make it inaccurate, update it in the same commit:

- Adding, removing, or renaming a workload → update the workloads table and the
  directory map.
- Adding, removing, or changing a public domain/route → update the Pangolin/Newt
  section and the workloads table (the source of truth is the `blueprintData`
  block in `sisyphus/apps/workloads/newt.yaml` — reflect it, don't duplicate
  logic from memory).
- Bumping a Helm chart or image version referenced in `CONTEXT.md` → update the
  version mentioned there, or remove the specific version if it's likely to drift
  (prefer describing *where* to check current versions over hardcoding them).
- Changing secrets/SOPS conventions, NFS server/share layout, or the bootstrap
  process → update the relevant section.
- Resolving one of the "known gotchas" (e.g. deduplicating Termix) → remove the
  note once it's no longer true.

Don't assume `CONTEXT.md` is accurate before relying on it for a task — spot-check
the specific files it references first, the same way you would with any memory.
If you find it's drifted from reality, fix it as part of your change.

## Verify, don't assume

Before making claims about how this repo works (routing, secrets, storage), grep
the actual files rather than inferring from convention or a similar-looking repo.
This repo has real deviations from what you'd expect (e.g. Pangolin config is
inline Helm values, not a separate CRD or UI-only config).
