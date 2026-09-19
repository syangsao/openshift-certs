# OpenShift Certificate Management

Ansible playbooks (and manual procedures) for managing TLS certificates on OpenShift clusters using ACME-generated certificates (acme.sh). Two certificate surfaces are covered:

- **Default ingress (apps)** — the wildcard cert served by the router for `*.apps.<domain>` (web console, CLI, and all applications under the `.apps` subdomain)
- **API server** — the named cert served by the kube-apiserver for `api.<domain>` so external clients can verify the API endpoint

## Playbooks

| Playbook | Purpose |
|---|---|
| [`update_apps_cert.yml`](update_apps_cert.yml) | Update the default ingress (apps) TLS certificate |

## Prerequisites

- **ansible-core** >= 2.14
- **oc** CLI with cluster-admin access
- **openssl** CLI
- **acme.sh** installed and configured (DNS provider: EasyDNS in this example)
- ACME certificate already issued for the apps wildcard domain (`*.apps.<domain>`)
- ACME certificate already issued for the API FQDN (`api.<domain>`)

## Issuing an ACME Certificate

Before running the playbook, issue or renew your certificates. You need two ACME certificates: a **wildcard** cert for the apps subdomain and a **single-domain** cert for the API FQDN.

```bash
# Install acme.sh if not already installed
curl -s https://get.acme.sh | sh

# 1) Wildcard certificate for the apps subdomain (DNS-01 challenge)
acme.sh --issue --dns dns_easydns --domain '*.apps.luke.syangsao.net' --force

# 2) Single-domain certificate for the API server FQDN (DNS-01 challenge)
acme.sh --issue --dns dns_easydns --domain 'api.luke.syangsao.net' --force

# Certificates are stored in:
#   Apps: ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer   (cert + CA chain)
#         ~/.acme.sh/*.apps.luke.syangsao.net_ecc/*.apps.luke.syangsao.net.key
#   API:  ~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer      (cert + CA chain)
#         ~/.acme.sh/api.luke.syangsao.net_ecc/api.luke.syangsao.net.key
```

Verify both certificates before proceeding:

```bash
# Apps wildcard — SAN must show *.apps.<domain>
openssl x509 -in ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -text | grep -A1 'Subject Alternative Name'

# API — SAN must show api.<domain>
openssl x509 -in ~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -text | grep -A1 'Subject Alternative Name'
```

Both `fullchain.cer` files must list the leaf certificate **first**, followed by intermediate certificates, and ending with the root CA. The private keys must be **unencrypted**.

---

## Automated: Using the Ansible Playbook

### Quick Start

```bash
ansible-playbook -i inventory.ini update_apps_cert.yml \
  -e kubeconfig_path=/path/to/kubeconfig \
  -e apps_domain=apps.luke.syangsao.net \
  -e acme_home=~/.acme.sh
```

### Dry Run (Preview Only)

```bash
ansible-playbook -i inventory.ini update_apps_cert.yml \
  -e apps_domain=apps.luke.syangsao.net \
  -e dry_run=true
```

### Playbook Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `kubeconfig_path` | Yes | `$KUBECONFIG` or `~/.kube/config` | Path to kubeconfig file |
| `apps_domain` | Yes | — | Apps domain (e.g. `apps.luke.syangsao.net`) |
| `acme_home` | No | `~/.acme.sh` | ACME.sh home directory |
| `ingress_namespace` | No | `openshift-ingress` | Router namespace |
| `ingress_controller_namespace` | No | `openshift-ingress-operator` | IngressController namespace |
| `ingress_controller_name` | No | `default` | IngressController name |
| `backup_dir` | No | `/tmp/openshift-certs-backup` | Backup destination |
| `dry_run` | No | `false` | Preview without applying changes |

### What the Playbook Does

