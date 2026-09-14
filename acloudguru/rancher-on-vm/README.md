# Rancher on A Cloud Guru VMs

## Machine type
- OS: OpenSUSE Leap 15.5
- size: 8

## 1. Check virtual ip address

Export env vars:
```bash
export HOSTNAME="fb3d98993e1c.mylabserver.com"
export PUBLIC_IP="52.214.179.233"
export PRIVATE_IP="172.31.27.185"
```

Check virtual ip address from my pc:
```bash
andreagrillo@KPDOK77R6M6Y9P ~ % host fb3d98993e1c.mylabserver.com
fb3d98993e1c.mylabserver.com has address 35.176.108.146
```

## 2. Fix grub configurations to enable cgroupv2
Copy the ssh key:
```bash
make copy-key
```

Login:
```bash
make ssh
```

Get administration permissions:
```bash
sudo su
```

Check cgroup version: tmpfs -> cgroupv1
```bash
fb3d98993e1c:/home/cloud_user # stat -fc %T /sys/fs/cgroup
tmpfs
```

use the openSUSE/GRUB method:
```bash
vi /etc/default/grub
```

Find:
```
GRUB_CMDLINE_LINUX="..."
```

and add:
```
systemd.unified_cgroup_hierarchy=1
```

For example:
```
GRUB_CMDLINE_LINUX=".... systemd.unified_cgroup_hierarchy=1"
```

Then regenerate GRUB:
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Then reboot the node:
```bash
reboot
```

Login again:
```bash
make ssh
```

Get sudo access:
```bash
sudo su
```

Login and check cgroup version again:
```bash
fb3d98993e1c:/home/cloud_user # stat -fc %T /sys/fs/cgroup
cgroup2fs
```

## 3. Install rke2 kubernetes cluster on the node

Create directory for rancher configurations:
```bash
mkdir -p /etc/rancher/rke2
```

Create config file:
```bash
cat > /etc/rancher/rke2/config.yaml <<EOF
tls-san:
  - $(hostname).mylabserver.com
EOF
```

Install the rke2-server service and the rke2 binary onto your machine:
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.36.4+rke2r1 sh -
```

Enable and start the rke2-server service:
```bash
systemctl enable --now rke2-server.service
```

Copy the kubeconfig file to a temporary location:
```bash
cp /etc/rancher/rke2/rke2.yaml /home/cloud_user/rke2.yaml
chown cloud_user:users /home/cloud_user/rke2.yaml
```

## 3. Download kubeconfig file

Exit from the machine:
```bash
exit
exit
```

Copy the kubeconfig file locally:
```bash
scp -i ./ssh/key cloud_user@${HOSTNAME}:/home/cloud_user/rke2.yaml ${HOME}/.kube/rancher.yaml
```

Replace the address in the kubeconfig file:
```bash
sed -i "" "s/127.0.0.1/${PUBLIC_IP}/g" ${HOME}/.kube/rancher.yaml
```

List nodes:
```bash
kubectl get nodes --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify

NAME           STATUS   ROLES                AGE   VERSION
fb3d98993e1c   Ready    control-plane,etcd   17m   v1.36.4+rke2r1
```

List pods:
```bash
kubectl get pods -A --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify

NAMESPACE     NAME                                                   READY   STATUS      RESTARTS      AGE
kube-system   cloud-controller-manager-fb3d98993e1c                  1/1     Running     0             17m
kube-system   etcd-fb3d98993e1c                                      1/1     Running     0             17m
kube-system   helm-install-rke2-canal-vrmsr                          0/1     Completed   0             17m
kube-system   helm-install-rke2-coredns-dzjv4                        0/1     Completed   0             17m
kube-system   helm-install-rke2-metrics-server-ddhlt                 0/1     Completed   0             17m
kube-system   helm-install-rke2-runtimeclasses-dwwrf                 0/1     Completed   0             17m
kube-system   helm-install-rke2-snapshot-controller-6lhcp            0/1     Completed   2 (16m ago)   17m
kube-system   helm-install-rke2-snapshot-controller-crd-6ndsj        0/1     Completed   0             17m
kube-system   helm-install-rke2-traefik-crd-9hcv7                    0/1     Completed   0             17m
kube-system   helm-install-rke2-traefik-js978                        0/1     Completed   2 (16m ago)   17m
kube-system   kube-apiserver-fb3d98993e1c                            1/1     Running     0             17m
kube-system   kube-controller-manager-fb3d98993e1c                   1/1     Running     0             17m
kube-system   kube-proxy-fb3d98993e1c                                1/1     Running     0             17m
kube-system   kube-scheduler-fb3d98993e1c                            1/1     Running     0             17m
kube-system   rke2-canal-6tzhj                                       2/2     Running     0             17m
kube-system   rke2-coredns-rke2-coredns-6b85489767-ctv9l             1/1     Running     0             17m
kube-system   rke2-coredns-rke2-coredns-autoscaler-8fcd75dfd-ckzks   1/1     Running     0             17m
kube-system   rke2-metrics-server-9d9d8b8f-px87z                     1/1     Running     0             16m
kube-system   rke2-snapshot-controller-5d85999559-s8nw4              1/1     Running     0             16m
kube-system   rke2-traefik-f6wdz                                     1/1     Running     0             16m
```

## 4. Generate certificates
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 365 -nodes -subj "CN=mylabserver.com"
```

