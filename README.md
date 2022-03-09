# autoscaling_ingress

## Install a self-managed K8S cluster
```
kubectl get no
```


## Install Helm
See https://helm.sh/docs/intro/install/
```
wget https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz
tar -zxvf helm-v3.8.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

## Install nginx ingress controller using helm
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
and then update ...
```
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx \
--set controller.metrics.enabled=true \
--set-string controller.podAnnotations."prometheus\.io/scrape"="true" \
--set-string controller.podAnnotations."prometheus\.io/port"="10254"
```
verify the updated values ...
```
helm get values ingress-nginx --namespace ingress-nginx
```
## Installing prometheus and grafana
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
```
helm repo add stable https://charts.helm.sh/stable
```
```
helm repo update
```
```
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace prometheus --create-namespace
```
```
$ kubectl get po -n prometheus
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          40s
prometheus-grafana-8568977b76-rszrx                      3/3     Running   0          56s
prometheus-kube-prometheus-operator-55b8575db7-mc86z     1/1     Running   0          56s
prometheus-kube-state-metrics-94f76f559-fr44q            1/1     Running   0          56s
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          39s
prometheus-prometheus-node-exporter-8ckhp                1/1     Running   0          56s
prometheus-prometheus-node-exporter-rhlgl                1/1     Running   0          56s
prometheus-prometheus-node-exporter-wj6p4                1/1     Running   0          56s
```
```
$ kubectl get svc -n prometheus
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   81s
prometheus-grafana                        ClusterIP   10.100.35.177    <none>        80/TCP                       97s
prometheus-kube-prometheus-alertmanager   ClusterIP   10.96.38.194     <none>        9093/TCP                     97s
prometheus-kube-prometheus-operator       ClusterIP   10.100.226.240   <none>        443/TCP                      97s
prometheus-kube-prometheus-prometheus     ClusterIP   10.98.42.127     <none>        9090/TCP                     97s
prometheus-kube-state-metrics             ClusterIP   10.100.72.229    <none>        8080/TCP                     97s
prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     80s
prometheus-prometheus-node-exporter       ClusterIP   10.105.67.143    <none>        9100/TCP                     97s
```
```
kubectl port-forward -n prometheus --address 0.0.0.0  svc/prometheus-grafana 3000:80 &   #username: admin  password: prom-operator
kubectl port-forward -n prometheus --address 0.0.0.0  svc/prometheus-operated 9090:9090 &
```
Default password for grafana `username: admin  password: prom-operator`
