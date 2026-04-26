# NKP Runbook — Change FQDN & Apply Custom TLS (Post-Deployment)

## Objective

Update an existing Nutanix Kubernetes Platform (NKP) deployment to:

* Use a custom FQDN (e.g. `nkp.domain.com`)
* Apply a trusted TLS certificate (e.g. wildcard `*.domain.com`)
* Ensure browser trust and proper HTTPS access

---

## Scope

* NKP **already deployed**
* Kommander (UI) already running
* DNS already prepared

---

## Prerequisites

* Valid TLS certificate:

  * `cert.pem`
  * `key.pem`
* kubectl access to cluster
* kubeconfig configured

```bash
export KUBECONFIG=<your-cluster>.conf
```

---

## Step 1 — Verify Cluster Access

```bash
kubectl get nodes
```

---

## Step 2 — Create TLS Secret

Namespace must match Kommander (`kommander`)

```bash
kubectl create secret tls nkp-ui-tls \
  --cert=cert.pem \
  --key=key.pem \
  -n kommander
```

---

## Step 3 — Update Kommander FQDN

Patch `KommanderCluster`:

```bash
kubectl patch kommandercluster host-cluster \
  -n kommander \
  --type merge \
  -p '{
    "spec": {
      "ingress": {
        "hostname": "nkp.domain.com"
      }
    }
  }'
```

---

## Step 4 — Verify Ingress Update

```bash
kubectl describe kommandercluster -n kommander host-cluster
```

Expected:

```
IngressAddressReady: True
IngressCertificateReady: True
```

---

## Step 5 — Patch Ingress TLS (Required)

NKP 2.17 does NOT support custom TLS via `KommanderCluster`.

Patch all ingresses manually:

```bash
for i in $(kubectl get ingress -n kommander -o name); do
  kubectl patch "$i" -n kommander --type merge -p '{
    "spec": {
      "tls": [
        {
          "hosts": ["nkp.domain.com"],
          "secretName": "nkp-ui-tls"
        }
      ]
    }
  }'
done
```

---

## Step 6 — Verify Certificate

### Option A — curl

```bash
curl -vk https://nkp.domain.com
```

### Option B — openssl

```bash
openssl s_client -connect nkp.domain.com:443 -servername nkp.domain.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

## Expected Result

* CN / SAN = `*.domain.com`
* No `traefik.localhost.localdomain`
* No `kommander-ca`

---

## Step 7 — DNS Validation

Ensure:

```
nkp.domain.com → <Ingress IP>
```

Check:

```bash
kubectl get svc -n kommander kommander-traefik
```

---

## Known Limitations

### 1. TLS patch is NOT persistent

* NKP controllers may overwrite ingress
* This is a **temporary workaround**

---

### 2. Default behavior

NKP uses:

* Internal CA (`kommander-ca`)
* Traefik default cert

---

### 3. Proper solution (production)

Use:

* cert-manager
* ClusterIssuer
* Kommander advanced configuration

---

## Troubleshooting

### Certificate not updated

Check ingress:

```bash
kubectl get ingress -n kommander -o yaml | grep secretName
```

---

### Still seeing default cert

Cause:

* Traefik using default TLS store

Next step:

* Patch Traefik deployment (advanced)

---

### DNS issues

```bash
nslookup nkp.domain.com
```

---

## Summary

| Task           | Status           |
| -------------- | ---------------- |
| Set FQDN       | KommanderCluster |
| Apply TLS      | Ingress patch    |
| Verify         | curl / openssl   |
| Persistent fix | cert-manager     |

---

## Recommendation

For production:

* Avoid manual ingress patch
* Implement cert-manager + ClusterIssuer

---

