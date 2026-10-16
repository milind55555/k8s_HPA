# Kubernetes HPA Demo on Minikube

A hands-on demo of the **Horizontal Pod Autoscaler (HPA)** in Kubernetes. It runs on a local Minikube cluster, generates CPU load on a sample app, and shows pods scaling **up** under load and **down** when load stops.

## Architecture

```
 load-generator pod  --HTTP-->  Service (php-apache)  -->  Deployment (1..10 pods)
                                                                  ^
 metrics-server --> CPU metrics --> HPA controller (target 50%) --+
```

## Repository Structure

```
.
├── README.md
├── php-apache.yaml       # Deployment (with CPU requests/limits) + Service
├── hpa.yaml              # HorizontalPodAutoscaler (autoscaling/v2)
└── load-generator.yaml   # Pod that sends continuous requests to create CPU load
```

## How HPA Works

- **metrics-server** collects pod CPU/memory usage.
- The HPA controller checks metrics every ~15 seconds.
- Formula: `desiredReplicas = ceil(currentReplicas x currentCPU / targetCPU)`
- CPU utilization is measured as a **percentage of the pod's CPU request**, so `resources.requests.cpu` is mandatory.
- Scale-up is fast; scale-down waits for a stabilization window (default 300s, set to 60s here).

## Prerequisites

- Docker (or another Minikube driver)
- kubectl
- Minikube
- At least 2 CPUs and 4 GB RAM free

## Quick Start

### 1. Start Minikube and enable metrics-server

```bash
minikube start --driver=docker --cpus=2 --memory=4096
minikube addons enable metrics-server
kubectl top nodes        # wait 1-2 min until values appear
```

### 2. Deploy the app and the HPA

```bash
kubectl apply -f php-apache.yaml
kubectl apply -f hpa.yaml
kubectl get hpa
```

Expected initially:

```
NAME         REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS
php-apache   Deployment/php-apache   cpu: 0%/50%   1         10        1
```

If TARGETS shows `<unknown>`, wait 30-60 seconds.

### 3. Watch (use separate terminals)

```bash
kubectl get hpa php-apache --watch
kubectl get pods -l app=php-apache --watch
kubectl top pods -l app=php-apache      # run repeatedly
```

### 4. Generate CPU load

```bash
kubectl apply -f load-generator.yaml
```

Or as a one-liner (stop with Ctrl+C):

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -it --rm -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

For heavier load, run 2-3 generators (`load-generator-1`, `load-generator-2`, ...).

### 5. Observe scale-up

Within 1-3 minutes CPU goes above the 50% target and replicas increase:

```
php-apache   Deployment/php-apache   cpu: 250%/50%   1   10   1
php-apache   Deployment/php-apache   cpu: 250%/50%   1   10   4
php-apache   Deployment/php-apache   cpu: 55%/50%    1   10   8
```

Inspect the scaling decisions:

```bash
kubectl describe hpa php-apache
kubectl get deployment php-apache
```

### 6. Stop the load and observe scale-down

```bash
kubectl delete pod load-generator --ignore-not-found
```

CPU drops to ~0% and after the stabilization window replicas return to 1.

## Verification Checklist

| # | Check | Command | Expected |
|---|-------|---------|----------|
| 1 | Cluster running | `kubectl get nodes` | Node Ready |
| 2 | Metrics available | `kubectl top nodes` | CPU/memory shown |
| 3 | App deployed | `kubectl get pods -l app=php-apache` | 1 pod Running |
| 4 | HPA created | `kubectl get hpa` | TARGETS shows a % |
| 5 | Under load | `kubectl top pods` | CPU above 100m per pod |
| 6 | Scaled up | `kubectl get hpa` | REPLICAS > 1 |
| 7 | Load removed | delete load-generator | CPU near 0% |
| 8 | Scaled down | `kubectl get hpa` | REPLICAS back to 1 |

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| TARGETS `<unknown>/50%` | Metrics not ready or no CPU request | Wait 1-2 min; confirm `requests.cpu` is set |
| `Metrics API not available` | metrics-server not running | `minikube addons enable metrics-server`; check `kubectl get pods -n kube-system` |
| HPA not scaling up | Load too low | Run more load generators |
| Pods stuck Pending | Not enough Minikube resources | Start Minikube with more CPU/memory or lower `maxReplicas` |
| Load pod cannot resolve `php-apache` | Wrong namespace | Run it in the same namespace as the Service |
| Slow scale-down | Default 300s stabilization window | Wait, or keep the `behavior.scaleDown` block in `hpa.yaml` |

Useful debug commands:

```bash
kubectl describe hpa php-apache
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs -n kube-system deploy/metrics-server
```

## Cleanup

```bash
kubectl delete -f load-generator.yaml --ignore-not-found
kubectl delete -f hpa.yaml -f php-apache.yaml
minikube stop      # or: minikube delete
```

## Key Takeaways

- HPA scales pod replicas; VPA resizes pods; Cluster Autoscaler scales nodes.
- HPA requires metrics-server and CPU requests on pods.
- Utilization is relative to the request, not the limit.
- `autoscaling/v2` also supports memory, custom, and external metrics.
- Let HPA own the replica count; do not hard-code `replicas` in a pipeline that re-applies the Deployment.

## References

- [Kubernetes HPA docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Minikube docs](https://minikube.sigs.k8s.io/docs/)
