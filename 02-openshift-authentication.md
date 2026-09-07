# OpenShift Authentication — L3 Corporate Administrator Guide

## 1. Authentication Architecture

OpenShift authentication can be understood as:

**User → Identity Provider (IdP) → OAuth Server → OpenShift User/Identity → Groups → RBAC → Authorization**

```text
                    Corporate Identity
                    ┌───────────────┐
                    │ LDAP / AD     │
                    │ Entra ID      │
                    │ GitHub        │
                    │ HTPasswd      │
                    └───────┬───────┘
                            │
                            │ Authentication
                            ▼
                    ┌─────────────────┐
                    │ OpenShift OAuth │
                    │     Server      │
                    └────────┬────────┘
                             │
                             │ Identity
                             ▼
                  ┌──────────────────────┐
                  │ OpenShift API Server │
                  └──────────┬───────────┘
                             │
                             │ RBAC
                             ▼
                  ┌──────────────────────┐
                  │ User / Groups / Roles │
                  └──────────────────────┘
```

> **Authentication = Who are you?**  
> **Authorization = What are you allowed to do?**

---

# 2. Main OpenShift Authentication Types

| Identity Provider | Typical Usage | Corporate Relevance |
|---|---|---|
| HTPasswd | Local users | Lab / POC / small environments |
| LDAP | Enterprise directory | Very common |
| Active Directory | Microsoft enterprise identity | Very common |
| OpenID Connect (OIDC) | SSO / cloud identity | Increasingly common |
| GitHub | Developer authentication | Development |
| GitLab | Developer authentication | CI/CD environments |
| Google | Google identity | Limited enterprise use |
| Keystone | OpenStack environments | OpenStack integration |
| Request Header | External authentication proxy | Specialized |
| Basic Auth | Legacy/special cases | Avoid for new designs |

For an OpenShift administrator, the most important are:

**HTPasswd → LDAP/AD → OIDC**

---

# 3. HTPasswd Authentication

HTPasswd is the simplest authentication method. OpenShift stores an htpasswd file in a Secret.

```text
OpenShift
   │
   ▼
OAuth Server
   │
   ▼
HTPasswd file
   │
   ├── admin
   ├── developer1
   └── developer2
```

## Create users

```bash
htpasswd -c -B users.htpasswd admin
htpasswd -B users.htpasswd developer1
```

## Create the Secret

```bash
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config
```

## OAuth configuration

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```

Apply:

```bash
oc apply -f oauth.yaml
```

Check:

```bash
oc get oauth cluster -o yaml
```

## L3 considerations

HTPasswd is easy but not ideal for large production environments.

### Good for

- Lab
- Training
- POC
- Disconnected environment
- Temporary administration

### Poor for

- Large production enterprise
- Thousands of users
- Centralized identity management
- SSO requirements
- Compliance-heavy environments

The main limitation is that user lifecycle and password management remain inside OpenShift.

---

# 4. LDAP Authentication

LDAP is one of the most important enterprise authentication methods.

```text
                    Corporate Network
                         │
                         ▼
                ┌────────────────┐
                │ LDAP / AD      │
                │ Directory      │
                └───────┬────────┘
                        │
                    LDAP/LDAPS
                        │
                        ▼
                ┌────────────────┐
                │ OpenShift OAuth│
                └───────┬────────┘
                        │
                        ▼
                    OpenShift
```

LDAP normally contains:

- Users
- Groups
- Attributes
- Passwords
- Organizational Units

Example:

```text
OU=Users
   ├── raj
   ├── amit
   └── john

OU=Groups
   ├── openshift-admins
   ├── openshift-developers
   └── openshift-readonly
```

---

# 5. LDAP Authentication Flow

Suppose:

```text
User: raj
Password: ********
```

The user opens the OpenShift console.

```text
Raj
 │
 │ username/password
 ▼
OpenShift OAuth
 │
 │ LDAP bind/search
 ▼
