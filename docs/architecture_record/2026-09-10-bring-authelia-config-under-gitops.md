# Bring Authelia's server config under GitOps, and loosen session lifetimes

Found and fixed 2026-09-10, prompted by wanting longer "stay signed in"
sessions — not a live incident. Two things were tangled together:

## The gap: `configuration.yml` was never actually deployed by git

`k8s/apps/authelia/configuration.yml` looked like the source of truth for
Authelia's config, but nothing deployed it. The chart
(`bjw-s-labs/app-template`) only *mounts* a ConfigMap named
`authelia-config` (`values.yaml` → `persistence.config`); it does not
create one, and no manifest under `k8s/` created it either. The live
ConfigMap was `kubectl create configmap authelia-config
--from-file=configuration.yml`'d by hand once during rollout step 4
(`docs/architecture_record/2026-08-13-migrate-the-sso-critical-path.md`)
and never touched again — the later OIDC work went into a separate
hand-applied Secret (`identity_providers.yml`), not this file. Editing the
tracked file and pushing would have done nothing. Same class of drift as
the CrowdSec one found on 2026-08-20
(`docs/architecture_record/2026-08-20-bring-crowdsec-under-gitops.md`).

**Fix:** replaced the bare `configuration.yml` with a real
`k8s/apps/authelia/configmap.yaml` (`kind: ConfigMap`, name
`authelia-config`, namespace `authelia`, config under `data`). The
recursive root app (`k8s/argocd/root-app.yaml`) picks it up as a plain
manifest, the same way it manages the `k8s/apps/ingress/*` Traefik
objects — Argo CD adopts the existing in-cluster ConfigMap on first sync.
The chart's mount is unchanged (`subPath: configuration.yml` from the
same ConfigMap name), so `values.yaml` needed no edit.

Secrets stay out of git as before: session/storage/JWT keys and the LDAP
bind password are in the `authelia-secrets` Secret (`envFrom`), and the
OIDC `identity_providers.yml` is its own hand-applied Secret.

One-time cost: a ConfigMap `subPath` mount does not update live and
Authelia does not hot-reload, so the running pod had to be restarted once
(`kubectl rollout restart deployment/authelia -n authelia`, or the Argo
UI "Restart" action) to pick up the new file. Future edits to the config
still need that restart, but the content now flows through git.

## The change: session lifetimes

Old `session.cookies[0]`: `expiration: 1h`, `inactivity: 5m`, no
`remember_me` — a bounce to the login page after five minutes away from
the keyboard, and a hard re-auth every hour regardless.

New:

| Key | Old | New | Why |
|---|---|---|---|
| `expiration` | `1h` | `8h` | Covers a normal day's session without a mid-task re-auth. |
| `inactivity` | `5m` | `2h` | The one that actually hurt — survives lunch, a meeting, a closed laptop lid. |
| `remember_me` | (unset) | `1M` | Enables the "Remember me" checkbox; opt-in 30-day session that ignores the inactivity timer. |

Kept deliberately moderate rather than maximal because every app is
internet-facing (`Internet → Traefik → CrowdSec → Authelia → app`), so
the un-remembered session is still a normal-feeling web session and the
long tail is behind an explicit per-login checkbox. `regulation`
(brute-force lockout) is unchanged.

## Verification

The check that matters: after the root app syncs, the `authelia` app
stays `Synced`/`Healthy` with no unexpected diff on the adopted
ConfigMap, and after the pod restart a fresh login shows the "Remember
me" checkbox and a session that outlives the old 5-minute idle window.
