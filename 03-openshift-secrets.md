# OpenShift Secrets — L3 Corporate-Level Guide

## 1. What is a Secret?

A **Secret** is a Kubernetes/OpenShift API object used to store sensitive configuration data separately from application manifests.

Common examples:

- Database usernames/passwords
- API tokens
- SSH private keys
- TLS certificates and private keys
- Private container-registry credentials
- Application credentials

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USERNAME: appuser
  DB_PASSWORD: "MySecretPassword123"
```

A Deployment can consume it:

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_PASSWORD
```

### Corporate architecture

```text
                    OpenShift API Server
                           |
                    +------v------+
                    |   Secret    |
                    | db-secret    |
                    +------+------+
                           |
                 -----------------------
                 |          |          |
                 v          v          v
              Pod-A      Pod-B      Job/CronJob
                 |
                 v
        DB_PASSWORD environment
```

---

## 2. Important: Secret ≠ Encryption

One of the most important L3 concepts:

> **Base64 encoding is not encryption.**

For example:

```yaml
data:
  password: TXlQYXNzd29yZA==
```

Decode it:

```bash
echo 'TXlQYXNzd29yZA==' | base64 -d
```

Output:

```text
MyPassword
```

Therefore:

- Base64 = encoding
- Encryption at rest = security control
- RBAC = access-control mechanism
- Audit = visibility/control over access

A Secret should not be considered secure merely because its value appears Base64 encoded.

---

# 3. Secret Object Structure

Typical Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
  namespace: production
type: Opaque

data:
  username: YWRtaW4=
  password: TXlQYXNzd29yZA==
```

Important fields:

| Field | Purpose |
|---|---|
| `apiVersion` | API version |
| `kind` | Secret |
| `metadata.name` | Secret name |
| `metadata.namespace` | Namespace/project |
| `type` | Secret type |
| `data` | Base64-encoded values |
| `stringData` | Plain-text input processed into Secret data |

---

# 4. `data` vs `stringData`

## Using `data`

Values must already be Base64 encoded:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=
  password: TXlQYXNzd29yZA==
```

Decode:

```bash
echo YWRtaW4= | base64 -d
```

Output:

```text
admin
```

## Using `stringData`

You can provide the original value:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: MyPassword123
```

Apply:

```bash
oc apply -f secret.yaml
```

### L3 recommendation

`stringData` is easier for manually written manifests, but **do not commit plaintext credentials to Git**.

---

# 5. Secret Types

Common Secret types:

```text
Secret
│
├── Opaque
├── kubernetes.io/basic-auth
├── kubernetes.io/ssh-auth
├── kubernetes.io/tls
├── kubernetes.io/dockerconfigjson
├── kubernetes.io/service-account-token
└── Custom Secret types
```

---

# 6. Opaque Secret

The most common Secret type:

```yaml
type: Opaque
```

Used for arbitrary application data.

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:
  DB_USER: appuser
  DB_PASSWORD: "P@ssw0rd123"
  DB_NAME: applicationdb
```

Apply:

```bash
oc apply -f db-secret.yaml
```

Check:

```bash
oc get secret db-secret
```

Example output:

```text
NAME         TYPE     DATA   AGE
db-secret    Opaque   3      10s
```

`DATA = 3` means the Secret contains three data entries.

---

# 7. Basic Authentication Secret

Type:

```yaml
type: kubernetes.io/basic-auth
```

Typically represents:

```text
username + password
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: "P@ssw0rd123"
```

Typical use cases:

- HTTP basic authentication
- Internal application credentials
- Repository credentials
- External service authentication

---

# 8. SSH Authentication Secret

Type:

```yaml
type: kubernetes.io/ssh-auth
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-secret
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

Typical use:

```text
Pod
 |
 +-- SSH private key
 |
 +-- Git/SSH server
```

### Corporate security warning

Treat SSH private keys as highly sensitive credentials.

Never:

- Commit private keys to Git
- Put private keys in a Dockerfile
- Expose private keys in logs
- Put credentials directly into Deployment manifests
- Share private keys through insecure channels

---

# 9. TLS Secret

Type:

```yaml
type: kubernetes.io/tls
```

Used for:

```text
TLS certificate
+
Private key
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-tls
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

  tls.key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
```

Expected keys:

```text
tls.crt
tls.key
```

Check:

```bash
oc get secret application-tls
```

---

# 10. TLS Secret and OpenShift Route

A common OpenShift architecture:

```text
Internet
   |
   | HTTPS
   v
OpenShift Router
   |
   | HTTP/HTTPS
   v
Service
   |
   v
