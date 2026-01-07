# Module 4: The Operator Lifecycle Manager (OLM)
## Lecture Transcript (~40 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Phase 2! We're moving from infrastructure to the platform layer.

Today's topic: Operators. And I promise you, by the end of this, you'll understand why Red Hat says 'Everything is an Operator.'"

[CHECK]

"Quick poll - how many of you have installed software on Kubernetes using Helm? ...And how many have had to manage day-2 operations like upgrades and backups? That's exactly what Operators solve."

---

### Section 1: What Problem Do Operators Solve? (8 mins)

[SLIDE - The Operations Problem]

"Let me tell you a story. You're the DBA. You need to run PostgreSQL on Kubernetes.

With regular Kubernetes, you:
1. Write Deployment YAML
2. Write Service YAML
3. Write PVC YAML
4. Deploy it... success!

But then..."

[SLIDE - Day 2 nightmare]

"Week 2: 'We need to add a replica'
Week 3: 'We need to upgrade PostgreSQL'
Week 4: 'We need to restore from backup'
Week 5: 'PostgreSQL is down at 3 AM and you need to fix it'

Kubernetes doesn't know ANYTHING about PostgreSQL. It just knows 'run these containers.'"

[PAUSE]

"This is where Operators come in."

[SLIDE - Operator pattern]

"An Operator is a piece of software that KNOWS how to run another piece of software.

A PostgreSQL Operator knows:
- How to set up replication
- How to do rolling upgrades  
- How to take and restore backups
- How to handle failures

It encodes human operator knowledge into code."

[CHECK]

"Makes sense? An Operator automates what a human operator would do."

---

### Section 2: OLM - The Operator Manager (7 mins)

[SLIDE - OLM diagram]

"So Operators are great. But who manages the Operators themselves? Enter OLM - Operator Lifecycle Manager."

[SLIDE - OLM components]

"Think of it like a package manager. apt, yum, brew... OLM does the same for Operators.

Key resources:

**CatalogSource** - Where to find Operators (like a repository)

**Subscription** - 'I want this Operator installed'

**InstallPlan** - 'Here's what will be installed'

**ClusterServiceVersion (CSV)** - The Operator version itself"

[SLIDE - Flow diagram]

"The flow is:
1. You create a Subscription
2. OLM checks the CatalogSource
3. OLM creates an InstallPlan
4. You approve it (or auto-approve)
5. OLM installs the CSV
6. Operator is running!"

---

### Section 3: LiveDemo - Installing an Operator (12 mins)

[SLIDE - Demo time]

"Enough theory. Let's install an Operator!"

[BROWSER - OpenShift Console]

[DEMO]

"First, let me show you the OperatorHub..."

[Navigate to Operators → OperatorHub]

"Look at all these! Red Hat Operators, Certified Operators, Community Operators... hundreds of them."

[Search for 'Web Terminal']

"We'll install the Web Terminal Operator. It's simple and useful - lets you run a terminal right in the browser."

[Click Install]

"Notice the options:
- **Update channel**: stable, fast, etc.
- **Install mode**: All namespaces or specific one
- **Update approval**: Automatic or Manual

For production? Manual. You want to control when Operators update."

[Click Install, then wait]

[TERMINAL - after install completes]

```bash
# Check if CSV installed
oc get csv -n openshift-operators

# Should show: web-terminal.vX.X.X   Succeeded
```

"See that 'Succeeded'? That means the Operator is running!"

```bash
# Check the Operator pod
oc get pods -n openshift-operators | grep web-terminal
```

"There's our Operator pod, happily running."

[BROWSER]

"Now look at the top right of the console... see that terminal icon? Click it!"

[Demo the web terminal]

"Boom! A terminal in your browser. The Operator created this capability."

---

### Section 4: Understanding CSV States (5 mins)

[SLIDE - CSV states]

"Let's talk about troubleshooting. CSVs can be in different states..."

[Table on slide]

"**Pending** - Waiting for something (dependencies, permissions)
**InstallReady** - Ready to install
**Installing** - In progress
**Succeeded** - Working!
**Failed** - Something broke"

[TERMINAL]

[DEMO]

```bash
# Check all CSVs
oc get csv --all-namespaces

# If one is not Succeeded, describe it
oc describe csv <name> -n <namespace>
```

"When troubleshooting, look at:
1. The Conditions section
2. The Events at the bottom
3. The operator pod logs"

```bash
# Check operator pod logs
oc logs <operator-pod> -n openshift-operators
```

---

### Section 5: Update Approval (5 mins)

[SLIDE - Manual vs Automatic]

"One more critical concept: update approval."

[SLIDE - Comparison]

"**Automatic Approval:**
- New version available? Install immediately
- Good for: Dev/Test environments
- Risk: Untested update in production

**Manual Approval:**
- New version available? Wait for human
- Good for: Production
- Control: You decide when to update"

[TERMINAL]

[DEMO]

```bash
# Check subscription settings
oc get subscription web-terminal -n openshift-operators -o yaml | grep installPlanApproval
```

[SLIDE - Approving updates]

"If manual, you need to approve InstallPlans:"

```bash
# Find pending install plans
oc get installplans -n openshift-operators

# Approve one
oc patch installplan <name> -n openshift-operators \
  --type merge --patch '{"spec":{"approved":true}}'
```

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 4 complete! Key points:

1. **Operators encode human knowledge** - Day 2 operations automated

2. **OLM manages Operators** - Like a package manager

3. **Subscription → InstallPlan → CSV** - The installation flow

4. **CSV states tell health** - Succeeded = good, anything else = investigate

5. **Manual approval for prod** - Control when Operators update"

[SLIDE - Next Module]

"Next: Cluster Updates. How to upgrade OpenShift itself without breaking everything!"

---

## 📝 Presenter Notes

- Have CRC running with console accessible
- Pre-search for "Web Terminal" in OperatorHub (it's quick to install)
- If Web Terminal is already installed, use a different operator like Grafana
- Common confusion: "What's the difference between Operator and Helm?" - Operators are lifecycle managers, Helm is just packaging
