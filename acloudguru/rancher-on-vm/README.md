# Rancher on A Cloud Guru VMs

## Machine type
- OS: OpenSUSE Leap 15.5
- size: 4

## 1. Check virtual ip address

Export env vars:
```bash
export HOSTNAME="fb3d98993e1c.mylabserver.com"
export PUBLIC_IP="35.179.121.214"
```

Check virtual ip address from my pc:
```bash
andreagrillo@KPDOK77R6M6Y9P ~ % host fb3d98993e1c.mylabserver.com
fb3d98993e1c.mylabserver.com has address 35.179.121.214
```

## 2. Fix grub configurations to enable cgroupv2
Login:
```bash
ssh cloud_user@${HOSTNAME}
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

Login and check cgroup version again:
```bash
fb3d98993e1c:/home/cloud_user # stat -fc %T /sys/fs/cgroup
cgroup2fs
```

## 3. Install rke2 kubernetes cluster on the node
Login:
```bash
ssh cloud_user@${HOSTNAME}
```

Get administration permissions:
```bash
sudo su
```

Create directory for rancher configurations:
```bash
mkdir -p /etc/rancher/rke2
```

Create config file:
```bash
cat > /etc/rancher/rke2/config.yaml <<EOF
tls-san:
  - $(hostname)
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

Copy the kubeconfig file:
```bash
scp -i ./key cloud_user@${HOSTNAME}:/home/cloud_user/rke2.yaml ${HOME}/.kube/rancher.yaml
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

## 6. Install MetalLB
Add helm repo:
```bash
helm repo add metallb https://metallb.github.io/metallb
```

```bash
helm install metallb metallb/metallb --create-namespace -n metallb-system --version 0.16.1 --kubeconfig=${HOME}/.kube/rancher.yaml --kube-insecure-skip-tls-verify

NAME: metallb
LAST DEPLOYED: Thu Sep  3 17:41:00 2026
NAMESPACE: metallb-system
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
MetalLB is now running in the cluster.
```

Verify pods:
```bash
kubectl get pods -n metallb-system -w --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
```

Advertise address pool:
```bash
cat <<EOF | kubectl apply --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: ingress-l2-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - ingress-ippool
EOF
```

Create address pool:
```bash
cat <<EOF | kubectl apply --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-ippool
  namespace: metallb-system
spec:
  addresses:
  - 172.31.100.169/32
  serviceAllocation:
    priority: 100
    serviceSelectors:
    - matchExpressions:
      - {key: app.kubernetes.io/name, operator: In, values: [rke2-ingress-nginx]}
EOF
```

Create nginx ingress:
```bash
cat <<EOF | kubectl apply --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify -f -
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      config:
        use-forwarded-headers: "true"
        enable-real-ip: "true"
      publishService:
        enabled: true
      service:
        enabled: true
        type: LoadBalancer
        externalTrafficPolicy: Local
EOF
```

kubectl get HelmChartConfig -n kube-system --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify
kubectl get svc -n kube-system --kubeconfig=${HOME}/.kube/rancher.yaml --insecure-skip-tls-verify