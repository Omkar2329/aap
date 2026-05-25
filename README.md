## *KEDA with Cron Scaler on VKS*

### Introduction

KEDA Cron scaler enables time-based autoscaling for Kubernetes workloads running on VKS. Unlike traditional CPU or memory autoscaling, the Cron scaler uses scheduled time windows to scale workloads predictably during business hours, testing windows, maintenance periods, or SLA-driven operational schedules.

This approach is particularly valuable in enterprise on-premises environments where cluster resources are finite and idle workloads consume shared infrastructure capacity.

This validation demonstrates:

- KEDA installation and CRD validation
- Cron trigger activation and deactivation
- Automatic HPA creation by KEDA
- Replica scaling for a sample workload
- Scale-to-zero capability on VKS
  
### Why Cron Scaler on VKS?

Enterprise workloads often require predictable scaling patterns rather than reactive autoscaling based only on CPU or memory. Development environments, demo platforms, batch-processing services, and office-hour applications do not require continuous runtime capacity.

KEDA Cron scaler solves this by:

- Scaling workloads only during active time windows
- Eliminating idle pod resource consumption
- Supporting declarative scheduling via Kubernetes manifests
- Providing GitOps-friendly automation
- Enabling scale-to-zero outside operational windows

For VKS environments running on finite infrastructure, this directly improves cluster utilization and prevents unnecessary resource contention between workloads.

### What is KEDA Cron Scaler?

The KEDA Cron scaler is an event-driven scaler that adjusts Kubernetes workload replicas according to cron-based schedules.

KEDA continuously reconciles a ScaledObject resource and creates a native Kubernetes HPA behind the scenes. During the configured schedule window:

- Desired replicas are applied
- HPA reconciles deployment scaling
- Workloads scale automatically

When the schedule window ends:

- Cooldown logic activates
- Replicas scale back down
- Idle workloads can return to zero pods

This provides predictable and repeatable autoscaling behavior without relying on workload metrics.

### Benefits of Cron Scaler
- Predictable scaling windows
- Zero idle resource consumption outside active schedules
- Declarative Kubernetes-native configuration
- Simple GitOps integration
- Lightweight operational overhead
- Enterprise-ready automation model
- Works in connected and air-gapped environments
- How Scaling Works Internally

The following flow explains how KEDA Cron scaling operates internally inside the VKS workload cluster:
```
KEDA Operator
      ↓
ScaledObject (Cron trigger)
      ↓
Active window starts
      ↓
Desired replicas applied
      ↓
KEDA creates/reconciles HPA
      ↓
Deployment scales up
      ↓
Window ends + cooldown
      ↓
Deployment scales down to 0
```
### VKS Platform with KEDA Cron Scaler Architecture

The diagram below represents how KEDA Cron scaler integrates into the VKS platform architecture.

Supervisor Cluster manages workload cluster lifecycle
KEDA runs inside workload clusters
Cron schedules are evaluated by KEDA Operator
HPA objects are automatically created
Deployments scale dynamically during active schedules
```
+-----------------------------------------------------------+
|                 VMware Cloud Foundation                   |
|                                                           |
|  +---------------------------------------------------+    |
|  |                Supervisor Cluster                 |    |
|  |                                                   |    |
|  |  Namespace Service / Workload Management          |    |
|  +------------------------+--------------------------+    |
|                           |                               |
|                           v                               |
|  +---------------------------------------------------+    |
|  |               VKS Workload Cluster                |    |
|  |                                                   |    |
|  |  +---------------------------------------------+  |    |
|  |  |               KEDA Operator                 |  |    |
|  |  +-------------------+-------------------------+  |    |
|  |                      |                            |    |
|  |                      v                            |    |
|  |  +---------------------------------------------+  |    |
|  |  |          ScaledObject (Cron Trigger)        |  |    |
|  |  +-------------------+-------------------------+  |    |
|  |                      |                            |    |
|  |                      v                            |    |
|  |  +---------------------------------------------+  |    |
|  |  |              Kubernetes HPA                 |  |    |
|  |  +-------------------+-------------------------+  |    |
|  |                      |                            |    |
|  |                      v                            |    |
|  |  +---------------------------------------------+  |    |
|  |  |         Sample Deployment / Pods            |  |    |
|  |  +---------------------------------------------+  |    |
|  |                                                   |    |
|  +---------------------------------------------------+    |
|                                                           |
+-----------------------------------------------------------+
```
### Setup
### Pre-Requisites
| Component |	Version |
|---|---|
|VMware Cloud Foundation (VCF)|	9.0.2|
|vSphere Kubernetes Service (VKS)|	3.6.x|
|KEDA	|2.19|
|Kubernetes	|v1.30+|
|Helm	|3.14.0|
|Harbor (Optional for Airgap)|	2.14.3|
|Istio|	1.28.5|
|Cert Manager|	1.19.4|

