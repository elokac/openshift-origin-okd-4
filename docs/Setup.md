Install OKD 4 that is the upstream version of Red Hat OpenShift 4.
This example is based on the environment like follows.

| NODE	                                               | HOSTNAME	           | IP ADDRESS	   | MEMORY  | vCPU	| STORAGE
| ---------------------------------------------------- |-----------------------|---------------|---------|------|----------
| Manager Node (DNS, HAProxy, Nginx) - CentOS-Stream-9 | apps	               | 16.16.0.13	   | 4GB	 | 4	|    50GB
| Bootstrap Node - Fedora CoreOS	                   | okd4-bootstrap	       | 16.16.0.14    | 16GB	 | 4	|    50GB
| Control Node-1 - Fedora CoreOS	                   | okd4-control-plane-1  | 16.16.0.15    | 16GB	 | 4	|    50GB
| Control Node-2 - Fedora CoreOS	                   | okd4-control-plane-2  | 16.16.0.16    | 16GB	 | 4	|    50GB
| Control Node-3 - Fedora CoreOS	                   | okd4-control-plane-3  | 16.16.0.17    | 16GB	 | 4	|    50GB

1. ## Install and Configure dnsmasq
We will begin by setting up Dnsmasq on the manager node to resolve the requests for the Kubernetes cluster

- ### Install dnsmasq

```[root@mngr-node ~]# dnf -y install dnsmasq```

- ### Configure Dnsmasq
```[root@mngr-node ~]# vim /etc/dnsmasq.conf```
```
port=53
domain-needed
bogus-priv

strict-order
expand-hosts
domain=okd.local.com
address=/apps.okd.local.com/16.16.0.13
```

my domain name is local.com and 16.16.0.13 resolves all the requests directed to apps.okd.local.com. Once the desired changes have been made, save the file and restart the service:

```
[root@mngr-node ~]# systemctl restart dnsmasq
[root@mngr-node ~]# systemctl enable --now dnsmasq
```

- ### Configure DNS Records
Now add the DNS records to /etc/hosts. Dnsmasq will reply to all requests using the records here.
```
[root@mngr-node ~]# vim /etc/hosts
```
```
16.16.0.13 api-int.okd.local.com               api 
16.16.0.14 okd4-bootstrap.okd.local.com        okd4-bootstrap
16.16.0.15 okd4-control-plane-1.okd.local.com  okd4-control-plane-1
16.16.0.16 okd4-control-plane-2.okd.local.com  okd4-control-plane-2
16.16.0.17 okd4-control-plane-3.okd.local.com  okd4-control-plane-3
```
Also modify /etc/resolv.conf:
```
search okd.local.com
nameserver 16.16.0.13
nameserver 8.8.8.8
```
Restart dnsmasq;

```[root@mngr-node ~]# systemctl restart dnsmasq```

Allow firewall;

```
[root@mngr-node ~]# firewall-cmd --permanent --add-port=53/udp
[root@mngr-node ~]# firewall-cmd --reload
```
Test if the service is working as desired;
```
[root@mngr-node ~]# dig api-int.okd.local.com
```

2. ## Install and Configure Nginx
This will used to serve the ignition file to other nodes

- ### Install nginx
```
[root@mngr-node ~]# dnf -y install nginx
```
- ### Configure Nginx
```
[root@mngr-node ~]#	vim /etc/nginx/nginx.conf
```
```
 server {
        listen       8080;
        listen       [::]:8080;
        server_name  okd.local.com;
        root         /usr/share/nginx/html;
    }
```
```
[root@mngr-node ~]# systemctl enable --now nginx
[root@mngr-node ~]# firewall-cmd --add-service=http
[root@mngr-node ~]#	firewall-cmd --runtime-to-permanent
success
```

3. ## Install and Configure HAProxy
Now, it's time to configure HAProxy which will be used to route traffic to individual Nodes.
- ### Install HAProxy
```
[root@mngr-node ~]# dnf install haproxy git -y
```
- ### Configure HAProxy
```
[root@mngr-node ~]# vim /etc/haproxy/haproxy.cfg
```
```
global
  log         127.0.0.1 local2
  pidfile     /var/run/haproxy.pid
  maxconn     4000
  daemon
  stats socket /var/lib/haproxy/stats
defaults
  mode                    http
  log                     global
  option                  dontlognull
  option http-server-close
  option                  redispatch
  retries                 3
  timeout http-request    10s
  timeout queue           1m
  timeout connect         10s
  timeout client          1m
  timeout server          1m
  timeout http-keep-alive 10s
  timeout check           10s
  maxconn                 3000
listen stats
  bind :9000
  stats uri /stats
  stats refresh 10000ms
listen api-server-6443
  bind *:6443
  mode tcp
  #option  httpchk GET /readyz HTTP/1.0
  #option  log-health-checks
  #balance roundrobin
  server okd4-bootstrap       okd4-bootstrap.okd.local.com:6443 check
  server okd4-control-plane-1 okd4-control-plane-1.okd.local.com:6443 check
  server okd4-control-plane-2 okd4-control-plane-2.okd.local.com:6443 check
  server okd4-control-plane-3 okd4-control-plane-3.okd.local.com:6443 check
listen machine-config-server-22623
  bind *:22623
  mode tcp
  server okd4-bootstrap       okd4-bootstrap.okd.local.com:22623 check inter 1s
  server okd4-control-plane-1 okd4-control-plane-1.okd.local.com:22623 check inter 1s
  server okd4-control-plane-2 okd4-control-plane-2.okd.local.com:22623 check inter 1s
  server okd4-control-plane-3 okd4-control-plane-3.okd.local.com:22623 check inter 1s
listen ingress-router-443
  bind *:443
  mode tcp
  balance source
  server okd4-control-plane-1 okd4-control-plane-1.okd.local.com:443 check inter 1s
  server okd4-control-plane-2 okd4-control-plane-2.okd.local.com:443 check inter 1s
  server okd4-control-plane-3 okd4-control-plane-3.okd.local.com:443 check inter 1s
  # server okd4-worker-node-1 okd4-worker-node-1.okd.local.com:443 check inter 1s
listen ingress-router-80
  bind *:80
  mode tcp
  balance source
  server okd4-control-plane-1 okd4-control-plane-1.okd.local.com:80 check inter 1s
  server okd4-control-plane-2 okd4-control-plane-2.okd.local.com:80 check inter 1s
  server okd4-control-plane-3 okd4-control-plane-3.okd.local.com:80 check inter 1s
  # server okd4-worker-node-1 okd4-worker-node-1.okd.local.com:80 check inter 1s
```
Modify SELinux:
```
setsebool -P haproxy_connect_any 1
setsebool -P httpd_can_network_connect on
setsebool -P httpd_graceful_shutdown on
setsebool -P httpd_can_network_relay on
setsebool -P nis_enabled on
semanage port -a -t http_port_t -p tcp 6443
semanage port -a -t http_port_t -p tcp 22623
semanage port -a -t http_port_t -p tcp 1936
```
Start and enable the service;
```
[root@mngr-node ~]# systemctl enable haproxy
[root@mngr-node ~]# systemctl start haproxy
[root@mngr-node ~]# systemctl status haproxy
```
Allow the OKD ports through the firewall;
```
[root@mngr-node ~]# firewall-cmd --add-port={6443/tcp,22623/tcp,1936/tcp}
[root@mngr-node ~]# firewall-cmd --add-service={http,https}
[root@mngr-node ~]# firewall-cmd --runtime-to-permanent
[root@mngr-node ~]# firewall-cmd --reload
success
```