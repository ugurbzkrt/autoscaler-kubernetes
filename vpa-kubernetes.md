# Vertical Pod Autoscaling (VPA) in Kubernetes

In my previous post, I mentioned Horizontal Pod Autoscaling (HPA) as a scaling feature for Kubernetes. In this post, I will talk about expanding the resources of a pod based on incoming traffic, rather than increasing the number of pods.

https://github.com/ugurbzkrt/autoscaler-kubernetes/blob/main/hpa-kubernetes.md

## Vertical Pod Autoscaling


What is Vertical Pod Autoscaler?

Vertical Pod autoscaling (VPA) ensures that a container’s resources are not under- or over-utilized. It recommends optimized CPU and memory requests/limits values, and can also automatically update them for you so that the cluster resources are efficiently used.

![alt text](https://github.com/ugurbzkrt/autoscaler-kubernetes/blob/main/images/vertical.png)

VPA recommends optimized CPU and memory requests/limits values (and automatically updates them for you so that the cluster resources are efficiently used). VPA won’t add more replicas of a Pod, but it increases the memory or CPU limits. This is useful when adding more replicas won’t help your solution. For example, sometimes you can’t scale a database just by adding more Pods. Still, you can make the database handle more connections by increasing the memory or CPU. You can use the VPA when your application serves heavyweight requests, which requires higher resources.

HPA can be useful when, for example, your application serves a large number of lightweight (i.e., low resource-consuming) requests. In that case, scaling the number of replicas can distribute the workload on each pod. The VPA, on the other hand, can be useful when your application serves heavyweight requests, which require higher resources.

HPA and VPA are incompatible. Do not use both together for the same set of pods. HPA uses the resource request and limits to trigger scaling, and in the meantime, VPA modifies those limits, so it will be a mess unless you configure the HPA to use either custom or external metrics.

## Architecture

VPA consists of 3 components:

- VPA admission controller
  Once you deploy and enable the Vertical Pod Autoscaler in your cluster, every pod submitted to the cluster goes through this webhook, which checks whether a VPA object is referencing it.

- VPA recommender
  The recommender pulls the current and past resource consumption (CPU and memory) data for each container from metrics-server running in the cluster and provides optimal resource recommendations based on it, so that a container uses only what it needs.

- VPA updater
  The updater checks at regular intervals if a pod is running within the recommended range. Otherwise, it accepts it for update, and the pod is evicted by the VPA updater to apply resource recommendation.


## Installation

If you are on Google Cloud Platform, you can simply enable vertical-pod-autoscaling:‍

```
gcloud container clusters update <cluster-name> --enable-vertical-pod-autoscaling
```

To install it manually follow below steps:

- You can refer to my previous HPA article for instructions on manually installing the Metric Server.

```
kubectl get deployment metrics-server -n kube-system
```

- Also, verify the API below is enabled:

```
kubectl api-versions | grep admissionregistration
admissionregistration.k8s.io/v1beta1
```

- Clone the kubernetes/autoscaler GitHub repository, and then deploy the Vertical Pod Autoscaler with the following command.
- https://github.com/kubernetes-sigs/metrics-server

```
git clone https://github.com/kubernetes/autoscaler.git
./autoscaler/vertical-pod-autoscaler/hack/vpa-up.sh
```

Verify that the Vertical Pod Autoscaler pods are up and running:

```
kubectl get po -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
vpa-admission-controller-68c748777d-ppspd   1/1     Running   0          7s
vpa-recommender-6fc8c67d85-gljpl            1/1     Running   0          8s
vpa-updater-786b96955c-bgp9d                1/1     Running   0          8s

kubectl get crd
verticalpodautoscalers.autoscaling.k8s.io 
```

## VPA using Resource Metrics

A. Setup: Create a Deployment and VPA resource

Use the same deployment config to create a new deployment with "--vm-bytes", "850M". Then create a VPA resource in Recommendation Mode with updateMode : Off

```yml
apiVersion: autoscaling.k8s.io/v1beta2
kind: VerticalPodAutoscaler
metadata:
 name: autoscale-tester-recommender
spec:
 targetRef:
   apiVersion: "apps/v1"
   kind:       Deployment
   name:       autoscale-tester
 updatePolicy:
   updateMode: "Off"
 resourcePolicy:
   containerPolicies:
   - containerName: autoscale-tester
     minAllowed:
       cpu: "500m"
       memory: "500Mi"
     maxAllowed:
       cpu: "4"
       memory: "8Gi"
```

- minAllowed is an optional parameter that specifies the minimum CPU request and memory request allowed for the container. 
- maxAllowed is an optional parameter that specifies the maximum CPU request and memory request allowed for the container.

B. Check the Pod’s Resource Utilization

Check the resource utilization of the pods. Below, you can see only ~50 Mi memory is being used out of 1000Mi and only ~30m CPU out of 1000m. This clearly indicates that the pod resources are underutilized.

```
Kubectl top po
NAME                            	CPU(cores)   MEMORY(bytes)   
autoscale-tester-5d6b48d64f-8zgb9   39m      	51Mi       	 
autoscale-tester-5d6b48d64f-npts4   32m      	50Mi       	 
autoscale-tester-5d6b48d64f-vctx5   35m      	50Mi 
```

If you describe the VPA resource, you can see the Recommendations provided. (It may take some time to show them.)

```yml
kubectl describe vpa autoscale-tester-recommender
Name:     	autoscale-tester-recommender
Namespace:	autoscale-tester
...
  Recommendation:
        Container Recommendations:
  	Container Name:  autoscale-tester
  	Lower Bound:
    	Cpu: 	500m
    	Memory:  500Mi
  	Target:
    	Cpu: 	500m
    	Memory:  500Mi
  	Uncapped Target:
    	Cpu: 	93m
    	Memory:  262144k
  	Upper Bound:
    	Cpu: 	4
    	Memory:  4Gi
```

C. Understand the VPA recommendations

Target: The recommended CPU request and memory request for the container that will be applied to the pod by VPA.

Uncapped Target: The recommended CPU request and memory request for the container if you didn’t configure upper/lower limits in the VPA definition. These values will not be applied to the pod. They’re used only as a status indication.

Lower Bound: The minimum recommended CPU request and memory request for the container. There is a --pod-recommendation-min-memory-mb flag that determines the minimum amount of memory the recommender will set—it defaults to 250MiB.

Upper Bound: The maximum recommended CPU request and memory request for the container.  It helps the VPA updater avoid eviction of pods that are close to the recommended target values. Eventually, the Upper Bound is expected to reach close to target recommendation.

```yml
 Recommendation:
        Container Recommendations:
  	Container Name:  autoscale-tester
  	Lower Bound:
    	Cpu: 	500m
    	Memory:  500Mi
  	Target:
    	Cpu: 	500m
    	Memory:  500Mi
  	Uncapped Target:
    	Cpu: 	93m
    	Memory:  262144k
  	Upper Bound:
    	Cpu: 	500m
    	Memory:  1274858485 
```

D. VPA processing with Update Mode Off/Auto

Now, if you check the logs of vpa-updater, you can see it's not processing VPA objects as the Update Mode is set as Off.

```
kubectl logs -f vpa-updater-675d47464b-k7xbx
1 updater.go:135] skipping VPA object autoscale-tester-recommender because its mode is not "Recreate" or "Auto"
1 updater.go:151] no VPA objects to process
```

VPA allows various Update Modes, detailed: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#quick-start

Let's change the VPA updateMode to “Auto” to see the processing.

As soon as you do that, you can see vpa-updater has started processing objects, and it's terminating all 3 pods.

```
kubectl logs -f vpa-updater-675d47464b-k7xbx
1 update_priority_calculator.go:147] pod accepted for update autoscale-tester/autoscale-tester-5d6b48d64f-8zgb9 with priority 1
1 update_priority_calculator.go:147] pod accepted for update autoscale-tester/autoscale-tester-5d6b48d64f-npts4 with priority 1
1 update_priority_calculator.go:147] pod accepted for update autoscale-tester/autoscale-tester-5d6b48d64f-vctx5 with priority 1
1 updater.go:193] evicting pod autoscale-tester-5d6b48d64f-8zgb9
1 event.go:281] Event(v1.ObjectReference{Kind:"Pod", Namespace:"autoscale-tester", Name:"autoscale-tester-5d6b48d64f-8zgb9", UID:"ed8c54c7-a87a-4c39-a000-0e74245f18c6", APIVersion:"v1", ResourceVersion:"378376", FieldPath:""}): 
type: 'Normal' reason: 'EvictedByVPA' Pod was evicted by VPA Updater to apply resource recommendation.
```

You can also check the logs of vpa-admission-controller:

```
kubectl logs -f vpa-admission-controller-bbf4f4cc7-cb6pb
Sending patches: [{add /metadata/annotations map[]} {add /spec/containers/0/resources/requests/cpu 500m} {add /spec/containers/0/resources/requests/memory 500Mi} {add /spec/containers/0/resources/limits/cpu 500m} {add /spec/containers/0/resources/limits/memory 500Mi} {add /metadata/annotations/vpaUpdates Pod resources updated by autoscale-tester-recommender: container 0: cpu request, memory request, cpu limit, memory limit} {add /metadata/annotations/vpaObservedContainers autoscale-tester}]

```

NOTE: Ensure that you have more than 1 running replicas. Otherwise, the pods won’t be restarted, and vpa-updater will give you this warning:

```
1 pods_eviction_restriction.go:209] too few replicas for ReplicaSet autoscale-tester/autoscale-tester1-7698974f6. Found 1 live pods
```

Now, describe the new pods created and check that the resources match the Target recommendations:

```
kubectl get po
NAME                            	READY   STATUS    	RESTARTS   AGE
autoscale-tester-5d6b48d64f-5dlb7   1/1 	Running   	0      	77s
autoscale-tester-5d6b48d64f-9wq4w   1/1 	Running   	0      	37s
autoscale-tester-5d6b48d64f-qrlxn   1/1 	Running   	0      	17s


kubectl describe po autoscale-tester-5d6b48d64f-5dlb7
Name:     	autoscale-tester-5d6b48d64f-5dlb7
Namespace:	autoscale-tester
...
	Limits:
  	cpu: 	500m
  	memory:  500Mi
	Requests:
  	cpu:    	500m
  	memory: 	500Mi
	Environment:  <none>
```

The Target Recommendation can not go below the minAllowed defined in the VPA spec.


‍E. Stress Loading Pods

Let’s recreate the deployment with memory request and limit set to 2000Mi and "--vm-bytes", "500M".

Gradually stress load one of these pods to increase its memory utilization.
You can login to the pod and run stress --vm 1 --vm-bytes 1400M --timeout 120000s.

```
kubectl top po
NAME                            	CPU(cores)   MEMORY(bytes)   
autoscale-tester-5d6b48d64f-5dlb7   1000m     	1836Mi       	 
autoscale-tester-5d6b48d64f-9wq4w   252m      	501Mi       	 
autoscale-tester-5d6b48d64f-qrlxn   252m      	501Mi 	
```

You will notice that the VPA recommendation is also calculated accordingly and applied to all replicas.

```
kubectl describe vpa autoscale-tester-recommender
Name:     	autoscale-tester-recommender
Namespace:	autoscale-tester
...
  Recommendation:
	Container Recommendations:
  	Container Name:  autoscale-tester
  	Lower Bound:
    	Cpu: 	500m
    	Memory:  500Mi
  	Target:
    	Cpu: 	500m
    	Memory:  628694953
  	Uncapped Target:
    	Cpu: 	49m
    	Memory:  628694953
  	Upper Bound:
    	Cpu: 	500m
    	Memory:  1553712527
```

Limits v/s Request
VPA always works with the requests defined for a container and not the limits. So, the VPA recommendations are also applied to the container requests, and it maintains a limit to request ratio specified for all containers.

For example, if the initial container configuration defines a 100m Memory Request and 300m Memory Limit, then when the VPA target recommendation is 150m Memory, the container Memory Request will be updated to 150m and Memory Limit to 450m.


### In my next article, I plan to touch upon the topic of Cluster Auto Scaling.

## References

https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

https://www.cncf.io/blog/2023/02/24/optimizing-kubernetes-vertical-pod-autoscaler-responsiveness/

https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html

https://cloud.google.com/blog/products/containers-kubernetes/google-kubernetes-engine-clusters-can-have-up-to-15000-nodes

https://www.velotio.com/engineering-blog/autoscaling-in-kubernetes-using-hpa-vpa
