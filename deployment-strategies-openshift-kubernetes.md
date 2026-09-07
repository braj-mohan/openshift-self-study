# Kubernetes / OpenShift Deployment Strategies

## 1. Overview

Deployment strategy defines how a new application version is released and how old Pods are replaced.

| Strategy | Native Kubernetes Deployment | Downtime | Main Use |
|---|---|---:|---|
| Recreate | Yes | Possible | Dev/test, incompatible versions |
| RollingUpdate | Yes | Usually no | Standard production deployments |
| Blue-Green | No (pattern) | No | Fast switch and rollback |
| Canary | No (pattern) | No | Gradual production rollout |
| A/B | No (pattern) | No | User/feature experiments |

---

## 2. Recreate

Old Pods are terminated before new Pods are created.

```yaml
strategy:
  type: Recreate
```

```text
Old Pods
   ↓
Deleted
   ↓
New Pods
```

**Pros:** Simple, useful when old/new versions cannot coexist.  
**Cons:** Can cause downtime.

---

## 3. RollingUpdate

The default Kubernetes Deployment strategy. Old Pods are gradually replaced by new Pods.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

```text
v1 v1 v1
  ↓
v1 v1 v2
  ↓
v1 v2 v2
  ↓
v2 v2 v2
```

### Important Parameters

- `maxUnavailable` — Maximum Pods that can be unavailable during rollout.
- `maxSurge` — Maximum extra Pods that can temporarily run above the desired replica count.

**Best for:** Normal production applications.

---

## 4. Blue-Green

Two complete application environments are maintained:

```text
Blue  → v1
Green → v2
```

Traffic is switched from Blue to Green after testing.

```text
Users
  ↓
Service / Route
  ↓
Green (v2)
```

**Pros:** Fast rollback, easy validation.  
**Cons:** Requires additional resources.

> Blue-Green is a deployment pattern, not a native `Deployment.spec.strategy` value.

---

## 5. Canary

A small percentage of traffic is sent to the new version first.

```text
90% → v1
10% → v2
```

If v2 is healthy, gradually increase traffic:

```text
90/10 → 70/30 → 50/50 → 0/100
```

**Best for:** High-risk production releases.

> Canary is a progressive-delivery pattern, not a native Kubernetes Deployment strategy.

---

## 6. A/B Deployment

Different users or groups are routed to different application versions.

```text
Users A → v1
Users B → v2
```

Routing can be based on user group, headers, cookies, geography, or other rules depending on the routing technology.

**Best for:** Feature testing and experiments.

---

## 7. OpenShift DeploymentConfig

OpenShift historically provided `DeploymentConfig` with strategies such as:

- Recreate
- Rolling
- Custom

For modern OpenShift applications, Kubernetes-native `Deployment` objects are generally preferred.

---

## 8. OpenShift Traffic Flow

```text
Internet
   ↓
Route / Ingress / Gateway
   ↓
Service
   ↓
Pods
```

OpenShift Routes can be useful when implementing traffic-based patterns such as Blue-Green and Canary.

---

## 9. Quick Comparison

| Strategy | Pod Replacement | Traffic Control | Rollback |
|---|---|---|---|
| Recreate | All at once | No | Easy |
| RollingUpdate | Gradual | Limited | Easy |
| Blue-Green | Separate environments | Excellent | Very fast |
| Canary | Gradual exposure | Excellent | Very fast |
| A/B | Multiple versions | Rule-based | Good |

---

## 10. Interview Quick Recall

- **Default Kubernetes strategy:** `RollingUpdate`
- **Native Deployment strategies:** `Recreate`, `RollingUpdate`
- **`maxUnavailable`:** Controls unavailable Pods.
- **`maxSurge`:** Controls extra Pods.
- **Blue-Green:** Switch traffic between two complete environments.
- **Canary:** Gradually expose the new version to users.
- **A/B:** Route different user groups to different versions.
- **RollingUpdate:** Controls gradual Pod replacement.
- **Canary/Blue-Green:** Primarily concern traffic/version exposure.
