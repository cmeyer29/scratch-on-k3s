# scratch-on-k3s
This repo describes how to implement a scratch game compiled to html within a nginx web server running on a k3s single node cluster

## Prerequisites
You need a small Virtual Machine with Ubuntu Desktop 22.04 LTS.

## Overview
For deploying the game there are several task to do
- Installing a standard k3s Single-Node-Clusters 
- Installing helm for the application deployment
- Installing the nginx Webserver-Deployments
- DNS Entry configuration (if you do not a have a DNS Server instead)
- Starting the Website

## k3s Deployment
### Installing a standard k3s Single-Node-Clusters 

```bash
## https://github.com/k3s-io/k3s/releases
## https://docs.k3s.io/security/hardening-guide
## https://docs.k3s.io/security/self-assessment
## https://docs.k3s.io/security/secrets-encryption
export K3S_VERSION="1.24.6+k3s1"
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"
export NODE_NAME=$(hostname -f)
export NODE_IP=$(ip -4 -o a | grep -i ens3 | tr -s ' ' | cut -d ' ' -f4 | cut -d '/' -f1)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v${K3S_VERSION}" sh -s - \
  --disable metrics-server \
  --node-name="${NODE_NAME}" \
  --tls-san="${NODE_IP}" \
  --disable-cloud-controller \
  --write-kubeconfig-mode=0644
sudo systemctl status k3s -l
sudo sed -i "s/127.0.0.1/${NODE_IP}/g" /etc/rancher/k3s/k3s.yaml
printf '\nKUBECONFIG="/etc/rancher/k3s/k3s.yaml"\n' | sudo tee -a /etc/environment
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ingressClass:
      enabled: true
      isDefaultClass: true
EOF

sudo systemctl restart k3s

```
## k3s Testing and Configuration
```bash
sudo systemctl status k3s -l
kubectl get nodes -A
kubectl get addon -A
kubectl get pods --all-namespace
echo "#####################"
echo "#####################"
echo "#####################"
echo "theKUBECONFIG variabel must be set manually! Please execute the following command!"
echo "export KUBECONFIG="/etc/rancher/k3s/k3s.yaml" "

```
### Helm Installation
In order to install helm you can use the following script

```bash
export ARCH="amd64"
export ARCH_X=$(uname -m)
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
[[ ${ARCH_X} == "x86_64" ]] \
  && (export ARCH="amd64" && export ARCH_BIT="64bit") \
  || (export ARCH=${ARCH_X} && export ARCH_BIT=${ARCH_X})
export HELM_VERSION="3.10.0"
wget -O /tmp/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz \
  https://get.helm.sh/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz
pushd /tmp
wget -O helm-v${HELM_VERSION}-linux-${ARCH}.tmp \
  https://get.helm.sh/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz.sha256sum
cat helm-v${HELM_VERSION}-linux-${ARCH}.tmp | grep helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz > helm-v${HELM_VERSION}-linux-${ARCH}.sha256sum
[[ "$(sha256sum -c helm-v${HELM_VERSION}-linux-${ARCH}.sha256sum)" == *"OK" ]] || exit 1
rm -rf helm-v${HELM_VERSION}-linux-${ARCH}.tmp \
       helm-v${HELM_VERSION}-linux-${ARCH}.sha256sum
popd
sudo tar -zxvf /tmp/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz \
  --strip-components=1 \
  -C /usr/local/bin \
  linux-${ARCH}/helm
rm -rf /tmp/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz
sudo chown root:root /usr/local/bin/helm
sudo chmod 755 /usr/local/bin/helm
sudo ln -s /usr/local/bin/helm /usr/local/bin/helm3
helm list --all-namespaces
```
### Installing the nginx Webserver-Deployments
For the deployment we are using the standard public nginx helm charts from bitnami. You can optionally fetch the whole chart data from the repository.
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install merrychristmas bitnami/nginx \
  --namespace merrychristmas \
  --create-namespace \
  --timeout 600s \
  --set global.storageClass="localPath" \
  --set cloneStaticSiteFromGit.enabled=true \
  --set cloneStaticSiteFromGit.repository="https://github.com/cmeyer29/scratch-on-k3s.git" \
  --set cloneStaticSiteFromGit.branch="master" \
  --set cloneStaticSiteFromGit.interval=3600 \
  --set service.type="ClusterIP" \
  --set ingress.enabled=true \
  --set ingress.hostname="merrychristmas.toyou" \
  --set ingress.path="/"  \
  --set ingress.ingressClassName="traefik"
```

### DNS Configuration on the host
you need to set a dns entry in the hosts data /etc/hosts 
```bash
sudo sed -i '/^127.0.0.1/ s/$/ merrychristmas.toyou /' /etc/hosts
```

### DNS Configuration within the CoreDNS ConfigMap
You need the set an entry within the CoreDNS ConfigMap for a functioning DNS Resolution
```bash
kubectl edit cm -n kube-system coredns
```
### Reloading the CoreDNS ConfigMap
in order to reload the changed ConfigMap we have to delete the current CoreDNS pod
```bash
kubectl delete pods -n kube-system $(kubectl get pods -n kube-system | grep -i coredns | cut -d' ' -f1)
```

### Testing
Testing via Curl
```bash
curl -k http://merrychristmas.toyou
```
### Starting the site 
```bash
firefox --new-window http://merrychristmas.toyou
```
![Image](src:images/christmas-game.jpg)
### Deinstall nginx
```bash
helm uninstall merrychristmas -n merrychristmas 
```
### Deinstall k3s
```bash
/usr/local/bin/k3s-uninstall.sh
```

### Sources

- [k3s Installation](https://docs.k3s.io/installation)
- [helm Installation](https://helm.sh/docs/helm/helm_install/)
- [Bitnami Nginx Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/nginx/)
- [Scratch 3.0](https://scratch.mit.edu/)
