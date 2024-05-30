# Create a k3s cluster on EC2 nodes

Create 2 EC2 instances:
- >= t3.small
- SUSE operating system
- with key pair for SSH
- security group for ports 22 and 6443
 
SSH into one instance and install **k3s-server**:
```
curl -sfL https://get.k3s.io | sh -
```

Get the token:
```
sudo /var/lib/rancher/k3s/server/node-token
```

Test that the cluster is created:
```
sudo kubectl get node
```

SSH into the other instance and install **k3s-agent**:
```
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

To check if the agent node can communicate with the server node:
```
curl -vk https://<k3s-server private ip>:6443/cacerts
```

After the k3s-agent is installed, check again the nodes on the server node:
```
sudo kubectl get node
```

# Add the Traefik Load Balancer port to the AWS security group
K3S automatically installs a Traefik ingress controller in the cluster.
It works like a Node Port service so it is necessary to add the port to the security group:
```
sudo kubectl get svc -A
```
Get the node port associated to port 80 and add it to the security group.
Now you can use the node port from one node to connect to the other node.

# Create a test app

Create a pod running Nginx:
```
sudo kubectl run nginx --image=nginx --restart=Never
```

# Create a LoadBalancer
Create a NLB Load Balancer on AWS and create a Target Group pointing to the node port of Traefik.
- listener rule: tcp/80
- target group: tcp/<traefik node port>
- tcp health check

## Resources
- https://docs.k3s.io/quick-start
