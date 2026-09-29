# Eclipse Che (upstream Dev Spaces) on RKE2 with Traefik

The parent [README](../README.md) targets OpenShift Dev Spaces. This folder has what you need to run the same Ansible workspace, including nested `podman` for `ansible-navigator`, on an RKE2 cluster that uses Traefik as its ingress controller.

## What is different from OpenShift

| OpenShift Dev Spaces | RKE2 + Traefik (this folder) |
| --- | --- |
| OpenShift OAuth logs users in | An OIDC provider (Keycloak below) logs users in, and the kube-apiserver must trust it (`00-rke2-server-config.yaml`) |
| Routes | Ingresses with `ingressClassName: traefik` and Traefik annotations instead of the operator's nginx defaults (`05-checluster-patch.yaml`) |
| nginx `proxy-read-timeout` / `ssl-redirect` annotations | Traefik entrypoint timeouts and an HTTP→HTTPS redirect (`01-rke2-traefik-config.yaml`) |
| `disableContainerRunCapabilities: false` (SCC + user namespaces) | The operator forces this setting off outside OpenShift, so the equivalent security context is set in the CheCluster (`05`) and `hostUsers: false` is set on the DevWorkspace Operator (`06`) |
| CRI-O annotation `io.kubernetes.cri-o.Devices: /dev/fuse,/dev/net/tun` | A device plugin advertises `squat.ai/fuse` and `squat.ai/tun`, and the devfile requests them (`02`, `devfile.yaml`) |
| Random UID with GID 0 | Pinned to UID 1000 (`user` in the image) (`05`) |
| Writable, delegated cgroups (`set-cgroups.sh`) | containerd mounts cgroups read-only, so podman runs nested containers without cgroups (devfile `postStart`) |

## Prerequisites

