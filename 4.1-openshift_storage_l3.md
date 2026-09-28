# Kubernetes & OpenShift Storage --- L3 Corporate Administrator Guide

## 1. Storage Architecture

The Kubernetes/OpenShift storage model separates application storage
requirements from the underlying storage implementation.

``` text
                    APPLICATION / POD
                           |
                           v
                    +-------------+
                    |     PVC     |
                    | "I need     |
                    | 100Gi RWO"  |
                    +------+------+
                           |
                           v
                    +-------------+
                    | StorageClass|
                    | "Which type |
                    | of storage?"|
                    +------+------+
                           |
                     CSI Provisioner
                           |
                           v
                    +-------------+
                    |     PV      |
                    | Actual K8s  |
                    | storage obj |
                    +------+------+
                           |
                           v
              +------------------------+
              |      CSI Driver        |
              +-----------+------------+
                          |
                          v
        +-------------------------------------+
        |       Physical / Virtual Storage    |
        | SAN / NAS / Ceph / ODF / EBS /      |
        | Azure Disk / vSphere / NFS / etc.   |
        +-------------------------------------+
```

Key distinction:

> **PVC is the application's request. PV represents storage allocated to
> satisfy that request. StorageClass describes how storage should be
> dynamically created. CSI connects Kubernetes/OpenShift to the storage
> platform.**

------------------------------------------------------------------------

## 2. What Is a Volume?

A Volume is storage made available to containers inside a Pod.

Example:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data

  volumes:
  - name: data
    emptyDir: {}
```

Here:

``` text
Pod
 |
 +-- nginx container
 |      |
 |      +-- /data
 |
 +-- emptyDir volume
```

Kubernetes volumes can provide temporary storage, shared storage between
containers, configuration data, secrets, and persistent application
data.

------------------------------------------------------------------------

## 3. Ephemeral vs Persistent Storage

``` text
Storage
|
+-- Ephemeral
|   +-- emptyDir
|   +-- ConfigMap
|   +-- Secret
|   +-- other temporary volume mechanisms
|
+-- Persistent
    +-- PV
    +-- PVC
    +-- StorageClass
    +-- CSI-backed storage
```

### Ephemeral Storage

Data is associated with the Pod lifecycle.

Useful for:

-   Temporary files
-   Application cache
-   Scratch space
-   Temporary processing

Not normally suitable for:

-   Databases
-   Critical application data
-   Data that must survive Pod replacement

### Persistent Storage

Persistent storage can survive Pod restart, deletion, and rescheduling
depending on the storage implementation and lifecycle configuration.

``` text
Pod A
  |
  v
PVC
  |
  v
PV
  |
  v
Storage Backend

Pod A deleted
  |
  v
Pod B created
  |
  v
Same PVC
  |
  v
Same data
```

------------------------------------------------------------------------

# 4. PersistentVolume (PV)

A **PersistentVolume (PV)** is a Kubernetes/OpenShift object
representing persistent storage available to the cluster.

PV is **cluster-scoped**.

``` bash
oc get pv
```

Example:

``` text
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY
pvc-12345678-aaaa-bbbb-cccc-123456789abc  100Gi      RWO            Delete
pvc-98765432-dddd-eeee-ffff-987654321abc  500Gi      RWX            Retain
```

A PV can contain:

-   Capacity
-   Access modes
-   Volume mode
-   StorageClass
-   Reclaim policy
-   CSI driver
-   Volume handle
-   Node affinity
-   Mount options

Example:

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  capacity:
    storage: 100Gi

  accessModes:
  - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: enterprise-block

  volumeMode: Filesystem
```

------------------------------------------------------------------------

# 5. PersistentVolumeClaim (PVC)

A **PVC is an application's request for persistent storage**.

PVC is **namespace-scoped**.

Example:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 100Gi

  storageClassName: enterprise-block
```

Think:

``` text
PV  = "I have 100 GiB storage"

PVC = "I need 100 GiB storage"

Pod = "I want to use this PVC"
```

------------------------------------------------------------------------

# 6. PV vs PVC

  Feature                 PV                         PVC
  ----------------------- -------------------------- -----------------------
  Full name               PersistentVolume           PersistentVolumeClaim
  Scope                   Cluster                    Namespace
  Created by              Admin / provisioner        Application/user
  Represents              Actual allocated storage   Request for storage
  Size                    Capacity                   Requested size
  Access mode             Capability                 Requirement
  Used directly by Pod?   No                         Yes
  StorageClass            Associated                 Requested
  Lifecycle               Independent of Pod         Independent of Pod

Important:

> **Pods consume PVCs, not PVs directly. PVCs bind to PVs.**

------------------------------------------------------------------------

# 7. Static Provisioning

In **static provisioning**, the administrator creates PVs manually
before the application requests them.

``` text
Storage Array
     |
     | Administrator creates storage
     v
    PV
     |
     | PVC binds
     v
    PVC
     |
     v
    Pod
```

Example:

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-nfs-pv
spec:
  capacity:
    storage: 100Gi

  accessModes:
  - ReadWriteMany

  persistentVolumeReclaimPolicy: Retain

  storageClassName: nfs-static

  nfs:
    server: 10.10.10.50
    path: /exports/application
```