Generate the CA private key:
```bash
openssl genrsa -out cert/ca.key 4096
```

Generate the CA certificate expiring in 1 year:
```bash
openssl req -x509 -new -nodes \
  -key cert/ca.key \
  -sha256 \
  -days 365 \
  -out cert/ca.crt \
  -subj "/CN=mylabserver.com"
```

Generate the TLS/server private key:
```bash
openssl genrsa -out cert/tls.key 2048
```

Create a certificate signing request
```bash
openssl req -new \
  -key cert/tls.key \
  -out cert/tls.csr \
  -subj "/CN=mylabserver.com"
```

Create tls.ext:
```bash
cat > cert/tls.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
DNS.1=${HOSTNAME}
IP.1=${PUBLIC_IP}
EOF
```

Sign the TLS certificate with your CA
```bash
openssl x509 -req \
  -in cert/tls.csr \
  -CA cert/ca.crt \
  -CAkey cert/ca.key \
  -CAcreateserial \
  -out cert/tls.crt \
  -days 825 \
  -sha256 \
  -extfile cert/tls.ext
```

Create cacerts.pem:
```bash
cat cert/ca.crt > cert/cacerts.pem
```

You should have:
```
ca.key       # CA private key
ca.crt       # CA certificate
cacerts.pem  # CA trust bundle
tls.key      # TLS/server private key
tls.crt      # TLS/server certificate
tls.csr      # CSR, optional after signing
ca.srl       # CA serial-number file, optional
```

Remove files that not useful anymore:
```bash
rm cert/ca.srl
rm cert/tls.csr
rm cert/tls.ext
```

Verify that the server certificate was signed by your CA:
```bash
openssl verify -CAfile cert/ca.crt cert/tls.crt

cert/tls.crt: OK
```

And verify that the private key matches the certificate:
```bash
openssl x509 -in cert/tls.crt -pubkey -noout | openssl sha256
openssl pkey -in cert/tls.key -pubout | openssl sha256

SHA2-256(stdin)= c87372ea8ccc24a8002fd2bb2601f9e2a6c7a9035f5a64307dd0af84e8c96c74
SHA2-256(stdin)= c87372ea8ccc24a8002fd2bb2601f9e2a6c7a9035f5a64307dd0af84e8c96c74
```

## 5. Create certificate secrets

Create namespace:
```bash
kubectl create ns cattle-system --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
```

Create tls secret:
```bash
kubectl create secret tls tls-rancher-ingress -n cattle-system --cert=cert/tls.crt --key=cert/tls.key --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
```

Create CA secret:
```bash
kubectl create secret generic tls-ca --from-file=cert/cacerts.pem -n cattle-system --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
```

Verify the secrets:
```bash
kubectl get secret -n cattle-system --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
```

## 6. Install Rancher server
Add repo:
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

Install the rancher helm with certificates from file:
```bash
helm upgrade --install rancher rancher-latest/rancher \
  --kubeconfig=${HOME}/.kube/rancher.yaml --kube-insecure-skip-tls-verify \
  --namespace cattle-system \
  --set hostname=${HOSTNAME} \
  --set bootstrapPassword=admin \
  --set replicas=1 \
  --version 2.15.1 \
  --set ingress.tls.source=secret \
  --set privateCA=true


Release "rancher" does not exist. Installing it now.
NAME: rancher
LAST DEPLOYED: Mon Sep  7 12:27:04 2026
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
Rancher Server has been installed. Rancher may take several minutes to fully initialize.

Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

## First Time Login

If you provided your own bootstrap password during installation, browse to https://fb3d98993e1c.mylabserver.com to get started.
If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:

echo https://fb3d98993e1c.mylabserver.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

To get just the bootstrap password on its own, run:

kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'


Happy Containering!
```

Monitor the deployment:
```bash
kubectl -n cattle-system rollout status deploy/rancher --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
kubectl -n cattle-system get po -w --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
kubectl -n cattle-system logs -f deploy/rancher --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
```

## When restarting the A Cloud Guru machine
Copy the kubeconfig file locally:
```bash
export HOSTNAME="fb3d98993e1c.mylabserver.com"
cd acloudguru/rancher-on-vm
scp -i ./ssh/key cloud_user@${HOSTNAME}:/home/cloud_user/rke2.yaml ${HOME}/.kube/rancher.yaml
```

Replace the address in the kubeconfig file:
```bash
export PUBLIC_IP="34.243.45.96"
sed -i "" "s/127.0.0.1/${PUBLIC_IP}/g" ${HOME}/.kube/rancher.yaml
```

## Cleanup
Uninstall rancher server:
```bash
helm uninstall rancher -n cattle-system --kubeconfig=${HOME}/.kube/rancher.yaml --kube-insecure-skip-tls-verify
```