Pod
```

A Route can use TLS:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp
spec:
  host: myapp.example.com
  to:
    kind: Service
    name: myapp
  tls:
    termination: edge
    certificate: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN PRIVATE KEY-----
      ...
      -----END PRIVATE KEY-----
```

In enterprise environments, certificates are commonly managed through standardized certificate-management processes rather than manually embedding certificates.

---

# 11. Docker Registry Secret

Very important for OpenShift administrators.

Type:

```yaml
type: kubernetes.io/dockerconfigjson
```

Used when Pods need to pull images from private registries.

Examples:

- Quay
- Harbor
- Enterprise registries
- Cloud container registries
- Private internal registries

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "registry.example.com": {
          "username": "myuser",
          "password": "mypassword",
          "auth": "..."
        }
      }
    }
```

Use it in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: app
    image: registry.example.com/myproject/myapp:1.0
  imagePullSecrets:
  - name: registry-secret
```

---

# 12. OpenShift Image Pull Secret

Instead of adding the Secret to every Pod, associate it with a ServiceAccount.

```bash
oc secrets link default registry-secret --for=pull
```

Check:

```bash
oc describe sa default
```

Typical architecture:

```text
Private Registry
       |
       | credentials
       v
Registry Secret
       |
       v
ServiceAccount
       |
       v
Deployment
       |
       v
Pod
       |
       v
Image Pull
```

This is generally cleaner for enterprise workloads.

---

# 13. ServiceAccount Token Secret

Historically you may encounter:

```yaml
type: kubernetes.io/service-account-token
```

These Secrets contain ServiceAccount authentication information.

Modern Kubernetes/OpenShift versions generally prefer **short-lived projected ServiceAccount tokens** for normal Pod authentication rather than relying on long-lived token Secrets.

Therefore:

> Do not assume every ServiceAccount has a permanent token Secret.

Check:

```bash
oc get sa
```

```bash
oc describe sa myapp
```

---

# 14. Using Secrets as Environment Variables

Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USERNAME: appuser
  DB_PASSWORD: "P@ssword123"
```

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-client
spec:
  replicas: 2
  selector:
    matchLabels:
      app: database-client

  template:
    metadata:
      labels:
        app: database-client

    spec:
      containers:
      - name: application
        image: example.com/app:1.0

        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USERNAME

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```

Inside the container:

```bash
echo $DB_USERNAME
```

Output:

```text
appuser
```

---

# 15. `envFrom`

Import all Secret keys as environment variables:

```yaml
containers:
- name: application
  image: example.com/app:1.0
  envFrom:
  - secretRef:
      name: db-secret
```

If the Secret contains:

```yaml
stringData:
  DB_USERNAME: appuser
  DB_PASSWORD: password
  DB_NAME: mydb
```

the container receives:

```text
DB_USERNAME
DB_PASSWORD
DB_NAME
```

### L3 consideration

`envFrom` is convenient, but explicit `secretKeyRef` can be preferable in enterprise workloads because it clearly documents which Secret keys the application actually needs.

---

# 16. Mounting a Secret as a Volume

Secrets can also be mounted as files.

Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  username: admin
  password: password123
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-demo
spec:
  containers:
  - name: app
    image: registry.access.redhat.com/ubi9/ubi

    volumeMounts:
    - name: secret-volume
      mountPath: /etc/app-secret
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

Inside the container:

```bash
ls -l /etc/app-secret
```

Expected files:

```text
username
password
```

Then:

```bash
cat /etc/app-secret/username
```

Output:

```text
admin
```

---

# 17. Environment Variable vs Volume

| Method | Example | Common use |
|---|---|---|
| Environment variable | `DB_PASSWORD` | Application credentials |
| Secret volume | `/etc/secrets/password` | Certificates, keys, files |
| `envFrom` | Import all keys | Multiple application settings |
| `imagePullSecrets` | Registry credentials | Private image pulls |

### Practical rule

Use environment variables when the application expects credentials as environment variables.

Use volume-mounted Secrets when the application expects files, especially:

- TLS certificates
- Private keys
- SSH keys
- Credential files
- Secret configuration files

---

# 18. Secret Updates and Pod Behavior

Suppose a Secret contains:

```text
password = oldpassword
```

The application consumes it.

Then the Secret changes:

```text
password = newpassword
```

## Environment variable

If consumed through:

```yaml
env:
- valueFrom:
    secretKeyRef:
```

the existing container environment does not automatically change.

Normally restart/redeploy the workload:

```bash
oc rollout restart deployment/myapp
```

## Secret volume

Secret volumes can be updated by Kubernetes/OpenShift mechanisms after the Secret changes.

However:

> The file may update, but the application must reread/reload it to use the new value.

Some applications only read credentials during startup.

---

# 19. Secret Rotation

Secrets should have a lifecycle.

```text
Create
  |
  v
Deploy
  |
  v
Use
  |
  v
Rotate
  |
  v
Validate
  |
  v
Revoke old credential
```

Example database credential rotation:

```text
1. Change/generate new DB credential
2. Update/create Secret
3. Reload/restart application
4. Verify all replicas
5. Monitor application
6. Revoke old credential
7. Audit the change
```

Do not blindly overwrite a production credential without considering:

- Existing connections
- Multiple replicas
- Application reload behavior
- Rollback
- External dependencies
- Credential validity

---

# 20. RBAC and Secrets

Anyone with permission to read Secrets can potentially obtain sensitive credentials.

Check your own permission:

```bash
oc auth can-i get secrets
```

Specific user:

```bash
oc auth can-i get secrets --as=user1
```

ServiceAccount:

```bash
oc auth can-i get secrets \
  --as=system:serviceaccount:production:myapp
```

### Corporate principle

> **Do not grant Secret access unless the workload or administrator actually needs it.**

Use least privilege.

---

# 21. Inspecting Secrets

List:

```bash
oc get secrets
```

All namespaces:

```bash
oc get secrets -A
```

Describe:

```bash
oc describe secret db-secret
```

YAML:

```bash
oc get secret db-secret -o yaml
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USERNAME: YXBwdXNlcg==
  DB_PASSWORD: cGFzc3dvcmQxMjM=
```

Decode one key:

```bash
oc get secret db-secret \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### Security warning

Be careful when decoding Secrets because the result may appear in:

- Shared terminals
- Screen recordings
- CI/CD logs
- Shell history
- Incident tickets
- Chat messages

---

# 22. Creating Secrets with `oc`

## From literals

```bash
oc create secret generic db-secret \
  --from-literal=username=appuser \
  --from-literal=password='P@ssword123'
```

## From a file

```bash
oc create secret generic app-secret \
  --from-file=config=/path/to/config
```

## TLS

```bash
oc create secret tls app-tls \
  --cert=tls.crt \
  --key=tls.key
```

## Docker registry

```bash
oc create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword'
```

---

# 23. Secret vs ConfigMap

| Feature | Secret | ConfigMap |
|---|---|---|
| Sensitive data | Yes | No |
| Password | Yes | No |
| API token | Yes | No |
| TLS private key | Yes | No |
| Non-sensitive application config | Sometimes | Yes |
| Base64 representation | Yes | ConfigMap data is generally plain text |
| RBAC | Yes | Yes |
| Encryption at rest | Can be configured | Can also be configured, but not intended as secrecy mechanism |

Simple rule:

```text
Non-sensitive configuration
        |
        v
    ConfigMap

Sensitive configuration
        |
        v
      Secret
```

Example:

```text
ConfigMap:
LOG_LEVEL=INFO
APP_PORT=8080

Secret:
DB_PASSWORD=xxxx
API_TOKEN=xxxx
TLS_PRIVATE_KEY=xxxx
```

---

# 24. Namespace Scope

Secrets are **namespace-scoped** resources.

Example:

```text
project-a
  |
  +-- db-secret

project-b
  |
  +-- db-secret
```

These are two separate Secret objects.

A Pod in `project-a` cannot simply reference `project-b/db-secret`.

The Secret normally needs to exist in the same namespace/project as the consuming Pod.

This provides an important namespace-level security boundary.

Check the current project:

```bash
oc project
```

Find a Secret across namespaces:

```bash
oc get secret -A | grep db-secret
```

---

# 25. Secret Ownership

A Deployment using a Secret does not automatically mean that the Deployment owns the Secret.

Explicit ownership can be established through:

```yaml
metadata:
  ownerReferences:
```

Use ownership carefully because Kubernetes garbage collection can remove dependent objects when owners are deleted.

In enterprise environments, Secret lifecycle ownership should be deliberate.

---

# 26. Immutable Secrets

Kubernetes supports immutable Secrets.

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: application-secret
type: Opaque
immutable: true
stringData:
  username: appuser
  password: password123
```

Advantages:

- Prevents accidental modification
- Makes credential lifecycle explicit
- Can reduce update/watch activity

If rotation is required, create/use a new Secret.

Example:

```text
db-secret-v1
      |
      v
db-secret-v2
```

Then update the workload to use the new Secret.

---

# 27. GitOps and Secrets

### Avoid this

```yaml
stringData:
  password: "ProductionPassword123"
```

and committing it directly to Git.

