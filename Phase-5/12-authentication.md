# Module 12: Authentication (Identity Providers)
## Lecture Transcript (~30 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Phase 5 - Security! We start with the most fundamental question:

Who are you? And how do we prove it?

That's authentication. Let's configure it!"

---

### Section 1: OpenShift Auth Flow (5 mins)

[SLIDE - Auth flow]

"When you run `oc login`, here's what happens:

1. oc contacts the OAuth server
2. OAuth says 'go authenticate with an Identity Provider'
3. You prove who you are (password, SSO, etc.)
4. OAuth gives you a token
5. You use that token for all API calls"

[SLIDE - Identity Providers]

"OpenShift supports many IDPs:
- **HTPasswd** - Simple file (dev/test)
- **LDAP** - Active Directory (enterprise)
- **OIDC** - Okta, Keycloak, Azure AD (modern)
- **GitHub** - For open source projects"

---

### Section 2: LiveDemo - HTPasswd IDP (12 mins)

[TERMINAL]

[DEMO]

```bash
# Create htpasswd file
htpasswd -c -B -b users.htpasswd admin admin123
htpasswd -B -b users.htpasswd developer dev123
htpasswd -B -b users.htpasswd viewer view123

# Create secret
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config
```

"Now configure OAuth to use it..."

```yaml
# Save as oauth.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: Local Users
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpasswd-secret
```

```bash
oc apply -f oauth.yaml

# Wait for rollout
oc get pods -n openshift-authentication -w
```

"Test the new user..."

```bash
oc login -u admin -p admin123
oc whoami
```

"We're in! Now grant cluster-admin..."

```bash
oc adm policy add-cluster-role-to-user cluster-admin admin
```

---

### Section 3: Enterprise IDPs (Theory) (8 mins)

[SLIDE - LDAP]

"For Active Directory, you'd configure LDAP:"

```yaml
spec:
  identityProviders:
    - name: ActiveDirectory
      type: LDAP
      ldap:
        url: "ldap://ad.company.com/ou=users,dc=company,dc=com?sAMAccountName"
        bindDN: "cn=service,dc=company,dc=com"
        bindPassword:
          name: ldap-secret
```

[SLIDE - OIDC]

"For Okta or Azure AD, use OpenID Connect:"

```yaml
spec:
  identityProviders:
    - name: Okta
      type: OpenID
      openID:
        clientID: your-client-id
        clientSecret:
          name: okta-secret
        issuer: https://company.okta.com
```

"In production, you'd use LDAP or OIDC. HTPasswd is just for learning!"

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 12 complete!

1. **OAuth handles all auth** - One configuration point

2. **HTPasswd for dev only** - Not scalable

3. **LDAP/OIDC for enterprise** - Centralized identity

4. **Remove kubeadmin** - After setting up admin user

5. **Groups sync from LDAP** - For team management"

[SLIDE - Next Module]

"Next: RBAC - What can authenticated users actually DO?"

---

## 📝 Presenter Notes

- HTPasswd changes require restarting auth pods - wait 1-2 minutes
- Common error: typo in htpasswd password vs login attempt
- Always grant cluster-admin before removing kubeadmin!
