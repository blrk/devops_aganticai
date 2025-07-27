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
```bash
blrk@blrk-lpc:~/devops_aganticai> kubectl get pods -n kube-system | grep metrics-server
metrics-server-7bb66479db-lpr2p   0/1     Running   0               13s
```


