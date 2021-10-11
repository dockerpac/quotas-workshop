# Kubernetes resource quotas workshop

## Check current Quotas

First let's check the quotas currently enforced in your Namespace

```sh
# ResourceQuota
kubectl describe resourcequota

Name:                   quota
Namespace:              default
Resource                Used  Hard
--------                ----  ----
limits.cpu              0     20
limits.memory           0     3Gi
persistentvolumeclaims  0     100
pods                    0     250
requests.cpu            0     10
requests.memory         0     3Gi
services.loadbalancers  0     0
services.nodeports      0     5

# LimitRange
kubectl describe limitrange

Name:       limitrange
Namespace:  default
Type        Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Pod         cpu       10m    2    -                -              10
Pod         memory    100Mi  8Gi  -                -              1
```

## Deployment without Request / Limit

Create a Deployment without any Resource.

```sh
kubectl apply -f 1-noquota.yaml
```

Deployment is created but Pods are NOT created !

```sh
# Deployment is created
kubectl get deploy
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
my-app   0/1     0            0           6s

# No Pod in Namespace
kubectl get pods
No resources found in default namespace.

# Check latest event
kubectl get events --sort-by='.lastTimestamp' | tail -1

Warning  FailedCreate  46s (x8 over 6m11s)  replicaset-controller  (combined from similar events): Error creating: pods "my-app-f5d98dc86-97l27" is forbidden: [minimum memory usage per Pod is 100Mi.  No request is specified, minimum cpu usage per Pod is 10m.  No request is specified, maximum cpu usage per Pod is 2.  No limit is specified, maximum memory usage per Pod is 8Gi.  No limit is specified, cpu max limit to request ratio per Pod is 10, but no request is specified or request is 0, memory max limit to request ratio per Pod is 1, but no request is specified or request is 0]
```

Now update the Deployment and add correct Request and Limits :

```yaml
  resources:
    limits: 
      cpu: "200m"
      memory: "600Mi"
    requests:
      cpu: "20m"
      memory: "600Mi"
```

```sh
kubectl apply -f 2-quota.yaml
deployment.apps/my-app configured

# Now Pods are created
kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
my-app-85b999b8c6-tt7wb   1/1     Running   0          25s
```

```sh
# Cleanup
kubectl delete -f 2-quota.yaml
```

## Deployment with wrong CPU Quota

Deploy an application that sets Request / Limits that doesn't respect the Cpu LimitRange Ratio

```sh
kubectl apply -f 3-wrongquotacpu.yaml
deployment.apps/wrong-quota-cpu created

# Pods are not created
kubectl get pods
No resources found in default namespace.

# Check events
kubectl get events --sort-by='.lastTimestamp' | tail -1

3s          Warning   FailedCreate              replicaset/my-app-6fb6f4d6f6                      (combined from similar events): Error creating: pods "my-app-6fb6f4d6f6-hm6jx" is forbidden: cpu max limit to request ratio per Pod is 10, but provided ratio is 20.000000
```

> :warning: The Ratio between Limits and Requests for CPU must be < 10 (ie : \<container cpu limit\> / \<container cpu request\> < 10)  

- Example 1  
If you want to set a CPU limit of 200m, you must set at least a CPU request of 20m
- Example 2  
If you want to set a CPU limit of 1vCPU (1000m), you must set at least a CPU request of 100m

Review the current range enforced in your Namespace. Limit/Request Ratio for CPU is 10.

```sh
kubectl describe limitrange

Namespace:  default
Type        Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Pod         cpu       10m    2    -                -              10
Pod         memory    100Mi  8Gi  -                -              1
```

```sh
# Cleanup
kubectl delete -f 3-wrongquotacpu.yaml
```

## Deployment with wrong MEMORY Quota

Deploy an application that sets Request / Limits that doesn't respect the Memory LimitRange Ratio

```sh
kubectl apply -f 4-wrongquotamemory.yaml
deployment.apps/wrong-quota-memory created

# Pods are not created
kubectl get pods
No resources found in default namespace.

# Check events
kubectl get events --sort-by='.lastTimestamp' | tail -1

3s          Warning   FailedCreate              replicaset/my-app-686cbfc57                       (combined from similar events): Error creating: pods "my-app-686cbfc57-ww7wb" is forbidden: memory max limit to request ratio per Pod is 1, but provided ratio is 6.000000
```

> :warning: The Ratio between Limits and Requests for Memory must be = 1 (ie : \<container memory limit\> = \<container memory request\>)  

- Example  
If you want to set a Memory Limit of 1Gi, you must set a Memory Request of 1Gi

```sh
# Cleanup
kubectl delete -f 4-wrongquotamemory.yaml
```

## Deployment when Quota of Namespace is full

First let's fill 80% of the Namespace quota with a dummy application

```sh
kubectl apply -f 5-fillquota.yaml
deployment.apps/fill-quota created
```

Now deploy your application with 2 replicas

```sh
kubectl apply -f 6-outofquota.yaml
deployment.apps/out-of-quota created

# Only 1 Pod is created
kubectl get pods | grep out-of-quota
out-of-quota-7f568f8c78-vvm2h   1/1     Running   0          2s

# Check events
kubectl get events --sort-by='.lastTimestamp' | tail -1

37s         Warning   FailedCreate        replicaset/out-of-quota-7f568f8c78   (combined from similar events): Error creating: pods "out-of-quota-7f568f8c78-c6cgt" is forbidden: exceeded quota: quota, requested: limits.memory=800Mi,requests.memory=800Mi, used: limits.memory=2400Mi,requests.memory=2400Mi, limited: limits.memory=3Gi,requests.memory=3Gi
```

Check that the quota of the Namespace has been exceeded.

```sh
TODO : review display
kubectl describe resourcequota

Name:                   quota
Namespace:              default
Resource                Used    Hard
--------                ----    ----
limits.cpu              300m    20
limits.memory           2400Mi  64Gi
persistentvolumeclaims  0       100
pods                    3       250
requests.cpu            30m     10
requests.memory         2400Mi  64Gi
services.loadbalancers  0       0
services.nodeports      0       5
```

Try to lower your Memory Request and Limit.

```sh
# Lower the memory Request and Limit
kubectl apply -f 7-outofquota-reviewed.yaml

# --> All Pods are now created
kubectl get pods | grep out-of-quota
out-of-quota-6d9c8b764-n9qjz   1/1     Running   0          6s
out-of-quota-6d9c8b764-rv5gq   1/1     Running   0          10s

# Cleanup
kubectl delete -f 5-fillquota.yaml -f 7-outofquota-reviewed.yaml
```

## What happens when the Limit is reached for my container ?

### CPU : throttling

### MEMORY : OutOfMemoryKill

## Pending Pods
Show current usage in cluster

## Appli impactée par une autre ?
Impact de la request


## Quota pendant rolling update
Terminating --> quota ?


## How to check your quotas ?
- Sysdig
- kube-capacity