PVC:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-pvc
spec:
  accessModes:
  - ReadWriteMany

  resources:
    requests:
      storage: 100Gi

  storageClassName: nfs-static
```

### When Static Provisioning Makes Sense

Useful when:

-   Existing storage already exists
-   Special storage needs manual allocation
-   Legacy SAN/NAS environments
-   Migration projects
-   Storage requires administrator approval
-   A specific LUN/export must be used

### Disadvantage

Large environments become difficult to manage manually.

For example:

``` text
5,000 applications
      |
      v
5,000 manually-created PVs
```

This is why enterprises normally prefer dynamic provisioning.

------------------------------------------------------------------------

# 8. Dynamic Provisioning

Dynamic provisioning automatically creates storage when a PVC is
submitted.

``` text
Developer
   |
   | PVC
   v
StorageClass
   |
   v
CSI Provisioner
   |
   v
Storage Platform
   |
   v
Volume created automatically
   |
   v
PV created
   |
   v
PVC -> PV
```

Example:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  storageClassName: fast-block

  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 200Gi
```

The administrator does not manually create the PV.

------------------------------------------------------------------------

# 9. StorageClass

A **StorageClass defines a class/profile of storage**.

For example:

``` text
StorageClass
|
+-- fast-block
|   +-- High IOPS SSD
|
+-- standard-block
|   +-- General-purpose disk
|
+-- shared-file
|   +-- RWX filesystem
|
+-- archive
    +-- Cheap large-capacity storage
```

StorageClasses can represent different:

-   Performance levels
-   Availability characteristics
-   Backup policies
-   Replication levels
-   Storage technologies
-   Cost tiers

------------------------------------------------------------------------

# 10. StorageClass Example

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-block

provisioner: csi.example.com

parameters:
  type: ssd
  iops: "5000"

reclaimPolicy: Delete

allowVolumeExpansion: true

volumeBindingMode: WaitForFirstConsumer
```

Important fields:

``` text
StorageClass
|
+-- provisioner
+-- parameters
+-- reclaimPolicy
+-- allowVolumeExpansion
+-- volumeBindingMode
+-- mountOptions
```

------------------------------------------------------------------------

# 11. Provisioner

The `provisioner` tells Kubernetes which storage driver is responsible
for creating the volume.

Example:

``` yaml
provisioner: ebs.csi.aws.com
```

AWS EBS CSI driver.

Example:

``` yaml
provisioner: csi.vsphere.vmware.com
```

VMware vSphere CSI.

A CSI-based Ceph implementation may use a provisioner such as:

``` yaml
provisioner: openshift-storage.rbd.csi.ceph.com
```

depending on the OpenShift Data Foundation deployment.

Conceptually:

``` text
StorageClass
      |
      v
CSI Driver
      |
      v
Storage Backend
```

------------------------------------------------------------------------

# 12. CSI --- Container Storage Interface

CSI is a standard interface that allows container orchestration
platforms to communicate with storage systems.

Without CSI, Kubernetes would need vendor-specific integration for:

-   Dell
-   NetApp
-   Pure
-   HPE
-   Ceph
-   AWS
-   Azure
-   GCP
-   VMware
-   Other storage vendors

CSI provides a standardized model:

``` text
             Kubernetes/OpenShift
                     |
                     | CSI
                     v
                CSI Driver
                     |
          +----------+----------+
          |          |          |
          v          v          v
        SAN        NAS        Cloud
      Storage     Storage     Storage
```

------------------------------------------------------------------------

# 13. CSI Architecture

A simplified enterprise CSI architecture:

``` text
                    Kubernetes API
                         |
                         v
                  CSI Controller
                         |
              +----------+---------+
              |          |          |
              v          v          v
        Provisioner  Attacher  Resizer
              |
              v
        Storage Backend


Node
 |
 +-- CSI Node Plugin
       |
       +-- Node Driver
             |
             v
        Storage Device
```

### CSI Controller

Responsible for control-plane storage operations such as:

``` text
CreateVolume
DeleteVolume
ControllerPublishVolume
ControllerExpandVolume
CreateSnapshot
```

### CSI Node Plugin

Normally runs on nodes as a DaemonSet.

Responsible for operations such as:

``` text
NodeStageVolume
NodePublishVolume
NodeUnpublishVolume
```

Simple mental model:

``` text
Controller = "Create/attach the storage"

Node plugin = "Make the storage available on this node"
```

------------------------------------------------------------------------

# 14. Access Modes

Important access modes:

``` text
RWO
ROX
RWX
RWOP
```

------------------------------------------------------------------------

## 14.1 RWO --- ReadWriteOnce

Read-write from a single node.

``` text
Read + Write
      |
      v
Single Node
```

Multiple Pods on the same node may be able to use the volume depending
on the storage implementation.

Example:

``` text
Node 1
 +-- Pod A --+
 +-- Pod B --+--> RWO volume
```

Common workloads:

-   PostgreSQL
-   MySQL
-   MongoDB
-   Elasticsearch data nodes
-   Single-instance applications

------------------------------------------------------------------------

## 14.2 RWOP --- ReadWriteOncePod

More restrictive than RWO.

``` text
One Pod
```

can mount the volume read-write.

Comparison:

``` text
RWO:

