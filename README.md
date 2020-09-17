# Monitoring on EKS
Monitoring Lab:
Prometheus, Grafana and Loki implementation

## Prerequisites
EKS cluster to work on

## Introduction to Prometheus

### Overview
Prometheus is an open-source systems monitoring and alerting toolkit

### Components
The Prometheus ecosystem consists of multiple components, many of which are optional:

* prometheus server which scrapes and stores time series data
* client libraries for instrumenting application code
* push gateway for supporting short-lived jobs
* exporters for services like HAProxy, StatsD, Graphite, etc.
* alertmanager to handle alerts
* various support tools

### Client Libraries
Before you can monitor your services, you need to add instrumentation to their code via one of the Prometheus client libraries. These implement the Prometheus metric types.
This lets you define and expose internal metrics via an HTTP endpoint on your application’s instance:
* Go
* Java or Scala
* Python
* Ruby
There are other third party libraries.

## Exporters
We use exporters to instrument applications we don’t have the source code for
For that we install exporters next to the applications we want to collect data from, usually on the host server if possible.  

![Image of Yaktocat](img/prometheus-exporters.png)  


The exporter receive a data request from prometheus, it gathers and formats the data and send it back to Prometheus

The **node exporter** gets metrics from kubernetes clusters.  

## Service Discovery
Once we have exporters and API endpoints for prometheus there are two ways we can define them as targets for scraping:  
- Define them in prometheus configuration  
- Use service discovery  

We are gonna use service discovery to inform us when we have new pods and services in our cluster and those will show up as targets within prometheus.  
To monitor kubernetes we need to access the kubernetes API and other endpoints.  

![Image of Yaktocat](img/prometheus-service-discovery.png)  


## Scraping
Prometheus is a pull base system
Pull from HTTP endpoints
Scraping is a request made to the http endpoint - The response is parsed and ingested into storage


## Step-by-step guide

1.  Create a namespace for prometheus and other monitoring tools to live in.
https://github.com/driprado/monitoring/blob/master/prometheus/namespaces.yml


a) Apply
````
root@ip-10-0-1-4:~/prometheus# kubectl apply -f namespaces.yml
namespace/monitoring created
root@ip-10-0-1-4:~/prometheus#
````

b) Check
```
root@ip-10-0-1-4:~/prometheus# kubectl get namespace
NAME              STATUS   AGE
default           Active   165m
kube-node-lease   Active   166m
kube-public       Active   166m
kube-system       Active   166m
monitoring        Active   2m8s
root@ip-10-0-1-4:~/prometheus#
```

2.  Create prometheus configmap
https://github.com/driprado/monitoring/blob/master/prometheus/prometheus-config-map.yml

A ConfigMap is an API object used to store non-confidential data.  
Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.  
A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.  

a) Get one of your nodes internal IP address:  
`kubectl describe nodes <node-name>`

b) Add one of the pods internal IP address as static_configs: targets 
```scrape_configs:
      - job_name: 'node-exporter'
        static_configs:
        - targets: ['<node-ip>:9100']
```

c) Apply configmap
```
root@ip-10-0-1-4:~/prometheus# kubectl apply -f prometheus-config-map.yml
configmap/prometheus-server-conf created
root@ip-10-0-1-4:~/prometheus#
```

d) Check
```
root@ip-10-0-1-4:~/prometheus# kubectl get configmaps -n monitoring
NAME                     DATA   AGE
prometheus-server-conf   1      89s
root@ip-10-0-1-4:~/prometheus#
````


3.  Create prometheus deployment
https://github.com/driprado/monitoring/blob/master/prometheus/prometheus-deployment.yml

a) Apply  
```
root@ip-10-0-1-4:~/prometheus# kubectl apply -f prometheus-deployment.yml
deployment.apps/prometheus-deployment created
root@ip-10-0-1-4:~/prometheus
```
b) Confirm  
```
root@ip-10-0-1-4:~/prometheus# kubectl get deployments -n monitoring
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
prometheus-deployment   1/1     1            1           10m
root@ip-10-0-1-4:~/prometheus#