### Deployment Manifests

### Namespace Manifest
```
apiVersion: v1
kind: Namespace
metadata:
  name: keda-cron-demo
  labels:
    app.kubernetes.io/name: keda-cron-demo
    app.kubernetes.io/part-of: keda-on-vks
    app.kubernetes.io/component: cron-scaler-demo
    app.kubernetes.io/managed-by: kubectl
    sunfire.io/validation-scope: keda-cron-scaler
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
```

### Deployment Manifest
```
cat << EOF   > "nginx-deploment.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  selector:
    matchLabels:
      app: nginx                        # Label used by the selector and service
  replicas: 2                           # Specifies the desired number of pods
  template:
    metadata:
      labels:
        app: nginx                      # Label used by the selector and service
    spec:
      containers:
      - name: nginx
        image: harbor.sunfire.lab/library/nginx:latest      # The NGINX container image
        ports:
        - containerPort: 80             # The port the container exposes
EOF
```
```
kubectl apply  -f "nginx-deploment.yaml"
```

### ScaledObject (Cron Trigger) Manifest
```
cat << EOF   > "nginx-cron-scaledobject.yaml"
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-cron-scaledobject
  namespace: nginx
spec:
  scaleTargetRef:
    name: nginx
  minReplicaCount: 1                    # Minimum pods during "off" hours
  maxReplicaCount: 5                    # Maximum pods KEDA can scale to
  triggers:
  - type: cron
    metadata:
    # TimeZone: UTC
      # timezone: UTC                   # mm hh dd MM DDD
      # start: 00 20 * * *              # Start at 09:00, Mon-Fri --> 1-5
      # end:   20 20 * * *              # End at 17:00, Mon-Fri
    # TimeZone: IST
      timezone: Asia/Kolkata
      start: 20 23 * * *                # Start at 09:00, Mon-Fri
      end:   25 23 * * *                # End at 17:00, Mon-Fri
      desiredReplicas: "3"
EOF
```
```
kubectl apply  -f "${PROJ_PATH}/nginx-cron-scaledobject.yaml"
```
### Script Catalogue
### 00-prereq-check.sh
```
#!/bin/bash

kubectl get pods -n keda
kubectl get crd | grep keda
```

### 01-deploy.sh
```
#!/bin/bash

kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/nginx-deploment.yaml
kubectl apply -f manifests/nginx-cron-scaledobject.yaml
```
### 02-watch-scale.sh
```
#!/bin/bash

kubectl get deployment cron-demo-app -n keda-cron-demo -w
```

