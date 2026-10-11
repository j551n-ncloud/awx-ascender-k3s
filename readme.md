# AWX and Ascender on Single Node K3s

An example implementation of AWX and Ascender (Ctrl IQ's AWX fork, https://github.com/ctrliq/ascender) on single node K3s.

This setup allows you to access AWX and Ascender over HTTPS from a remote host.

## Table of Contents
- [Requirements](#requirements)
- [Deployment Instructions](#deployment-instructions)
- [Troubleshooting Issues](#troubleshooting-issues)
- [Useful Commands](#useful-commands)
- [Ascender](#ascender)

## Requirements

**Computing resources:**
- Both AMD64 (x86_64) with x86-64-v2 or cpu type Host support, and ARM64 (aarch64) are supported
- 4 GiB RAM minimum
- It's recommended to add more CPUs and RAM (like 4 CPUs and 8 GiB RAM or more) to avoid performance and job scheduling issues

**Storage resources:**
- At least 10 GiB for `/var/lib/rancher` for a fresh install
- The actual consumption depends on your environment and use case, so keep an eye on it and add more capacity if needed

## Deployment Instructions

Everything is driven by the `deploy-awx.yml` Ansible playbook, run from your machine (the controller) against the target host over SSH. There is no standalone shell script anymore, and no manual firewalld/no_proxy setup either: the playbook installs K3s, fixes K3s's proxy environment, and opens the firewalld rules K3s/AWX need on its own.

```bash
git clone https://github.com/j551n-ncloud/awx-ascender-k3s.git
cd awx-ascender-k3s/kubernetes
```

### Set the target host

Point `inventory.ini` at the host you're deploying to:

```ini
[awx_host]
k3s-node.example.com ansible_user=root ansible_become=true ansible_python_interpreter=/usr/bin/python3
```

### Configure proxy and LDAP

`kubernetes/awx.yml` contains example outbound proxy and LDAP settings (`example.com`, `CHANGE_ME`). Replace them with your own values, or delete the blocks you don't need. Use the same `NO_PROXY` value in the `Fix k3s systemd environment's NO_PROXY` task in `deploy-awx.yml`.

If the target host needs a proxy to reach the internet, put a script exporting `http_proxy`, `https_proxy` and `no_proxy` at `/etc/profile.d/proxy.sh` on that host (or change `proxy_env_file` in the playbooks). The playbooks source it before downloading K3s, the AWX operator and the Ascender installer, and skip it if it doesn't exist.

### Certificate

If you need to create a self-signed certificate, you can do so with this command (or point the playbook at an existing cert/key elsewhere):

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout yourdomain.key -out yourdomain.crt -subj "/CN=example.com" -addext "subjectAltName=DNS:example.com"
```

### Run the playbook

K3s includes Traefik ingress by default, so no separate nginx-ingress installation is needed. Check the [AWX-Operator releases](https://github.com/ansible/awx-operator/releases) for the latest version to enter when prompted.

The playbook installs K3s itself (skipped automatically if it's already set up), so run it straight from the `kubernetes` folder:

```bash
ansible-playbook -i inventory.ini deploy-awx.yml --ask-pass
```

You'll be prompted for the hostname, AWX version, and whether the cert/key are in the playbook folder or elsewhere. Certificate/key paths are answered relative to this machine, not the remote host, since that's where the prompt is answered.

K3s and `kubectl` live on the target host, not the controller, so the commands below (and everything in Troubleshooting/Useful Commands) are run there, e.g. `ssh root@k3s-node.example.com`.

Once deployment is complete, you can retrieve the admin password:

```bash
kubectl get secret awx-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode ; echo
```

### Troubleshooting Issues

**Check Resources:**

```bash
kubectl -n awx get awx,all,ingress,secrets
```

**Create Admin Password Secret (if not created automatically):**

```bash
kubectl -n awx create secret generic awx-admin-password --from-literal=password=<your-admin-password>
```

**Verify Ingress:**

```bash
kubectl get endpoints -n awx
```

Check if the endpoint for `awx-service` is available on port 80. If not, you may need to update `~/awx-deploy/awx-ingress-tls.yaml` on the target host.

## Useful Commands

```bash
kubectl get namespaces
kubectl -n awx get awx,all,ingress,secrets
kubectl get svc -n <namespace>
kubectl describe pod <pod_name> -n <namespace>
kubectl describe service <service_name> -n <namespace>
kubectl get events -n <namespace>
kubectl -n awx get all
kubectl -n awx logs -f deployments/awx-operator-controller-manager
```

## Ascender

`deploy-ascender.yml` deploys Ascender onto the same K3s cluster, in its own `ascender` namespace. Unlike AWX, Ascender doesn't document a manual kustomize/CR path, so this playbook drives Ascender's own installer ([ctrliq/ascender-install](https://github.com/ctrliq/ascender-install)) instead of reimplementing it. It expects K3s to already be set up (run `deploy-awx.yml` first, or install K3s yourself), and reuses the same LDAP and outbound proxy settings as `kubernetes/awx.yml` (edit `ldap_settings` and `proxy_settings` in the playbook), applied after install via the Settings API since Ascender's installer has no config for either.

Point the `[ascender_host]` group in `inventory.ini` at the target host (defaults to the same VM as `[awx_host]`), then run:

```bash
ansible-playbook -i inventory.ini deploy-ascender.yml --ask-pass
```

You'll be prompted for the hostname, Ascender version (check the [Ascender releases](https://github.com/ctrliq/ascender/releases) for the latest), an admin password, the LDAP bind password, and the cert/key, same as `deploy-awx.yml`. Installation (including the initial Django migrations) can take 10-20 minutes.

Once complete, browse to `https://<hostname>/` and log in as `admin` with the password you chose. Useful commands, on the target host:

```bash
kubectl -n ascender get awx,all,ingress,secrets
kubectl -n ascender logs -f deployments/ascender-app-task
```

**Note:** `deploy-ascender.yml` prompts for the LDAP bind password, but `kubernetes/awx.yml` holds it in plaintext. Don't commit a real one; consider templating it from Ansible Vault instead.

---

Written by Johannes Nguyen

