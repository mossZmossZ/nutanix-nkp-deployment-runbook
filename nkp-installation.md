# NKP Deployment Runbook
> NKP v2.17.1 — Nutanix Kubernetes Platform · Air-gapped · Harbor Registry

---

## 🔧 Environment Variables (Edit Before Use)

```bash
# ── Domain & Registry ────────────────────────────────────────────
export DOMAIN="example.local"
export REGISTRY_HOSTNAME="registry.example.local"
export REGISTRY_IP="192.168.1.10"
export REGISTRY_PROJECT="nkp"
export REGISTRY_URL="https://${REGISTRY_IP}/${REGISTRY_PROJECT}"
export REGISTRY_USERNAME="admin"
export REGISTRY_PASSWORD="<Registry-Password>"
export REGISTRY_CA="/home/nutanix/certs/${DOMAIN}.crt"
export REGISTRY_CERT="/home/nutanix/certs/${DOMAIN}.cert"
export REGISTRY_KEY="/home/nutanix/certs/${DOMAIN}.key"
export SSH_KEY_FILE="/home/nutanix/certs/rsa_key"

# ── Harbor VM ────────────────────────────────────────────────────
export HARBOR_VM_USER="nutanix"
export HARBOR_VM_PASS="<Harbor-VM-Password>"
export HARBOR_VERSION="v2.15.0"

# ── Bastion VM ───────────────────────────────────────────────────
export BASTION_USER="nutanix"
export BASTION_PASS="<Bastion-VM-Password>"

# ── NKP Cluster ──────────────────────────────────────────────────
export CLUSTER_NAME="nkp-mgmt-cluster"
export NUTANIX_PC_ENDPOINT="<Lab-Assign>"
export NUTANIX_USER="<Lab-Assign>"
export NUTANIX_PASSWORD='<Lab-Assign>'
export CONTROL_PLANE_IP="<Lab-Assign>"
export LB_IP_RANGE="<Lab-Assign>"
export IMAGE_NAME="nkp-rocky-9.5-release-1.32.3-20250605191120.qcow2"
export PRISM_ELEMENT_CLUSTER_NAME="<Lab-Assign>"
export SUBNET_NAME="<Lab-Assign>"
export NUTANIX_STORAGE_CONTAINER_NAME="default"
export KUBECONFIG_PATH="/home/nutanix/nkp-v2.17.1/${CLUSTER_NAME}.conf"

# ── Cluster Sizing ───────────────────────────────────────────────
export CP_REPLICAS="1"
export CP_VCPUS="4"
export CP_CORES_PER_VCPU="1"
export CP_MEMORY_GB="12"

export WORKER_REPLICAS="3"
export WORKER_VCPUS="12"
export WORKER_CORES_PER_VCPU="1"
export WORKER_MEMORY_GB="32"
```

---

## 1 · Harbor VM — Generate TLS Certificates

```bash
mkdir certs && cd certs

# CA key & certificate
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
  -subj "/C=TH/ST=BKK/L=BKK/O=example/OU=nkp/CN=*.${DOMAIN} Root CA" \
  -key ca.key -out ca.crt

# Server key & CSR
openssl genrsa -out ${DOMAIN}.key 4096
openssl req -sha512 -new \
  -subj "/C=TH/ST=BKK/L=BKK/O=example/OU=nkp/CN=${DOMAIN}" \
  -key ${DOMAIN}.key -out ${DOMAIN}.csr

# SAN extension file
cat > v3.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1=${REGISTRY_HOSTNAME}
DNS.2=${DOMAIN}
IP.1=${REGISTRY_IP}
EOF

# Sign the certificate
openssl x509 -req -sha512 -days 3650 \
  -extfile v3.ext \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -in ${DOMAIN}.csr \
  -out ${DOMAIN}.crt

# Convert to .cert format (PEM)
openssl x509 -inform PEM -in ${DOMAIN}.crt -out ${DOMAIN}.cert
```

---

## 2 · Harbor VM — Trust & Docker Setup

```bash
# Trust the CA system-wide
sudo cp ${REGISTRY_CA} /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Configure Docker to trust registry
sudo mkdir -p /etc/docker/certs.d/${DOMAIN}/
sudo cp ${REGISTRY_CERT} /etc/docker/certs.d/${DOMAIN}/
sudo cp ${REGISTRY_KEY}  /etc/docker/certs.d/${DOMAIN}/
sudo cp ${REGISTRY_CA}   /etc/docker/certs.d/${DOMAIN}/

sudo systemctl restart docker
```