One Node
 +-- Pod A
 +-- Pod B


RWOP:

One Pod
```

RWOP is useful when strict single-Pod ownership is required.

------------------------------------------------------------------------

## 14.3 ROX --- ReadOnlyMany

Multiple nodes can mount the volume, but read-only.

``` text
        +-- Pod A
        |
PV -----+-- Pod B
        |
        +-- Pod C

READ ONLY
```

Useful for:

-   Static content
-   Reference data
-   Software repositories
-   Shared read-only datasets

------------------------------------------------------------------------

## 14.4 RWX --- ReadWriteMany

Multiple nodes can mount the volume read-write.

``` text
                 +-- Node 1 / Pod A
                 |
RWX Storage -----+-- Node 2 / Pod B
                 |
                 +-- Node 3 / Pod C
```

Typical backends:

-   NFS
-   CephFS
-   EFS
-   Azure Files
-   NetApp
-   Enterprise NAS

Useful for:

-   Shared application files
-   CMS
-   Web uploads
-   Reports
-   Shared build artifacts

------------------------------------------------------------------------

# 15. Access Mode Does Not Mean Performance

Important interview point:

> **RWX does not mean faster or better than RWO.**

Access modes describe **how storage can be accessed**, not how fast it
is.

For example:

``` text
RWX + slow NFS
```

could be much slower than:

``` text
RWO + high-performance NVMe
```

------------------------------------------------------------------------

# 16. VolumeMode

Two important volume modes:

``` text
volumeMode
|
+-- Filesystem
|
+-- Block
```

### Filesystem

Default mode.

``` yaml
volumeMode: Filesystem
```

The application gets a mounted filesystem such as:

``` text
/mnt/data
```

Most applications use this mode.

### Block

``` yaml
volumeMode: Block
```

The application gets a raw block device instead of a mounted filesystem.

Conceptually:

``` text
Filesystem:

/mnt/data


Block:

/dev/<device>
```

Useful for specialized applications that understand raw block devices.

------------------------------------------------------------------------

# 17. Reclaim Policy

Reclaim policy controls what happens to storage after the PVC is
deleted.

Modern policies:

``` text
Delete
Retain
```

`Recycle` is deprecated for modern Kubernetes environments.

------------------------------------------------------------------------

## 17.1 Delete

``` text
PVC deleted
   |
   v
PV deleted
   |
   v
Backend volume deleted
```

Useful for:

-   Development
-   Testing
-   CI/CD
-   Temporary environments
-   Disposable application data

Example:

``` yaml
reclaimPolicy: Delete
```

------------------------------------------------------------------------

## 17.2 Retain

``` text
PVC deleted
   |
   v
PV/storage retained
```

Useful for:

-   Production databases
-   Critical business data
-   Forensic data
-   Migration
-   Data requiring manual approval before deletion

Example:

``` yaml
reclaimPolicy: Retain
```

### Important

> **Retain is not a backup policy.**

``` text
Retain != Backup
```

You still need:

-   Backups
-   Snapshots
-   Replication
-   Disaster recovery
-   Application-level recovery

------------------------------------------------------------------------

# 18. volumeBindingMode

Important values:

``` text
Immediate
WaitForFirstConsumer
```

### Immediate

Storage can be provisioned as soon as the PVC is created.

``` text
PVC
 |
 v
Storage provisioned
```

Potential problem:

Storage may be created in a topology or availability zone incompatible
with the node where the Pod eventually runs.

### WaitForFirstConsumer

Storage provisioning waits until the scheduler has information about the
Pod's placement.

``` text
PVC
 |
 v
Pod created
 |
 v
Scheduler determines suitable node/topology
 |
 v
Storage provisioned appropriately
```

This is particularly important for topology-aware storage.

------------------------------------------------------------------------

# 19. allowVolumeExpansion

Example:

``` yaml
allowVolumeExpansion: true
```

Suppose:

``` text
Current PVC = 100Gi
```

You can increase it to:

``` text
200Gi
```

Example:

``` bash
oc edit pvc database-pvc
```

Change:

``` yaml
resources:
  requests:
    storage: 200Gi
```

Important:

> Volume expansion is supported by compatible storage drivers, but
> shrinking a PVC is generally not supported.

------------------------------------------------------------------------

# 20. Enterprise Storage Types

Think about storage in these categories:

``` text
Storage
|
+-- Block Storage
|
+-- File Storage
|
+-- Object Storage
|
+-- Local / NVMe
|
+-- Software-Defined Storage
```

------------------------------------------------------------------------

# 21. Block Storage

Examples:

-   SAN
-   Fibre Channel
-   iSCSI
-   AWS EBS
-   Azure Disk
-   GCP Persistent Disk
-   VMware vSphere
-   Ceph RBD
-   NVMe-oF

Characteristics:

``` text
High IOPS
Low latency
Filesystem created on block device
Often RWO
```

Ideal for:

-   Databases
-   VM disks
-   Transactional applications
-   High-performance workloads

Example:

``` text
PostgreSQL
    |
    v
PVC 500Gi RWO
    |
    v
Premium SSD
    |
    v
CSI
    |
    v
