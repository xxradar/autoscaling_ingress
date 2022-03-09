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
kubectl port-forward svc/grafana-server 3000 -n prometheus --address  0.0.0.0 &
kubectl port-forward svc/prometheus-server 9090 -n prometheus --address 0.0.0.0 &
```
