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
kubectl port-forward -n prometheus --address 0.0.0.0  svc/prometheus-grafana 3000:80 &
kubectl port-forward -n prometheus --address 0.0.0.0  svc/prometheus-operated 9090:9090 &
```
Default password for grafana `username: admin  password: prom-operator`

## Making sure prometheus and grafana can obtain the ingress nginx metrics

```
kubectl apply -f -<<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-controller-metrics
  namespace: prometheus
  labels:
    app: nginx-ingress
    release: prometheus
spec:
  endpoints:
  - interval: 30s
    port: metrics
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  namespaceSelector:
    matchNames:
    - ingress-nginx
EOF
```
## Deploy a sample app
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
spec:
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: stefanprodan/podinfo
        ports:
        - containerPort: 9898
---
apiVersion: v1
kind: Service
metadata:
  name: podinfo
spec:
  ports:
    - port: 80
      targetPort: 9898
      nodePort: 30001
  selector:
    app: podinfo
  type: ClusterIP
EOF
```
## Deploy the ingress resource
```
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo
spec:
  ingressClassName: nginx
  rules:
    - host: "example.com"
      http:
        paths:
          - backend:
              service:
                name: podinfo
                port:
                  number: 80
            path: /
            pathType: Prefix
EOF
```
Test the ingress ...
```
$ kubectl get svc ingress-nginx-controller -n ingress-nginx
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.111.11.74   <pending>     80:30191/TCP,443:31432/TCP   3h14m
```
```
curl  -H "Host: example.com" http://127.0.0.1:30191
{
  "hostname": "podinfo-5d76864686-dcmdx",
  "version": "6.0.3",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.3",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.9",
  "num_goroutine": "6",
  "num_cpu": "4"
}
```
## Deploy LOCUST to generate traffic
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: locust-script
data:
  locustfile.py: |-
    from locust import HttpUser, task, between

    class QuickstartUser(HttpUser):
        wait_time = between(0.7, 1.3)

        @task
        def hello_world(self):
            self.client.get("/", headers={"Host": "example.com"})
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust
spec:
  selector:
    matchLabels:
      app: locust
  template:
    metadata:
      labels:
        app: locust
    spec:
      containers:
        - name: locust
          image: locustio/locust
          ports:
            - containerPort: 8089
          volumeMounts:
            - mountPath: /home/locust
              name: locust-script
      volumes:
        - name: locust-script
          configMap:
            name: locust-script
---
apiVersion: v1
kind: Service
metadata:
  name: locust
spec:
  ports:
    - port: 8089
      targetPort: 8089
      nodePort: 30015
  selector:
    app: locust
  type: LoadBalancer
EOF
```