### 03-validate.sh
```
#!/bin/bash

kubectl get namespace keda-cron-demo --show-labels
kubectl get deployment nginx -n keda-cron-demo
kubectl get scaledobject nginx-cron-scaledobject -n keda-cron-demo
kubectl get hpa -n keda-cron-demo
```
#### Example Output:
```
NAME                      SCALETARGETKIND      SCALETARGETNAME   MIN   MAX   READY   ACTIVE   FALLBACK   PAUSED   TRIGGERS   AUTHENTICATIONS   AGE
nginx-cron-scaledobject   apps/v1.Deployment   nginx             1     5     True    True     False      False    cron                         168m
```
```
NAME                               REFERENCE          TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-nginx-cron-scaledobject   Deployment/nginx   1/1 (avg)   1         5         3          168m
```
### 04-cleanup.sh
```
#!/bin/bash

kubectl delete -f manifests/nginx-cron-scaledobject.yaml --ignore-not-found=true
kubectl delete -f manifests/nginx-deploment.yaml --ignore-not-found=true
kubectl delete -f manifests/00-namespace.yaml --ignore-not-found=true
```

### Validation Flow

|Step|Activity	|Command|	Success Criteria|
|---|---|---|---|
|1|	Check KEDA pods & CRDs|	./scripts/00-prereq-check.sh|	KEDA pods running and CRDs visible|
|2|	Deploy demo resources|	./scripts/01-deploy.sh|	Namespace, deployment and ScaledObject created|
|3|	Validate namespace labels|	kubectl get ns keda-cron-demo --show-labels|	Labels visible|
|4|	Validate HPA creation|	kubectl get hpa -n keda-cron-demo	|HPA created by KEDA|
|5|	Watch scaling events|	./scripts/02-watch-scale.sh	|Replicas scale from 0 → 3 → 0|
|6|	Inspect ScaledObject|	./scripts/03-validate.sh|	Ready=True and Active=True during window|
|7|	Cleanup environment|	./scripts/04-cleanup.sh|	Namespace and resources deleted|

## Expected Scaling Behaviour
### 1. Initial State
```
kubectl get deployment -n keda-cron-demo
```
#### Expected:
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
nginx            0/0     0            0           15s
```
### 2. During Active Cron Window
```
kubectl get deployment -n keda-cron-demo
```
#### Expected:
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
nginx            3/3     3            3           3m
```
### 3. After Cron Window Ends
```
kubectl get deployment  -n keda-cron-demo
```
#### Expected:
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
nginx            0/0     0            0           10m
```
## Troubleshooting
|Issue|	Possible Cause|	Resolution|
|---|---|---|
|Namespace labels missing|	Namespace manifest not applied properly|	Reapply 00-namespace.yaml|
|ScaledObject not Ready	|Invalid target reference or CRD issue|	Validate YAML and deployment name|
|No scale-out event|	Incorrect cron expression or timezone	|Verify cron schedule|
|HPA not created	|KEDA operator issue|	Inspect KEDA operator logs|
|Scale-down delayed	|Cooldown period active|	Wait for cooldown expiry|
|Pods stuck Pending|	Resource constraints	|Check worker node capacity|

### Validation Commands
#### Verify KEDA Components
```
kubectl get pods -n keda
```
#### Verify ScaledObject
```
kubectl get scaledobject -n keda-cron-demo
```
#### Verify HPA
```
kubectl get hpa -n keda-cron-demo
```
#### Describe ScaledObject
```
kubectl describe scaledobject cron-demo-app -n keda-cron-demo
```
### Complete Workflow
```
Deploy Namespace
        ↓
Deploy Application
        ↓
Create ScaledObject
        ↓
KEDA reconciles Cron trigger
        ↓
HPA automatically created
        ↓
Cron schedule activates
        ↓
Deployment scales from 0 → 3
        ↓
Cron window ends
        ↓
Cooldown period starts
        ↓
Deployment scales from 3 → 0
```
## Conclusion

KEDA Cron scaler provides predictable, lightweight, and enterprise-friendly autoscaling for VKS environments. By enabling schedule-based scaling and scale-to-zero capability, platform teams can significantly improve cluster efficiency while maintaining declarative Kubernetes-native operations.

#### This validation confirms that:
- KEDA integrates cleanly with VKS
- Cron triggers function correctly
- HPAs are automatically managed
Scale-to-zero works reliably
Scheduled autoscaling reduces idle resource consumption on shared infrastructure clusters
