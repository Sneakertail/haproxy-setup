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

# Tell ArgoCD to stop forcing HTTPS and run in "insecure" mode. we have a proxy (KGateway) sitting in front of it that will handle our TLS certificates.
kubectl patch configmap argocd-cmd-params-cm -n argocd -p '{"data": {"server.insecure": "true"}}'
kubectl rollout restart deployment argocd-server -n argocd

OR

kubectl edit deployment argocd-server -n argocd
containers:
- name: argocd-server
  args:
  - --insecure


# CLI
#curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
#sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
#rm argocd-linux-amd64

# Login
argocd admin initial-password -n argocd
argocd login <IP:PORT> --username admin --password <initial-password> --insecure
argocd account update-password
# argocd123

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

## Argo-rollout

```sh
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl get pods -n argo-rollouts

curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

kubectl argo rollouts version

kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/dashboard-install.yaml
kubectl get svc -n argo-rollouts
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

```
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

## Create Cert

```sh
# Getting a Real Certificate via Let's Encrypt
sudo apt update
sudo apt install certbot

sudo systemctl stop haproxy
sudo certbot certonly --standalone -d sneakertail.online -d www.sneakertail.online
sudo systemctl start haproxy


---
root@ip-172-31-0-82:/home/ubuntu# dig +short sneakertail.online
65.1.191.5
root@ip-172-31-0-82:/home/ubuntu# sudo systemctl stop haproxy
root@ip-172-31-0-82:/home/ubuntu# sudo certbot certonly --standalone -d sneakertail.online -d www.sneakertail.online
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for sneakertail.online and www.sneakertail.online

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/sneakertail.online/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/sneakertail.online/privkey.pem
This certificate expires on 2026-07-25.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
root@ip-172-31-0-82:/home/ubuntu# sudo systemctl start haproxy
---


sudo bash -c 'cat /etc/letsencrypt/live/sneakertail.online/fullchain.pem /etc/letsencrypt/live/sneakertail.online/privkey.pem > /etc/ssl/private/sneakertail.pem'
sudo chmod 600 /etc/ssl/private/sneakertail.pem
sudo nano /etc/haproxy/haproxy.cfg

---
frontend http_front
    bind *:80
    mode http
    http-request redirect scheme https unless { ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/ssl/private/sneakertail.pem
    mode http
    
    option forwardfor
    http-request add-header X-Forwarded-Proto https
    
    default_backend kgateway_http_back

backend kgateway_http_back
    mode http
    balance roundrobin
    server worker1 172.31.0.178:31828 check
    server worker2 172.31.0.231:31828 check
---

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy

----------------------------

----------------------------
sudo certbot certonly --manual --preferred-challenges dns -d sneakertail.online -d "*.sneakertail.online"
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name:

_acme-challenge.sneakertail.online.

with the following value:

SkJ2YXpnqC5cdji7fGnfUuqSVod6WArE3MgC_6-Znk0

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.sneakertail.online.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
sudo bash -c 'cat /etc/letsencrypt/live/sneakertail.online-0001/fullchain.pem /etc/letsencrypt/live/sneakertail.online-0001/privkey.pem > /etc/ssl/private/sneakertail.pem'
sudo systemctl reload haproxy

```

## HeadLamp

```sh
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm install my-headlamp headlamp/headlamp --namespace kube-system
---
root@ip-172-31-0-49:/home/ubuntu/infra# kubectl get all -A | grep headlamp
kube-system       pod/my-headlamp-69f8584f88-2tqtk                        1/1     Running   0          2m39s
kube-system       service/my-headlamp                               ClusterIP      10.102.139.64    <none>        80/TCP                       2m39s
kube-system       deployment.apps/my-headlamp                        1/1     1            1           2m39s
kube-system       replicaset.apps/my-headlamp-69f8584f88                        1         1         1       2m39s
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: headlamp-route
  namespace: kube-system
spec:
  parentRefs:
  - name: my-gateway        # <-- change to your Gateway name
    namespace: default      # <-- change if your Gateway is in another namespace
  hostnames:
  - "headlamp.yourdomain.com"   # <-- optional (use if you have DNS)
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: my-headlamp
      port: 80
```

Authentication

```
kubectl -n kube-system create serviceaccount headlamp-admin
kubectl create clusterrolebinding headlamp-admin --serviceaccount=kube-system:headlamp-admin --clusterrole=cluster-admin
kubectl create token headlamp-admin -n kube-system
```

## NFS Server

```sh
sudo apt update
sudo apt install nfs-kernel-server -y

mkdir -p /var/nfs
chown -R nobody:nogroup /var/nfs
chown -R 777 /var/nfs

vi /etc/exports
/var/nfs  *(rw,sync,no_subtree_check,no_root_squash)
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server

/var/nfs/team3  <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

## CSI Driver

```sh
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system
```
