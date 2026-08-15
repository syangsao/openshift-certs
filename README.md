# OpenShift Certificate Management

Ansible playbooks for managing TLS certificates on OpenShift clusters using ACME-generated certificates (acme.sh).

## Playbooks

| Playbook | Purpose |
|---|---|
| [`update_apps_cert.yml`](update_apps_cert.yml) | Update the default ingress (apps) TLS certificate |

## Prerequisites

- **ansible-core** >= 2.14
- **oc** CLI with cluster-admin access
- **openssl** CLI
- **acme.sh** installed and configured (DNS provider: EasyDNS in this example)
- ACME certificate already issued for the apps wildcard domain

## Issuing an ACME Certificate

Before running the playbook, issue or renew your wildcard certificate:

```bash
# Install acme.sh if not already installed
curl -s https://get.acme.sh | sh

# Issue wildcard certificate using EasyDNS DNS-01 challenge
acme.sh --issue --dns dns_easydns --domain '*.apps.luke.syangsao.net' --force

# Certificates are stored in:
#   ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer   (certificate + CA chain)
#   ~/.acme.sh/*.apps.luke.syangsao.net_ecc/*.apps.luke.syangsao.net.key   (private key)
```

Verify the certificate before proceeding:

```bash
openssl x509 -in ~/.acme.sh/*.apps.luke.syangsao.net_ecc/fullchain.cer \
  -noout -subject -dates -text | grep -A1 'Subject Alternative Name'
```

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

## Troubleshooting

| Issue | Resolution |
|---|---|
| `oc version --short` not recognized | Remove `--short` flag — not supported on oc < 4.21 |
| Certificate not serving after patch | Wait for router rollout to complete; check `oc get ingresscontroller default -n openshift-ingress-operator` |
| SAN mismatch error | Re-issue the ACME cert with the correct wildcard domain |
| `Could not find certificate from stdin` | The apps domain is not reachable from the playbook host — verify from a host that can reach the cluster routers |
| Secret already exists | The playbook removes the old `router-custom-certs` secret before creating a new one |

## Security

- No credentials, tokens, or cluster-specific values are hardcoded
- All sensitive paths are passed as variables
- Backups are created with `0600` file permissions

## References

- [OpenShift: Replacing the default ingress certificate](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/scalability_and_performance/replacing-default-ingress-certificate)
- [acme.sh documentation](https://github.com/acmesh-official/acme.sh)
