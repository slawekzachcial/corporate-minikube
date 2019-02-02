# Corporate Minikube

If ...

* You work in a shop that uses Windows workstations
* Your company network requires you to use HTTP proxy to access the Internet
* You love Linux and have a virtual machine running your favorite distribution
  and tools
* Your Linux VM has recent Docker installed
* You want to run [Kubernetes Minikube](https://kubernetes.io/docs/setup/minikube/)
  in you Linux VM

... this page may help you.

## TL;DR

Once you installed Minikube and `kubectl` using your distribution's package
manager ...

```
PROXY_HOST=proxy.host
PROXY_PORT=8080
VM_IP=10.0.2.15

SERVICE_CLUSTER_IP_ADDRESSES="$(echo 10.96.0.{1..255} | sed 's/ /,/g')"
SERVICE_CLUSTER_IP_RANGE="10.96.0.0/24"

export CHANGE_MINIKUBE_NONE_USER=true
export http{,s}_proxy=http://${PROXY_HOST}:${PROXY_PORT}
export no_proxy="localhost,127.0.0.1,${VM_IP},${SERVICE_CLUSTER_IP_ADDRESSES}"
sudo -E minikube start \
  --vm-driver=none \
  --apiserver-ips=127.0.0.1 \
  --apiserver-name=localhost \
  --logtostderr -v 0 \
  --service-cluster-ip-range="${SERVICE_CLUSTER_IP_RANGE}"
kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.10 --port=8080
kubectl expose deployment hello-minikube --type=NodePort
kubectl get pod
curl $(minikube service hello-minikube --url)
kubectl delete services hello-minikube
kubectl delete deployment hello-minikube
sudo -E minikube stop
sudo -E minikube delete
```

## Explanation

`PROXY_HOST` and `PROXY_PORT` should contain the hostname and port number of
your corporate HTTP proxy.

`VM_IP` is the IP address of your Linux VM. You should be able to find it using
`ifconfig` or `ip addr`.

Setting `CHANGE_MINIKUBE_NONE_USER=true` will make Minikube to move `.kube` and
`.minikube` from root directory to your home and set files ownership to your
user. Once `minikube start` completes it will print the commands you would need
to do otherwise - so you will know what exactly Minkube did for you.

`SERVICE_CLUSTER_IP_ADDRESSES="$(echo 10.96.0.{1..255} | sed 's/ /,/g')"` - the
right side of the assignment will be expanded to value
`10.96.0.1,10.96.0.2,...,10.96.0.255`. These IP addresses will be assigned by
Minikube to the services it will run as we will start it with
`--service-cluster-ip-range` parameter - see below. We use the value of
`SERVICE_CLUSTER_IP_ADDRESSES` in `no_proxy` variable to ensure that any
requests to those IP are made directly rather than going through the HTTP proxy.
Note that the name of the variable (`SERVICE_CLUSTER_IP_ADDRESSES`) is not
important - you could name it `FOO` as long as you properly reference it in
`no_proxy` value.

`SERVICE_CLUSTER_IP_RANGE="10.96.0.0/24"` is the value passed in
`--service-cluster-ip-range` parameter when starting Minikube. It is the IP
address range that Minikube will use to assign service addresses. The important
thing is that this IP range must cover all the IP addresses that were generated
when setting `SERVICE_CLUSTER_IP_ADDRESSES` variable above, which in this case
would be 10.96.0.1, 10.96.0.2, …, 10.96.0.255.

`export http{,s}_proxy=http://${PROXY_HOST}:${PROXY_PORT}` will set `http_proxy`
and `https_proxy` variables to your corporate proxy. The `http{,s}_proxy` is
[BASH brace expansion](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_03_04.html)
trick for the lazy typers like me.

`export no_proxy="localhost,127.0.0.1,${VM_IP},${SERVICE_CLUSTER_IP_ADDRESSES}"`
sets the list of IP addresses the connections to which must not go through HTTP
proxy. The list includes all services addresses assigned by Minikube.

The things to note in `minikube start` command are:
* `sudo -E` starts Minikube as root (required) and ensures that it can connect
  to the Internet to download its components. **IMPORTANT** - your Docker
  installation must be also configured to allow pulling images through [HTTP
  proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)
* `--vm-driver=none --apiserver-ips=127.0.0.1 --apiserver-name=localhost` tells
  Minikube to use Docker engine you have running on your Linux VM rather than a
  hypervisor
* `--service-cluster-ip-range="${SERVICE_CLUSTER_IP_RANGE}"` overwrites the
  default value of `10.96.0.0/16` and matches the IP addresses we used in
  `no_proxy` variable
* `--logtostderr -v 0` are useful debug options that helped me to figure out why
  my Minikube was not starting initially and what I should put in `no_proxy` to
  make it work.
