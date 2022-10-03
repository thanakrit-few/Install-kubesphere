## Step 1 install and config HAproxy
```
yum install -y haproxy
```
```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
vi /etc/haproxy/haproxy.cfg
```
```
global
    log /dev/log  local0 warning
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
   
   stats socket /var/lib/haproxy/stats
   
defaults
  log global
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000
   
frontend kube-apiserver
  bind *:6444
  mode tcp
  option tcplog
  default_backend kube-apiserver
   
backend kube-apiserver
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server kube-apiserver-1 IP node1:6443 check # Replace the IP address with your own.
    server kube-apiserver-2 IP node2:6443 check # Replace the IP address with your own.
    server kube-apiserver-3 IP node3:6443 check # Replace the IP address with your own.
```
```
haproxy -f /etc/haproxy/haproxy.cfg -c
setsebool -P haproxy_connect_any on
systemctl disable firewalld
systemctl stop firewalld
systemctl start haproxy
systemctl status haproxy
```

## Step 2 install and config keepalived

```
yum install -y keepalived
```
```
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
vi /etc/keepalived/keepalived.conf
```
```
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
   
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
   
vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface eth0                       # Network card
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 172.16.0.2      # The IP address of this machine
  unicast_peer {
    172.16.0.3                         # The IP address of peer machines
  }
   
  virtual_ipaddress {
    172.16.0.10/24                  # The VIP address
  }
   
  track_script {
    chk_haproxy
  }
}
```
```
systemctl enable keepalived
systemctl restart keepalived
```

## Step 3 install package and install kubesphere

```
sudo yum -y install conntrack socat
sudo yum -y install ebtables ipset 
```
```
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.2.2 sh -
chmod +x kk
./kk create config --with-kubernetes v1.23.7 --with-kubesphere v3.3.0 
vi config-sample.yaml 
```
```
apiVersion: kubekey.kubesphere.io/v1alpha2 
kind: Cluster 
metadata: 
  name: sample 
spec: 
  hosts: 
  - {name: Nodename, address: IP Node, internalAddress: IP Node, user: root, password: "Password"} 
  - {name: Nodename, address: IP Node, internalAddress: IP Node, user: root, password: "Password"} 
  - {name: Nodename, address: IP Node, internalAddress: IP Node, user: root, password: "Password"} 
  roleGroups: 
    etcd: 
    - Masternode1
    - Masternode2
    - Masternode3
    control-plane: 
    - Masternode1
    - Masternode2
    - Masternode3
    worker: 
    - Workernode1 
    - Workernode2
  controlPlaneEndpoint: 
    ## Internal loadbalancer for apiservers 
    # internalLoadbalancer: haproxy 
    domain: lb.kubesphere.local 
    address: "VIP" 
    port: 6444 
  kubernetes: 
    version: v1.23.7 
    clusterName: cluster.local 
    autoRenewCerts: true 
    containerManager: containerd 
  etcd: 
    type: kubekey 
  network: 
    plugin: calico 
    kubePodsCIDR: 10.233.64.0/18 
    kubeServiceCIDR: 10.233.0.0/18 
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni 
    multusCNI: 
      enabled: false 
  registry: 
    privateRegistry: "" 
    namespaceOverride: "" 
    registryMirrors: [] 
    insecureRegistries: [] 
  addons: [] 
```
```
./kk create cluster --with-kubernetes v1.23.7 --with-kubesphere v3.3.0 -f config-sample.yaml
```

>> Check log of kubesphere
```
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -f 
```