Enterprise block storage
```

------------------------------------------------------------------------

# 22. File Storage

Examples:

-   NFS
-   SMB/CIFS
-   NetApp
-   Dell PowerScale
-   CephFS
-   AWS EFS
-   Azure Files
-   GCP Filestore

Characteristics:

``` text
Shared filesystem
Often RWX
Multiple clients
```

Ideal for:

-   Shared files
-   Web uploads
-   CMS
-   Documents
-   Reports
-   Shared application content

Example:

``` text
Pod A --+
Pod B --+--> RWX --> NFS
Pod C --+
```

------------------------------------------------------------------------

# 23. Object Storage

Examples:

-   Amazon S3
-   Azure Blob Storage
-   Google Cloud Storage
-   MinIO
-   Ceph Object Gateway
-   NetApp StorageGRID

Object storage is normally accessed through APIs rather than as a normal
POSIX filesystem.

Typical interfaces:

``` text
HTTP
REST API
S3 API
SDK
```

Excellent for:

-   Backups
-   Logs
-   Images
-   Videos
-   Documents
-   Data lakes
-   Archives
-   Large unstructured data

Example:

``` text
Application
    |
    v
S3 API
    |
    v
Object Storage
    |
    +-- images
    +-- backups
    +-- logs
    +-- documents
```

------------------------------------------------------------------------

# 24. Local Storage / NVMe

Local storage is physically attached to a node.

``` text
Worker Node
|
+-- CPU
+-- RAM
+-- NVMe SSD
      |
      v
   Local PV
```

Advantages:

-   Very low latency
-   Very high IOPS
-   No remote storage network hop

Disadvantages:

-   Node-dependent
-   Failure management
-   Data mobility challenges
-   Capacity tied to nodes
-   More complex topology

Useful for:

-   High-performance workloads
-   Caches
-   Databases
-   Analytics
-   Logging

------------------------------------------------------------------------

# 25. Software-Defined Storage

Examples:

-   Red Hat OpenShift Data Foundation (ODF)
-   Ceph
-   Portworx
-   Longhorn
-   Other distributed storage platforms

Conceptually:

``` text
Node 1 -- disks
Node 2 -- disks
Node 3 -- disks
             |
             v
   Software-defined storage
             |
             v
      Distributed pool
```

------------------------------------------------------------------------

# 26. ODF / Ceph Concept

A simplified architecture:

``` text
             OpenShift
                 |
             PVC request
                 |
                 v
                CSI
                 |
       +---------+---------+
       |         |         |
       v         v         v
     RBD       CephFS    Object
    Block       File     Storage
       |         |         |
       +---------+---------+
                 |
                 v
                Ceph
                 |
        +--------+--------+
        |        |        |
      Node 1   Node 2   Node 3
        |        |        |
       SSD      SSD      SSD
```

This provides distributed storage rather than depending on a single
external NAS/SAN.

------------------------------------------------------------------------

# 27. Storage Selection by Application

## Database

For a latency-sensitive database:

``` text
Block
+
SSD/NVMe
+
RWO
+
High IOPS
+
Low latency
```

Example:

``` text
PostgreSQL
   |
   v
PVC 500Gi
   |
   v
RWO
   |
   v
Premium SSD
   |
   v
Enterprise block storage
```

Avoid automatically using inexpensive shared file storage for databases
without performance and consistency testing.

------------------------------------------------------------------------

## Web Application

Suppose there are 10 replicas and all require shared uploaded files:

``` text
Web Pod 1 --+
Web Pod 2 --+
Web Pod 3 --+
Web Pod 4 --+--> RWX --> File Storage
Web Pod 5 --+
Web Pod 6 --+
```

Possible storage:

-   NFS
-   CephFS
-   EFS
-   Azure Files
-   NetApp

For cloud-native applications, object storage may be better for user
uploads:

``` text
Web Pods
   |
   v
Object Storage
   |
   v
S3-compatible storage
```

------------------------------------------------------------------------

# 28. Logging

Avoid putting all logs on expensive high-IOPS block storage.

A better architecture can be:

``` text
Application
    |
    v
Log Collector
    |
    +--> Hot logs --> Fast storage
    |
    +--> Older logs --> Object storage
```

This is a form of storage tiering.

------------------------------------------------------------------------

# 29. Backup

Backups are often well suited to object storage because it can provide:

-   Large capacity
-   Low cost/GB
-   Retention
-   Versioning
-   Immutability
-   Replication
-   Long-term storage

Architecture:

``` text
OpenShift
   |
   v
Backup Platform
   |
   v
Object Storage
   |
   +-- Daily
   +-- Weekly
   +-- Monthly
   +-- Long-term
