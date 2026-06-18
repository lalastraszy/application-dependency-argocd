# Example for application dependencies in Argo CD

This repo demonstrates two approaches to deploying applications in order using Argo CD.

---

## Approach 1: App of Apps with Sync Waves

If you use the app-of-apps pattern, all applications are deployed in parallel by default. Sync waves let you control the order on **first deploy**, but subsequent syncs (e.g. updating resources) are not ordered — child apps with `selfHeal: true` sync simultaneously.

### Setup

Patch `argocd-cm` to enable Application health monitoring (required for wave progression):

```bash
kubectl patch cm/argocd-cm --type=merge -n argocd --patch-file patch-argocd-cm.yaml
```

Add [sync wave annotations](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) to your apps. See the [apps folder](apps) for examples.

Deploy the App of Apps:

```bash
kubectl apply -f all-apps.yml
```

**Limitation:** sync wave ordering only applies when the parent app syncs its child `Application` objects. Changes to the underlying manifests are picked up by each child app's `selfHeal` independently, causing simultaneous syncs.

---

## Approach 2: ApplicationSet with Progressive Sync (RollingSync)

Progressive sync enforces step ordering on **every sync**, including resource updates. Step 2 apps will not begin syncing until all step 1 apps are `Healthy`.

### Requirements

- Argo CD >= 2.6
- Progressive syncs enabled in `argocd-cmd-params-cm`:

```bash
kubectl patch cm/argocd-cmd-params-cm --type=merge -n argocd --patch-file patch-argocd-cmd-params-cm.yaml
kubectl rollout restart deployment argocd-applicationset-controller -n argocd
```

> **Note:** the flag must go in `argocd-cmd-params-cm`, not `argocd-cm`. The ApplicationSet controller reads its configuration from `argocd-cmd-params-cm`.

### Deploy

```bash
kubectl apply -f progressive-appset.yml
```

This creates two applications:

| App                  | Namespace            | Step                      |
|----------------------|----------------------|---------------------------|
| `progressive-api`    | `progressive-api`    | 1 — syncs first           |
| `progressive-frontend` | `progressive-frontend` | 2 — waits for api Healthy |

### Visible delay

The `progressive-api` deployment includes a 60-second init container, making the ordering clearly observable:

```bash
watch kubectl get applications progressive-api progressive-frontend -n argocd
```

You will see `progressive-api` sync and become `Healthy` before `progressive-frontend` begins syncing.