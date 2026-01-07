# Module 13: RBAC & Multi-Tenancy
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Authentication asks 'Who are you?' RBAC asks 'What can you do?'

Today we master Role-Based Access Control and set up multi-tenant environments!"

---

### Section 1: RBAC Building Blocks (7 mins)

[SLIDE - RBAC model]

"RBAC has four key resources:

**Role** - Permissions in ONE namespace
**ClusterRole** - Permissions cluster-wide
**RoleBinding** - Connects users to Roles
**ClusterRoleBinding** - Connects users to ClusterRoles"

[SLIDE - Example]

"Think of it like a keycard system:
- Role = 'Access to 3rd floor only'
- ClusterRole = 'Access to all floors'
- Binding = 'Give this keycard to Bob'"

---

### Section 2: Built-in Roles (5 mins)

[SLIDE - Common roles]

"OpenShift provides these out of the box:

**cluster-admin** - God mode. Everything.
**admin** - Full namespace access (except quotas)
**edit** - Create/modify resources (no RBAC)
**view** - Read-only access
**self-provisioner** - Can create projects"

[TERMINAL]

```bash
# See what a role allows
oc describe clusterrole admin
oc describe clusterrole view
```

---

### Section 3: LiveDemo - Grant Permissions (12 mins)

[TERMINAL]

[DEMO]

```bash
# Create a project
oc new-project rbac-demo

# Grant admin to a user in this project
oc adm policy add-role-to-user admin developer -n rbac-demo

# Grant view to another
oc adm policy add-role-to-user view viewer -n rbac-demo
```

"Test the permissions..."

```bash
# As developer - can create
oc login -u developer -p dev123
oc new-app nginx -n rbac-demo
# Works!

# As viewer - read only
oc login -u viewer -p view123
oc get pods -n rbac-demo  
# Works!

oc delete pod nginx-xxx -n rbac-demo
# Error: forbidden!
```

"Perfect! RBAC is working."

---

### Section 4: Multi-Tenancy (8 mins)

[SLIDE - Multi-tenant pattern]

"Multi-tenancy = Multiple teams sharing one cluster.

Pattern:
1. Each team gets their own namespace(s)
2. Each team gets admin on THEIR namespaces only
3. Use Groups for easier management"

```bash
# Create team namespaces
oc new-project team-frontend
oc new-project team-backend

# Create groups
oc adm groups new frontend-team
oc adm groups add-users frontend-team alice bob

# Grant admin to GROUP
oc adm policy add-role-to-group admin frontend-team -n team-frontend
```

"Now Alice and Bob are admins of team-frontend, but can't touch team-backend!"

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 13 complete!

1. **Roles define permissions** - Verbs on resources

2. **Bindings connect users** - To roles

3. **Use groups** - Easier than individual users

4. **Least privilege** - Grant minimum needed

5. **Namespace isolation** - Team A can't see Team B"

[SLIDE - Next Module]

"Next: Pod Security - SCCs and why Helm charts fail on OpenShift!"

---

## 📝 Presenter Notes

- `oc auth can-i --list` shows current user's permissions
- Groups are key for enterprise - always prefer groups over users
- Common mistake: granting cluster-admin when admin would suffice
