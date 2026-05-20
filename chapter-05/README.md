# Chapter 5 — Pods

**Book:** The Kubernetes Workshop, Packt (2019)
**Cluster:** K3s v1.35.4+k3s1 (3-node, Rocky Linux 9.7)
**Nodes:** k8s-control (control-plane), k8s-worker1, k8s-worker2

---

## What is a Pod?

A pod is the basic building block of Kubernetes and the smallest unit of
deployment. Just like we define a process as a program in execution, we
can define a pod as a running process in the Kubernetes world.

Pods are the smallest unit of replication in Kubernetes. A pod can have
any number of containers running in it. A pod is basically a wrapper
around containers running on a node.

### Why Pods Instead of Individual Containers?

Using pods instead of individual containers has several benefits:

- Containers in a pod share **volumes**
- Containers in a pod share **Linux namespaces** and **cgroups**
- Each pod has a **unique IP address**
- The **port space is shared** by all containers in that pod
- Different containers inside a pod can communicate with each other
  using their corresponding ports on **localhost**

Ideally, we should use multiple containers in a pod only when we want
them to be managed and located together in the Kubernetes cluster.

---

## Pod Configuration

Every pod configuration file has four main components:

```yaml
apiVersion: v1        # Version of Kubernetes API
kind: Pod             # Type of Kubernetes object
metadata:             # Information that uniquely identifies the object
  name: pod-name
spec:                 # Specification of the pod
  containers:
    - name: container-name
      image: container-image
```

### apiVersion
Version of the Kubernetes API we are going to use.
For Pods always v1 (stable, unchanged since Kubernetes 1.0)

### kind
The kind of Kubernetes object we are creating.
apiVersion + kind together uniquely identify the schema.

### metadata
Data ABOUT the resource. name is mandatory.
- Pod name: max 253 characters
- Allowed: lowercase letters (a-z), digits (0-9), hyphens (-), dots (.)
- Must be unique within a namespace

### spec
Specification of our pod — container name, image, volumes,
resource requests, probes, etc.
Different for different types of objects.

---

## Namespaces

Kubernetes supports namespaces to create multiple virtual clusters
within the same physical cluster.

**Use cases:**
- Separate environments for different teams
- Scoping object names (same name allowed in different namespaces)
- Resource isolation

**Default namespaces in every cluster:**

| Namespace | Purpose |
|-----------|---------|
| default | Default namespace for objects with no namespace specified |
| kube-system | Kubernetes system components (CoreDNS, Traefik, etc.) |
| kube-public | Readable by all users, public cluster info |
| kube-node-lease | Node heartbeat objects for health detection |

**Two ways to specify namespace:**
1. CLI flag: kubectl --namespace kube-public create -f pod.yaml
2. In YAML: metadata.namespace: kube-public

---

## Container Configuration Fields

Inside spec.containers, these fields can be set:

| Field | Description |
|-------|-------------|
| name | Name of the container within the pod |
| image | Docker image name |
| command | Command to run (overrides image ENTRYPOINT) |
| args | Arguments to the command (overrides image CMD) |
| ports | List of ports to expose from the container |
| env | List of environment variables |
| resources | CPU and memory requests and limits |

---

## Resource Management

### requests vs limits

```yaml
resources:
  requests:
    memory: "64Mi"    # Minimum guaranteed — scheduler uses this
    cpu: "250m"       # 250 millicores = 0.25 CPU core
  limits:
    memory: "128Mi"   # Maximum allowed — kernel enforces this
    cpu: "500m"       # Container throttled if exceeded
```

**requests** — Scheduling time concept
- Scheduler places pod only on nodes with enough free resources
- Container is guaranteed to get at least this much
- Think: hotel room reservation

**limits** — Runtime concept
- Memory exceeded → OOMKilled (container dies)
- CPU exceeded → Throttled (container slows, does not die)
- Think: walls of the room

**Rule:** requests = typical usage, limits = absolute maximum

### CPU Units
- 1 = 1 full CPU core
- 500m = 500 millicores = 0.5 core
- 250m = 250 millicores = 0.25 core

### Memory Units
- Mi = Mebibytes (preferred)
- Gi = Gibibytes
- M = Megabytes (avoid — ambiguous)

---

## Pod Lifecycle Phases

| Phase | Meaning |
|-------|---------|
| Pending | Submitted to cluster, containers not created yet |
| Running | Pod assigned to node, at least one container running |
| Succeeded | All containers terminated with success (exit code 0) |
| Failed | At least one container terminated with failure |
| Unknown | State cannot be determined |

---

## Probes / Health Checks

A probe is a health check configured to check the health of containers.

