# Monitoring Setup

## metrics-server

```sh
kubectl create namespace monitoring

helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm install metrics-server metrics-server/metrics-server \
  --namespace monitoring \
  --set args={--kubelet-insecure-tls}
```

prometheus

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring
```

garafana

```sh
# Loki with promtail

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki-stack \
  --namespace monitoring

helm install grafana grafana/grafana \
  --namespace monitoring

kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
# c4tuJe6uDHX1jA4ezAO8XPxdbjKf92SjerkRuid9
```
  ```
