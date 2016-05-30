# Startup scripts for IASSIST cluster

Below are the steps used to create www.iassist.ndslabs.org cluster.

## Provision nodes, install Kubernetes servies
Start the deploy-tools container
```
docker run -v `pwd`:/root/SAVED_AND_SENSITIVE_VOLUME --name deploy-test-cluster -it ndslabs/deploy-tools:nds244 bash
```

Run the ansible playbook (inventory in ansible/inventory):
```
ansible-playbook -i inventory/iassist1.5 playbooks/ndslabs-k8.yml
```

## Manual steps 

SSH to master.  Stop the addons service and correct the node labels:
```
ssh -i SAVED_AND_SENSITIVE_VOLUME/iassist1.pem core@192.168.100.67

sudo systemctl stop kube-addons.service

kubectl label node iassist1-node1 ndslabs-node-role=compute
kubectl label node iassist1-node2 ndslabs-node-role=compute
kubectl label node iassist1-node3 ndslabs-node-role=compute
kubectl label node iassist1-node4 ndslabs-node-role=compute
kubectl label node iassist1-loadbal ndslabs-node-role=loadbalancer
```

Clone this repo. Start the DNS service
```
git clone https://github.com/craig-willis/ndslabs-iassist-startup

kubectl create -f addons/namespace.yaml
kubectl create -f addons/dns/skydns-svc.yaml 
kubectl create -f addons/dns/skydns-rc.yaml 
kubectl get pods --namespace=kube-system

#Wait for fluent and dns to be ready
```

Start the cluster monitoring services:
```
kubectl create -f addons/cluster-monitoring/heapster-service.yaml
kubectl create -f addons/cluster-monitoring/heapster-controller.yaml
kubectl create -f addons/cluster-monitoring/influxdb-grafana-controller.yaml
kubectl create -f addons/cluster-monitoring/influxdb-service.yaml
kubectl create -f addons/cluster-monitoring/grafana-service.yaml
```


##Setup Gluster

Start Gluster server daemonsets:
```
kubectl create -f gluster-server-ds.yaml
kubectl get pods -o wide

kubectl exec -it <pod> bash

#Nodes
#192.168.100.61
#192.168.100.63

gluster peer probe 192.168.100.63

gluster volume create ndslabs transport tcp 192.168.100.61:/media/brick0/brick 192.168.100.63:/media/brick0/brick
gluster volume start ndslabs
```

Start Gluster client daemonsets
```
kubectl create -f gluster-client-ds.yaml
```

For each node 60, 62, 64, 65
```
mkdir -p /ndslabs/data
```

For each client pod
```
kubectl exec -it <pod>
mount -t glusterfs 192.168.100.61:/ndslabs /ndslabs/data
```

On one node, create the volumes directory and open permissions (for now)
```
mkdir /ndslabs/data/volumes
chmod -R a+rw /ndslabs/data/volumes
```

##Start NDSLabs services

Start API server and GUI
```
kubectl create -f apiserver.yaml
kubectl create -f gui.yaml
```

Start the loadbalancer and default ingress rules
```
kubectl create -f default-backend.yaml
kubectl create -f loadbalancer.yaml
kubectl create -f default-ingress.yaml
```

Manually add security groups


Cache images
```
for image in `cat images`
do
  docker pull $image
done
```

