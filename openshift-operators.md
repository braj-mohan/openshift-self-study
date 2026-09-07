# OpenShift Operators — L3 Corporate Administrator Guide

## 1. What is an Operator?

An **Operator is a Kubernetes-native controller that manages the complete lifecycle of an application or platform component**.

A useful mental model is:

> **Operator = Controller + Domain knowledge + Custom Resource (CR) + Reconciliation loop**

An Operator can automate:

- Installation
- Configuration
- Upgrades
- Scaling
- Certificate management
- Backup/restore
- Failover
- Health monitoring
- Configuration reconciliation
- Version compatibility
- Day-2 operations
- Recovery from failures

For example, instead of manually managing a complex database with Deployments, Services, PVCs, Secrets, configuration, backups, and recovery procedures, an Operator can expose a higher-level resource such as:

```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: production-db
spec:
  replicas: 3
  storage: 500Gi
  version: "16"
```

The Operator watches this object and creates/manages the required Kubernetes resources.

---

# 2. Why were Operators created?

Kubernetes already has controllers, but generic controllers do not necessarily understand application-specific operational knowledge.

For example, a generic Kubernetes controller knows how to maintain Pods:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

It does not inherently understand all database operations such as:

- Primary/replica relationships
- Database failover
- Replication recovery
- Application-specific upgrades
- Backup consistency
- Database-specific health checks

An Operator packages this domain knowledge into a Kubernetes-native controller.

```text
Kubernetes
   |
   | Generic orchestration
   ↓
Pods / Services / Deployments

Operator
   |
   | Application-specific operational knowledge
   ↓
Database / Kafka / Storage / Monitoring / etc.
```

---

# 3. Operator Architecture

A typical Operator architecture looks like:

```text
                 OpenShift API Server
                         |
                         |
                  Custom Resource
                         |
                         ↓
              ┌─────────────────────┐
              │      Operator       │
              │                     │
              │ Controller          │
              │ Reconciliation Loop │
              │ Application Logic   │
              └──────────┬──────────┘
                         |
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
      Deployment       Service         PVC
          ↓              ↓              ↓
        Pods          Networking      Storage
```

The most important component is the **reconciliation loop**.

---

# 4. Desired State vs Actual State

Suppose a Custom Resource specifies:

```yaml
spec:
  replicas: 3
```

Desired state:

```text
3 replicas
```

If actual state is also:

```text
3 replicas
```

the Operator has nothing to correct.

If an administrator manually deletes one Pod:

```text
Desired state = 3
Actual state  = 2
```

The Operator detects the difference and takes corrective action.

```text
Desired State
      |
      ↓
Operator
      |
      ↓
Compare desired vs actual
      |
      ├── Desired = Actual
      |       ↓
      |     Do nothing
      |
      └── Desired ≠ Actual
              ↓
       Corrective action
```

This process is called **reconciliation**.

### L3 interview statement

> An Operator continuously reconciles the desired state represented by Kubernetes API objects with the observed cluster state and takes corrective action whenever drift occurs.

---

# 5. Operator vs Controller

Operators and controllers are closely related.

| Controller | Operator |
|---|---|
| Watches resources | Watches resources |
| Uses reconciliation | Uses reconciliation |
| Kubernetes-native | Kubernetes-native |
| Usually manages generic Kubernetes resources | Usually manages an application/platform |
| May not contain deep application knowledge | Contains domain-specific operational knowledge |
| Example: Deployment controller | Example: PostgreSQL Operator |

Important:

> **Every Operator is fundamentally controller-based, but not every controller is an Operator.**

---

# 6. CRD — CustomResourceDefinition

Operators commonly use **CustomResourceDefinitions (CRDs)** to extend the Kubernetes API.

A CRD defines a new resource type.

Examples:

```text
PostgreSQL
Kafka
Database
Elasticsearch
```

After an Operator installs a CRD, Kubernetes understands the new resource type.

Check CRDs:

```bash
oc get crd
```

Example:

```text
postgresqls.database.example.com
```

You can then create a Custom Resource:

```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: mydb
spec:
  replicas: 3
```

Apply it:

```bash
oc apply -f postgres.yaml
```

---

# 7. CRD vs CR

This is a common interview question.

### CRD

Defines the **type** of resource.

```text
CRD = Definition of a new API resource
```

### CR

Creates an **instance** of that resource.

```text
CR = Actual object created from the CRD
```

Analogy:

```text
Class  → Object
CRD    → CR
```

Example:

```text
CRD:
PostgreSQL

CR:
production-db
```

---

# 8. Operator Lifecycle

A mature Operator can manage:

```text
Install
   ↓
Configure
   ↓
Deploy
   ↓
Observe
   ↓
Reconcile
   ↓
Upgrade
   ↓
Recover
   ↓
Scale
   ↓
Backup/Restore
```

This is why Operators are particularly valuable for **Day-2 operations**.

---

# 9. Kubernetes Operators vs OpenShift Operators

Operators are fundamentally a **Kubernetes concept**.

OpenShift builds on Kubernetes and provides an integrated Operator management ecosystem.

```text
Kubernetes
 └── Operators
      ├── CRDs
      ├── Controllers
      └── Custom Resources

OpenShift
 └── Kubernetes
      +
      ├── OLM
      ├── OperatorHub
      ├── CatalogSources
      ├── Subscriptions
      ├── InstallPlans
      ├── CSVs
      └── OpenShift-specific Operators
```

The important distinction:

> **Operators are Kubernetes-native; OpenShift adds an integrated Operator lifecycle/catalog ecosystem and uses Operators extensively to manage the OpenShift platform itself.**

---

# 10. Operator Lifecycle Manager — OLM

**Operator Lifecycle Manager (OLM)** manages Operators installed through the OLM ecosystem.

It helps with:

- Installation
- Upgrade
- Dependency management
- Operator version management
- Operator metadata
- Permissions
- Lifecycle management

A simplified lifecycle:

```text
OperatorHub
     |
     ↓
CatalogSource
     |
     ↓
Package / Bundle
     |
     ↓
Subscription
     |
     ↓
InstallPlan
     |
     ↓
CSV
     |
     ↓
Operator Deployment
     |
     ↓
CRDs
     |
     ↓
Custom Resources
```

---

# 11. OperatorHub

OpenShift Console provides:

```text
Administration
    ↓
Operators
    ↓
OperatorHub
```

OperatorHub provides a catalog of Operators available for installation.

Depending on OpenShift version/catalog configuration, you can inspect available packages with:

```bash
oc get packagemanifests
```

A useful search:

```bash
oc get packagemanifests | grep -i prometheus
```

---

# 12. Subscription

A **Subscription** tells OLM that you want a particular Operator package and channel.

Conceptual example:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: example-operator
spec:
  channel: stable
  name: example-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Check:

```bash
oc get subscription -A
```

Think:

> **Subscription = I want this Operator/channel managed by OLM.**

---

# 13. InstallPlan

An InstallPlan represents the resources/actions OLM intends to perform to install or upgrade an Operator.

Check:

```bash
oc get installplan -A
```

An InstallPlan can involve:

```text
CRDs
RBAC
ServiceAccounts
Deployments
CSV
Dependencies
```

If an Operator installation is stuck, always inspect:

```bash
oc get subscription -A
oc get installplan -A
oc get csv -A
```

---

# 14. CSV — ClusterServiceVersion

CSV is an important OLM resource.

Check:

```bash
oc get csv -A
```

Example:

```text
NAME
my-operator.v1.2.0
```

CSV contains Operator installation/lifecycle metadata, including information such as:

- Operator version
- Install strategy
- Required CRDs
- Provided CRDs
- RBAC
- Deployment
- Upgrade information
- Webhooks
- Dependencies

For L3 troubleshooting:

```bash
oc describe csv <csv-name> -n <namespace>
```

---

# 15. OperatorGroup

An **OperatorGroup** defines the namespace scope in which an OLM-managed Operator can operate.

Check:

```bash
oc get operatorgroup -A
```

Conceptually:

```text
OperatorGroup
       |
       ├── Namespace A
       ├── Namespace B
       └── Namespace C
```

A common production problem is:

> Operator is installed successfully, but it does not reconcile a CR in a particular namespace.

Check:

```bash
oc get operatorgroup -A
```

The Operator may not be watching that namespace.

---

# 16. Namespace-scoped vs Cluster-scoped Operators

Operators can be configured to watch:

### One namespace

```text
Operator
   ↓
Namespace A
```

### Multiple namespaces

```text
Operator
   ↓
Namespace A
Namespace B
Namespace C
```

### Cluster-wide

```text
Operator
   ↓
Entire cluster
```

The exact scope depends on the Operator's design and OLM configuration.

---

# 17. OpenShift Cluster Operators