LDAP Server
 │
 │ Authentication successful
 ▼
OAuth Server
 │
 │ creates OpenShift identity
 ▼
OpenShift
 │
 │ RBAC lookup
 ▼
Roles / Groups
```

LDAP answers:

> Is the username/password valid?

OpenShift answers:

> What is this user allowed to do?

These are separate responsibilities.

---

# 6. LDAP Configuration Example

A simplified LDAP configuration:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: corporate-ldap
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        DN:
        - dn
        UID:
        - uid
        Email:
        - mail
        Name:
        - cn
        PreferredUsername:
        - uid

      ca:
        name: ldap-ca

      insecure: false

      url: ldaps://ldap.example.com:636/ou=users,dc=example,dc=com?uid
```

## Production principle

Prefer **LDAPS** or otherwise protected LDAP communication.

```text
LDAP  → TCP 389
LDAPS → TCP 636
```

For production, verify:

- TLS certificate
- CA chain
- Hostname validation
- LDAP bind
- LDAP search
- Firewall rules
- DNS

---

# 7. Active Directory Authentication

Active Directory is extremely common in corporate environments.

OpenShift commonly integrates with AD through the LDAP identity provider.

```text
Corporate AD
     │
     ├── Users
     ├── Groups
     └── Service Accounts
          │
          │ LDAPS
          ▼
     OpenShift OAuth
```

Example AD groups:

```text
OpenShift-Admins
OpenShift-Developers
OpenShift-Viewers
```

These groups can be mapped to OpenShift RBAC.

```text
AD Group
    │
    ▼
OpenShift Group
    │
    ▼
ClusterRoleBinding / RoleBinding
    │
    ▼
OpenShift Permissions
```

---

# 8. Corporate RBAC Example

Suppose AD contains:

```text
OpenShift-Admins
OpenShift-Developers
OpenShift-Viewers
```

Possible mapping:

```text
OpenShift-Admins
        │
        ▼
cluster-admin

OpenShift-Developers
        │
        ▼
edit

OpenShift-Viewers
        │
        ▼
view
```

Commands:

```bash
oc adm policy add-cluster-role-to-group \
  cluster-admin OpenShift-Admins
```

```bash
oc adm policy add-cluster-role-to-group \
  edit OpenShift-Developers
```

```bash
oc adm policy add-cluster-role-to-group \
  view OpenShift-Viewers
```

## Corporate best practice

Prefer:

```text
Corporate Group
      ↓
OpenShift Group
      ↓
RBAC Role
```

instead of:

```text
Individual User
      ↓
RBAC Role
```

This makes employee onboarding, offboarding and access reviews easier.

---

# 9. OpenID Connect (OIDC)

OIDC is increasingly important for modern enterprise environments.

Common enterprise IdPs include:

- Microsoft Entra ID
- Keycloak
- Red Hat SSO
- Okta
- Ping Identity
- Other OIDC-compliant providers

Architecture:

```text
                  Identity Provider
                 ┌──────────────────┐
                 │ Entra ID /       │
                 │ Keycloak / Okta  │
                 └────────┬─────────┘
                          │
                         OIDC
                          │
                          ▼
                  ┌───────────────┐
                  │ OpenShift     │
                  │ OAuth         │
                  └───────┬───────┘
                          │
                          ▼
                    OpenShift API
```

OIDC uses OAuth 2.0 concepts and ID tokens to communicate identity information.

---

# 10. OIDC Authentication Example

Suppose the organization uses Microsoft Entra ID.

```text
User
 │
 ▼
OpenShift Console
 │
 │ redirect
 ▼
Entra ID
 │
 │ authentication
 │ MFA
 │ Conditional Access
 │
 ▼
OIDC authorization
 │
 ▼
OpenShift OAuth
 │
 ▼
OpenShift
```

Benefits:

