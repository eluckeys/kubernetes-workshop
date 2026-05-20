# Exercise 5.01 — Creating a Pod with a Single Container

**Chapter:** 5 — Pods
**Book:** The Kubernetes Workshop, Packt (2019)
**Cluster:** K3s v1.35.4+k3s1 (3-node, Rocky Linux 9.7)
**Date:** May 2026

---

## What This Exercise Does

Creates the simplest possible pod — one container running nginx.
This is the foundation exercise. Every Kubernetes concept in the
remaining chapters builds on what you learn here.

---

## YAML File

single-container-pod.yaml

---

## Commands Used

```bash
# Create the pod
kubectl create -f single-container-pod.yaml

# Watch status change live
kubectl get pods --watch

# Full details + Events timeline
kubectl describe pod first-pod

# Shell inside the running container
kubectl exec -it first-pod -- /bin/sh

# View container logs
kubectl logs first-pod

# Delete the pod
kubectl delete pod first-pod
```

---

## What I Observed

| Field | Value |
|-------|-------|
| Node selected by scheduler | k8s-worker__ |
| Pod IP assigned | 10.42.x.x |
| Time to Running | ~__ seconds |
| Image pull time | ~__ seconds |

Fill in the above values from your kubectl describe output.

### STATUS Progression
```
Pending → ContainerCreating → Running
```

### Events Section from kubectl describe
```
Scheduled → Successfully assigned default/first-pod to k8s-worker__
Pulling   → Pulling image "nginx"
Pulled    → Successfully pulled image "nginx"
Created   → Created container my-first-container
Started   → Started container my-first-container
```

### Inside the Container (kubectl exec)
- nginx process confirmed running via ps aux
- nginx served default HTML page via wget localhost

---

## Key Learnings

**Four-key structure** — apiVersion, kind, metadata, spec applies to
every Kubernetes object. Learn it once, use it forever.

**Pod is ephemeral** — when deleted, nothing recreates it automatically.
This is exactly why Deployments exist (Chapter 7).

**Pod IP is temporary** — changes every time pod is recreated.
This is exactly why Services exist (Chapter 8).

**kubectl describe Events** — primary debugging tool. Always check
this first when something goes wrong with a pod.

**kubectl create returns immediately** — pod/first-pod created means
accepted into etcd, NOT that it is running. Always watch STATUS.

---

## Troubleshooting

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| Pod stuck in Pending | kubectl describe → Events → FailedScheduling | Check node resources |
| CrashLoopBackOff | kubectl logs first-pod | Check container error output |
| ImagePullBackOff | kubectl describe → Events | Check image name spelling |
| ErrImagePull | kubectl describe → Events | Check Docker Hub connectivity |

---

## 2026 vs Book Difference

Book uses Minikube — shows only one node (minikube/10.0.2.15).
Our cluster shows k8s-worker1 OR k8s-worker2 in the Node field.
Real scheduling decision happening across two worker nodes.
This is authentic Kubernetes behavior Minikube cannot demonstrate.