OpenShift uses Operators heavily to manage its own platform.

Examples include:

- Cluster Version Operator
- Machine Config Operator
- Cluster Network Operator
- DNS Operator
- Ingress Operator
- Authentication Operator
- Cluster Storage Operator
- Cloud Credential Operator
- Image Registry Operator
- Monitoring-related Operators
- Console Operator
- Machine API Operator
- Node Tuning Operator
- Service CA Operator

These platform Operators are different from ordinary add-on Operators installed through an OLM Subscription.

---

# 18. Cluster Version Operator — CVO

The **Cluster Version Operator (CVO)** is one of the most important OpenShift Operators.

It manages the lifecycle/version of OpenShift components.

Conceptually:

```text
CVO
 |
 ├── API Server
 ├── Scheduler
 ├── Controller Manager
 ├── etcd
 ├── OAuth
 ├── Ingress
 ├── Networking
 ├── Monitoring
 ├── Machine Config
 └── Other cluster operators
```

Check:

```bash
oc get clusterversion
```

Detailed:

```bash
oc describe clusterversion
```

Check cluster Operators:

```bash
oc get co
```

---

# 19. ClusterOperator

`co` is the short name for:

```text
ClusterOperator
```

Run:

```bash
oc get co
```

This displays OpenShift platform Operator health.

Typical columns:

```text
NAME
VERSION
AVAILABLE
PROGRESSING
DEGRADED
SINCE
```

Healthy example:

```text
NAME             AVAILABLE   PROGRESSING   DEGRADED
authentication   True        False         False
console          True        False         False
dns              True        False         False
etcd             True        False         False
ingress          True        False         False
network          True        False         False
```

For detailed information:

```bash
oc describe co network
```

---

# 20. `oc api-resources` vs `oc get co`

This distinction is extremely important for L3 administrators.

## `oc api-resources`

```bash
oc api-resources
```

asks:

> **What resource types does the API server support?**

It can show resources such as:

```text
pods
services
deployments
nodes
routes
projects
machineconfigs
clusteroperators
clusterversions
...
```

It describes **resource types**, not individual objects.

Useful filters:

```bash
oc api-resources | grep -i operator
```

```bash
oc api-resources --namespaced=false
```

```bash
oc api-resources --namespaced=true
```

---

## `oc get co`

```bash
oc get co
```

asks:

> **What ClusterOperator objects exist and what is their current health?**

It shows instances such as:

```text
authentication
console
dns
etcd
ingress
network
machine-config
monitoring
...
```

### Key difference

```text
oc api-resources
        ↓
"What RESOURCE TYPES exist?"

oc get co
        ↓
"What INSTANCES of ClusterOperator exist?"
```

Analogy:

```text
API resource type:
Deployment

Instances:
nginx
httpd
myapp
```

Similarly:

```text
API resource type:
ClusterOperator

Instances:
network
dns
ingress
authentication
...
```

---

# 21. How `oc api-resources` relates to `oc get co`

Run:

```bash
oc api-resources | grep -w ClusterOperator
```

You should see something similar to:

```text
clusteroperators   co   config.openshift.io/v1   false   ClusterOperator
```

This tells you:

```text
Resource type = ClusterOperator
Short name    = co
API version   = config.openshift.io/v1
Namespaced    = false
Kind          = ClusterOperator
```

Then:

```bash
oc get co
```

returns the instances of that resource type.

Conceptually:

```text
API resource definition
        |
        ↓
ClusterOperator
        |
        ↓
Instances
        |
        ├── authentication
        ├── console
        ├── dns
        ├── etcd
        ├── ingress
        ├── network
        └── ...
```

This is analogous to:

```text
oc api-resources
        ↓
pods

oc get pods
        ↓
pod-1
pod-2
pod-3
```

---

# 22. Default OpenShift 4.18 Cluster Operators

The exact set can vary depending on installation platform, capabilities, and configuration. The following are important OpenShift 4.18 platform Operators/Cluster Operators:

