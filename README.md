# argocd-tls

Gets a real TLS certificate onto the Argo CD server running as the managed
`Microsoft.ArgoCD` AKS cluster extension, replacing its self-signed default.

## Why this exists

Argo CD's server loads its serving certificate from a Kubernetes `Secret`
named `argocd-server-tls` in its own namespace (`argocd`), read once at
process startup. There's no config value on the AKS extension (or the
upstream `argo-cd` Helm chart it wraps) to supply this — the secret is always
expected to be created out of band.

The `argocd-server` pod itself is owned by the managed extension's Helm
release, so nothing here mounts a `SecretProviderClass` onto it directly.
Instead, this repo deploys a tiny helper `Deployment` in the `argocd`
namespace whose only job is to mount a `SecretProviderClass`. That's enough
to make the Secrets Store CSI driver pull `proj1devargocdcert` out of Azure
Key Vault (`proj1devkv01`) and sync it into the `argocd-server-tls` secret,
which Argo CD then serves.

(We looked at using [External Secrets Operator](https://external-secrets.io)
instead, since it wouldn't need a trigger pod at all - a controller pulls the
secret directly via workload identity. Rolled back: ESO isn't an AKS-managed
add-on like the Secrets Store CSI driver is, so it'd be a whole extra
self-maintained component just for this one cert. Worth revisiting if more
"sync a KV secret into a plain K8s Secret" needs show up later.)

## Files

| File | Purpose |
|---|---|
| `serviceaccount.yaml` | `argocdtlssa` ServiceAccount, federated to the `argocdwi` Azure managed identity via workload identity |
| `secretproviderclass.yaml` | `argocdtlsspc` SecretProviderClass - pulls the `proj1devargocdcert` object from Key Vault and syncs it into the `argocd-server-tls` secret |
| `deployment.yaml` | `argocdtlsdeployment` - a near-empty pod that mounts the `SecretProviderClass`, which is what actually triggers the CSI driver sync |

Deployed by Argo CD itself (`argocd_application.argocd_tls_sync` in
[project1](https://github.com/hswg94/project1)'s `modules/argocd/main.tf`),
synced automatically from the repo root.

## After the cert is (re)synced: restart argocd-server

Argo CD reads `argocd-server-tls` once at boot, not on change. After the
first sync, or after `proj1devargocdcert` renews, roll the deployment:

```bash
kubectl -n argocd rollout restart deployment/argocd-server
```

This isn't automated yet. See project1's notes on
[Reloader](https://github.com/stakater/Reloader) if that stops being
acceptable (annotate `argocd-server` to auto-restart on secret change) —
untested here, and worth confirming the extension's periodic Helm
reconciliation doesn't strip the annotation back out.

## Cert renewal

`proj1devargocdcert` is issued via the project's ADCS pipeline
(`certmngr-code/certmngr-vmapp/makecert.py`), not by anything in this repo.
Re-run it on `crtmgrvm` to renew.