- **RKE2 on Kubernetes 1.33 or later.** User namespaces (`UserNamespacesSupport`) and `ProcMountType` are enabled by default from 1.33 on. On older versions, turn on the feature gates shown in `00-rke2-server-config.yaml`.
- **A node kernel that supports idmapped mounts for user-namespaced pods.** Linux 6.3 or later is the upstream requirement. Some enterprise kernels backport it; check your distribution.
- `kubectl`, `helm` and [`chectl`](https://github.com/che-incubator/chectl).
- [cert-manager](https://cert-manager.io) with a `ClusterIssuer`. To get a wildcard certificate from ACME you need a DNS-01 solver.
- DNS records pointing at the RKE2 ingress nodes or load balancer:
  - `keycloak.example.com`
  - `che.example.com` and the wildcard `*.che.example.com` (workspace endpoints use `<hash>.che.example.com`)

Replace `example.com`, `my-cluster-issuer` and the passwords and secrets in every file before you apply them.

## Install

### 1. Configure RKE2 (all server nodes)

1. Merge `00-rke2-server-config.yaml` into `/etc/rancher/rke2/config.yaml`.
2. If Keycloak's certificate comes from a private CA, copy that CA to `/var/lib/rancher/rke2/server/tls/oidc-ca.crt`. If the CA is public, remove the `oidc-ca-file` line.
3. Copy `01-rke2-traefik-config.yaml` to `/var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml`.
4. Restart RKE2:

   ```bash
   systemctl restart rke2-server
   ```

The API server starts even if the OIDC issuer is not reachable yet. It picks the issuer up once Keycloak is running.

### 2. Expose `/dev/fuse` and `/dev/net/tun`

```bash
kubectl apply -f 02-generic-device-plugin.yaml
# Every node should now report the new resources:
kubectl get nodes -o custom-columns=NAME:.metadata.name,FUSE:.status.allocatable.squat\\.ai/fuse,TUN:.status.allocatable.squat\\.ai/tun
```

### 3. OIDC provider (skip if you already have one)

```bash
kubectl apply -f 03-keycloak.yaml
kubectl wait --for=condition=ready pod -l app=keycloak -n keycloak --timeout=300s

kubectl exec deploy/keycloak -n keycloak -- bash -c \
  "/opt/keycloak/bin/kcadm.sh config credentials --server http://localhost:8080 --realm master --user admin --password admin && \
   /opt/keycloak/bin/kcadm.sh create realms -s realm=che -s enabled=true && \
   /opt/keycloak/bin/kcadm.sh create clients -r che -s clientId=k8s-client -s id=k8s-client \
     -s 'redirectUris=[\"https://che.example.com/*\"]' -s secret=eclipse-che && \
   /opt/keycloak/bin/kcadm.sh create users -r che -s username=test -s email=test@example.com \
     -s enabled=true -s emailVerified=true && \
   /opt/keycloak/bin/kcadm.sh set-password -r che --username test --new-password test"
```

`emailVerified=true` matters. When the username claim is `email`, the kube-apiserver rejects tokens whose email is not verified.

### 4. TLS for Che

```bash
kubectl apply -f 04-che-tls-certificate.yaml
kubectl wait --for=condition=ready certificate/che-tls -n eclipse-che --timeout=300s
```

If your issuer is a private CA, give Che the CA as well, so that it trusts Keycloak and its own endpoints:

```bash
kubectl create configmap custom-ca --from-file=ca.crt=my-ca.crt -n eclipse-che
kubectl label configmap custom-ca -n eclipse-che \
  app.kubernetes.io/part-of=che.eclipse.org app.kubernetes.io/component=ca-bundle
```

### 5. Deploy Che

```bash
chectl server:deploy --platform k8s --domain che.example.com \
  --skip-cert-manager \
  --che-operator-cr-patch-yaml 05-checluster-patch.yaml
```

### 6. Enable user namespaces for workspaces

chectl installs the DevWorkspace Operator. Once it is running, apply:

```bash
kubectl apply -f 06-devworkspace-operator-config.yaml
```

### 7. Start a workspace

Open `https://che.example.com` and log in as the Keycloak user. Point Che at the RKE2 devfile. The repository's root `devfile.yaml` is the OpenShift one.

```text
https://che.example.com/#https://github.com/<you>/Ansible-Development?devfilePath=devspaces/rke2/devfile.yaml
```

Alternatively, copy `devfile.yaml` over the root `devfile.yaml` in your own fork.

## Verify

Open a terminal in the workspace:

```bash
id                               # uid=1000(user)
cat /proc/self/uid_map           # NOT "0 0 4294967295", which would mean no user namespace
ls -l /dev/fuse /dev/net/tun
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal echo nested containers work
ansible-navigator --version
```

## Pod Security Admission

The workspace pods need the `privileged` Pod Security level: `procMount: Unmasked` and `seccompProfile: Unconfined` are not allowed at `baseline`.

- By default RKE2 enforces `privileged`, so you don't need to change anything.
- With `profile: cis`, RKE2 enforces `restricted`. Che names workspace namespaces `<username>-che`. Pre-create each one with the Che labels and a PSA exemption:

  ```bash
  kubectl create namespace test-che
  kubectl label namespace test-che \
    app.kubernetes.io/part-of=che.eclipse.org \
    app.kubernetes.io/component=workspaces-namespace \
    pod-security.kubernetes.io/enforce=privileged
  kubectl annotate namespace test-che che.eclipse.org/username=test@example.com
  ```

## Troubleshooting

- **Workspace pod stays Pending with `Insufficient squat.ai/fuse`.** The device plugin is not running on that node. Check `kubectl -n kube-system get ds generic-device-plugin`.
- **Pod rejected because of `procMount`.** `hostUsers: false` did not reach the pod. Check that the file from step 6 is in the DevWorkspace Operator's namespace, and that the cluster runs Kubernetes 1.33 or later (or has the feature gates enabled).
- **`sudo: unknown uid 1234`, or podman `cannot find UID/GID mappings`.** `podSecurityContext` from `05-checluster-patch.yaml` is not applied. Run `kubectl get checluster -n eclipse-che -o yaml` to check.
- **Mount errors (`fuse-overlayfs`, `/proc`) on Ubuntu or RHEL nodes.** Check that `appArmorProfile: Unconfined` is set, or uncomment `seLinuxOptions: container_engine_t` on SELinux-enforcing nodes.
- **Editor disconnects every 60 s.** The Traefik timeouts from step 1 are not active. Check with `kubectl -n kube-system get helmchartconfig rke2-traefik -o yaml`.
- **Login loop or `401` from the dashboard.** The `--oidc-*` API server flags do not match `networking.auth.*` in the CheCluster, or the kube-apiserver cannot reach or verify the issuer. Look for `oidc` in the kube-apiserver logs.