| Operator | Main responsibility |
|---|---|
| Cluster Baremetal Operator | Bare-metal host provisioning |
| Cloud Credential Operator | Cloud credentials / CredentialsRequests |
| Cluster Authentication Operator | OAuth/authentication |
| Cluster Autoscaler Operator | Cluster autoscaling |
| Cloud Controller Manager Operator | Cloud-provider integration |
| Cluster CAPI Operator | Cluster API functionality |
| Cluster Config Operator | Cluster configuration |
| Cluster CSI Snapshot Controller Operator | CSI volume snapshots |
| Cluster Image Registry Operator | Internal image registry |
| Cluster Machine Approver Operator | Machine CSR approval |
| Cluster Monitoring Operator | Monitoring stack |
| Cluster Network Operator | Cluster networking |
| Cluster Samples Operator | OpenShift sample images/templates |
| Cluster Storage Operator | Storage/CSI configuration |
| Cluster Version Operator | OpenShift lifecycle/upgrades |
| Console Operator | Web console |
| Control Plane Machine Set Operator | Control-plane machine management |
| DNS Operator | Cluster DNS |
| etcd Cluster Operator | etcd management |
| Ingress Operator | Ingress/router |
| Insights Operator | Red Hat Insights integration |
| Kubernetes API Server Operator | Kubernetes API server |
| Kubernetes Controller Manager Operator | Kubernetes controller manager |
| Kubernetes Scheduler Operator | Kubernetes scheduler |
| Kubernetes Storage Version Migrator Operator | Storage-version migration |
| Machine API Operator | Machine lifecycle |
| Machine Config Operator | Node OS/configuration |
| Marketplace Operator | Operator catalog/marketplace |
| Node Tuning Operator | Node performance tuning |
| OpenShift API Server Operator | OpenShift API server |
| OpenShift Controller Manager Operator | OpenShift controllers |
| OLM Operators | Operator lifecycle management components |
| OpenShift Service CA Operator | Service CA certificates |
| vSphere Problem Detector Operator | vSphere-specific problem detection |

**Important:** Do not assume every Operator in a reference list appears on every OpenShift 4.18 cluster. Platform, installation type, optional capabilities, and configuration can affect the actual result.

Always verify your cluster with:

```bash
oc get co
```

---

# 23. Operators You Should Know for L3 Interviews

You do not need to memorize all Operators equally.

## Core control plane

```text
CVO
etcd
kube-apiserver
kube-controller-manager
kube-scheduler
openshift-apiserver
openshift-controller-manager
```

## Networking

```text
Cluster Network Operator
DNS Operator
Ingress Operator
```

## Node / Machine

```text
Machine API Operator
Machine Config Operator
Node Tuning Operator
Machine Approver Operator
Cluster Autoscaler Operator
```

## Storage

```text
Cluster Storage Operator
CSI Snapshot Controller
Kubernetes Storage Version Migrator
```

## User-facing platform

```text
Console Operator
Authentication Operator
Image Registry Operator
Samples Operator
Service CA Operator
```

## Observability

```text
Cluster Monitoring Operator
Insights Operator
```

## Operator management

```text
Marketplace Operator
OLM
OLM v1
```

---

# 24. Application/Add-on Operators

Examples of important OpenShift ecosystem Operators include:

### OpenShift GitOps

Based around Argo CD.

```text
Git
 ↓
Argo CD
 ↓
OpenShift/Kubernetes
 ↓
Applications
```

Typical GitOps resources include:

```text
Application
ApplicationSet
AppProject
```

---

### OpenShift Pipelines

Based around Tekton.

```text
Git
 ↓
Pipeline
 ↓
Tasks
 ↓
Build/Test
 ↓
Deploy
```

Typical resources include:

```text
Pipeline
PipelineRun
Task
TaskRun
```

---

### OpenShift Service Mesh

Provides capabilities such as:

- Traffic management
- Service-to-service security
- Observability
- mTLS

Conceptually:

```text
Service A
   |
 Sidecar
   |
Service Mesh
   |
 Sidecar
   |
Service B
```

---

### OpenShift Virtualization

Allows virtual machines to run on OpenShift.

Conceptually:

```text
OpenShift
   |
   ├── Containers
   |
   └── Virtual Machines
          ↓
       KubeVirt
```

---

### AMQ Streams

Red Hat's Kafka offering based on the Strimzi project.

Conceptually:

```text
Kafka CR
   ↓
AMQ Streams Operator
   ↓
Kafka infrastructure
```

Depending on the version and architecture, the Operator manages Kafka brokers, configuration, storage, users, topics, and related resources.

---

### OpenShift Data Foundation

Provides enterprise storage capabilities.

Conceptually:

```text
Application
     ↓
PVC
     ↓
CSI
     ↓
ODF
     ↓
Ceph
     ↓
Storage
```

---

### Advanced Cluster Management