Even if the file is deleted later, the credential may remain in Git history.

Enterprise approaches commonly include:

```text
External Secret Manager
        |
        v
Secret synchronization
        |
        v
OpenShift Secret
        |
        v
Application
```

Examples of technologies/patterns:

- HashiCorp Vault
- Cloud secret managers
- External Secrets Operator
- Sealed Secrets
- Enterprise GitOps secret-management workflows

The exact architecture depends on organizational security standards.

---

# 28. Encryption at Rest

An L3 administrator must distinguish:

```text
Base64 encoding
        ≠
Encryption
```

Conceptually:

```text
Application
     |
     v
OpenShift API
     |
     v
Secret
     |
     v
Encryption at Rest
     |
     v
Cluster datastore
```

Encryption at rest protects stored data.

But:

> Encryption at rest does not replace RBAC.

A user authorized to retrieve a Secret through the API can still potentially receive its value.

---

# 29. Corporate Example — Production Application

Example architecture:

```text
OpenShift Cluster
        |
        +-- project: ecommerce-prod
                  |
                  +-- Deployment: order-api
                  |
                  +-- Secret: db-credentials
                  |
                  +-- Secret: payment-api-token
                  |
                  +-- Secret: order-api-tls
                  |
                  +-- ServiceAccount: order-api-sa
```

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-api

  template:
    metadata:
      labels:
        app: order-api

    spec:
      serviceAccountName: order-api-sa

      containers:
      - name: order-api
        image: registry.example.com/ecommerce/order-api:2.5

        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password

        - name: PAYMENT_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: payment-api-token
              key: token

        volumeMounts:
        - name: tls
          mountPath: /etc/order-api/tls
          readOnly: true

      volumes:
      - name: tls
        secret:
          secretName: order-api-tls
```

Architecture:

```text
                   OpenShift
                       |
                ecommerce-prod
                       |
                 +-----+------+
                 |            |
             Deployment    Secrets
                 |            |
             order-api        +-- db-credentials
                 |            |
                 |             +-- payment-api-token
                 |            |
                 |             +-- order-api-tls
                 |
                 v
              3 Pods
             /  |  \
            /   |   \
           v    v    v
         Pod1 Pod2 Pod3
           |
      +----+----------------+
      |                     |
      v                     v
 Environment             Volume
 variables               /etc/order-api/tls
      |                     |
 DB credentials       TLS certificate/key
```

This is closer to the design/operational discussion expected from an L3 OpenShift administrator.

---

# 30. L3 Troubleshooting Methodology

Suppose an application reports:

```text
Database authentication failed
```

Follow a structured process.

## Step 1 — Check Secret

```bash
oc get secret db-secret
```

## Step 2 — Check Secret keys

```bash
oc get secret db-secret -o yaml
```

Verify expected keys:

```text
DB_USERNAME
DB_PASSWORD
```

## Step 3 — Check Deployment reference

```bash
oc get deployment myapp -o yaml
```

Look for:

```yaml
secretKeyRef:
```

Verify:

```text
Secret name
Key name
Namespace/project
```

## Step 4 — Check Pods

```bash
oc get pod
```

## Step 5 — Check Pod events

```bash
oc describe pod <pod-name>
```

## Step 6 — Check application logs

```bash
oc logs <pod-name>
```

## Step 7 — Decode only when authorized and necessary

```bash
oc get secret db-secret \
-o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Do not expose the credential unnecessarily.

---

# 31. Common Secret Reference Failure

Secret:

```yaml
stringData:
  DB_PASSWORD: password123
```

Deployment:

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_PASS
```

Problem:

```text
Secret key       = DB_PASSWORD
Deployment asks  = DB_PASS
```

Correct:

```yaml
key: DB_PASSWORD
```

This is a common operational issue.

---

# 32. Secret Missing from Namespace

Deployment:

```yaml
secretKeyRef:
  name: db-secret
```

But:

```bash
oc get secret db-secret
```

returns:

```text
Error from server (NotFound)
```

Possible causes:

```text
Secret doesn't exist
Wrong namespace/project
Typo in Secret name
Deployment is in another project
GitOps synchronization failed
```

Check:

```bash
oc project
```

and:

```bash
oc get secret -A | grep db-secret
```

---

# 33. Secret + ServiceAccount

Common corporate pattern:

```text
                 ServiceAccount
                       |
             +---------+---------+
             |                   |
             v                   v
       Image Pull Secret    Application Secret
             |                   |
             +---------+---------+
                       |
                       v
                      Pod
```

Example ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: application-sa
```

Link registry Secret:

```bash
oc secrets link application-sa registry-secret --for=pull
```