1. **Checks current apps certificate** — extracts and inspects expiry from the `router-ca` secret
2. **Validates ACME certificate** — verifies `~/.acme.sh/` has a valid cert for `*.apps.<domain>`
3. **Backs up configuration** — saves IngressController YAML and current cert secret before changes
4. **Creates TLS secret** — deploys `router-custom-certs` secret from ACME cert
5. **Updates IngressController** — patches `defaultCertificate` to use the new secret
6. **Waits for rollout** — monitors router deployment rollout completion
7. **Validates** — connects to the apps domain and verifies the live cert matches the ACME cert

---

## Manual: Step-by-Step Without the Playbook

If you prefer to update the apps certificate manually, follow these steps.

### Step 1: Check Current Certificate Status

```bash
# Get the current default certificate secret name
oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate.name}'

# Extract and inspect the current certificate
oc get secret router-ca -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -subject -dates
```

### Step 2: Verify ACME Certificate Exists

```bash
# Check the ACME certificate is valid and not expired
openssl x509 -in ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -checkend 0

# Verify the SAN matches your apps domain
openssl x509 -in ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  -noout -text | grep -A1 'Subject Alternative Name'
```

### Step 3: Backup Current Configuration

```bash
# Backup the IngressController configuration
oc get ingresscontroller default -n openshift-ingress-operator -o yaml \
  > /tmp/ingresscontroller_backup.yaml

# Backup the current certificate secret
oc get secret router-ca -n openshift-ingress -o yaml \
  > /tmp/router-ca_backup.yaml
```

### Step 4: Create TLS Secret from ACME Certificate

```bash
oc create secret tls router-custom-certs \
  --cert=~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  --key=~/.acme.sh/*.apps.luke.syangsao.net_ecc/*.apps.luke.syangsao.net.key \
  -n openshift-ingress
```

### Step 5: Update IngressController

```bash
oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate":{"name":"router-custom-certs"}}}'
```

### Step 6: Wait for Rollout

```bash
oc rollout status deploy/router-default -n openshift-ingress --timeout=300s
```

### Step 7: Verify the New Certificate

```bash
# From a host that can reach the cluster routers:
echo | openssl s_client -connect apps.luke.syangsao.net:443 \
  -servername apps.luke.syangsao.net 2>/dev/null | \
  openssl x509 -noout -subject -dates

# Or verify the secret on the cluster:
oc get secret router-custom-certs -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -subject -dates
```

### Rollback (If Something Goes Wrong)

```bash
# Restore the IngressController to use the original secret
oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate":{"name":"router-ca"}}}'

# Wait for rollout
oc rollout status deploy/router-default -n openshift-ingress --timeout=300s
```

---

## Manual: Adding the API Server Certificate

The default API server certificate is issued by an internal OpenShift cluster CA, so clients outside the cluster cannot verify it. Add a **named certificate** so the kube-apiserver returns your public-CA cert when the client requests `api.<domain>` (e.g. via SNI through a load balancer or reverse proxy).

> **Warning:** Do **not** add a named certificate for the internal load balancer hostname `api-int.<cluster>.<base_domain>` — doing so leaves the cluster in a degraded state.

### API Step 1: Check Current State

```bash
# Current named certificates (empty on a fresh cluster)
oc get apiserver cluster -o jsonpath='{.spec.servingCerts.namedCertificates}'
```

### API Step 2: Verify the ACME API Certificate

```bash
openssl x509 -in ~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -checkend 0

# SAN must show api.<domain>
openssl x509 -in ~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer \
  -noout -text | grep -A1 'Subject Alternative Name'
```

### API Step 3: Create the TLS Secret

Create the secret in `openshift-config` (this is where the kube-apiserver reads named certificates from):

```bash
oc create secret tls api-luke-custom-cert \
  --cert=~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer \
  --key=~/.acme.sh/api.luke.syangsao.net_ecc/api.luke.syangsao.net.key \
  -n openshift-config
```

### API Step 4: Patch the APIServer to Reference the Secret

```bash
oc patch apiserver cluster --type=merge -p '{
  "spec": {
    "servingCerts": {
      "namedCertificates": [
        {
          "names": ["api.luke.syangsao.net"],
          "servingCertificate": {"name": "api-luke-custom-cert"}
        }
      ]
    }
  }
}'
```