Used for multi-cluster management.

```text
                 Hub Cluster
                     |
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
   Cluster 1      Cluster 2      Cluster 3
   OpenShift      OpenShift      Kubernetes
```

---

# 25. How an Operator Actually Works

Suppose you create:

```yaml
kind: Database
metadata:
  name: prod-db
spec:
  replicas: 3
  version: "16"
```

You run:

```bash
oc apply -f database.yaml
```

### Step 1 — API server stores the CR

```text
oc apply
   ↓
API Server
   ↓
Database CR
```

### Step 2 — Operator watches the API

```text
Operator
   ↓
Watch Database CR
```

### Step 3 — Operator observes the CR

```text
Database/prod-db
```

### Step 4 — Operator compares state

```text
Desired state
      vs
Observed state
```

### Step 5 — Operator creates/updates resources

```text
Deployment/StatefulSet
Service
PVC
Secret
ConfigMap
```

### Step 6 — Kubernetes controllers create Pods

```text
Operator
   ↓
StatefulSet
   ↓
Pods
```

### Step 7 — Operator observes health

```text
Pods Running?
Storage available?
Database healthy?
Replication healthy?
```

### Step 8 — Operator updates status

```yaml
status:
  phase: Ready
```

---

# 26. What Happens When an Administrator Manually Changes an Operator-managed Resource?

Suppose the desired configuration is:

```text
replicas = 3
```

An administrator manually changes the generated Deployment:

```bash
oc scale deployment database --replicas=1
```

The Operator may observe:

```text
Desired = 3
Actual  = 1
```

and restore the desired configuration.

Therefore:

> **Do not manually modify resources owned by an Operator unless the product documentation explicitly says the modification is supported.**

Prefer:

```text
Modify Custom Resource
        ↓
Operator observes change
        ↓
Reconciliation
        ↓
Generated resources updated
```

instead of:

```text
Modify generated Deployment
        ↓
Operator reconciles it
        ↓
Manual change may disappear
```

---

# 27. How to Identify Operator-owned Resources

Inspect the resource:

```bash
oc get <resource> <name> -o yaml
```

Look for:

```yaml
metadata:
  ownerReferences:
```

Example:

```yaml
ownerReferences:
- apiVersion: example.com/v1
  kind: Database
  name: prod-db
```

This can show the ownership relationship.

Also inspect labels and annotations:

```bash
oc get deployment <name> -o yaml
```

Look for Operator-specific metadata.

---

# 28. L3 Operator Troubleshooting

Suppose:

```bash
oc apply -f database.yaml
```

but the application does not become Ready.

Do not immediately restart Pods.

Use a structured approach.

## Step 1 — Inspect the CR

```bash
oc get database prod-db -o yaml
```

Check:

```yaml
status:
```

Also:

```bash
oc describe database prod-db
```

---

## Step 2 — Check Operator Pod

```bash
oc get pods -n <operator-namespace>
```

---

## Step 3 — Check CSV

```bash
oc get csv -n <operator-namespace>
```

---

## Step 4 — Check Subscription

```bash
oc get subscription -n <operator-namespace>
```

---

## Step 5 — Check InstallPlan

```bash
oc get installplan -n <operator-namespace>
```

---

## Step 6 — Check Operator logs

```bash
oc logs deployment/<operator-deployment> -n <operator-namespace>
```

or:

```bash
oc logs pod/<operator-pod> -n <operator-namespace>
```

Previous container:

```bash
oc logs pod/<operator-pod> -n <operator-namespace> --previous
```

---

## Step 7 — Check events

```bash
oc get events -n <namespace> --sort-by=.lastTimestamp
```

Events can reveal:

- Scheduling failures
- Permission problems
- Mount failures
- Image problems
- Webhook failures
- Resource constraints

---

## Step 8 — Check RBAC

```bash
oc get sa -n <operator-namespace>
```

```bash
oc get role -n <operator-namespace>
```

```bash
oc get rolebinding -n <operator-namespace>
```

For cluster-wide permissions:

```bash
oc get clusterrole
oc get clusterrolebinding
```

---

# 29. Scenario: CSV Stuck in Pending

Typical lifecycle:

```text
Subscription
      ↓
InstallPlan
      ↓
CSV
      ↓
Pending
```

Investigate:

```bash
oc get subscription
oc get installplan
oc get csv
```

Potential causes:

