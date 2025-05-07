# How to build a PromQL (Prometheus Query Language)

This repository is here to guide you through the GitHub tutorial that goes hand-in-hand with a video available on YouTube and a detailed blog post on my website. 
Together, these resources are designed to give you a complete understanding of the topic.

Here are the links to the related assets:
- YouTube Video: [How to build a PromQL (Prometheus Query Language)](https://www.youtube.com/watch?v=hvACEDjHQZE)
- Blog Post: [How to build a PromQL (Prometheus Query Language](https://isitobservable.io/observability/prometheus/how-to-build-a-promql-prometheus-query-language)

Feel free to explore the materials, star the repository, and follow along at your own pace.


## PromQL tutorial
<p align="center"><img src="/image/k8sprom.png" width="40%" alt="Prometheus Logo" /></p>

This repository showcases the usage of Prometheus by using GKE with the HipsterShop


## Prerequisites
The following tools need to be installed on your machine :
- jq
- kubectl
- git
- gcloud (if you're using GKE)
- Helm
  
### 1. Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2. Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone the GitHub repo
```
git clone https://github.com/isItObservable/Episode-5---PromQL
cd Episode-5---PromQL
```
### 4. Deploy Prometheus and Grafana
#### HipsterShop
```
cd hipstershop
./setup.sh
```
#### Prometheus
```
helm install prometheus stable/prometheus-operator
```
#### Expose Grafana
```
kubectl get svc
kubectl edit svc prometheus-grafana
```
change to type NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  labels:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.0.3
    helm.sh/chart: grafana-5.3.0
  name: prometheus-grafana
  namespace: default
  resourceVersion: "89873265"
  selfLink: /api/v1/namespaces/default/services/prometheus-grafana
spec:
  clusterIP: IPADRESSS
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30806
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Deploy the ingress by making sure to replace the service name of your Grafana
```
cd ..\grafana
kubectl apply -f ingress.yaml
```
Get the login user and password of Grafana
* For the password :
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
* For the login user:
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode
```
Get the IP address of your Grafana
```
kubectl get ingress grafana-ingress -ojson | jq  '.status.loadBalancer.ingress[].ip'
```
### 5. Tutorial on PromQL
The tutorial will be using metrics exposed by the Kube state metric and node exporter.

#### Label Filtering
Let's focus on the metric kube_pod_status_phase 
Open Grafana, click Explore
Let's just look at the label exposed on this metric
<p align="center"><img src="/image/explore.PNG" width="80%" alt="PromLens Logo" /></p>

The labels exposed on this metric are :
* endpoint
* instance
* job
* namespace
* phase
* pod
* service

Let's try to count the number of failed pods per service in the hipster-shop namespace over the last 5 minutes

Let's start by using label filtering
```
kube_pod_status_phase{phase="Failed", namespace="hipster-shop"}
```

Now let's use the function to count the number of running pods per namespace
```
count(kube_pod_status_pahse{phase="Running"} ) by(namespace)
```

Now that we have a counter, we can use the rate to have the number of pods in failure/s over the last 10m
```
rate(sum(kube_pod_status_pahse{phase="Failed"} ) by(namespace) [10m])
```

#### Operation
Let's pick a Counter to utilize the rate function

Let's take the example of the metric: node_disk_io_time_seconds_total
```
rate(node_disk_io_time_seconds_total [5m]) 
```
Let's use a Gauge metric to use histogram_quantile
The metric that we could use is: etcd_lease_object_counts_bucket
```
histogram_quantile(0.95, sum(rate(etcd_lease_object_counts_bucket[5m])) by (le))
```

Count the number of objects created in our cluster
For that, we can use metrics exposed in etcd
```
sum(rate(etcd_object_counts [5m] ) ) by (resource)
```


#### Highlight of an interesting solution: PromLens  
If you're not comfortable building your PromQL, there is a solution that will help you design your PromQL:
[PromLens](https://promlens.com/)
<p align="center"><img src="/image/promlens.png" width="80%" alt="PromLens Logo" /></p>
