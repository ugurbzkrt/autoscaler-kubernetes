# How to set up Autoscaling and Metric Server in Kubernetes?

- Assuming you have a cluster, I would like to proceed with the steps only.

- Firstly, in order to use HPA (Horizontal Pod Autoscaler), we need to install Metric Server, which is necessary for viewing the relevant metrics.

- Example deployment and service files;

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 500Mi
            cpu: 100m
          requests:
            memory: 250Mi
            cpu: 80m

---

apiVersion: v1
kind: Service
metadata:
  name: php-apache-service
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
    nodePort: 30002
  selector:
    run: php-apache 
  type: NodePort	
```

- Deploy this file.

```
kubectl apply -f php-apache.yaml
```

- Get the pods and service.

```
kubectl get po

kubectl get svc
```

- On opening browser (http://:) we see

```
OK!
```

- Alternatively, you can use;

```
curl <public-worker node-ip>:<node-port>
OK!
```

- Do not forget to open the Port in the security group or Firewall of your node instance.

Note: You can add multiple simple applications. The name of the newly added one can be 'web-deployment'.

# AutoScaling

## Benefits of Autoscaling


```
To understand better where autoscaling would provide the most value, let’s start with an example. Imagine you have a 24/7 production service with a load that is variable in time, where it is very busy during the day in the US, and relatively low at night. Ideally, we would want the number of nodes in the cluster and the number of pods in deployment to dynamically adjust to the load to
meet end user demand. The new Cluster Autoscaling feature together with Horizontal Pod Autoscaler can handle this for you automatically.
```

- Add watch board to verify the latest status of Cluster by below Commands.(This is Optional as not impacting the Functionality of Cluster). Observe in a separate terminal.

```
$ kubectl get service,hpa,pod -o wide
$ watch -n1 !!
```

## Create Horizontal Pod Autoscaler

- Now that the server is running, we will create the autoscaler using kubectl autoscale. The following command will create a Horizontal Pod Autoscaler that maintains between 2 and 10 replicas of the Pods controlled by the php-apache deployment we created in the first step of these instructions. Roughly speaking, HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 50% (since each pod requests 200 milli-cores by kubectl run), this means average CPU usage of 100 milli-cores). See here for more details on the algorithm.

- Now activate the HPAs;

```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=2 --max=10 
kubectl autoscale deployment web-deployment --cpu-percent=50 --min=3 --max=5 
```

- or we can use yaml files.

```
pwd
/home/ubuntu/microservices
mkdir auto-scaling && cd auto-scaling
cat << EOF > hpa-php-apache.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50

EOF
```
```
$ cat << EOF > hpa-web.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: web-deployment
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 3
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50 
EOF
```

```
$ kubectl apply -f hpa-php-apache.yaml
$ kubectl apply -f hpa-web.yaml
```

Let's look at the status;

```
watch -n3 kubectl get service,hpa,pod -o wide
```


- php-apache Pod number increased to 2, minimum number.
- web-deployment Pod number increased to 3, minimum number.
- The HPA line under TARGETS shows <unknown>/50%. The unknown means the HPA can't idendify the current use of CPU.

We may check the current status of autoscaler by running:

```
kubectl get hpa
```

```
$ kubectl describe hpa
```
- The metrics can't be calculated. So, the metrics server should be uploaded to the cluster.

# Install Metric Server

- First Delete the existing Metric Server if any.

```
$ kubectl delete -n kube-system deployments.apps metrics-server
```

- Get the Metric Server form GitHub.

```
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

- Edit the file components.yaml. You will select the Deployment part in the file. Add the below line to containers.args field under the deployment object.

```
        - --kubelet-insecure-tls
```

```
apiVersion: apps/v1
kind: Deployment
......
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-insecure-tls    # <------ Here.
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
......	
```
- Add metrics-server to your Kubernetes instance.

```
$ kubectl apply -f components.yaml
```

- Wait 1-2 minute or so.

- Verify the existace of metrics-server run by below command

```
$ kubectl -n kube-system get pods
```

- Verify metrics-server can access resources of the pods and nodes.

```
$ kubectl top pods
```

- Look at the the values under TARGETS. The values are changed from <unknown>/50% to 1%/50% & 2%/50%, means the HPA can now idendify the current use of CPU.

- If it is still <unknown>/50%, check the spec.template.spec.containers.resources.request field of deployment.yaml files. It is required to define this field. Otherwise, the autoscaler will not take any action for that metric.

> For per-pod resource metrics (like CPU), the controller fetches the metrics from the resource metrics API for each Pod targeted by the HorizontalPodAutoscaler. Then, if a target utilization value is set, the controller calculates the utilization value as a percentage of the equivalent resource request on the containers in each Pod.
 
> Please note that if some of the Pod's containers do not have the relevant resource request set, CPU utilization for the Pod will not be defined and the autoscaler will not take any action for that metric.

# Increase load

- Now, we will see how the autoscaler reacts to increased load. We will start a container, and send an infinite loop of queries to the php-apache service (please run it in a different terminal):

- First look at the services.

```
kubectl get svc
```

```
kubectl run -it --rm load-generator --image=busybox /bin/sh  

Hit enter for command prompt

while true; do wget -q -O- http://<puplic ip>:<port number of php-apache-service>; done 
```


Within a minute or so, we should see the higher CPU load by executing:

- Open new terminal and check the hpa.

```
$ kubectl get hpa 
```

On the watch board:

```
watch -n3 kubectl get service,hpa,pod -o wide
```


- Now, let's introduce load for to-do web app with load-generator pod as follows (in a couple of terminals):

```
$ kubectl exec -it load-generator -- /bin/sh
/ # while true; do wget -q -O- http://<puplic ip>:<port number of web-service> > /dev/null; done
```

Watch table

```
$ watch -n3 kubectl get service,hpa,pod -o wide

Every 3.0s: kubectl get service,hpa,pod -o wide  
```

Stop load

- We will finish our example by stopping the user load.

- In the terminal where we created the container with busybox image, terminate the load generation by typing Ctrl + C. Close the load introducing terminals grafecully and observe the behaviour at the watch board.

- Then we will verify the result state (after a minute or so):


Additionally, if you want to visually monitor resources and real-time communication within the pod architecture, you can use Weave Scope, which offers a simple installation process and is free of charge.

- https://www.weave.works/docs/scope/latest/installing/
- https://github.com/wave-k8s/wave


## In the next step, I plan to share how to manage scaling with DDoS attacks. Enjoy reading.












