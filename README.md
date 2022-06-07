![Course Cover](Cover.png "Dive Into Kubernetes Introduction")

# Dive Into Kubernetes - Introduction - Command Execution
Resources and Commands used during the Dive Into Kubernetes - Introduction Course

## Updates / Remarks

* On Kubernetes 1.24, you may also need to remove the new taint of 'node-role.kubernetes.io/control-plane:NoSchedule' with ```kubectl taint nodes $k8s_master node-role.kubernetes.io/control-plane:NoSchedule-```

* During the course I state and visualise that Dockershim is deprecated in Kubernetes 1.24, the terminology should have been 'Deprecated in 1.20 and Removed in 1.24'

## Web Links -

### containerd

https://github.com/containerd/containerd

https://github.com/containerd/containerd/releases

### nerdctl

https://github.com/containerd/nerdctl

https://github.com/containerd/nerdctl/releases

### CNI Plugins

https://github.com/containernetworking/plugins

https://github.com/containernetworking/plugins/releases

### Kubeadm Install

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm

### CNI Plugins Bridge

https://www.cni.dev/plugins/current/main/bridge

## Commands -

```
# Let's switch to a root user
sudo bash

# Change to /tmp
cd /tmp

# We'll run an apt update to populate apt
apt update

# And we'll install containerd
apt install containerd

# Enable systemd to start on reboot
systemctl enable containerd
systemctl start containerd
systemctl status containerd

# Install nerdctl
wget https://github.com/containerd/nerdctl/releases/download/v0.18.0/nerdctl-0.18.0-linux-amd64.tar.gz
tar zxvf nerdctl-0.18.0-linux-amd64.tar.gz nerdctl
mv nerdctl /usr/local/bin

# Install CNI Plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
mkdir -p /opt/cni/bin/
tar zxvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/

# Run a Docker like command with Nerdctl
nerdctl run --rm -it ubuntu bash

# Install and configure kubeadm
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

sudo modprobe br_netfilter

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Initialise kubeadm with a pod-network-cidr
kubeadm init --pod-network-cidr=10.10.0.0/16

# Configure kubeadm with a kubeconfig
export KUBECONFIG=/etc/kubernetes/admin.conf

# Set an alias (update accordingly)
k8s_master=ip-172-31-93-131
echo $k8s_master

# Create a CNI configuration
vim /etc/cni/net.d/10-bridge.conf ; watch kubectl get nodes -o wide
## Paste the following and save
{
    "cniVersion": "0.3.1",
    "name": "mynet",
    "type": "bridge",
    "bridge": "mynet0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.10.0.0/16"
    }
}

# Show mynet0 bridge
ip addr

# Investigate node taints
kubectl describe node $k8s_master | more

# Remove taint
kubectl taint nodes $k8s_master node-role.kubernetes.io/master:NoSchedule-
# If you are using Kubernetes 1.24 you'll also need to remove this taint
kubectl taint nodes $k8s_master node-role.kubernetes.io/control-plane:NoSchedule-

## Not covered in the course, you may also need to remove the following taint if it is set (changed in 1.24)
kubectl taint nodes $k8s_master node-role.kubernetes.io/control-plane:NoSchedule-

# Check taint is removed
kubectl describe node $k8s_master | more

# Create a resources holder
mkdir /etc/kubernetes/resources

# Create nginx pod yaml and save
kubectl run nginx --image=nginx --dry-run=client -o yaml > /etc/kubernetes/resources/nginx_pod.yaml

# Run pod and watch them create
kubectl run nginx --image=nginx; watch kubectl get pods -o wide

# Curl the ip from above
curl 10.10.0.4

# Look behind the scenes in nerdctl
nerdctl ps -a
nerdctl namespace ls
nerdctl -n k8s.io ps -a

# Manually create a pod in Docker
docker run -d --ipc=shareable --name pause -p 8080:80 k8s.gcr.io/pause:3.6
docker run -d --name mysql -e MYSQL_DATABASE=exampledb -e MYSQL_USER=exampleuser -e MYSQL_PASSWORD=examplepass -e MYSQL_RANDOM_ROOT_PASSWORD=1 --net=container:pause --ipc=container:pause --pid=container:pause --platform linux/amd64 mysql:5.7
docker run -d --name wordpress -e WORDPRESS_DB_HOST="127.0.0.1" -e WORDPRESS_DB_USER=exampleuser -e WORDPRESS_DB_PASSWORD=examplepass -e WORDPRESS_DB_NAME=exampledb --net=container:pause --ipc=container:pause --pid=container:pause wordpress

# Optional clean up
docker stop pause mysql wordpress
docker rm pause mysql wordpress

# Back to kubernetes, hide pause output
nerdctl -n k8s.io ps -a | grep -v pause

# Terminate the nginx pod (adjust accordingly with your container id)
nerdctl -n k8s.io stop 85ca5e7d7ac6; watch kubectl get pods -o wide

# Repeat but target the pause container associated with this pod
nerdctl -n k8s.io ps -a | grep nginx
nerdctl -n k8s.io stop d7ec7a6c1ebf; watch kubectl get pods -o wide

# Cleanup
kubectl delete pod/nginx --grace-period=-1

# Create a deployment and get information
kubectl create deployment nginx --image=nginx --port 80
kubectl get deployment
kubectl get replicaset
kubectl get pods -o wide

# Investigate behind the scenes
nerdctl -n k8s.io ps -a | grep nginx

# Scale replicas
kubectl scale deployment/nginx --replicas=2; watch kubectl get pods -o wide
nerdctl -n k8s.io ps -a | grep nginx

# Stop both of the pause containers associated with the deployment (both ip addresses will change)
nerdctl -n k8s.io stop c2a0709a26df 5cacb2710a7f; watch kubectl get pods -o wide

# Expose the deployment as a ClusterIP service
kubectl expose deployment/nginx --type=ClusterIP
kubectl get service

# Curl the service ip (adjust accordingly)
curl 10.104.248.88

# Again terminate the pause containers and see that the service still works as expected (adjust accordingly)
nerdctl -n k8s.io ps -a | grep nginx | grep pause | grep Up
nerdctl -n k8s.io stop c2a0709a26df 5cacb2710a7f; watch kubectl get pods -o wide
curl 10.104.248.88

# Check DNS Resolution
kubectl run --rm -i --tty curl --image=curlimages/curl --restart=Never -- sh
curl nginx.default.svc.cluster.local
exit

# Cleanup
kubectl delete deployment/nginx
kubectl delete service/nginx

# Show components currently running
nerdctl -n k8s.io ps -a | grep -v 'k8s.gcr.io/pause' | grep Up
kubectl get all -A

# Show the kubelet running
ps -ef | grep kubelet | grep -v kube-apiserver | grep -v grep
systemctl status kubelet

# Show all components, export and terminate
kubectl get all -A
kubectl -n kube-system get daemonset.apps/kube-proxy -o yaml > /etc/kubernetes/resources/kube-proxy.yaml
kubectl -n kube-system delete daemonset.apps/kube-proxy
kubectl -n kube-system get deployment.apps/coredns -o yaml > /etc/kubernetes/resources/coredns.yaml
kubectl -n kube-system delete deployment.apps/coredns
kubectl -n kube-system get service/kube-dns -o yaml > /etc/kubernetes/resources/kube-dns.yaml
kubectl -n kube-system delete service/kube-dns
kubectl get all -A

# Move the static pod declarations
cd /etc/kubernetes/manifests
ls -l
mv * ../resources

# kubectl will now error as expected
kubectl get nodes

# Existing containers will have transitioned to Created in Nerdctl
nerdctl -n k8s.io ps -a

# Stop the kubelet
systemctl stop kubelet

# Create 2 convenient nerdctl aliases
alias nerdctl_stopall="nerdctl -n k8s.io ps -a | grep -v CONTAINER | cut -d ' ' -f 1 | xargs -n1 -i sh -c 'nerdctl -n k8s.io stop {} || true'"
alias nerdctl_rmall="nerdctl -n k8s.io ps -a | grep -v CONTAINER | cut -d ' ' -f 1 | xargs -n1 -i sh -c 'nerdctl -n k8s.io rm {} || true'"

# Stop all containers and clean up
nerdctl_stopall
nerdctl_rmall
nerdctl ps -a
nerdctl -n k8s.io ps -a

# Check containerd is running
systemctl status containerd

# Refamiliarise ourself with the CNI plugin
cat /etc/cni/net.d/10-bridge.conf

# Start the kubelet
systemctl start kubelet
ps -ef | grep kubelet

# Check the kubelet config (look at the static manifests path)
cat /var/lib/kubelet/config.yaml

# At present, nothing is running
nerdctl -n k8s.io ps -a

# Using our exported nginx pod manifest, move it to the static directory and watch it start
cd /etc/kubernetes/manifests
mv ../resources/nginx_pod.yaml . ; watch nerdctl -n k8s.io ps -a

# Inspect the container with nerdctl (adjust command accordingly)
nerdctl -n k8s.io inspect 95ab1009f84f

# And curl the corresponding ip address
curl 10.10.0.9

# Watch what the kubelet does, if we repeatedly stop these containers outside of the kubelet (repeat as desired)
nerdctl_stopall; watch nerdctl -n k8s.io ps -a

# Move the nginx pod and cleanup
mv nginx_pod.yaml ../resources/
nerdctl_stopall; nerdctl_rmall

# Start etcd using a static pod
mv ../resources/etcd.yaml .; watch nerdctl -n k8s.io ps -a

# Install etcd-client
apt install etcd-client

# Grab the etcd running parameters
ps -ef | grep etcd

# Use the output to populate etcd table command (change the endpoints)
ETCDCTL_API=3 etcdctl --endpoints https://172.31.33.172:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key  /etc/kubernetes/pki/etcd/server.key --write-out=table --endpoints=$ENDPOINTS endpoint status

# List etcd keys
ETCDCTL_API=3 etcdctl --endpoints https://172.31.33.172:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key  /etc/kubernetes/pki/etcd/server.key  get / --prefix --keys-only  | grep -v ^$ | head -10

# Put an etcd key
ETCDCTL_API=3 etcdctl --endpoints https://172.31.33.172:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key  /etc/kubernetes/pki/etcd/server.key put foo bar

# Get an etcd key
ETCDCTL_API=3 etcdctl --endpoints https://172.31.33.172:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key  /etc/kubernetes/pki/etcd/server.key get foo

# Enable the api-server as a Static Pod
mv ../resources/kube-apiserver.yaml .; watch nerdctl -n k8s.io ps -a

# Get the nodes
kubectl get nodes

# Try to run a pod (it won't schedule)
kubectl run nginx --image=nginx; watch kubectl get pods -o wide

# Enable the kube-scheduler, watch the pod spring into life
mv ../resources/kube-scheduler.yaml .; watch kubectl get pods -o wide

# Cleanup
kubectl delete pod/nginx --grace-period=0

# Create a deployment, it won't start at this point
kubectl create deployment nginx --image=nginx --port=80; watch kubectl describe deployment/nginx

# Enable the controller manager and watch the deployment spring into life (replicas will be provisioned)
mv ../resources/kube-controller-manager.yaml .; watch kubectl describe deployment/nginx

# And check our pods
kubectl get pods -o wide

# And curl the ip address
curl 10.10.0.23

# Expose the Deployment and attempt to curl the service (will fail)
kubectl expose deployment/nginx --type=ClusterIP
kubectl get service
curl 10.103.249.211

# Enable kube-proxy and retry
kubectl apply -f /etc/kubernetes/resources/kube-proxy.yaml; watch nerdctl -n k8s.io ps -a
curl 10.103.249.211

# Check DNS using curlimages (will fail)
kubectl run --rm -i --tty curl --image=curlimages/curl --restart=Never -- sh
curl nginx.default.svc.cluster.local
exit

# Enable coredns and kube-dns
kubectl apply -f /etc/kubernetes/resources/coredns.yaml
kubectl apply -f /etc/kubernetes/resources/kube-dns.yaml

# Check DNS again
kubectl run --rm -i --tty curl --image=curlimages/curl --restart=Never -- sh
curl nginx.default.svc.cluster.local
exit
```