- Dependency problems
- Catalog problems
- RBAC issues
- CRD conflicts
- OperatorGroup problems
- Manual approval required

---

# 30. Scenario: Operator Pod CrashLoopBackOff

Check:

```bash
oc get pods -n <operator-namespace>
```

Then:

```bash
oc describe pod <pod> -n <operator-namespace>
```

Logs:

```bash
oc logs <pod> -n <operator-namespace>
```

Previous container:

```bash
oc logs <pod> -n <operator-namespace> --previous
```

Investigate:

```text
RBAC
Secret
ConfigMap
CRD
API compatibility
Resource limits
Dependencies
Webhooks
Network connectivity
```

---

# 31. Scenario: CR Exists but Nothing Happens

Example:

```bash
oc get database
```

shows:

```text
prod-db
```

but no Pods are created.

Check in this order:

```text
Is the Operator running?
       ↓
Is the CRD correct?
       ↓
Is the API version correct?
       ↓
Is the Operator watching this namespace?
       ↓
Is OperatorGroup correct?
       ↓
Are RBAC permissions correct?
       ↓
Are dependencies available?
       ↓
Are reconciliation errors present?
```

Useful commands:

```bash
oc get crd
oc get operatorgroup -A
oc get csv -A
oc get pods -A
oc get events -A --sort-by=.lastTimestamp
```

---

# 32. Scenario: Operator Continuously Changes Your Configuration

This often indicates:

```text
Desired state ≠ Manual state
```

Inspect the Custom Resource:

```bash
oc get <CR> <name> -o yaml
```

Then inspect the generated resource:

```bash
oc get <generated-resource> <name> -o yaml
```

Compare them.

The Operator may simply be performing normal reconciliation.

---

# 33. Critical L3 Distinction: CVO vs OLM

Do not confuse these mechanisms.

## CVO

The **Cluster Version Operator** manages OpenShift platform lifecycle and coordinates OpenShift component versions.

```text
CVO
 ↓
OpenShift platform Operators
 ↓
OpenShift platform
```

## OLM

**Operator Lifecycle Manager** manages Operators installed through the OLM ecosystem.

```text
Subscription
 ↓
InstallPlan
 ↓
CSV
 ↓
Operator
 ↓
CRDs / CRs
```

Simplified:

```text
OpenShift platform
       |
       ↓
      CVO
       |
       ↓
Cluster Operators

Add-on Operators
       |
       ↓
      OLM
       |
       ↓
Subscription / InstallPlan / CSV
```

---

# 34. Do Not Confuse These Resources

An L3 administrator should clearly understand:

```text
Operator
ClusterOperator
OperatorGroup
Subscription
InstallPlan
CSV
CRD
CR
```

Their roles are different.

```text
                    Operator ecosystem
                           |
                 ┌─────────┴─────────┐
                 ↓                   ↓
                OLM                Platform
                 |                   |
          Subscription              CVO
                 ↓                   |
          InstallPlan                ↓
                 ↓             ClusterOperator
                CSV
                 ↓
              Operator
                 ↓
                CRD
                 ↓
                 CR
                 ↓
       Application/platform resources
```

---

# 35. Important Command Set for OpenShift 4.18

Use these commands regularly in a lab.

## Version

```bash
oc version
```

## All API resource TYPES

```bash
oc api-resources
```

## Find Operator-related API resources

```bash
oc api-resources | grep -i operator
```

## Find ClusterOperator resource type

```bash
oc api-resources | grep -w ClusterOperator
```

## List ClusterOperators

```bash
oc get co
```

## Detailed ClusterOperator status

```bash
oc get co -o wide
```

## Inspect a ClusterOperator

```bash
oc describe co network
```

## List CRDs

```bash
oc get crd
```

## List OLM subscriptions

```bash
oc get subscription -A
```

## List installed CSVs

```bash
oc get csv -A
```

## List OperatorGroups

```bash
oc get operatorgroup -A
```

## List CatalogSources

```bash
oc get catalogsource -A
```

## List available Operator packages

```bash
oc get packagemanifests -n openshift-marketplace
```

---

# 36. Useful L3 Cluster Health Check

A simple first-level health check:

```bash
oc get co
```

Healthy example:

```text
AVAILABLE    = True
PROGRESSING  = False
DEGRADED     = False
```

If an Operator shows:

```text
AVAILABLE=False
```

investigate why the component is unavailable.

If:

```text
PROGRESSING=True
```

the component is currently changing/updating.

If:

```text
DEGRADED=True
```

there is a reported degraded condition.

Start with:

```bash
oc describe co <operator-name>
```

Then follow the status/message to the responsible component.

---

# 37. L3 Mental Model

Memorize this:

```text
                         OpenShift
                            |
                    Kubernetes control plane
                            |
                    OpenShift API Server
                            |
              ┌─────────────┴─────────────┐
              ↓                           ↓
       Resource Types                Resources
              |                           |
       oc api-resources                 oc get
              |                           |
      ClusterOperator               network
      Deployment                    ingress
      Pod                           dns
      Service                       authentication
      Route                         ...
      CRD
      CR
      ...
```

For an Operator:

```text
                 Custom Resource
                       |
                       ↓
                 Kubernetes API
                       |
                       ↓
                    Operator
                       |
                Reconciliation
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Deployment     Service       PVC
          ↓            ↓            ↓
         Pods       Networking    Storage
                       |
                       ↓
                    Status
                       |
                       └──────→ Custom Resource
```

---

# 38. The Golden Rules for L3 Administration

### Rule 1

> **Treat the Custom Resource as the desired-state interface for an Operator.**

### Rule 2

> **Do not manually modify Operator-owned generated resources unless explicitly supported.**

### Rule 3

> **Always understand who owns a resource before modifying it.**

Check:

```bash
oc get <resource> <name> -o yaml
```

and inspect:

```yaml
metadata:
  ownerReferences:
```

### Rule 4

> **For OpenShift platform health, start with `oc get co`.**

### Rule 5

> **For OLM Operator installation problems, inspect Subscription → InstallPlan → CSV → Operator Pod → CR.**

### Rule 6

> **For Operator reconciliation problems, inspect CR status, Operator logs, events, RBAC, scope, dependencies, and webhooks.**

---

# 39. Interview Rapid-Fire

### Q: What is an Operator?

> A Kubernetes-native controller that embeds application/platform operational knowledge and continuously reconciles desired and actual state.

### Q: Operator vs Controller?

> An Operator is a specialized controller pattern focused on managing an application or platform component using domain-specific knowledge.

### Q: What is a CRD?

> A CRD extends the Kubernetes API by defining a new resource type.

### Q: What is a CR?

> A Custom Resource is an instance of a CRD.

### Q: What is reconciliation?

> The continuous process of comparing desired state with observed state and taking corrective action.

### Q: What is OLM?

> Operator Lifecycle Manager manages the lifecycle of OLM-installed Operators, including installation, upgrades, dependencies, and permissions.

### Q: What is a Subscription?

> It declares the desired Operator package/channel that OLM should manage.

### Q: What is an InstallPlan?

> It represents the resources/actions OLM plans to perform to install or upgrade an Operator.

### Q: What is CSV?

> ClusterServiceVersion contains metadata and installation/lifecycle information for an Operator version.

### Q: What is OperatorGroup?

> It defines the namespace scope in which an OLM-managed Operator operates.

### Q: What is CVO?

> Cluster Version Operator manages the OpenShift platform version and coordinates the lifecycle of OpenShift's platform components.

### Q: What does `oc get co` show?

> Instances of the `ClusterOperator` resource and their health conditions.

### Q: What does `oc api-resources` show?

> Resource types supported by the API server.

### Q: Why are they different?

> `oc api-resources` shows resource TYPES; `oc get co` shows INSTANCES of the specific `ClusterOperator` type.

---

# 40. Final Memory Trick

Remember this chain:

```text
CR
 ↓
"I want this state"
 ↓
Operator
 ↓
"How to achieve it?"
 ↓
Reconciliation
 ↓
Kubernetes resources
 ↓
Pods / Services / PVCs / ConfigMaps / Secrets
 ↓
Observed state
 ↓
Operator checks again
 ↓
Correct drift
```

For OpenShift:

```text
OpenShift Platform
        ↓
       CVO
        ↓
Cluster Operators
        ↓
OpenShift platform components


Add-on Operators
        ↓
       OLM
        ↓
Subscription
        ↓
InstallPlan
        ↓
CSV
        ↓
Operator
        ↓
CRD
        ↓
CR
        ↓
Application resources
```

### One-line L3 interview answer

> **Kubernetes Operators extend the Kubernetes control loop with application-specific operational knowledge. In OpenShift, platform Operators manage core OpenShift services under the Cluster Version Operator, while OLM provides lifecycle management for add-on Operators.**