---

## 3 · Harbor VM — Install Harbor

```bash
cd /home/nutanix/

# Download & extract
curl -Lo harbor-offline-installer-${HARBOR_VERSION}.tgz \
  "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-offline-installer-${HARBOR_VERSION}.tgz"
tar -zxvf harbor-offline-installer-${HARBOR_VERSION}.tgz

cd harbor
cp harbor.yml.tmpl harbor.yml

# Edit harbor.yml — set these values:
#   hostname: ${REGISTRY_HOSTNAME}
#   certificate: /home/nutanix/certs/${DOMAIN}.crt
#   private_key: /home/nutanix/certs/${DOMAIN}.key
vi harbor.yml

./prepare
sudo ./install.sh --with-trivy
```

> **Harbor UI:** `https://${REGISTRY_IP}`  
> Default admin password: `Harbor12345` → change to `${REGISTRY_PASSWORD}`

---

## 4 · Bastion VM — Copy Certs & Trust

```bash
# Copy certs from Harbor VM
mkdir -p /home/nutanix/certs
scp -r root@${REGISTRY_IP}:/home/nutanix/certs/ /home/nutanix/

# Trust the CA
sudo cp /home/nutanix/certs/${DOMAIN}.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Configure Docker
sudo mkdir -p /etc/docker/certs.d/${DOMAIN}/
sudo cp /home/nutanix/certs/${DOMAIN}.cert /etc/docker/certs.d/${DOMAIN}/
sudo cp /home/nutanix/certs/${DOMAIN}.key  /etc/docker/certs.d/${DOMAIN}/
sudo cp /home/nutanix/certs/${DOMAIN}.crt  /etc/docker/certs.d/${DOMAIN}/

sudo systemctl restart docker
```

---

## 5 · Bastion VM — Install CLI Tools

### kubectl

```bash
# Install bash-completion (if not already installed)
dnf install -y bash-completion

# Install kubectl (latest stable)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl

# Enable bash completion & alias
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl"                   >> ~/.bashrc
echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
```

### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

source <(helm completion bash)
echo "source <(helm completion bash)" >> ~/.bashrc
```

### k9s

```bash
wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar zxvf k9s_Linux_amd64.tar.gz
mv k9s /usr/local/bin/
rm -f k9s_Linux_amd64.tar.gz
```

---

## 6 · Bastion VM — Generate SSH Key Pair

```bash
# Generate ed25519 key (no passphrase)
ssh-keygen -t ed25519 -f ${SSH_KEY_FILE} -N ''

# Display public key (copy this for reference)
cat ${SSH_KEY_FILE}.pub
```

---

## 7 · Bastion VM — Push NKP Bundles to Registry

```bash
# Push Konvoy image bundle
nkp push bundle \
  --bundle ./container-images/konvoy-image-bundle-v2.17.1.tar \
  --to-registry=${REGISTRY_URL} \
  --to-registry-username=${REGISTRY_USERNAME} \
  --to-registry-password=${REGISTRY_PASSWORD} \
  --to-registry-ca-cert-file=${REGISTRY_CA}

# Push Kommander image bundle
nkp push bundle \
  --bundle ./container-images/kommander-image-bundle-v2.17.1.tar \
  --to-registry=${REGISTRY_URL} \
  --to-registry-username=${REGISTRY_USERNAME} \
  --to-registry-password=${REGISTRY_PASSWORD} \
  --to-registry-ca-cert-file=${REGISTRY_CA}