root@ip-10-0-1-4:~/prometheus# kubectl get pods -n monitoring
NAME                                    READY   STATUS    RESTARTS   AGE
prometheus-deployment-5bdddc4bf-q4pwm   2/2     Running   0          10m
root@ip-10-0-1-4:~/prometheus#
```


4.  Create prometheus service
https://github.com/driprado/monitoring/blob/master/prometheus/prometheus-service.yml


a) Apply
```
root@ip-10-0-1-4:~/prometheus# kubectl apply -f prometheus-service.yml
service/prometheus-service created
root@ip-10-0-1-4:~/prometheus#
````

b) Check
```
root@ip-10-0-1-4:~/prometheus# kubectl get services -n monitoring
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
prometheus-service   NodePort   172.20.17.230   <none>        32767:32767/TCP   2m42s
root@ip-10-0-1-4:~/prometheus#
````

*// Doesn’t work, troubleshooting:*
```
kubectl logs -n monitoring prometheus-deployment-5bdddc4bf-q4pwm prometheus
level=error ts=2020-09-15T05:03:50.510518612Z caller=main.go:216 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:270: Failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:monitoring:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope"
level=error ts=2020-09-15T05:03:50.510602322Z caller=main.go:216 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:268: Failed to list *v1.Endpoints: endpoints is forbidden: User \"system:serviceaccount:monitoring:default\" cannot list resource \"endpoints\" in API group \"\" at the cluster scope"
level=error ts=2020-09-15T05:03:50.510694889Z caller=main.go:216 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:354: Failed to list *v1.Node: nodes is forbidden: User \"system:serviceaccount:monitoring:default\" cannot list resource \"nodes\" in API group \"\" at the cluster scope"
level=error ts=2020-09-15T05:03:50.51077653Z caller=main.go:216 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:270: Failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:monitoring:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope"
level=error ts=2020-09-15T05:03:50.512711339Z caller=main.go:216 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:269: Failed to list *v1.Service: services is forbidden: User \"system:serviceaccount:monitoring:default\" cannot list resource \"services\" in API group \"\" at the cluster scope"
````

c) fix:
Create rbac file:
https://github.com/driprado/monitoring/blob/master/prometheus/prometheus-rbac.yml

*//Even so, NodePort type of service on EKS won’t be accessible publicly, change the service to be available via LB:*
```
root@ip-10-0-1-4:~/prometheus# cat prometheus-service.yml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'

spec:
  selector:
    app: prometheus-server
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 9090 # Internally used by EKS
root@ip-10-0-1-4:~/prometheus#
```
*prometheus GUI should now be accessible via LoadBalancer endpoint:*

