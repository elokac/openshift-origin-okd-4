Install OKD 4 that is the upstream version of Red Hat OpenShift 4.
This example is based on the environment like follows.

| NODE	                                               | HOSTNAME	           | IP ADDRESS	   | MEMORY  | vCPU	| STORAGE
| ---------------------------------------------------- |-----------------------|---------------|---------|------|----------
| Manager Node (DNS, HAProxy, Nginx) - CentOS-Stream-9 | apps	               | 16.16.0.13	   | 4GB	 | 4	|    50GB
| Bootstrap Node - Fedora CoreOS	                   | okd4-bootstrap	       | 16.16.0.14    | 16GB	 | 4	|    50GB
| Control Node-1 - Fedora CoreOS	                   | okd4-control-plane-1  | 16.16.0.15    | 16GB	 | 4	|    50GB
| Control Node-2 - Fedora CoreOS	                   | okd4-control-plane-2  | 16.16.0.16    | 16GB	 | 4	|    50GB
| Control Node-3 - Fedora CoreOS	                   | okd4-control-plane-3  | 16.16.0.17    | 16GB	 | 4	|    50GB

1. ### Dnsmasq : Install and Configure
```[root@mngr-node ~]# dnf -y install dnsmasq```

2. ### Configure Dnsmasq
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

3. ### Configure DNS Records
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