# Load bootstrap image locally
docker load -i konvoy-bootstrap-image-v2.17.1.tar
```

---

## 8 · Bastion VM — Deploy NKP Cluster

### Verify Variables

```bash
echo "CLUSTER_NAME               = $CLUSTER_NAME"
echo "NUTANIX_PC_ENDPOINT        = $NUTANIX_PC_ENDPOINT"
echo "NUTANIX_USER               = $NUTANIX_USER"
echo "NUTANIX_PASSWORD           = $NUTANIX_PASSWORD"
echo "CONTROL_PLANE_IP           = $CONTROL_PLANE_IP"
echo "LB_IP_RANGE                = $LB_IP_RANGE"
echo "IMAGE_NAME                 = $IMAGE_NAME"
echo "PRISM_ELEMENT_CLUSTER_NAME = $PRISM_ELEMENT_CLUSTER_NAME"
echo "SUBNET_NAME                = $SUBNET_NAME"
echo "NUTANIX_STORAGE_CONTAINER  = $NUTANIX_STORAGE_CONTAINER_NAME"
echo "REGISTRY_URL               = $REGISTRY_URL"
echo "REGISTRY_USERNAME          = $REGISTRY_USERNAME"
echo "REGISTRY_PASSWORD          = $REGISTRY_PASSWORD"
echo "REGISTRY_CA                = $REGISTRY_CA"
echo "SSH_KEY_FILE               = $SSH_KEY_FILE"
echo "--- Sizing ---"
echo "CP   replicas=${CP_REPLICAS}  vcpu=${CP_VCPUS}  cores=${CP_CORES_PER_VCPU}  mem=${CP_MEMORY_GB}GB"
echo "WRKR replicas=${WORKER_REPLICAS}  vcpu=${WORKER_VCPUS}  cores=${WORKER_CORES_PER_VCPU}  mem=${WORKER_MEMORY_GB}GB"
```

### Create Cluster

```bash
nkp create cluster nutanix -c $CLUSTER_NAME -v 6 \
  --endpoint https://${NUTANIX_PC_ENDPOINT}:9440 \
  --control-plane-endpoint-ip $CONTROL_PLANE_IP \
  --control-plane-vm-image $IMAGE_NAME \
  --control-plane-prism-element-cluster $PRISM_ELEMENT_CLUSTER_NAME \
  --control-plane-subnets $SUBNET_NAME \
  --control-plane-replicas $CP_REPLICAS \
  --control-plane-vcpus $CP_VCPUS \
  --control-plane-cores-per-vcpu $CP_CORES_PER_VCPU \
  --control-plane-memory $CP_MEMORY_GB \
  --worker-vm-image $IMAGE_NAME \
  --worker-prism-element-cluster $PRISM_ELEMENT_CLUSTER_NAME \
  --worker-subnets $SUBNET_NAME \
  --worker-replicas $WORKER_REPLICAS \
  --worker-vcpus $WORKER_VCPUS \
  --worker-cores-per-vcpu $WORKER_CORES_PER_VCPU \
  --worker-memory $WORKER_MEMORY_GB \
  --kubernetes-service-load-balancer-ip-range ${LB_IP_RANGE}-${LB_IP_RANGE} \
  --csi-storage-container $NUTANIX_STORAGE_CONTAINER_NAME \
  --ssh-public-key-file ${SSH_KEY_FILE}.pub \
  --insecure \
  --self-managed \
  --airgapped \
  --timeout=0 \
  --registry-mirror-url $REGISTRY_URL \
  --registry-mirror-cacert $REGISTRY_CA \
  --registry-mirror-username=$REGISTRY_USERNAME \
  --registry-mirror-password=$REGISTRY_PASSWORD
```

---

## 9 · Post-Deploy — Access NKP Dashboard

```bash
nkp get dashboard --kubeconfig="${KUBECONFIG_PATH}"
```

---

## 📋 Quick Reference

| Item | Value |
|---|---|
| Domain | `example.local` |
| Registry Hostname | `registry.example.local` |
| Registry IP | `192.168.1.10` |
| Registry Project URL | `https://192.168.1.10/nkp` |
| Harbor UI | `https://192.168.1.10` |
| Cert Path | `/home/nutanix/certs/` |
| SSH Key Path | `/home/nutanix/certs/rsa_key` |
| Harbor Version | `v2.15.0` |
| NKP Version | `v2.17.1` |
| Control Plane Replicas | `$CP_REPLICAS` (default: 1) |
| Control Plane vCPU / RAM | `$CP_VCPUS` vCPU / `$CP_MEMORY_GB` GB |
| Worker Replicas | `$WORKER_REPLICAS` (default: 3) |
| Worker vCPU / RAM | `$WORKER_VCPUS` vCPU / `$WORKER_MEMORY_GB` GB |