```

------------------------------------------------------------------------

# 30. High-Performance Analytics

For analytics workloads, consider:

-   NVMe
-   SSD
-   Distributed high-performance storage
-   High throughput
-   High IOPS

Choose based on the actual workload rather than simply storage capacity.

------------------------------------------------------------------------

# 31. Storage Performance Metrics

Two systems can both provide 1 TB but behave very differently.

### Storage A

``` text
1 TB
1,000 IOPS
20 ms latency
```

### Storage B

``` text
1 TB
100,000 IOPS
1 ms latency
```

Same capacity, very different application behavior.

Therefore:

> **Never select enterprise storage based only on GB/TB capacity.**

Important metrics:

``` text
Capacity
IOPS
Latency
Throughput
Bandwidth
Availability
Durability
Replication
```

------------------------------------------------------------------------

# 32. IOPS

IOPS = Input/Output Operations Per Second.

Example:

``` text
10,000 IOPS
```

means approximately 10,000 storage operations per second under the
relevant workload conditions.

Databases often care heavily about:

``` text
IOPS + latency
```

------------------------------------------------------------------------

# 33. Throughput

Throughput is the rate at which data can be transferred.

Example:

``` text
500 MB/s
```

Throughput matters heavily for:

-   Large sequential reads
-   Large sequential writes
-   Video processing
-   Analytics
-   Backup
-   ETL

A database may care more about:

``` text
IOPS + latency
```

while backup workloads may care more about:

``` text
Throughput
```

------------------------------------------------------------------------

# 34. Latency

Latency is the time required to complete an I/O operation.

Generally:

``` text
Lower latency -> faster response for latency-sensitive workloads
```

Typical conceptual hierarchy:

``` text
NVMe
  |
  v
Very low latency

SSD SAN
  |
  v
Low latency

HDD
  |
  v
Higher latency

Object Storage
  |
  v
Different API/access model
```

Exact performance depends on the platform and workload.

------------------------------------------------------------------------

# 35. Storage Cost Strategy

Do not put everything on premium SSD.

Use storage tiers:

``` text
                 Storage Tier
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    Premium        Standard        Archive
       |              |              |
      NVMe           SSD            Object
    High IOPS       General        Cheap
    High cost       Medium         Low
```

Example:

  Workload             Storage           Access   Performance           Relative Cost
  -------------------- ----------------- -------- --------------------- --------------------
  PostgreSQL           Premium block     RWO      Very high             High
  Normal application   Standard block    RWO      Medium                Medium
  Shared files         File/NFS/CephFS   RWX      Medium                Medium
  Backup               Object            API      Throughput-oriented   Low
  Archive              Object/archive    API      Low                   Very low
  Cache                Local NVMe        Local    Very high             Workload dependent

------------------------------------------------------------------------

# 36. Corporate StorageClass Strategy

Avoid creating many random StorageClasses.

Create classes around business requirements:

``` text
StorageClasses
|
+-- prod-fast
|   +-- Production databases
|
+-- prod-standard
|   +-- Normal applications
|
+-- shared-rwx
|   +-- Shared filesystem workloads
|
+-- dev-standard
|   +-- Development
|
+-- archive
    +-- Long-term / low-cost storage
```

Developers should not need to understand:

``` text
SAN LUN 47
Ceph pool xyz
NetApp aggregate abc
AWS availability zone
```

They can simply request:

``` yaml
storageClassName: prod-fast
```

------------------------------------------------------------------------

# 37. Enterprise StorageClass Example

Conceptual example:

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prod-fast

provisioner: csi.vendor.example

parameters:
  type: ssd
  replication: "3"
  iops: "10000"

reclaimPolicy: Retain

allowVolumeExpansion: true

volumeBindingMode: WaitForFirstConsumer
```

Application PVC:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: prod-fast

  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 500Gi
```

------------------------------------------------------------------------

# 38. Pod Using a PVC

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1

  selector:
    matchLabels:
      app: app

  template:
    metadata:
      labels:
        app: app

    spec:
      containers:
      - name: app
        image: nginx

        volumeMounts:
        - name: application-data
          mountPath: /data

      volumes:
      - name: application-data
        persistentVolumeClaim:
          claimName: postgres-data
```

Complete chain:

``` text
Deployment
   |
   v
Pod
   |
   v
PVC
   |
   v
PV
   |
   v
CSI
   |
   v
Storage Backend
```

------------------------------------------------------------------------

# 39. PVC Pending Troubleshooting

Suppose:

``` bash
oc get pvc
```

returns:

``` text
NAME           STATUS    VOLUME
database-pvc   Pending
```

Do not immediately blame the storage array.

Follow:

``` text
PVC
 |
 v
StorageClass
 |
 v
CSI
 |
 v
Provisioner
 |
 v
Backend
 |
 v
Node
 |
 v
Mount
```

First:

``` bash
oc describe pvc database-pvc
```

Check Events.

Then:

``` bash
oc get sc
```

Check:

-   StorageClass exists
-   Correct provisioner
-   Default StorageClass
-   Parameters

Then:

``` bash
oc describe sc <sc-name>
```

CSI:

``` bash
oc get pods -A | grep -i csi
```

Storage platform:

``` bash
oc get pods -A
```

and inspect the relevant storage namespace.

Events:

``` bash
oc get events -A --sort-by='.lastTimestamp'
```

------------------------------------------------------------------------

# 40. PVC Pending Troubleshooting Tree

