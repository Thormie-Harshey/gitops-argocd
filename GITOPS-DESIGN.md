# GitOps Design

## Context
Deployments were previously made by 8 engineers running `kubectl apply` by hand. This caused configuration drift, left no audit trail, and led to a production incident from a wrong manifest. ArgoCD is now the single deployment mechanism, and the `main` branch of this repository is the single source of truth.

## 1. Preventing direct `kubectl apply` in production
We use defence in depth: three independent layers.

**Layer 1: Kubernetes RBAC (preventive control).** Engineers get read-only access to the `production` namespace through the built-in `view` ClusterRole. They can see pods and logs, but any `kubectl apply`, `scale` or `delete` is refused as Forbidden. Only ArgoCD's service account can change production.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: engineers-read-only
  namespace: production
subjects:
  - kind: Group
    name: engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

**Layer 2: Git branch protection.** Changes reach production only through a pull request to `main` with at least one approval. A CODEOWNERS file makes the platform team a required reviewer for production changes:

```
/gitops-config/overlays/production/  @example-org/platform-team
```

Every change therefore becomes a reviewed, signed-off commit, which gives a full audit trail and an easy rollback with `git revert`.

**Layer 3: ArgoCD self-heal (corrective control).** If a change bypasses the first two layers, ArgoCD reverts it to match Git. This was tested by scaling production to 5 replicas by hand; ArgoCD restored 3 replicas within about 2 seconds (Screenshot 07).

**Further hardening:** an admission controller such as Kyverno can reject any change to the `production` namespace not made by ArgoCD's service account, including changes by cluster admins.

## 2. Emergency hotfixes
**Level 1: Fast-track through Git (default).** Create a hotfix branch, open a pull request, get one approval from the on-call lead, merge, then click Sync in ArgoCD rather than waiting for its next Git check (about every 3 minutes by default). The fix stays reviewed and recorded.

**Level 2: We can decide to use thisonly when Git or ArgoCD is unavailable.**
1. The on-call engineer requests a time-limited account with write access to `production`. The request is logged in the incident ticket.
2. Disable auto-sync on the affected Application. Otherwise self-heal reverts the hotfix within seconds.
3. Apply the fix with `kubectl`. The Application shows OutOfSync, which signals that Git is behind.
4. Within 24 hours, commit the same fix to Git through a pull request, until the Application shows Synced.
5. Re-enable auto-sync and revoke the break-glass access.
6. Hold a short post-incident review to find out why Level 1 was not possible.

## 3. Self-healing
**How it works.** ArgoCD's application-controller continuously compares the live cluster with the state built from Git. With `automated.selfHeal: true`, any difference (drift) is reverted to Git's version within seconds. This is separate from automated sync, which applies new Git commits (checked about every 3 minutes by default). Tested in Screenshot 07: production scaled to 5 replicas by hand was restored to 3 within about 2 seconds.

**When to disable it:**
1. **Break-glass hotfixes**, so the manual fix is not reverted (see section 2).
2. **Live troubleshooting**, when a temporary change is needed to investigate a problem.
3. **When another controller owns a field.** For example, a HorizontalPodAutoscaler changing replicas would fight self-heal. Rather than disabling self-heal entirely, we ignore only that field, so self-heal still protects everything else:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

In every case, self-heal is disabled for one Application only, never globally, and re-enabled as soon as the reason has passed.

## 4. RBAC in ArgoCD
ArgoCD has its own RBAC, separate from Kubernetes RBAC (section 1). It controls what each person can do inside ArgoCD.

**Each team gets its own AppProject**, which limits where its apps may come from and deploy to:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/example-org/team-a-config.git
  destinations:
    - server: https://kubernetes.default.svc
      namespace: team-a-*
```

**RBAC policies grant each team rights over its own project only**, in the `argocd-rbac-cm` ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:team-a-dev, applications, get, team-a/*, allow
    p, role:team-a-dev, applications, sync, team-a/*, allow
    g, team-a-developers, role:team-a-dev
    p, role:team-b-dev, applications, get, team-b/*, allow
    p, role:team-b-dev, applications, sync, team-b/*, allow
    g, team-b-developers, role:team-b-dev
    g, platform-team, role:admin
```

**Result:**
- Team A can view and sync any app in `team-a`, but cannot sync team B's apps.
- Everyone else gets read-only access by default, so they can view all apps but change none.
- Developers cannot create or delete Applications. Only the platform team (`role:admin`) can.
- Access is granted per project, not per app, so new apps are covered without editing the policy (least privilege is enforced by the project boundary).