root@ip-10-0-1-4:~/prometheus# kubectl describe service
```
Name:                     prometheus-service
Namespace:                monitoring
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration:
                            {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"prometheus.io/port":"9090","prometheus.io/scrape":"true"},"name":"promethe...
                          prometheus.io/port: 9090
                          prometheus.io/scrape: true
Selector:                 app=prometheus-server
Type:                     LoadBalancer
IP:                       172.20.17.230
LoadBalancer Ingress:     ac073f3c3653049d8bdfc89c638f73f6-1873445490.ap-southeast-2.elb.amazonaws.com
Port:                     <unset>  80/TCP
TargetPort:               9090/TCP
NodePort:                 <unset>  30025/TCP
Endpoints:                10.0.2.18:9090
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  Type                  15m   service-controller  NodePort -> LoadBalancer
  Normal  EnsuringLoadBalancer  15m   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   15m   service-controller  Ensured load balancer
root@ip-10-0-1-4:~/prometheus#

root@ip-10-0-1-4:~/prometheus# curl ac073f3c3653049d8bdfc89c638f73f6-1873445490.ap-southeast-2.elb.amazonaws.com
<a href="/graph">Found</a>.

root@ip-10-0-1-4:~/prometheus#


Port 9090 and internal node IP are used by EKS:
root@ip-10-0-1-4:~/prometheus# curl 10.0.2.18:9090
<a href="/graph">Found</a>.

root@ip-10-0-1-4:~/prometheus#
````






5 - Configure kube-state-metrics
https://github.com/kubernetes/kube-state-metrics
kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects.


Updated: https://github.com/driprado/monitoring/blob/master/prometheus/kube-state-metrics.yml

a) Apply
root@ip-10-0-1-4:~/prometheus# kubectl apply -f kube-state-metrics.yml
service/kube-state-metrics unchanged
deployment.apps/kube-state-metrics created
root@ip-10-0-1-4:~/prometheus#



b) Check
root@ip-10-0-1-4:~/prometheus# kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
kube-state-metrics-57c85c5bdc-gj2k6     1/1     Running   0          4m40s
prometheus-deployment-5bdddc4bf-bcx75   2/2     Running   0          98m
root@ip-10-0-1-4:~/prometheus#

root@ip-10-0-1-4:~/prometheus#
root@ip-10-0-1-4:~/prometheus# kubectl get deployments
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
kube-state-metrics      1/1     1            1           4m57s
prometheus-deployment   1/1     1            1           98m
root@ip-10-0-1-4:~/prometheus#

c) Check GUI
http://ac073f3c3653049d8bdfc89c638f73f6-1873445490.ap-southeast-2.elb.amazonaws.com/targets

Look if up or down: kubernetes-service-endpointshttp://<one-of-internal-enis>:8080/metrics













===========================================================================================
Configuring Prometheus
https://acloud.guru/course/97037e05-88ed-41a1-92ee-f5a8080318c2/learn/2efa7efd-5c5d-40c4-8711-e70cd3f83985/e6865df3-b8d4-4fe0-abb7-fa4d2070f8de/watch?backUrl=~2Fcourses

Prometheus config file vs UI targets info

in /etc/prometheus/prometheus.yml on host, or prometheus-config-map.yml on our repository:

We can see:
- job_name: kubernetes-apiservers
  scrape_interval: 5s
  scrape_timeout: 5s
  metrics_path: /metrics
  scheme: https
  kubernetes_sd_configs:
  - api_server: null
    role: endpoints
    namespaces:
      names: []
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: false
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: default;kubernetes;https
    replacement: $1
    action: keep

This translates into:
￼
EKS automatically deploys 2 api servers to your cluster for redundancy.


- job_name: kubernetes-cadvisor
  scrape_interval: 5s
  scrape_timeout: 5s
  metrics_path: /metrics
  scheme: https
  kubernetes_sd_configs:
  - api_server: null
    role: node
    namespaces:
      names: []
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: false
  relabel_configs:
  - separator: ;
    regex: __meta_kubernetes_node_label_(.+)
    replacement: $1
    action: labelmap
  - separator: ;
    regex: (.*)
    target_label: __address__
    replacement: kubernetes.default.svc:443
    action: replace
  - source_labels: [__meta_kubernetes_node_name]
    separator: ;
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor # https://kubernetes.default.svc:443/api/v1/nodes/ip-10-0-2-223.ap-southeast-2.compute.internal/proxy/metrics/cadvisor
    action: replace
￼
# One for each node:
root@ip-10-0-1-4:~# kubectl get nodes -n monitoring
NAME                                            STATUS   ROLES    AGE   VERSION
ip-10-0-1-38.ap-southeast-2.compute.internal    Ready    <none>   29h   v1.17.9-eks-4c6976
ip-10-0-2-223.ap-southeast-2.compute.internal   Ready    <none>   27h   v1.17.9-eks-4c6976
root@ip-10-0-1-4:~#

// cAdvisor: is an open source agent that monitors resource usage and analyzes the performance of containers. It’s built into kubernetes.

===========================================================================================
Setting Up Grafana

1 - Create grafana deployment
Original: https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/grafana/grafana-deployment.yml
Updated: https://github.com/driprado/monitoring/blob/master/grafana/grafana-deployment.yml

a) Create an directory and the yaml file on your cluster host
root@ip-10-0-1-4:~# mkdir grafana && cd grafana
root@ip-10-0-1-4:~/grafana# vi grafana-deployment.yml

b) Apply
root@ip-10-0-1-4:~/grafana# kubectl apply -f grafana-deployment.yml
deployment.apps/grafana created
root@ip-10-0-1-4:~/grafana#

c) Check
root@ip-10-0-1-4:~/grafana# kubectl get deployments -n monitoring
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
grafana                 1/1     1            1           115s
kube-state-metrics      1/1     1            1           22h
prometheus-deployment   1/1     1            1           23h
root@ip-10-0-1-4:~/grafana#

root@ip-10-0-1-4:~/grafana# kubectl get pods -n monitoring
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-6fff4b5dc4-p69t8                1/1     Running   0          2m47s
kube-state-metrics-57c85c5bdc-gj2k6     1/1     Running   0          22h
prometheus-deployment-5bdddc4bf-bcx75   2/2     Running   0          23h
root@ip-10-0-1-4:~/grafana#

2 - Create a service so Grafana is publicly available
Original: https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/grafana/grafana-service.yml
Updated: 

a) Apply
root@ip-10-0-1-4:~/grafana# kubectl apply -f grafana-service.yml
service/grafana-service created

b) Check:
root@ip-10-0-1-4:~/grafana# kubectl get services -n monitoring
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)          AGE
grafana-service      LoadBalancer   172.20.20.122   a02c76bbaf4e942778c3c6f8e0af69b1-194853664.ap-southeast-2.elb.amazonaws.com    3000:30565/TCP   2m23s
kube-state-metrics   ClusterIP      172.20.197.89   <none>                                                                         8080/TCP         22h
prometheus-service   LoadBalancer   172.20.17.230   ac073f3c3653049d8bdfc89c638f73f6-1873445490.ap-southeast-2.elb.amazonaws.com   80:30025/TCP     25h
root@ip-10-0-1-4:~/grafana#

root@ip-10-0-1-4:~/grafana# curl  a02c76bbaf4e942778c3c6f8e0af69b1-194853664.ap-southeast-2.elb.amazonaws.com:3000
<a href="/login">Found</a>.

root@ip-10-0-1-4:~/grafana#

root@ip-10-0-1-4:~/grafana# kubectl describe service grafana-service
Name:                     grafana-service
Namespace:                monitoring
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration:
                            {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"grafana-service","namespace":"monitoring"},"spec":{"ports":[{"por...
Selector:                 app=grafana
Type:                     LoadBalancer
IP:                       172.20.20.122
LoadBalancer Ingress:     a02c76bbaf4e942778c3c6f8e0af69b1-194853664.ap-southeast-2.elb.amazonaws.com
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  30565/TCP
Endpoints:                10.0.1.217:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  10m   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   10m   service-controller  Ensured load balancer
root@ip-10-0-1-4:~/grafana#

GUI should be available:
http://a02c76bbaf4e942778c3c6f8e0af69b1-194853664.ap-southeast-2.elb.amazonaws.com:3000/login
admin/passeword

3 - Setup a prometheus data source in grafana
Grafana > Data source > Add data source 
Name: Prometheus
URL: Prometheus URL (your LB)
Access type: direct
Add. ===========================================================================================
Node Exporter

1 - Create node-exporter deamonset and service:
root@ip-10-0-1-4:~/prometheus# vi prometheus-node-exporter.yml

a) Apply
root@ip-10-0-1-4:~/prometheus# kubectl apply -f prometheus-node-exporter.yml
daemonset.apps/node-exporter created
service/node-exporter unchanged
root@ip-10-0-1-4:~/prometheus#

b) Check
root@ip-10-0-1-4:~/prometheus# kubectl get pods -n monitoring
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-6fff4b5dc4-p69t8                1/1     Running   0          119m
kube-state-metrics-57c85c5bdc-gj2k6     1/1     Running   0          24h
node-exporter-bc4wh                     1/1     Running   0          3m28s
node-exporter-hbkss                     1/1     Running   0          3m28s
prometheus-deployment-5bdddc4bf-bcx75   2/2     Running   0          25h
root@ip-10-0-1-4:~/prometheus#

root@ip-10-0-1-4:~/prometheus# kubectl get services -n monitoring
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)          AGE
grafana-service      LoadBalancer   172.20.20.122   a02c76bbaf4e942778c3c6f8e0af69b1-194853664.ap-southeast-2.elb.amazonaws.com    3000:30565/TCP   110m
kube-state-metrics   ClusterIP      172.20.197.89   <none>                                                                         8080/TCP         24h
node-exporter        ClusterIP      172.20.11.2     <none>                                                                         9100/TCP         7m8s
prometheus-service   LoadBalancer   172.20.17.230   ac073f3c3653049d8bdfc89c638f73f6-1873445490.ap-southeast-2.elb.amazonaws.com   80:30025/TCP     26h
root@ip-10-0-1-4:~/prometheus#

Should be UP now on GUI:
￼
===========================================================================================
Creating a Grafana Dashboard
https://acloud.guru/course/97037e05-88ed-41a1-92ee-f5a8080318c2/learn/2efa7efd-5c5d-40c4-8711-e70cd3f83985/0083fe68-0168-4026-ba43-f48f4f36a322/watch?backUrl=~2Fcourses

This dashboard is based on this existing one: https://grafana.com/grafana/dashboards/3131/reviews

1 - Import dashboard
Grafana icon > Dashboards > import
Select prometheus data source > Save and Open
===========================================================================================
Instrumenting Applications
https://learn.acloud.guru/course/97037e05-88ed-41a1-92ee-f5a8080318c2/learn/278a0009-cfc5-44de-a6ea-4458d571fc23/b8c1d097-0fa9-409e-846c-d27003e66e86/watch

===========================================================================================
Recording Rules
https://learn.acloud.guru/course/97037e05-88ed-41a1-92ee-f5a8080318c2/learn/c3fc52f9-951a-4d94-ad22-b2c32a0cf309/605b5d7a-c5a9-4f1f-8550-a2d1f07d98c6/watch
https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#:~:text=Defining%20recording%20rules&text=To%20include%20rules%20in%20Prometheus,SIGHUP%20to%20the%20Prometheus%20process.

1 - Update prometheus configMap to handle the rules:
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/readrules/prometheus-config-map.yml


a) Apply
root@ip-10-0-1-4:~/prometheus/rules# kubectl apply -f prometheus-config-map.yml
configmap/prometheus-server-conf configured
root@ip-10-0-1-4:~/prometheus/rules#

2 - Create a configMap for rules
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/readrules/prometheus-read-rules-map.yml

a) Apply
root@ip-10-0-1-4:~/prometheus/rules# kubectl apply -f prometheus-read-rules-map.yml
configmap/prometheus-read-rules-conf created
root@ip-10-0-1-4:~/prometheus/rules#

3 - Update prometheus deployment
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/readrules/prometheus-deployment.yml
https://github.com/driprado/monitoring/blob/master/prometheus/rules/prometheus-deployment.yml

a) Apply
root@ip-10-0-1-4:~/prometheus/rules# kubectl apply -f prometheus-deployment.yml
deployment.apps/prometheus-deployment configured
root@ip-10-0-1-4:~/prometheus/rules#

b) Check
root@ip-10-0-1-4:~/prometheus/rules# kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
grafana-6fff4b5dc4-p69t8              1/1     Running   0          22h
kube-state-metrics-57c85c5bdc-gj2k6   1/1     Running   0          44h
node-exporter-bc4wh                   1/1     Running   0          20h
node-exporter-hbkss                   1/1     Running   0          20h
prometheus-deployment-577b78c-jh4gk   2/2     Running   0          7m48s
root@ip-10-0-1-4:~/prometheus/rules#
===========================================================================================
Setting Up Alertmanager
https://learn.acloud.guru/course/97037e05-88ed-41a1-92ee-f5a8080318c2/learn/0fbc8fb2-8803-4680-871b-1e4f2a5f3b49/0280064c-23d4-4d48-86e2-f8b8034efb66/watch

Overview
Alerting rules are configured in the prometheus server
When an alerting rule threshold is reached, an alert is triggered in alert manager which in return sends a message to the notification system (slack/PD/…)

1 - Configure slack for notifications
a) Create an workspace
b) Create a channels > Connect an app > View App Directory > Build > Start Building
c) Give app name and select your workspace, it will generate your app credentials and optios
d) under Add features and functionality select Incoming Webhooks > turn it on
e) Under Incoming Webhooks click Add New Webhook to Workspace, select your channel, Allow.
f) Copy Webhook URL and add it to alertmanager configmap(2) under api_url, add also your slack user and the channel
https://hooks.slack.com/services/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


2 - Create alertmanager configmap
https://github.com/driprado/monitoring/blob/master/alertmanager/alertmanager-configmap.yml

a)Apply
root@ip-10-0-1-4:~/alertmanager# kubectl apply -f alertmanager-configmap.yml
configmap/alertmanager-conf created
root@ip-10-0-1-4:~/alertmanager#

3 - Create alertmanager deployment
https://github.com/driprado/monitoring/blob/master/alertmanager/alermanager-deployment.yml

a) Apply
root@ip-10-0-1-4:~/alertmanager# kubectl apply -f alertmanager-depoloyment.yml
deployment.apps/alertmanager created
root@ip-10-0-1-4:~/alertmanager#

b) Check
root@ip-10-0-1-4:~/alertmanager# kubectl get deployments -n monitoring
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
alertmanager            1/1     1            1           2m57s
grafana                 1/1     1            1           23h
kube-state-metrics      1/1     1            1           46h
prometheus-deployment   1/1     1            1           47h
root@ip-10-0-1-4:~/alertmanager#

root@ip-10-0-1-4:~/alertmanager# kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-7499587cd7-pp9bq         2/2     Running   0          3m19s
grafana-6fff4b5dc4-p69t8              1/1     Running   0          23h
kube-state-metrics-57c85c5bdc-gj2k6   1/1     Running   0          46h
node-exporter-bc4wh                   1/1     Running   0          22h
node-exporter-hbkss                   1/1     Running   0          22h
prometheus-deployment-577b78c-jh4gk   2/2     Running   0          91m
root@ip-10-0-1-4:~/alertmanager#


4 - Update prometheus configMap
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/alertmanager/prometheus-config-map.yml

a) Apply
root@ip-10-0-1-4:~/alertmanager# kubectl apply -f prometheus-config-map-v1.3.yml
configmap/prometheus-server-conf configured
root@ip-10-0-1-4:~/alertmanager#

b) Check
root@ip-10-0-1-4:~/alertmanager# kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-7499587cd7-pp9bq         2/2     Running   0          20m
grafana-6fff4b5dc4-p69t8              1/1     Running   0          24h
kube-state-metrics-57c85c5bdc-gj2k6   1/1     Running   0          46h
node-exporter-bc4wh                   1/1     Running   0          22h
node-exporter-hbkss                   1/1     Running   0          22h
prometheus-deployment-577b78c-jh4gk   2/2     Running   0          108m
root@ip-10-0-1-4:~/alertmanager#
# Just checking if everything is running fine

5 - Update prometheus rules configMap
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/alertmanager/prometheus-rules-config-map.yml
https://github.com/driprado/monitoring/blob/master/alertmanager/prometheus-rules-config-map-v1.3.yml

a) Apply
root@ip-10-0-1-4:~/alertmanager# kubectl apply -f prometheus-rules-config-map-v1.3.yml
configmap/prometheus-rules-conf created
root@ip-10-0-1-4:~/alertmanager#

b) Check
root@ip-10-0-1-4:~/alertmanager# kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-7499587cd7-pp9bq         2/2     Running   0          24m
grafana-6fff4b5dc4-p69t8              1/1     Running   0          24h
kube-state-metrics-57c85c5bdc-gj2k6   1/1     Running   0          46h
node-exporter-bc4wh                   1/1     Running   0          22h
node-exporter-hbkss                   1/1     Running   0          22h
prometheus-deployment-577b78c-jh4gk   2/2     Running   0          113m
root@ip-10-0-1-4:~/alertmanager#


6 - Update prometheus deployment to accomodate alertmanager
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/alertmanager/prometheus-deployment.yml
https://github.com/driprado/monitoring/blob/master/alertmanager/prometheus-deployment-v1.3.yml

a) Apply
root@ip-10-0-1-4:~/alertmanager# kubectl apply -f prometheus-deployment-v1.3.yml
deployment.apps/prometheus-deployment configured
root@ip-10-0-1-4:~/alertmanager#

b) Check
root@ip-10-0-1-4:~/alertmanager# kubectl get deployments -n monitoring
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
alertmanager            1/1     1            1           35m
grafana                 1/1     1            1           24h
kube-state-metrics      1/1     1            1           46h
prometheus-deployment   1/1     1            1           2d
root@ip-10-0-1-4:~/alertmanager#

root@ip-10-0-1-4:~/alertmanager# kubectl get pods  -n monitoring
NAME                                     READY   STATUS    RESTARTS   AGE
alertmanager-7499587cd7-pp9bq            2/2     Running   0          36m
grafana-6fff4b5dc4-p69t8                 1/1     Running   0          24h
kube-state-metrics-57c85c5bdc-gj2k6      1/1     Running   0          46h
node-exporter-bc4wh                      1/1     Running   0          22h
node-exporter-hbkss                      1/1     Running   0          22h
prometheus-deployment-784b48cf5f-8mmsg   2/2     Running   0          3m26s
root@ip-10-0-1-4:~/alertmanager#

c) Check alerts on Prometheus GUI > Alerts
￼

7 - Create alertmanager service
https://github.com/linuxacademy/content-kubernetes-prometheus-env/blob/master/alertmanager/alertmanager-service.yml
https://github.com/driprado/monitoring/blob/master/alertmanager/alertmanager-service.yml
30000-32767
31111 - new nodePort
8081 - old nodePort


a) Apply
root@ip-10-0-1-4:~/alertmanager# kubectl apply -f alertmanager-service.yml
service/alertmanager created
root@ip-10-0-1-4:~/alertmanager#

b) Check Alertmanager endpoint on Prometheus GUI
Status > Runtime and Build Information
￼
Should also update status under Alerts
￼
Should also send the existing active alert to slack:

￼

Should be able to access Alertmanager GUI on LB:9093
==================================================================================================================================
Writing Prometheus Alerting Rules
* https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/#:~:text=elements'%20label%20sets.-,Defining%20alerting%20rules,same%20way%20as%20recording%20rules.&text=The%20optional%20for%20clause%20causes,as%20firing%20for%20this%20element.
* https://blog.pvincent.io/2017/12/prometheus-blog-series-part-5-alerting-rules/
* https://alex.dzyoba.com/blog/prometheus-alerts/

# The redis alert exists because it expects a redis server to exist
1 - spin up a redis service to quiet the alert
2 - remove the alert-rule from prometheus configuration (configMap)

==================================================================================================================================
￼