``` text
PVC Pending
|
+-- Is StorageClass specified?
|      |
|      +-- No -> Check default StorageClass
|
+-- Does StorageClass exist?
|      |
|      +-- No -> Fix StorageClass
|
+-- Is provisioner healthy?
|      |
|      +-- No -> Troubleshoot CSI
|
+-- Is backend reachable?
|      |
|      +-- No -> Network/storage issue
|
+-- Is capacity available?
|      |
|      +-- No -> Storage capacity issue
|
+-- Is topology correct?
|      |
|      +-- No -> Check WaitForFirstConsumer/topology
|
+-- Is access mode supported?
|      |
|      +-- No -> Select compatible storage
|
+-- Check Events
```

------------------------------------------------------------------------

# 41. Pod Pending Because of Storage

Scenario:

``` text
PVC
 |
 v
Bound
 |
 v
Pod Pending
```

PVC is not necessarily the problem.

Run:

``` bash
oc describe pod <pod>
```

Potential reasons:

-   Volume node affinity conflict
-   Insufficient storage
-   Node topology mismatch
-   CSI attach limit
-   CSI node plugin issue
-   Volume attachment problem

------------------------------------------------------------------------

# 42. Pod Running but Mount Failed

Common errors:

``` text
FailedMount
FailedAttachVolume
MountVolume.SetUp failed
```

Troubleshooting path:

``` text
Pod
 |
 v
Kubelet
 |
 v
CSI Node Plugin
 |
 v
Storage Network
 |
 v
Storage Backend
```

Commands:

``` bash
oc describe pod <pod>
```

``` bash
oc get pods -A | grep -i csi
```

``` bash
oc get nodes
```

Then inspect the relevant CSI node/controller logs.

------------------------------------------------------------------------

# 43. FailedAttachVolume vs FailedMount

### FailedAttachVolume

Usually indicates an attachment/controller/backend problem.

Think about:

``` text
CSI controller
Storage backend
Volume attachment
Cloud volume
Node attachment
```

### FailedMount

The volume may already be attached, but mounting it on the node is
failing.

Think about:

``` text
CSI node plugin
Filesystem
Permissions
Device
Mount options
SELinux
Network
```

------------------------------------------------------------------------

# 44. Storage and SELinux

OpenShift security makes this especially important.

Suppose an NFS volume is mounted but the application receives:

``` text
Permission denied
```

Do not assume normal Linux permissions are the only cause.

Investigate:

``` text
UID/GID
SELinux
SCC
Mount options
NFS export permissions
Application user
```

------------------------------------------------------------------------

# 45. Storage and SCC

The application may run as a non-root UID.

Conceptually:

``` text
Pod
 |
 v
SCC
 |
 v
Security Context
 |
 v
UID
 |
 v
Volume permissions
```

If the NFS backend expects one UID/GID while OpenShift runs the
container with another permitted UID, the application can get:

``` text
Permission denied
```

Useful inspection:

``` bash
oc get pod <pod> -o yaml
```

``` bash
oc get pod <pod> -o jsonpath='{.spec.securityContext}'
```

Also check:

-   Storage-side permissions
-   NFS export configuration
-   SELinux behavior
-   Application UID/GID

------------------------------------------------------------------------

# 46. Storage Snapshots

Modern Kubernetes/OpenShift storage architectures can use:

``` text
VolumeSnapshot
VolumeSnapshotClass
VolumeSnapshotContent
```

Conceptually:

``` text
PVC
 |
 v
PV
 |
 v
Snapshot
 |
 +--> Restore
       |
       v
     New PVC
```

Useful for:

-   Database recovery
-   Testing
-   Environment cloning
-   Application migration
-   Point-in-time recovery

Important:

> A storage snapshot is not automatically a complete
> application-consistent backup.

For databases, application consistency matters.

------------------------------------------------------------------------

# 47. PVC Clone

Conceptually:

``` text
Source PVC
    |
    v
Clone
    |
    v
New PVC
```

Useful for:

``` text
Production DB
     |
     v
   Clone
     |
     v
Testing/UAT
```

Availability depends on the CSI driver and storage platform.

------------------------------------------------------------------------

# 48. Storage Topology

Modern storage can be topology-dependent.

Example:

``` text
Region
|
+-- AZ-A
|   +-- Storage A
|
+-- AZ-B
|   +-- Storage B
|
+-- AZ-C
    +-- Storage C
```

If the Pod is scheduled in AZ-A but its volume exists only in AZ-C,
depending on the storage technology the Pod may fail to start or may
experience undesirable topology behavior.

This is why:

``` yaml
volumeBindingMode: WaitForFirstConsumer
```

and topology-aware CSI drivers are important.

------------------------------------------------------------------------

# 49. Storage High Availability

Do not confuse:

``` text
PVC HA
```

with:

``` text
Application HA
```

Example:

``` text
PostgreSQL
   |
   v
RWO PVC
```

The storage system may support reattachment after a node failure, but
that alone does not make PostgreSQL an HA application.

True application HA may require:

``` text
Application replication
+
Multiple Pods/nodes
+
Storage redundancy
+
Pod anti-affinity
+
Backups
+
DR
```

------------------------------------------------------------------------

# 50. Storage Redundancy Layers

Think about HA at multiple layers:

``` text
Application
     |
     v
Pod redundancy
     |
     v
Kubernetes node redundancy
     |
     v
CSI/storage redundancy
     |
     v
Disk redundancy
     |
     v
Datacenter redundancy
     |
     v
DR site
```

