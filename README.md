# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

## PromQL tutorial
<p align="center"><img src="/image/k8sprom.png" width="40%" alt="Prometheus Logo" /></p>
Repository containing the files for the Episode 5 of Is it Observable : PromQL


This repository showcase the usage of the Prometheus  by using GKE with :
- the HipsterShop


## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud containr clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
```
git clone https://github.com/isItObservable/Episode-5---PromQL
cd Episode-5---PromQL
```
### 4. Deploy Prometheus & Grafana
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
Deploy the ingress by making sure to replace the service name of your grafan
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
Get the ip adress of your Grafana
```
kubectl get ingress grafana-ingress -ojson | jq  '.status.loadBalancer.ingress[].ip'
```
### 5. Tutorial on PromQL
The tutorial will be using metrics expose by the Kube state metric and node exporter.

#### Label Filtering
Let's focus on the metric kube_pod_status_phase 
Open Grafana, click on Explore
Let's just look on the label exposed on this metric
<p align="center"><img src="/image/explore.png" width="80%" alt="PromLens Logo" /></p>

The label exposed on this metric are :
* endpoint
* instance
* job
* namespace
* phase
* pod
* service

Let's try to to count the number of Failed pods per services in the hipster-shop namespace over the last 5minutes

let's start by using label filtering
```
kube_pod_status_phase{phase="Failed", namespace="hipster-shop"}
```

Now let's use the function to count the number of running pods per namespaces
```
count(kube_pod_status_pahse{phase="Running"} ) by(namespace)
```

Now that we have counter, we can use rate to have the number of pods in failure /s over the last 10m
```
rate(sum(kube_pod_status_pahse{phase="Failed"} ) by(namespace) [10m])
```

#### Operation
Let's pick a Counter to utilize the rate function

let's take the example of the metric : node_disk_io_time_seconds_total
```
rate(node_disk_io_time_seconds_total [5m]) 
```
let's use a Gauge metric to use histogram_quantile
The metric that we could use is : etcd_lease_object_counts_bucket
```
histogram_quantile(0.95, sum(rate(etcd_lease_object_counts_bucket[5m])) by (le))
```

count the number of objects created in our cluster
we can use for that metrics expose in etcd
```
sum(rate(etcd_object_counts [5m] ) ) by (resource)
```


#### Highlight of an interesting solution : PromLens  
If you are not confortable of building your PromQL, there is a solution that will help you designing your Promql :
[PromLens](https://promlens.com/)
<p align="center"><img src="/image/promlens.png" width="80%" alt="PromLens Logo" /></p>