### API Step 5: Wait for the Rollout

Adding a named certificate **for the first time** triggers the kube-apiserver-operator to roll out a new revision of the API server pods (no node reboots). Watch the operator until `PROGRESSING` returns to `False` and `AVAILABLE` is `True`:

```bash
oc get clusteroperators kube-apiserver -w
```

> **Note:** On a freshly installed cluster, `kube-apiserver` may already be `PROGRESSING=True` for an unrelated reason (e.g. the node installer still converging to the install revision). The cert rollout is complete once **all** kube-apiserver pods serve the new certificate — verify with Step 6 rather than relying on the operator condition alone.

### API Step 6: Verify the New Certificate Is Served

```bash
# From a host that can reach the API endpoint:
echo | openssl s_client -connect api.luke.syangsao.net:6443 \
  -servername api.luke.syangsao.net 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates
# The issuer should now be your public CA (e.g. ZeroSSL), not the internal cluster CA.
```

### API Step 7: Update Client Trust (Important)

Once the API server serves the public-CA certificate, **clients that previously trusted the internal cluster CA will fail to verify** the API endpoint (`x509: certificate signed by unknown authority`). This includes `oc`/`kubectl` using a kubeconfig whose `certificate-authority-data` is the internal cluster CA.

Point your kubeconfig's cluster trust at the public CA chain (the same `fullchain.cer` used for the secret):

```bash
oc config set-cluster <context-cluster> \
  --certificate-authority=~/.acme.sh/api.luke.syangsao.net_ecc/fullchain.cer
```

Any other client (scripts, CI, monitoring agents) that talks to `api.<domain>:6443` must also trust the public CA. Internal cluster workloads are unaffected — they continue to use the internal service CA.

### API Rollback (If Something Goes Wrong)

Remove the named certificate entry to restore the default internal-CA behavior:

```bash
oc patch apiserver cluster --type=merge -p '{
  "spec": {
    "servingCerts": {
      "namedCertificates": []
    }
  }
}'

# Restore your kubeconfig to trust the internal cluster CA again:
oc config set-cluster <context-cluster> \
  --certificate-authority-data="$(oc get secret <cluster-ca-secret> -n openshift-config-managed -o jsonpath='{.data:.ca\.bundle\.crt}' | base64 -d)"
```

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| `oc version --short` not recognized | Remove `--short` flag — not supported on oc < 4.21 |
| Certificate not serving after patch (apps) | Wait for router rollout to complete; check `oc get ingresscontroller default -n openshift-ingress-operator` |
| SAN mismatch error | Re-issue the ACME cert with the correct wildcard/FQDN domain |
| `Could not find certificate from stdin` | The apps domain is not reachable from the playbook host — verify from a host that can reach the cluster routers |
| Secret already exists | The playbook removes the old `router-custom-certs` secret before creating a new one |
| API still serves internal CA after patch | The kube-apiserver revision hasn't rolled out yet — wait for all pods to restart, then re-check with `openssl s_client` |
| `x509: certificate signed by unknown authority` after API cert change | Your client/kubeconfig still trusts the internal cluster CA — update it to trust the public CA chain (see API Step 7) |
| API intermittently fails verification during rollout | During the apiserver rollout, pods serve different certs (old vs new). Wait until **all** pods serve the new cert before relying on a single CA in the kubeconfig |
| `kube-apiserver` stuck `PROGRESSING=True` | On fresh clusters this is often the node installer converging to the install revision, not the cert change — check the condition message and verify the served cert directly |

## Security

- No credentials, tokens, or cluster-specific values are hardcoded
- All sensitive paths are passed as variables
- Backups are created with `0600` file permissions

## References

- [OpenShift: Replacing the default ingress certificate](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/scalability_and_performance/replacing-default-ingress-certificate)
- [OpenShift: Adding API server certificates (named certificates)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/security_and_compliance/index#api-server-certificates)
- [acme.sh documentation](https://github.com/acmesh-official/acme.sh)
