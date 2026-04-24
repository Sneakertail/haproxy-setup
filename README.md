# Network Setup

## sneakertail.online

- Add DNS Record to godaddy

| Type | Name | Value |
| --- | --- | --- |
| A | @ | <PUBLIC_IP_OF_HAPROXY_EC2> |

## Kgateway

```sh
# Control plane
snap install helm --classic

# 1. Install standard Kubernetes Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# 2. Install Kgateway CRDs
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.2.1 kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds

# 3. Install Kgateway and expose via NodePort for HAProxy
helm upgrade -i -n kgateway-system kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.2.1 \
  --set gateway.service.type=NodePort \
  --set gateway.service.httpPortNodePort=32080 \
  --set gateway.service.httpsPortNodePort=32443

kubectl get pods -n kgateway-system
kubectl get gatewayclass
kubectl get svc -n kgateway-system
```

```sh
sudo apt update
sudo apt install -y certbot

# Create Cert
sudo certbot certonly --standalone -d sneakertail.online -d www.sneakertail.online

sudo apt install haproxy -y
sudo vi /etc/haproxy/haproxy.cfg
```

```sh
frontend http_frontend
        bind *:80
        default_backend worker_nodes
backend worker_nodes
        balance roundrobin
        server s1 172.31.1.91:xxxx check
        server s2 172.31.1.128:xxxx check
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