- SSO
- MFA
- Centralized authentication
- Conditional access
- Account lifecycle management
- Centralized auditing

---

# 11. OIDC Configuration Concept

A simplified configuration:

```yaml
spec:
  identityProviders:
  - name: entra-id
    mappingMethod: claim
    type: OpenID
    openID:
      clientID: openshift
      clientSecret:
        name: oidc-client-secret
      claims:
        preferredUsername:
        - preferred_username
        name:
        - name
        email:
        - email
      issuer: https://identity.example.com
```

Production configuration must correctly define:

- Issuer
- Client ID
- Client Secret
- Redirect URI
- TLS trust
- Claims
- Groups
- Token configuration

---

# 12. Authentication vs Authorization

This is a critical L3 concept.

Suppose:

```text
Braj logs into OpenShift
```

### Authentication

Question:

> Who is Braj?

Answer:

```text
braj@example.com
```

### Authorization

Question:

> What is Braj allowed to do?

Example:

```text
Project: production

get pods          → YES
create pods       → YES
delete namespace  → NO
cluster-admin     → NO
```

Therefore:

```text
Authentication
       ↓
Identity
       ↓
Groups
       ↓
RBAC
       ↓
Authorization
```

---

# 13. OpenShift Identity

After authentication, OpenShift creates an identity relationship.

Check identities:

```bash
oc get identities
```

Example:

```text
NAME
ldap:john
ldap:raj
htpasswd:admin
```

Check users:

```bash
oc get users
```

Example:

```text
NAME
john
raj
admin
```

Important:

```text
Identity ≠ User
```

An **Identity** represents the authentication relationship with an IdP.

A **User** represents the OpenShift user.

---

# 14. mappingMethod

A key L3 configuration is:

```yaml
mappingMethod: claim
```

OpenShift needs to determine which authenticated identity maps to which OpenShift user.

Common mapping methods include:

```text
claim
lookup
add
generate
```

Conceptually:

```text
External Identity
       ↓
mappingMethod
       ↓
OpenShift User
```

---

# 15. Multiple Identity Providers

A production cluster can have multiple IdPs.

Example:

```text
OpenShift OAuth
      │
      ├── Corporate LDAP
      │
      ├── Entra ID / OIDC
      │
      └── HTPasswd
```

Example:

```yaml
identityProviders:

- name: corporate-ldap
  type: LDAP

- name: corporate-oidc
  type: OpenID

- name: break-glass
  type: HTPasswd
```

This can provide flexibility, but every IdP should have a clearly defined purpose and security model.

---

# 16. Break-Glass Authentication

A controlled emergency authentication mechanism can be maintained for situations where the primary corporate IdP is unavailable.

Normal:

```text
Corporate SSO
     │
     ▼
OpenShift
```

Emergency:

```text
Corporate SSO
      X
      │
      ▼
Break-glass authentication
      │
      ▼
OpenShift
```

A break-glass account should have:

- Strong credential protection
- Limited ownership
- Documented approval process
- Auditing
- Periodic validation
- Controlled access
- Credential rotation

Do not casually create permanent `cluster-admin` accounts.

---

# 17. Request Header Authentication

This is a specialized architecture where an external proxy authenticates the user.

```text
User
 │
 ▼
Corporate Proxy
 │
 │ authentication
 ▼
OpenShift
```

The proxy may pass trusted headers such as:

```text
X-Remote-User
X-Remote-Group
```

Security is critical.

If an attacker can bypass the proxy and directly submit trusted headers, they could potentially impersonate another user.

Therefore:

```text
Internet
   │
   X
   │
Proxy
   │
   ▼
OpenShift
```

Direct bypass access must be prevented.

---

# 18. GitHub Authentication

GitHub can act as an identity provider.

```text
Developer
    │
    ▼
GitHub
    │
    ▼
OpenShift OAuth
    │
    ▼
OpenShift
```

Useful for:

- Development clusters
- Open-source projects
- Developer platforms
- POCs

For large corporate environments, an enterprise IdP such as AD/Entra ID/OIDC is usually more appropriate.

---

# 19. GitLab Authentication

Similar architecture:

```text
Developer
    │
    ▼
GitLab
    │
    ▼
OpenShift OAuth
    │
    ▼
OpenShift
```

Useful when GitLab is already part of the organization's developer identity ecosystem.

---

# 20. Choosing an Authentication Provider

| Environment | Recommended approach |
|---|---|
| Personal lab | HTPasswd |
| Training cluster | HTPasswd |
| Small internal cluster | HTPasswd / LDAP |
| AD-based enterprise | LDAP/AD |
| Modern SSO enterprise | OIDC |
| Entra ID organization | OIDC |
| Keycloak organization | OIDC |
| Developer/open-source environment | GitHub |
| GitLab-centric environment | GitLab |
| Special proxy architecture | Request Header |

---

# 21. L3 Authentication Troubleshooting

When a user cannot log in:

```text
User cannot authenticate
```

Do not immediately modify the OAuth configuration.

Follow the authentication chain.

## Step 1 — Check OAuth configuration

```bash
oc get oauth cluster -o yaml
```

Check:

- identityProviders
- mappingMethod
- type
- issuer
- LDAP URL
- claims
- CA
- Secret references

---

## Step 2 — Check OAuth pods

```bash
oc get pods -n openshift-authentication
```

Check logs:

```bash
oc logs -n openshift-authentication \
  deployment/oauth-openshift
```

Or inspect the relevant OAuth pod:

```bash
oc get pods -n openshift-authentication
oc logs -n openshift-authentication <oauth-pod>
```

---

## Step 3 — Check Authentication ClusterOperator

```bash
oc get co authentication
```

Example healthy state:

```text
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED
authentication   4.18.x    True        False         False
```

Also:

```bash
oc describe co authentication
```

---

# 22. LDAP Troubleshooting

LDAP authentication depends on several layers:

```text
DNS
 ↓
Network
 ↓
TCP 636
 ↓
TLS
 ↓
LDAP bind
 ↓
LDAP search
 ↓
User attribute
 ↓
OpenShift identity mapping
```

Useful checks from an approved troubleshooting host:

### DNS

```bash
nslookup ldap.example.com
```

### Network

```bash
nc -vz ldap.example.com 636
```

### TLS

```bash
openssl s_client \
  -connect ldap.example.com:636
```

Verify:

- Certificate
- CA chain
- Hostname
- TLS handshake
- Network connectivity

---

# 23. OIDC Troubleshooting

For OIDC, check:

```text
OpenShift
   │
   ├── issuer
   ├── client ID
   ├── client secret
   ├── redirect URI
   ├── TLS trust
   └── claims
```

A common scenario:

```text
User successfully authenticates at IdP
             ↓
OpenShift rejects login
```

This can indicate:

- Incorrect issuer
- Incorrect client configuration
- Redirect URI mismatch
- Incorrect claims
- TLS trust issue
- User/group mapping issue

---

# 24. Authentication ClusterOperator

OpenShift has an Authentication ClusterOperator.

Check:

```bash
oc get co authentication
```

Detailed:

```bash
oc describe co authentication
```

A healthy operator should generally report:

```text
AVAILABLE   PROGRESSING   DEGRADED
True        False         False
```

This should be one of the first checks when cluster authentication behaves unexpectedly.

---

# 25. OAuth Configuration

The cluster OAuth configuration is available through:

```bash
oc get oauth cluster
```

Detailed:

```bash
oc get oauth cluster -o yaml
```

Example:

```yaml
spec:
  identityProviders:
  - name: corporate-ldap
    type: LDAP
```

The OAuth server is managed by OpenShift control-plane components rather than being deployed manually as a normal application.

---

# 26. Authentication and OpenShift Console