Important:

``` text
RAID alone != HA
Storage replication alone != Application HA
PVC alone != Backup
```

------------------------------------------------------------------------

# 51. Storage Types: Practical Selection

  -----------------------------------------------------------------------
  Requirement                         Recommended Approach
  ----------------------------------- -----------------------------------
  High-IOPS database                  Premium block / NVMe

  Normal database                     SSD block

  Shared application files            RWX file storage

  Large backup                        Object storage

  Archive                             Object/archive tier

  Temporary cache                     emptyDir/local NVMe

  High-performance analytics          NVMe/distributed high-performance
                                      storage

  VM-style workload                   Block

  Shared configuration                ConfigMap/Secret

  Multi-Pod shared writes             RWX file/distributed filesystem

  Large media files                   Object storage

  Critical production data            Replicated enterprise storage +
                                      backup
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 52. Do Not Use PVC for Everything

Suppose an application needs:

``` text
Configuration
Secrets
Temporary cache
User uploads
Database
Logs
```

Do not automatically create a PVC for each.

A better architecture can be:

``` text
Configuration -> ConfigMap
Secrets        -> Secret
Temporary      -> emptyDir
Database       -> Block PVC
User uploads   -> Object Storage
Logs           -> Logging platform/Object Storage
```

This can improve:

-   Cost
-   Performance
-   Scalability
-   Backup strategy
-   Operational simplicity

------------------------------------------------------------------------

# 53. Example Corporate OpenShift Storage Architecture

``` text
                     USERS
                       |
                       v
                 OpenShift Route
                       |
                       v
                +--------------+
                | Web Pods     |
                | 3 replicas   |
                +------+-------+
                       |
              +--------+--------+
              |                 |
              v                 v
         Object Storage       Redis
         User uploads        Cache
              |
              v
        S3-compatible
           Storage

                       |
                       v
                   API Pods
                       |
                       v
                  PostgreSQL
                       |
                       v
                 RWO Premium SSD
                       |
                       v
                   CSI Driver
                       |
                       v
                Enterprise SAN
```

This is usually more scalable than putting every data type into one
large shared filesystem.

------------------------------------------------------------------------

# 54. Cost Optimization

Suppose an enterprise has 100 TB of data.

Do not automatically purchase:

``` text
100 TB premium SSD
```

Instead classify data:

``` text
5 TB   -> Premium SSD
20 TB  -> Standard SSD
25 TB  -> Standard/HDD
50 TB  -> Object/archive
```

Use:

``` text
Hot data
  |
  v
Fast storage

Warm data
  |
  v
Standard storage

Cold data
  |
  v
Object/archive
```

This is storage tiering.

------------------------------------------------------------------------

# 55. StorageClass Naming Strategy

Avoid:

``` text
sc1
sc2
storage-new
teststorage
abc-sc
```

Prefer meaningful names:

``` text
prod-block-premium
prod-block-standard
prod-file-rwx
dev-block-standard
backup-object
```

Naming should communicate the intended workload and performance/cost
tier.

------------------------------------------------------------------------

# 56. What an L3 OpenShift Administrator Should Monitor

Storage administration is not only:

``` bash
oc get pvc
```

Monitor:

``` text
Capacity
IOPS
Latency
Throughput
PVC utilization
PV utilization
CSI health
Node storage
Filesystem usage
Storage backend health
Replication
Disk failures
Snapshots
Volume expansion
Mount failures
Attach failures
```

Also monitor:

``` text
Capacity Forecasting
```

Example:

``` text
Current storage = 70 TB
Growth = 5 TB/month
Available = 20 TB
```

You should forecast when the storage platform will reach a critical
threshold rather than waiting for:

``` text
No space left on device
```

------------------------------------------------------------------------

# 57. Useful OpenShift Commands

### StorageClasses

``` bash
oc get storageclass
oc get sc
oc describe sc <storageclass>
```

### PVs

``` bash
oc get pv
oc describe pv <pv-name>
```

### PVCs

``` bash
oc get pvc -A
oc get pvc -n <namespace>
oc describe pvc <pvc> -n <namespace>
```

### Pods

``` bash
oc get pods -n <namespace>
oc describe pod <pod> -n <namespace>
```

### Events

``` bash
oc get events -A --sort-by='.lastTimestamp'
```

### Nodes

``` bash
oc get nodes
oc describe node <node>
```

### CSI-related Pods

``` bash
oc get pods -A | grep -i csi
```

### Storage-related API resources

``` bash
oc api-resources | grep -i storage
```

### PVC YAML

``` bash
oc get pvc <pvc> -o yaml
```

### PV YAML

``` bash
oc get pv <pv> -o yaml
```

### StorageClass YAML

``` bash
oc get sc <sc> -o yaml
```

------------------------------------------------------------------------

# 58. Complete Storage Lifecycle

A strong L3 administrator should be able to explain this complete
lifecycle:

``` text
Developer creates PVC
        |
        v
PVC references StorageClass
        |
        v
StorageClass selects provisioner
        |
        v
CSI provisioner receives request
        |
        v
Storage backend creates volume
        |
        v
PV object created
        |
        v
PVC <-> PV Bound
        |
        v
Pod references PVC
        |
        v
Scheduler selects node
        |
        v
CSI controller attaches volume
        |
        v
CSI node plugin mounts volume
        |
        v
Container accesses filesystem
```

