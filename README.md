In Kubernetes, each container can read and write to its own file system. But when a container is restarted, all data is lost. Therefore, containers that need to maintain state would store data in a persistent storage such as Network File System (NFS). What’s already stored in NFS isn't deleted when a pod, which might contain one or more containers, is destroyed. Also, an NFS can be accessed from multiple pods at the same time, so an NFS can be used to share data between pods. 

This behavior is really useful when containers or applications need to read configuration data from a single shared file system or when multiple containers need to read from and write data to a single shared file system.

Oracle Cloud Infrastructure File Storage provides a durable, scalable, and distributed enterprise-grade network file system that supports NFS version 3 along with Network Lock Manager (NLM) for a locking mechanism. You can connect to File Storage from any bare metal, virtual machine, or container instance in your virtual cloud network (VCN). You can also access a file system from outside the VCN by using Oracle Cloud Infrastructure FastConnect or an Internet Protocol Security (IPSec) virtual private network (VPN). File Storage is a fully managed service so you don't have to worry about hardware installations and maintenance, capacity planning, software upgrades, security patches,  and so on. You can start with a file system that contains only a few kilobytes of data and grows to handle 8 exabytes of data.


This workshop explains how to use File Storage (sometimes referred to as FSS) with Container Engine for Kubernetes and we will see how pods in different nodes share the same File Storage file system.

![image](https://user-images.githubusercontent.com/57708209/139249533-8242109c-2d84-42e5-9904-e7d2abb80291.png)



Create a OKE Cluster (using Quick create and default settings) which creates Kubernetes API Endpoint in public subnet(oke-k8sApiEndpoint-subnet-quick-OKEFile-67ad59148-regional), worker nodes in private subnet (oke-nodesubnet-quick-OKEFile-67ad59148-regional	)and load balancer in seperate public subnet(oke-svclbsubnet-quick-OKEFile-67ad59148-regional).

![image](https://user-images.githubusercontent.com/57708209/139245792-2e21718e-0fae-4f39-ad57-e620f1f75ba0.png)

Now Create a File Storage with default settings and Mount target in same VCN as used above (in my case oke-vcn-quick-OKEFile-67ad59148 and public subnet as oke-k8sApiEndpoint-subnet-quick-OKEFile-67ad59148-regional as shown)

![image](https://user-images.githubusercontent.com/57708209/139246412-9978065e-5f75-495f-aa44-e193f321f4a8.png)

In this scenario, the mount target that exports the file system is in a different subnet (oke-k8sApiEndpoint-subnet-quick-OKEFile-67ad59148-regional) than the instance you want to mount the file system to(oke-nodesubnet-quick-OKEFile-67ad59148-regional	). Security rules must be configured for both the mount target and the instance either in a security list for each subnet, or a network security group (NSG) for each resource.

Set up the following the following security rules for the mount target. Specify the instance IP address or CIDR block as the source for ingress rules and the destination for egress rules:

Stateful ingress from ALL ports in the source instance CIDR block to TCP ports 111, 2048, 2049, and 2050.
Stateful ingress from ALL ports in the source instance CIDR block to UDP ports 111 and 2048.
Stateful egress from TCP ports 111, 2048, 2049, and 2050 to ALL ports in the destination instance CIDR block.
Stateful egress from UDP port 111 ALL ports in the destination instance CIDR block.
 
 ![image](https://user-images.githubusercontent.com/57708209/139247379-bafbfc06-7473-4a6e-aea2-123d9848eb6c.png)
 
 (Last 4 rules are added new)

![image](https://user-images.githubusercontent.com/57708209/139247424-2bd2fee5-b73e-4f58-b15c-54ba5592279c.png)

 (Last 3 rules are added new)

Next, set up the following security rules for the instance. Specify the mount target IP address or CIDR block as the source for ingress rules and the destination for egress rules:


Stateful ingress from source mount target CIDR block TCP ports 111, 2048, 2049, and 2050 to ALL ports.
Stateful ingress from source mount target CIDR block UDPport 111 to ALL ports.
Stateful egress from ALL ports to destination mount target CIDR block TCP ports 111, 2048, 2049, and 2050.
Stateful egress from ALL ports to destination mount target CIDR block UDP ports 111 and 2048.


![image](https://user-images.githubusercontent.com/57708209/139247587-5d6f5fc9-a7e6-48b9-8d63-92d795a1f316.png)

(Last3 are added new)

![image](https://user-images.githubusercontent.com/57708209/139247696-0247f060-62e2-41b6-9695-7395721fd9a6.png)

 (Last 4 rules are added new)
 
 For reference  may refer : https://docs.oracle.com/en-us/iaas/Content/File/Tasks/securitylistsfilestorage.htm
 
## High-Level Steps
Create storage class.
Create a persistent volume (PV).
Create a persistent volume claim (PVC).
Create a pod to consume the PVC.


Now go to your cluster and click Access Cluster and run Kubectl get nodes , to check if you are able to fetch your cluster nodes

oci ce cluster create-kubeconfig --cluster-id ocid1.cluster.oc1.phx.aaaaaaaah6wvn73ex4jfsalinitsvgcuhfwdvl7qibi4dvmlpcrxpo4uegaq --file $HOME/.kube/config --region us-phoenix-1 --token-version 2.0.0  --kube-endpoint PUBLIC_ENDPOINT
kubectl get nodes
vim StorageClass.yaml
kubectl create -f StorageClass.yaml 

vim PersistentVolume.yaml
kubectl create -f PersistentVolume.yaml 
vim PersitentVolumeClaim.yaml
kubectl create -f PersitentVolumeClaim.yaml 
kubectl get pvc oke-fsspvc

Label your nodes: get the ip address of 2 of your nodes and label as  node1 and node2

kubectl label node 10.0.10.93 nodeName=node1
kubectl label node 10.0.10.236 nodeName=node2

The following pod (oke-fsspod) on Worker Node 1 (node1) consumes the file system PVC (oke-fsspvc).
vim okefsspod.yaml
kubectl create -f okefsspod.yaml 
kubectl get pods oke-fsspod -o wide

Write to the File System by Using kubectl exec:

kubectl exec -it oke-fsspod bash
Execute :
echo "Hello from POD1" >> /usr/share/nginx/html/hello_world.txt
cat /usr/share/nginx/html/hello_world.txt
![image](https://user-images.githubusercontent.com/57708209/139248584-c38ce85d-85c3-44e3-be44-82622e0a7ff1.png)

Repeat the Process with the Other Pod
Ensure that this file system can be mounted into the other pod (oke-fsspod2), which is on Worker Node 2 (node2):

vim okefsspod2.yaml
cp okefsspod.yaml okefsspod2.yaml
vim okefsspod2.yaml
kubectl create -f okefsspod2.yaml 
kubectl get pods oke-fsspod oke-fsspod2 -o wide
TEST if new pod can read from the shared file storage:

kubectl exec -it oke-fsspod2 -- cat /usr/share/nginx/html/hello_world.txt
![image](https://user-images.githubusercontent.com/57708209/139249130-44f82d98-f99f-4c80-9e52-afbba223d670.png)


You can also test that the newly created pods can write to the share:
kubectl exec -it oke-fsspod2 -- cat /usr/share/nginx/html/hello_world.txt
Execute:
echo "Hello from POD2" >> /usr/share/nginx/html/hello_world.txt
cat /usr/share/nginx/html/hello_world.txt
![image](https://user-images.githubusercontent.com/57708209/139248864-e72abc40-4b67-4f12-b100-600a87456943.png)


