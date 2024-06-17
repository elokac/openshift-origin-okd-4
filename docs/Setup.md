Install OKD 4 that is the upstream version of Red Hat OpenShift 4.
This example is based on the environment like follows.

| NODE	                                               | HOSTNAME	           | IP ADDRESS	   | MEMORY  | vCPU	| STORAGE
| ---------------------------------------------------- |-----------------------|---------------|---------|------|----------
| Manager Node (DNS, HAProxy, Nginx) - CentOS-Stream-9 | apps	               | 16.16.0.13	   | 4GB	 | 4	|    50GB
| Bootstrap Node - Fedora CoreOS	                   | okd4-bootstrap	       | 16.16.0.14    | 16GB	 | 4	|    50GB
| Control Node-1 - Fedora CoreOS	                   | okd4-control-plane-1  | 16.16.0.15    | 16GB	 | 4	|    50GB
| Control Node-2 - Fedora CoreOS	                   | okd4-control-plane-2  | 16.16.0.16    | 16GB	 | 4	|    50GB
| Control Node-3 - Fedora CoreOS	                   | okd4-control-plane-3  | 16.16.0.17    | 16GB	 | 4	|    50GB

1. ## Dnsmasq : Install and Configure
```[root@mngr-node ~]# dnf -y install dnsmasq```

2. ## Configure Dnsmasq
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