### Liveness Probe
**Question:** Is this container still alive and functional?
**Action on failure:** Kill container and restart (per restartPolicy)
**Use case:** Detect deadlocks, stuck states, unrecoverable errors

### Readiness Probe
**Question:** Is this container ready to receive traffic?
**Action on failure:** Remove from Service endpoints — NO kill, NO restart
**Use case:** Slow startup, cache warming, dependency waiting

### Key Difference

```
Liveness failure  → KILL + RESTART (self-healing)
Readiness failure → TRAFFIC GATE (no kill, just stop routing)
```

### Probe Implementation Types

**Command Probe (exec)**
Exit code 0 = success, non-zero = failure

**HTTP Request Probe**
HTTP 200-399 = success, others = failure

**TCP Socket Probe**
Connection established = success

### Probe Configuration Fields

| Field | Description | Default |
|-------|-------------|---------|
| initialDelaySeconds | Wait before first probe | 0 |
| periodSeconds | How often to probe | 10 |
| failureThreshold | Failures before action | 3 |
| timeoutSeconds | Probe timeout | 1 |

### Best Practices

- Liveness initialDelaySeconds must be greater than app startup time.
  Too small causes crash loop — app never gets chance to fully start.

- Readiness initialDelaySeconds can be small.
  Failure means just not ready, no harm in frequent polling.

- Readiness failureThreshold — be careful.
  Too low pulls pod out of rotation permanently on temporary blip.

---

## Restart Policy

Specified in spec.restartPolicy:

| Policy | Behavior | Use Case |
|--------|----------|----------|
| Always | Always restart (default) | Long-running services |
| OnFailure | Restart only on failure | Batch jobs |
| Never | Never restart | One-off tasks, debugging |

---

## Multi-Container Pods — Sidecar Pattern

Multiple containers in one pod share:
- Same IP address
- Same port space
- Same storage volumes
- Same scheduling fate (always on same node)
- Communicate via localhost

**Sidecar pattern:**
- Main container: business logic
- Sidecar container: infrastructure concern
  (logging, metrics, config reload, certificate rotation)

---

## Key kubectl Commands — Chapter 5

```bash
# Create pod from YAML
kubectl create -f pod.yaml

# Watch pod status change live
kubectl get pods --watch

# Full details + Events timeline
kubectl describe pod <name>

# Follow logs in real time
kubectl logs <pod> -f

# Logs from specific container in multi-container pod
kubectl logs <pod> <container> -f

# Shell inside container
kubectl exec -it <pod> -- /bin/sh

# Shell in specific container
kubectl exec -it <pod> -c <container> -- /bin/bash

# Port forward — development only
kubectl port-forward pod/<name> 8080:80

# Delete pod
kubectl delete pod <name>

# Namespace operations
kubectl get namespaces
kubectl --namespace <ns> get pods
kubectl create namespace <name>

# Change default namespace for all commands
kubectl config set-context $(kubectl config current-context) --namespace <namespace>

# Reset back to default
kubectl config set-context $(kubectl config current-context) --namespace default
```

---

## Exercises in This Chapter

| Exercise | Topic | Status |
|----------|-------|--------|
| 5.01 | Creating a Pod with a Single Container | ✅ |
| 5.02 | Creating a Pod in Different Namespace via CLI | ✅ |
| 5.03 | Creating a Pod in Different Namespace via YAML | ✅ |
| 5.04 | Changing Namespace for All kubectl Commands | ✅ |
| 5.05 | Pod Running a Container with a Command | ✅ |
| 5.06 | Pod Running a Container That Exposes a Port | ✅ |
| 5.07 | Pod with Resource Requirements | ✅ |
| 5.08 | Pod with Unmet Resource Requirements | ✅ |
| 5.09 | Pod with Multiple Containers | ✅ |
| 5.10 | Liveness Probe — restartPolicy Always | ✅ |
| 5.11 | Liveness Probe — restartPolicy Never | ✅ |
| 5.12 | Readiness Probe | ✅ |
| Activity 5.01 | Deploying a Full Application Pod | ✅ |

---

## Chapter Summary

In this chapter we explored various components of pod configuration
and learned when to use what. We can now:

- Create a pod and choose right values for various fields
- Use namespaces to organize and isolate workloads
- Configure resource requests and limits for proper scheduling
- Implement liveness probes for self-healing
- Implement readiness probes for traffic gating
- Use sidecar pattern for multi-container pods
- Diagnose pod issues using kubectl describe Events section

In the next chapter (Chapter 6), we will learn how to add labels and
annotations to pods and use them to identify, search, and organize pods.