Typical console authentication flow:

```text
Browser
   │
   ▼
OpenShift Console
   │
   ▼
OAuth
   │
   ▼
Identity Provider
   │
   ▼
Authenticated session
   │
   ▼
Console
   │
   ▼
Kubernetes API
```

CLI:

```bash
oc login https://api.cluster.example.com:6443
```

Check authenticated user:

```bash
oc whoami
```

Example:

```text
raj
```

You can display the current token with:

```bash
oc whoami --show-token
```

Treat the token as a credential. Do not expose it in tickets, chat, shell history or scripts.

---

# 27. Enterprise Group-Based Access Model

A mature OpenShift environment normally avoids managing permissions user-by-user.

### Less desirable

```text
User → Role
```

### Preferred

```text
Corporate Group
       ↓
OpenShift Group
       ↓
RBAC Role
       ↓
Project / Cluster
```

Example:

```text
AD
│
├── OCP-PROD-ADMIN
│       ↓
│   cluster-admin
│
├── OCP-PROD-DEV
│       ↓
│   edit
│
└── OCP-PROD-VIEW
        ↓
       view
```

Employee leaves organization:

```text
Remove user from corporate group
          ↓
OpenShift access disappears
```

This is significantly easier to manage and audit.

---

# 28. Corporate Reference Architecture

A modern enterprise design can look like:

```text
                  ┌─────────────────────┐
                  │ Microsoft Entra ID   │
                  │ / Corporate IdP      │
                  └──────────┬──────────┘
                             │
                            OIDC
                             │
                             ▼
                  ┌─────────────────────┐
                  │ OpenShift OAuth     │
                  └──────────┬──────────┘
                             │
                       Authenticated
                          Identity
                             │
                             ▼
                  ┌─────────────────────┐
                  │ OpenShift User      │
                  │ + Groups            │
                  └──────────┬──────────┘
                             │
                            RBAC
                             │
                             ▼
               ┌──────────────────────────┐
               │ Projects / Namespaces    │
               │                          │
               │ dev → edit               │
               │ prod → restricted        │
               │ ops → admin              │
               └──────────────────────────┘
```

For a traditional Microsoft enterprise, AD/LDAP integration remains highly relevant. For organizations standardized on modern SSO, OIDC with an enterprise IdP is often the cleaner architecture.

---

# 29. L3 Interview Answer

If asked:

> **Explain OpenShift authentication.**

A strong answer:

> OpenShift uses an OAuth server to authenticate users through configured identity providers. The identity provider can be HTPasswd, LDAP/Active Directory, OpenID Connect, GitHub, GitLab and others. Once the user is authenticated, OpenShift maps the external identity to an OpenShift User and potentially Groups. Authorization is handled separately through Kubernetes RBAC. In an enterprise environment, I would normally integrate OpenShift with the corporate identity platform, preferably using LDAPS for directory integration or OIDC for centralized SSO, and map corporate groups to OpenShift RBAC rather than granting permissions directly to individual users.

For troubleshooting:

> If authentication fails, I check the Authentication ClusterOperator, OAuth configuration, OAuth pod logs, IdP connectivity, DNS, firewall, TLS certificates, LDAP bind/search configuration, or OIDC issuer/claims and redirect configuration.

---

# 30. Quick L3 Memory Model

```text
              AUTHENTICATION
                     │
                     ▼
          Identity Provider
      ┌────────┬────────┬────────┐
      │ LDAP   │ OIDC   │ HTPass │
      └────────┴────────┴────────┘
                     │
                     ▼
               OAuth Server
                     │
                     ▼
              OpenShift User
                     │
                  Groups
                     │
                     ▼
              Kubernetes RBAC
                     │
                     ▼
              AUTHORIZATION
```

## One-line interview memory trick

> **IdP proves identity → OAuth establishes identity → User/Group represents identity → RBAC decides permissions.**
