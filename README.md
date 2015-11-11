# Introduction

We are going to turn on three CoreOS VMs under vagrant and set them under various configurations to show off different failure domains of Kubernetes and how to handle them in production.

```
git clone https://github.com/coreos/coreos-vagrant
cd coreos-vagarnt
git clone https://github.com/philips/real-world-kubernetes
sed -e 's%num_instances=1%num_instances=3%g' < config.rb.sample > config.rb
```

NOTE: please use the latest version of CoreOS alpha box

```
vagrant box update --box coreos-alpha
vagrant box update --box coreos-alpha --provider vmware_fusion
```

Now lets startup the hosts

```
vagrant up
vagrant status
```

And configure ssh to talk to the new vagrant hosts correctly:

```
vagrant ssh-config > ssh-config
alias ssh="ssh -F ssh-config"
alias scp="scp -F ssh-config"
```

This should show three healthy CoreOS hosts launched.

# etcd clustering

For etcd we are going to scale the cluster from a single machine up to a three machine cluster. Then we will fail a machine and show everything is still working.

## Single Machine

Setup an etcd cluster with a single machine on core-01. This is as easy as starting the etcd2 service on CoreOS.

```
vagrant up
vagrant ssh-config > ssh-config
vagrant ssh core-01
sudo systemctl start etcd2
systemctl status etcd2
```

Confirm that you can write into etcd now:

```
etcdctl set kubernetes rocks
etcdctl get kubernetes
```

Now, we can confirm the cluster configuration with the `etcdctl member list` subcommand.

```
etcdctl member list
```

By default etcd will listen on localhost and not advertise a public address. We need to fix this before adding additional members. First, tell the cluster the members new address. The IP is the default IP for core-01 in coreos-vagrant.

```
etcdctl member update ce2a822cea30bfca http://172.17.8.101:2380
Updated member with ID ce2a822cea30bfca in cluster
```

Let's reconfigure etcd to listen on public ports to get it ready to cluster. 

```
sudo su 
sudo mkdir /etc/systemd/system/etcd2.service.d/
cat  <<EOM > /etc/systemd/system/etcd2.service.d/10-listen.conf
[Service]
Environment=ETCD_NAME=core-01
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://172.17.8.101:2379
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
EOM
```

All thats left is to restart etcd and the reconfiguration should be complete.

```
sudo systemctl daemon-reload
sudo systemctl restart etcd2
etcdctl get kubernetes
```

## Add core-02 to the Cluster

Now that core-01 is ready for clustering lets add our first additional cluster member, core-02.

```
vagrant ssh core-02
etcdctl --peers http://172.17.8.101:2379 set /foobar baz
```

Login to core-02 and lets add it to the cluster:

```
etcdctl --peers http://172.17.8.101:2379 member add core-02 http://172.17.8.102:2380
```

The above command will dump out a bunch of initial configuration information. Next, we will put that configuration information into the systemd unit file for this member:

```
sudo su
mkdir /etc/systemd/system/etcd2.service.d/
cat  <<EOM > /etc/systemd/system/etcd2.service.d/10-listen.conf
[Service]
Environment=ETCD_NAME=core-02
Environment=ETCD_INITIAL_CLUSTER=core-01=http://172.17.8.101:2380,core-02=http://172.17.8.102:2380
Environment=ETCD_INITIAL_CLUSTER_STATE=existing

Environment=ETCD_ADVERTISE_CLIENT_URLS=http://172.17.8.102:2379
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
EOM
```

```
sudo systemctl daemon-reload
sudo systemctl restart etcd2
etcdctl member list
```

Now, at this point the cluster is in an unsafe configuration. If either machine fails etcd will stop working.

```
sudo systemctl stop etcd2
exit
vagrant ssh core-01
sudo systemctl set kubernetes bad
sudo systemctl get kubernetes
exit
vagrant ssh core-02
sudo systemctl start etcd2
sudo systemctl set kubernetes awesome
```

## Add core-03 to the Cluster

To get out of this unsafe configuration lets add a third member. After adding this third member this cluster will be able to survive single machine failures.

```
vagrant ssh core-03
etcdctl --peers http://172.17.8.101:2379 member add core-03 http://172.17.8.103:2380
```

```
sudo su
mkdir /etc/systemd/system/etcd2.service.d/
cat  <<EOM > /etc/systemd/system/etcd2.service.d/10-listen.conf
[Service]
Environment=ETCD_NAME=core-03
Environment=ETCD_INITIAL_CLUSTER=core-01=http://172.17.8.101:2380,core-02=http://172.17.8.102:2380,core-03=http://172.17.8.103:2380
Environment=ETCD_INITIAL_CLUSTER_STATE=existing

Environment=ETCD_ADVERTISE_CLIENT_URLS=http://172.17.8.103:2379
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
EOM

```

```
sudo systemctl daemon-reload
sudo systemctl restart etcd2
etcdctl member list
```

## Surviving Machine Failure

Now we can have a single machine fail, like core-01, and have the cluster configure to set and retrieve values.

```
vagrant destroy core-01
vagrant ssh core-02
etcdctl set foobar asdf
```

