### Predictive Scaling for an customer Service
#### Components:
* Sample Application: A simple Nginx deployment representing our customer application frontend.
* Metrics Server: Required for HPA to work.
* Prometheus: To collect metrics from our custom metric exporter.
* Prometheus Adapter: To expose custom metrics from Prometheus to the Kubernetes Custom Metrics API, which HPA can consume.
* Python "AI Agent" (Simple Script):
    * Reads current time and date.
    * Has a simple rule-based "prediction" logic (e.g., increase load prediction during peak hours, or if a "flash sale" flag is set).
    * Pushes this predicted load as a custom metric to Prometheus.
* Horizontal Pod Autoscaler (HPA): Configured to scale the Nginx deployment based on our custom "predicted_load" metric.

#### Deploy Metrics Server (Ignore If already Present)
* Create metric server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```
* Verify the status of metric server pod 
```bash
blrk@blrk-lpc:~/devops_aganticai> kubectl get pods -n kube-system | grep metrics-server
metrics-server-7bb66479db-lpr2p   0/1     Running   0               13s
```
#### Deploy Prometheus and Prometheus Adapter
* Add Prometheus repo and update it 
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
* Install Prometheus
```bash
helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace
```
* List Prometheus resources 
```bash
kubectl get all -n monitoring 
NAME                                                     READY   STATUS    RESTARTS   AGE
pod/prometheus-alertmanager-0                            1/1     Running   0          3m59s
pod/prometheus-kube-state-metrics-57d654d7bf-bdppd       1/1     Running   0          3m59s
pod/prometheus-prometheus-node-exporter-slpvp            1/1     Running   0          3m59s
pod/prometheus-prometheus-node-exporter-x29tk            1/1     Running   0          3m59s
pod/prometheus-prometheus-pushgateway-784c485d55-5smr2   1/1     Running   0          3m59s
pod/prometheus-server-7756f6c55d-5xbkp                   2/2     Running   0          3m59s

NAME                                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/prometheus-alertmanager               ClusterIP   10.111.239.73   <none>        9093/TCP   4m
service/prometheus-alertmanager-headless      ClusterIP   None            <none>        9093/TCP   4m
service/prometheus-kube-state-metrics         ClusterIP   10.99.113.146   <none>        8080/TCP   4m
service/prometheus-prometheus-node-exporter   ClusterIP   10.96.240.136   <none>        9100/TCP   4m
service/prometheus-prometheus-pushgateway     ClusterIP   10.109.123.83   <none>        9091/TCP   4m
service/prometheus-server                     ClusterIP   10.100.15.80    <none>        80/TCP     4m

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/prometheus-prometheus-node-exporter   2         2         2       2            2           kubernetes.io/os=linux   3m59s

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-kube-state-metrics       1/1     1            1           3m59s
deployment.apps/prometheus-prometheus-pushgateway   1/1     1            1           3m59s
deployment.apps/prometheus-server                   1/1     1            1           3m59s

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-kube-state-metrics-57d654d7bf       1         1         1       3m59s
replicaset.apps/prometheus-prometheus-pushgateway-784c485d55   1         1         1       3m59s
replicaset.apps/prometheus-server-7756f6c55d                   1         1         1       3m59s

NAME                                       READY   AGE
statefulset.apps/prometheus-alertmanager   1/1     3m59s
```
* Install Prometheus Adapter (to expose custom metrics)
* Note: We'll use a custom values file to configure the adapter to expose our predicted_load metric.
* Create a file named prometheus-adapter-values.yaml
```bash
cat <<EOF > prometheus-adapter-values.yaml
rules:
  default: false 
  # Configure rules for external metrics
  external:
    - seriesQuery: '{__name__="predicted_load"}' # Matches the metric name generated by your Python agent
      name:
        matches: "predicted_load" # The exact name of your metric
        as: "predicted_load_per_pod" # The name it will be exposed as in the Custom Metrics API
      metricsQuery: "predicted_load" # The PromQL query to retrieve the metric value
      # HPA will target `predicted_load_per_pod` without requiring it to be tied to a specific pod label in the adapter rules.

# Ensure the adapter can reach Prometheus
prometheus:
  url: http://prometheus-prometheus.monitoring.svc # Use the internal service name for Prometheus
  port: 9090
EOF
```
* Verify deployments
```bash
kubectl get pods -n monitoring
NAME                                                 READY   STATUS    RESTARTS   AGE
prometheus-adapter-5c76b47bd4-8zclf                  1/1     Running   0          8m27s
prometheus-alertmanager-0                            1/1     Running   0          16m
prometheus-kube-state-metrics-57d654d7bf-bdppd       1/1     Running   0          16m
prometheus-prometheus-node-exporter-slpvp            1/1     Running   0          16m
prometheus-prometheus-node-exporter-x29tk            1/1     Running   0          16m
prometheus-prometheus-pushgateway-784c485d55-5smr2   1/1     Running   0          16m
prometheus-server-7756f6c55d-5xbkp  
```
* 
```bash
kubectl get apiservice v1beta1.custom.metrics.k8s.io -o yaml -- Not working 
```
#### Deploy Sample Nginx Application
* Create application.yaml and add the following manifest
```bash
# application.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer-frontend
  labels:
    app: customer-rontend
spec:
  replicas: 1 
  selector:
    matchLabels:
      app: customer-frontend
  template:
    metadata:
      labels:
        app: customer-frontend
      annotations:
        prometheus.io/scrape: "true" # Ensure Prometheus scrapes this pod
        prometheus.io/port: "80"
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: customer-frontend
spec:
  selector:
    app: customer-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
```bash
kubectl apply -f application.yaml
deployment.apps/customer-frontend created
```
* Verify the status of customer application
```bash
kubectl get all

NAME                                     READY   STATUS              RESTARTS   AGE
pod/customer-frontend-664c8cd878-qgh2x   0/1     ContainerCreating   0          7s

NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/customer-frontend   ClusterIP   10.98.161.87   <none>        80/TCP    7s
service/kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP   7d18h

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/customer-frontend   0/1     1            0           9s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/customer-frontend-664c8cd878   1         1         0       8s
```




