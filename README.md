# Network Setup

## sneakertail.online

- Add DNS Record to godaddy

| Type | Name | Value |
| --- | --- | --- |
| A | @ | <PUBLIC_IP_OF_HAPROXY_EC2> |

## ArgoCD

```sh
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

#CLI
#curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
#sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
#rm argocd-linux-amd64

#Login
argocd admin initial-password -n argocd
argocd login <IP:PORT> --username admin --password <initial-password> --insecure
argocd account update-password

kubectl config get-contexts -o name
argocd cluster add <CONTEXT>
argocd cluster list
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/Sneakertail/sneakertail-charts.git'
    targetRevision: dev
    path: infra
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Kgateway

```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.3.0-main kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds

helm upgrade -i -n kgateway-system kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
--version v2.3.0-main

kubectl get pods -n kgateway-system 
kubectl get gatewayclass kgateway 
kubectl get crds 
kubectl get all -n kgateway-system 
```

gateway.yaml
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kgateway
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All
```

## HA Proxy

0```sh
sudo apt update
sudo apt install -y certbot

# Create Cert
sudo certbot certonly --standalone -d sneakertail.online -d www.sneakertail.online

sudo apt install haproxy -y
sudo vi /etc/haproxy/haproxy.cfg
```

```sh
frontend http_front
    bind *:80
    mode http
    default_backend kgateway_http_back

frontend https_front
    bind *:443
    mode tcp
    default_backend kgateway_https_back

backend kgateway_http_back
    mode http
    balance roundrobin
    server worker1 172.31.0.178:31828 check
    server worker2 172.31.0.231:31828 check

backend kgateway_https_back
    mode tcp
    balance roundrobin
    server worker1 172.31.0.178:32767 check
    server worker2 172.31.0.231:32767 check
```

```sh
sudo systemctl restart haproxy
```

```sh
apt install nfs-kernel-server -y
mkdir -p /var/nfs/team3
chown -R nobody:nogroup /var/nfs/team3
chown -R 777 /var/nfs/team3
vi /etc/exports
/var/nfs/team3  *(rw,sync,no_subtree_check,no_root_squash)
sudo exportfs -a
systemctl restart nfs-kernel-server
sudo exportfs -v
/var/nfs/team3  <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```