## Automatic Bootstrapping

This exercise was designed to get you comfortable with etcd bringup and reconfiguration. In environments where you have deterministic IP address you can use [static cluster bringup](https://coreos.com/etcd/docs/latest/clustering.html#static). In environments with dynamic IPs you can use an [etcd discovery](https://coreos.com/etcd/docs/latest/clustering.html#discovery).

## Cleanup

For the next exercise we are only going to use a single member etcd cluster. Lets destroy the machines and bring up clean hosts:

```
vagrant destroy -f core-01
vagrant destroy -f core-02
vagrant destroy -f core-03
```

# Disaster Recovery of etcd

In most all environments etcd will be replicated. But, etcd is generally holding onto critical data so you should plan for backups and disaster recovery. This example will cover restoring etcd from backup.

## Start a Cluster and Destroy

Bring up the cluster of three machines

```
vagrant up
vagrant ssh-config > ssh-config
```

Startup a single machine etcd cluster on core-01 and launch a process that will write a key now every 5 seconds with the current date.

```
ssh core-02
systemctl start etcd2
sudo systemd-run /bin/sh -c 'while true; do  etcdctl set now "$(date)"; sleep 5; done'
exit
```

Backup the etcd cluster state to a tar file and save it on the local filesystem. In a production cluster this could be done with a tool like rclone in a container to save it to an object store or another server.

```
ssh core-02 sudo tar cfz - /var/lib/etcd2 > backup.tar.gz
ssh core-02 etcdctl get now
vagrant destroy -f core-02
vagrant up core-02
vagrant ssh-config > ssh-config
```

## Restore from Backup

First, lets restore the data from the etcd member to the new host location:

```
scp backup.tar.gz core-01:
ssh core-01
tar xzvf backup.tar.gz
sudo su
mv var/lib/etcd2/member /var/lib/etcd2/
chown -R etcd /var/lib/etcd2
```

Next we need to tell etcd to start but to only use the data, not the cluster configuration. We do this by setting a flag called FORCE_NEW_CLUSTER. This is somthing like "single user mode" on a Linux host.

```
mkdir -p /run/systemd/system/etcd2.service.d
cat  <<EOM > /run/systemd/system/etcd2.service.d/10-new-cluster.conf
[Service]
Environment=ETCD_FORCE_NEW_CLUSTER=1
EOM
systemctl daemon-reload
systemctl restart etcd2
```

To ensure we don't accidently reset the cluster configuration in the future, remove the force new cluster option and flush it from systemd.

```
rm /run/systemd/system/etcd2.service.d/10-new-cluster.conf
systemctl daemon-reload
```

Now, we should have our database fully recovered with the application data intact. From here we can rebuild the cluster using the methods from the first section.

```
etcdctl member list
etcdctl get now
```

## Cleanup

```
vagrant destroy -f core-01
vagrant destroy -f core-02
vagrant destroy -f core-03
```

# Securing etcd

Now that we have good practice with cluster operations of etcd under network partition, adding/removing members, and backups lets add transport security to the machine that will act as our etcd machine: core-01.

```
vagrant up
vagrant ssh-config > ssh-config
```

## Generate Certificate Authority

First, lets generate a certificate authority and some certificates signed by that authority. You can take a look at the makefile but it essentially using the [cfssl](https://github.com/cloudflare/cfssl) tool to generate a CA and an etcd cert signed by that CA.

```
pushd real-world-kubernetes/tls-setup
make install-cfssl
make
popd
```

Now drop the certs onto the host:

```
scp -r real-world-kubernetes/tls-setup/certs core-01:
```

## Use CA with etcd

Install the newly generated certificates onto the host. In a real-world environment this would be done with a cloud-config or installed on first boot.

```
ssh core-01
sudo su
mkdir /etc/etcd
mv certs/etcd* /etc/etcd
chown -R etcd: /etc/etcd
cp certs/ca.pem /etc/ssl/certs/
/usr/bin/c_rehash
exit
exit
```

Finally, prepare etcd to use a certificate and key file that are dropped onto the host.

```
ssh core-01
sudo su
mkdir /etc/systemd/system/etcd2.service.d/
cat  <<EOM > /etc/systemd/system/etcd2.service.d/10-listen.conf
[Service]
Environment=ETCD_NAME=core-01
Environment=ETCD_ADVERTISE_CLIENT_URLS=https://core-01:2379
Environment=ETCD_LISTEN_CLIENT_URLS=https://0.0.0.0:2379
Environment=ETCD_CERT_FILE=/etc/etcd/etcd.pem
Environment=ETCD_KEY_FILE=/etc/etcd/etcd-key.pem
EOM
systemctl daemon-reload
systemctl restart etcd2
exit
exit
```

## Test with etcdctl

With everything in place we should be able to set a key over a secure connection:

```
ssh core-01
etcdctl --peers https://core-01:2379 --ca-file certs/ca.pem set kubernetes is-ready
```

# Running Kubernetes API Server

Now that we fully understand etcd and how to operate it securely and in clusters lets bringup a Kubernetes API server.


```
CONTROLLER=core-01
```

Get the basic configuration files in place on the server.

```
scp -r real-world-kubernetes/k8s-setup ${CONTROLLER}:
scp -r real-world-kubernetes/k8s-srv-setup ${CONTROLLER}:
```

Then copy them over to the right locations and restart the kubelet to have it bootstrap the API server.


```
ssh ${CONTROLLER}
sudo su
mkdir -p /etc/kubernetes/ssl/
cp certs/ca.pem /etc/kubernetes/ssl/
cp certs/apiserver* /etc/kubernetes/ssl/
cp k8s-setup/kubelet.service /etc/systemd/system/kubelet.service
mkdir -p /etc/kubernetes/manifests/
cp k8s-setup/kube-*.yaml /etc/kubernetes/manifests/
mkdir -p /srv/kubernetes/manifests
cp k8s-srv-setup/*.yaml /srv/kubernetes/manifests
systemctl daemon-reload
systemctl restart kubelet.service
systemctl enable kubelet
exit
exit
```

## Test API

At this point the API should be up and available. But, we need a DNS entry to point at; so lets set that up first on our workstations. NOTE: this IP will change based on host configuration. 

```
export CORE_01_IP=$(cat ssh-config | grep HostName | awk '{print $2}' | head -n1)
sudo -E /bin/sh -c 'echo "${CORE_01_IP} core-01" >> /etc/hosts'
```

With DNS configured we can try `kubectl` with our pre-made configuration file:

```
export KUBECONFIG=real-world-kubernetes/kubeconfig
kubectl get pods
```

If all goes well we shoudl get an empty list of pods! Now, lets add some worker nodes.

# Kubernetes API Server Under etcd Failure

## Temporary Partition

Lets start a really boring job on the cluster that just sleeps forever. This goes through just fine:

```
kubectl run pause --image=gcr.io/google_containers/pause
```

Next, we will stop etcd simulating a partition:

```
ssh core-01 sudo systemctl stop etcd2
```

Attempting to do any API call to the server is going to fail blocked on etcd. This behavior is identical to if you had a web service and stopped its SQL database.

```
kubectl describe rc pause
```

Lets start up etcd and get things going:

```
ssh core-01 sudo systemctl start etcd2
``` 

After a few seconds the API server should start responding and we should be able to get the status of our replication controller:

```
kubectl describe rc pause
```

## Data-loss and Restore

Lets run a really boring application in this cluster with no nodes:

```
kubectl run pause --image=gcr.io/google_containers/pause
```

And take a quick backup of etcd:

```
ssh core-01 sudo tar cfz - /var/lib/etcd2 > backup.tar.gz
```

```
kubectl scale rc pause --replicas=5
kubectl describe rc pause
```

```
ssh core-01 sudo systemctl stop etcd2
```

```
ssh core-01
sudo su
mkdir tmp
mv /etc/kubernetes/manifests/kube-* tmp/
rm -Rf /var/lib/etcd2/*
exit
docker ps
exit
```

```
scp backup.tar.gz core-01:
ssh core-01
tar xzvf backup.tar.gz
sudo su
mv var/lib/etcd2/member /var/lib/etcd2/
chown -R etcd /var/lib/etcd2
systemctl start etcd2.service
mv tmp/* /etc/kubernetes/manifests
exit
etcdctl --peers https://core-01:2379 --ca-file certs/ca.pem set kubernetes is-ready
exit
```

```
kubectl describe rc pause
kubectl scale rc pause --replicas=1
```

# Kubernetes Workers 

## Setup Workers

Lets setup core-02 and core-03 as the worker machines.

```
WORKER=core-02
```

```
scp -r real-world-kubernetes/worker-setup ${WORKER}:
scp -r real-world-kubernetes/tls-setup/certs ${WORKER}:
ssh ${WORKER}
sudo su
mkdir -p /etc/kubernetes/ssl/ /etc/kubernetes/manifests
cp certs/ca.pem /etc/kubernetes/ssl/
cp certs/worker* /etc/kubernetes/ssl/
cp worker-setup/kubelet.service /etc/systemd/system/kubelet.service
cp worker-setup/kube-*.yaml /etc/kubernetes/manifests/
cp worker-setup/worker-kubeconfig.yaml /etc/kubernetes
systemctl daemon-reload
systemctl restart kubelet.service
systemctl enable kubelet
exit
exit
```

Now re-run the above after changing the worker variable to setup core-03:

```
WORKER=core-03
```

At this point we should see two machines listed in the set of machines

```
kubectl get nodes
```

## Individual Worker Failure


```
kubectl describe rc pause
kubectl scale rc pause --replicas=10
```

```
vagrant halt core-01
```

Now after about one minute we should notice everything has been moved off of core-03. Why? It is because of a thing called the 

```
kubectl describe node core-03 core-02
```

# High Availability of API Server

This is easy, the API server is trivially horizontally scalable.

Set the controller to core-02 and re-run the controller provisioning steps from before against this host.

```
CONTROLLER=core-02
```

Now test out that the kubernetes services are running.

```
kubectl -s https://core-02 get pods
```

# High Availability of Scheduler/Controller-Manager

```

```

# Authenticating API Server