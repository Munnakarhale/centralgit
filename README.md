KUBERNETES HPA & VPA CONCEPTS
SCALING: Adjusting the resources based on varying load on application.
Pods can be scaled in following 2 ways in K8s:
1) Horizontal Scaling: Increasing or Decreasing the number of pods or
number of VMs based on application load.
• Manual Scaling: By changing replicas count in deployment.yaml
• Automatic Scaling: By using Horizontal pods Autoscaler

Horizontal Scaling
2) Vertical Scaling: Increasing or decreasing the resources(CPU,
Memory etc) of the same pod or VM based on traffic to application.
• Manual Scaling: By changing container requests and limits in
deployment.yaml
• Automatic Scaling: By using Vertical pods Autoscaler

 Vertical Scaling
1) Horizontal Pods Autoscaler(HPA): It is a Kubernetes object that
automatically scales out(increase) or sclaes in(decrease) the number of
Haseeb Ullah
pods based on Resource usage or Custom metrics. It runs as a control
loop, checking metrics every 15 seconds.

 Flow with Default Resource Metrics (CPU & Memory)
Note: Metric Server is the pre-requisite for HPA
Practical:
Deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
            limits:
              cpu: "200m"




Service .yaml:
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
"
Apply;
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-Service .yaml
Verify;

Lets create a HPA object:
kubectl autoscale deployment nginx-deploy --
cpu=’50%’ --min=1 --max=5
It states that at least 1 replica must run, and if CPU usage exceeds 50% of
the request, scale up to a maximum of 5 replicas.
Haseeb Ullah
Lets apply some CPU load on the pods using below command from 2
different tabs:
kubectl run -it --rm load-generator-1 --
image=busybox -- /bin/sh -c "while true; do
wget -q -O- http://nginx-svc; done"
Behavior:
• As load increases, CPU utilization rises.
• HPA will scale from 1 pod up to max 5 if needed.
• Once load decreases (stop load-generator), HPA scales back down.
CPU Utilization:
Scaling out of Replicas:
Kubectl Events:
Haseeb Ullah
Applied Load generator-2;
kubectl run -it --rm load-generator-2 --
image=busybox -- /bin/sh -c "while true; do
wget -q -O- http://nginx-s
vc; done"
CPU Utilization:
Scaling out of Replicas:
Kubectl Events:
Kubectl describe hpa nginx-deploy
Haseeb Ullah
Lets Terminate Load generator-2 and observe scaling in behaviour;
CPU Utilization:
Scaling in of Replicas:
Kubectl Events:
Conclusion:
HPA automatically scales out and scales in the number of replicas on the
cluster based on load/traffic received by the application.
2) Vertical Pods Autoscaler(VPA):
It automatically adjust the CPU and memory requests and limits of pods
based on their actual usage.
• It increases or decreases the CPU & memory reservations of pods.
• Pods always have enough resources to perform efficiently.
• Wasted resources are minimized, avoiding over-provisioning.
• Removes guesswork. No need to manually tune CPU/memory
requests.
Haseeb Ullah
Prequisite: VPA recommender, Updater, Admission controller and metric
Server.
Practical:
deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
 name: nginx-deploy
spec:
 replicas: 2
 selector:
 matchLabels:
 app: nginx
 template:
 metadata:
 labels:
 app: nginx
 spec:
 containers:
 - name: nginx
 image: nginx
 resources:
 requests:
 cpu: "100m"
 limits:
 cpu: "200m"
 ports:
 - containerPort: 80
Service.yaml:
apiVersion: v1
kind: Service
metadata:
 name: nginx-svc
spec:
 selector:
Haseeb Ullah
 app: nginx
 ports:
 - protocol: TCP
 port: 80
 targetPort: 80
 type: ClusterIP
vpa.yaml:
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
 name: nginx-vpa
spec:
 targetRef:
 apiVersion: "apps/v1"
 kind: Deployment
 name: nginx-deploy
 updatePolicy:
 # updateMode options:
 # "Off" - VPA only recommends
resources, does NOT apply them.
 # "Initial" - VPA sets recommended
resources at pod creation, no changes after.
 # "Auto" - VPA automatically updates
resources and restarts pods as needed.
 updateMode: "Auto"
Apply;
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-Service .yaml
Verify;
Haseeb Ullah
We have set this CPU request and limit in yaml manifest:

Lets create VPA and observe its behaviour;
kubectl apply -f vpa.yaml
Verify;
The moment we have created VPA; This event took place:
Now the CPU requests and limits are adjusted to following values by VPA.

Lets apply some CPU load on the pods using below command from 2
different tabs:
kubectl run -it --rm load-generator-1 --
image=busybox -- /bin/sh -c "while true; do
wget -q -O- http://nginx-svc; done"
Now Because of increasing CPU load on pods, VPA has set new CPU
request, limits values to pods:

Haseeb Ullah
Kubectl Events:
The VPA evicted one of the running pods, updated the CPU requests and
limits, and then the ReplicaSet controller created new pods with these
updated resource values.
Kubecetl get pods
Now the CPU request, Limits are set as below in each pods:

Conclusion: VPA dynamically adjusts the pod’s resource requests and
limits based on the application’s varying load, while keeping the number
of pods unchanged.