Application Secret is referenced independently:

```yaml
env:
- name: API_TOKEN
  valueFrom:
    secretKeyRef:
      name: application-secret
      key: API_TOKEN
```

---

# 34. Corporate Secret Security Model

A production Secret-management model should consider:

```text
              Secret Security
                    |
      +-------------+-------------+
      |             |             |
     RBAC        Encryption      Audit
      |             |             |
      +-------------+-------------+
                    |
             Secret lifecycle
                    |
          +---------+---------+
          |                   |
       Rotation          External Vault
```

Also consider:

- Least privilege
- Namespace isolation
- ServiceAccount permissions
- API audit logs
- Secret rotation
- External secret managers
- Git repository scanning
- CI/CD log masking
- Developer access restrictions

---

# 35. Important `oc` Commands — Interview Cheat Sheet

### List Secrets

```bash
oc get secrets
```

### All namespaces

```bash
oc get secrets -A
```

### Describe

```bash
oc describe secret db-secret
```

### YAML

```bash
oc get secret db-secret -o yaml
```

### JSON

```bash
oc get secret db-secret -o json
```

### Decode a key

```bash
oc get secret db-secret \
-o jsonpath='{.data.password}' | base64 -d
```

### Create generic Secret

```bash
oc create secret generic db-secret \
--from-literal=username=appuser \
--from-literal=password='password123'
```

### Create TLS Secret

```bash
oc create secret tls app-tls \
--cert=tls.crt \
--key=tls.key
```

### Create registry Secret

```bash
oc create secret docker-registry registry-secret \
--docker-server=registry.example.com \
--docker-username=user \
--docker-password='password'
```

### Link registry Secret to ServiceAccount

```bash
oc secrets link default registry-secret --for=pull
```

### Check authorization

```bash
oc auth can-i get secrets
```

### Check ServiceAccount authorization

```bash
oc auth can-i get secrets \
--as=system:serviceaccount:production:myapp
```

---

# 36. L3 Interview Questions

## Q1. Is Secret data encrypted?

**Answer:** Not merely because it is a Secret. Secret values are Base64 encoded in the API representation. Base64 is not encryption. Encryption at rest is an additional security control, and RBAC is still required.

## Q2. Difference between `data` and `stringData`?

```text
data
  -> Base64-encoded input

stringData
  -> Plain-text input
  -> API server processes it into Secret data
```

## Q3. What happens when a Secret used as an environment variable changes?

Existing container environment variables do not automatically change. The workload generally needs a restart/rollout to receive the new value.

## Q4. Can a Secret be shared across namespaces?

Secrets are namespace-scoped. A normal Secret reference does not directly reference a Secret in another namespace. Create/synchronize the Secret in each consuming namespace.

## Q5. Secret vs ConfigMap?

```text
ConfigMap -> non-sensitive configuration
Secret    -> sensitive data
```

## Q6. How do you provide private registry credentials?

Create a:

```text
kubernetes.io/dockerconfigjson
```

Secret and use it as an `imagePullSecret`, or associate it with the appropriate ServiceAccount.

## Q7. How would you rotate a production database password?

A strong L3 answer:

```text
1. Generate/change credential in the DB
2. Update/create the Secret
3. Coordinate application reload/restart
4. Verify all replicas
5. Monitor application errors
6. Revoke old credential
7. Verify no workload still depends on old credential
8. Audit the rotation
```

---

# 37. Most Important L3 Mental Model

Remember Secrets through five layers:

```text
                 SECRET
                    |
      +-------------+-------------+
      |             |             |
     TYPE          DATA         ACCESS
      |             |             |
 Opaque/TLS/   password/key    RBAC
 Registry/SSH      token
      |             |
      +------+------+ 
             |
          CONSUMPTION
             |
       +-----+------+
       |            |
       v            v
   Environment    Volume
    Variable       Mount
       |            |
       +-----+------+
             |
             v
          WORKLOAD
             |
             v
       Application
```

For corporate L3 administration, add:

```text
                    SECRET
                       |
       +---------------+---------------+
       |               |               |
      RBAC        Encryption       Rotation
       |               |               |
       +---------------+---------------+
                       |
                 External Vault
                       |
                       v
                  GitOps / CI-CD
```

## Key takeaway

> **A Secret is an API object for delivering sensitive data to workloads — not a complete security solution by itself.**

An L3 OpenShift administrator should understand the complete lifecycle:

```text
Creation
   ↓
Access Control
   ↓
Consumption
   ↓
Monitoring/Auditing
   ↓
Rotation
   ↓
Revocation
   ↓
Secure Decommissioning
```
