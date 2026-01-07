# Module 19: OpenShift GitOps (ArgoCD)
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to the final phase! We start with GitOps.

One question: Where is your cluster's configuration stored? If you answered 'in my head' or 'in random YAML files'... we need to talk about GitOps!"

---

### Section 1: What is GitOps? (6 mins)

[SLIDE - GitOps concept]

"GitOps = Git is the source of truth

Everything in your cluster is defined in Git. Every change goes through Git. The cluster PULLS from Git and stays in sync."

[SLIDE - GitOps workflow]

"1. Developer pushes change to Git
2. ArgoCD sees the change
3. ArgoCD applies to cluster
4. Cluster matches Git

No more `kubectl apply` from laptops!"

---

### Section 2: Install OpenShift GitOps (8 mins)

[TERMINAL]

[DEMO]

```bash
# Install from OperatorHub
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for pods
oc get pods -n openshift-gitops -w
```

"Get the ArgoCD URL and password..."

```bash
# URL
oc get route openshift-gitops-server -n openshift-gitops

# Admin password
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

[BROWSER]

"Login to ArgoCD with admin and that password."

---

### Section 3: Create Application (12 mins)

[SLIDE - ArgoCD Application]

"An Application in ArgoCD connects:
- A Git repo (source)
- A namespace (destination)"

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/your/repo.git
    path: manifests
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

"Key points:
- **source**: Where to get manifests
- **destination**: Where to deploy
- **syncPolicy.automated**: Auto-apply changes
- **selfHeal**: Revert manual changes"

[DEMO - browser]

"Watch in ArgoCD UI: Create the app, see it sync, all resources appear!"

---

### Section 4: Self-Healing (4 mins)

[SLIDE - Self-healing]

"The magic: Try changing something manually..."

```bash
oc scale deployment demo --replicas=10 -n gitops-demo
```

"Watch ArgoCD UI... it will revert to whatever's in Git!"

"That's self-healing. Git is truth. Manual changes get undone."

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 19 complete!

1. **Git is source of truth** - All config in version control

2. **ArgoCD syncs continuously** - Cluster matches Git

3. **Self-healing** - Manual changes reverted

4. **Audit trail** - Git history = change log

5. **ApplicationSets** - Multi-environment deployments"

[SLIDE - Next Module]

"Final module: OpenShift Virtualization - VMs on Kubernetes!"

---

## 📝 Presenter Notes

- GitOps is increasingly required in enterprises
- Self-healing demo is the 'wow' moment
- If no Git repo ready, use ArgoCD's example repos