When the application is deleted:

``` text
Pod deleted
   |
   v
Volume unmounted
   |
   v
Volume detached
   |
   v
PVC deleted
   |
   v
ReclaimPolicy evaluated
   |
   +-- Delete -> backend volume deleted
   |
   +-- Retain -> storage preserved
```

------------------------------------------------------------------------

# 59. L3 Storage Decision Framework

Whenever someone asks:

> "Which storage should we use?"

Ask five questions.

### 1. What is the workload?

``` text
Database?
Web?
Logs?
Backup?
Analytics?
Cache?
```

### 2. What is the access pattern?

``` text
RWO?
RWX?
ROX?
RWOP?
```

### 3. What performance is required?

``` text
IOPS?
Latency?
Throughput?
```

### 4. What availability is required?

``` text
Single node?
Multi-node?
Replication?
Availability Zone?
DR?
```

### 5. What is the cost requirement?

``` text
Premium?
Standard?
Archive?
Object?
```

Then select:

``` text
Workload
   |
   v
Access Pattern
   |
   v
Performance
   |
   v
Availability
   |
   v
Security
   |
   v
Cost
   |
   v
StorageClass
   |
   v
PVC
```

------------------------------------------------------------------------

# 60. Interview-Level Answer

If an interviewer asks:

> **"Explain persistent storage in OpenShift."**

A strong L3 answer:

> OpenShift separates application storage requirements from the
> underlying storage implementation using PVC, PV, StorageClass and CSI.
> A PVC is the application's storage request, while a PV represents the
> provisioned persistent storage. With static provisioning,
> administrators create PVs manually; with dynamic provisioning, a PVC
> references a StorageClass and the associated CSI provisioner
> dynamically creates the backend volume and PV. StorageClasses can
> represent different performance or availability tiers such as premium
> block, standard block or RWX file storage. Access modes such as RWO,
> RWX, ROX and RWOP define how a volume can be consumed, while
> volumeMode defines Filesystem or raw Block access. ReclaimPolicy
> controls what happens to the storage after the PVC is deleted. In an
> enterprise OpenShift environment, storage should be selected based on
> workload characteristics such as IOPS, latency, throughput, access
> pattern, topology, availability, backup requirements, security and
> cost rather than simply storage capacity.

------------------------------------------------------------------------

# 61. Final L3 Mental Model

Memorize this:

``` text
                         OPENSHIFT
                             |
              +--------------+--------------+
              |                             |
             POD                           PVC
              |                             |
              |                    "I need 500Gi RWO"
              |                             |
              |                             v
              |                       StorageClass
              |                             |
              |                         Provisioner
              |                             |
              |                            CSI
              |                             |
              |              +--------------+--------------+
              |              |              |              |
              |              v              v              v
              |            Block          File           Object
              |              |              |              |
              |          SAN/EBS/        NFS/CephFS      S3/Blob
              |          Ceph RBD/       EFS/NetApp
              |          vSphere
              |              |
              +--------------+--------------+
                             |
                             v
                            PV
                             |
                             v
                           Data
```

## Core L3 principle

Do not ask only:

> "How many GB does the application need?"

Ask:

> **What data? What I/O pattern? What access mode? What
> IOPS/latency/throughput? What failure domain? What backup/DR
> requirement? What security requirements? What cost tier?**

That is the difference between basic PVC administration and enterprise
storage architecture.

## Quick Reference

``` text
Volume
  -> Storage exposed to a Pod

PV
  -> Cluster-level representation of persistent storage

PVC
  -> Namespace-level request for persistent storage

StorageClass
  -> Defines a storage tier and dynamic provisioning behavior

Provisioner
  -> CSI driver responsible for provisioning

CSI
  -> Standard interface between Kubernetes/OpenShift and storage

RWO
  -> ReadWriteOnce

RWOP
  -> ReadWriteOncePod

ROX
  -> ReadOnlyMany

RWX
  -> ReadWriteMany

Filesystem
  -> Mounted filesystem

Block
  -> Raw block device

Delete
  -> Delete storage after PVC lifecycle according to policy

Retain
  -> Preserve storage for manual recovery/reclamation

Immediate
  -> Provision without waiting for a consumer

WaitForFirstConsumer
  -> Provision with Pod scheduling/topology information

Block storage
  -> Databases / high-performance workloads

File storage
  -> Shared filesystem / RWX

Object storage
  -> Backup / archive / media / unstructured data

Local NVMe
  -> Very low latency / high IOPS

ODF/Ceph
  -> Distributed software-defined storage
```

## References

-   Kubernetes Persistent Volumes:
    https://kubernetes.io/docs/concepts/storage/persistent-volumes/
-   Kubernetes Storage Classes:
    https://kubernetes.io/docs/concepts/storage/storage-classes/
-   Kubernetes Volumes:
    https://kubernetes.io/docs/concepts/storage/volumes/
-   OpenShift 4.18 Storage Overview:
    https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/storage-overview
-   OpenShift 4.18 Persistent Storage:
    https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/understanding-persistent-storage
