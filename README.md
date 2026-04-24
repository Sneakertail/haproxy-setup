# Network Setup

## sneakertail.online

- Add DNS Record to godaddy
|Type|Name|Value|
|---|---|---|
|A|@|<PUBLIC_IP_OF_HAPROXY_EC2>|

```sh
sudo apt update
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
