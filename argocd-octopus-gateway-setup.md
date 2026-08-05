# Setting up ArgoCD on an Ubuntu VM with the Octopus Argo CD Gateway

This guide sets up a single-node ArgoCD instance on an Ubuntu VM (e.g. running in Parallels on Apple Silicon) and connects it to Octopus Deploy via the Octopus Argo CD Gateway. Intended for **non-production / test environments** only.

## Prerequisites

- An Ubuntu VM with at least 2 CPU cores and 4 GB RAM
- Outbound internet access from the VM (no inbound access required — the gateway only dials out)
- An Octopus Deploy Cloud instance with permission to manage Infrastructure

## 1. Install k3s

k3s is a lightweight Kubernetes distribution — ideal for a single-node test cluster.

```bash
curl -sfL https://get.k3s.io | sh -
```

Point `kubectl` at the k3s config (k3s writes it to a different location than `kubectl` expects by default):

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

Verify:

```bash
kubectl get nodes
```

You should see the node in `Ready` state. If `kubectl` complains about `localhost:8080`, the kubeconfig step above didn't take — re-check it, or `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` for the current session.

## 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 3. Expose the ArgoCD UI permanently

By default the `argocd-server` service is `ClusterIP` only. For repeat access without re-running `kubectl port-forward` every time, switch it to `NodePort`:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

Pin it to a fixed port (random ports get reassigned if the service is recreated):

```bash
kubectl edit svc argocd-server -n argocd
```

Under the port named `https` (443), set `nodePort: 30443` and confirm `type: NodePort`.

If ufw is enabled, open the port:

```bash
sudo ufw allow 30443/tcp
```

**Finding the VM's IP** — `ip a` is noisy because k3s creates several virtual interfaces (`cni0`, `flannel.1`, `docker0`, `veth*`). Skip the noise with:

```bash
ip route get 1.1.1.1 | grep -oP 'src \K\S+'
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Now the UI is reachable at `https://<VM-IP>:30443` (username `admin`) any time, from the VM or any machine on the same network. Expect a self-signed cert warning — fine for a test instance, but **do not expose this on the open internet**.

## 4. Install the ArgoCD CLI

```bash
uname -m
```

- `aarch64` → download `argocd-linux-arm64` (this is what you'll get on Apple Silicon Parallels VMs by default)
- `x86_64` → download `argocd-linux-amd64`

```bash
curl -sSL -o argocd-linux-<arch> https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-<arch>
sudo install -m 555 argocd-linux-<arch> /usr/local/bin/argocd
rm argocd-linux-<arch>
```

Log in:

```bash
argocd login <VM-IP>:30443 --username admin --password '<password from step 3>' --insecure
```

## 5. Create a scoped ArgoCD user for Octopus

Octopus needs read access to Applications/Clusters/Logs — not a login. Create a dedicated `octopus` account with `apiKey` capability only:

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"accounts.octopus":"apiKey","accounts.octopus.enabled":"true"}}'
```

Confirm:

```bash
argocd account list
```

Grant it read (and optionally sync) permissions via RBAC:

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p '{"data":{"policy.csv":"p, octopus, applications, get, *, allow\np, octopus, applications, sync, *, allow\np, octopus, clusters, get, *, allow\np, octopus, logs, get, */*, allow\n"}}'
```

> This overwrites the entire `policy.csv` key. If `argocd-rbac-cm` already has custom rules, use `kubectl edit configmap argocd-rbac-cm -n argocd` and append these lines instead of patching. Drop the `sync` line if Octopus should only read state, not trigger syncs.

Generate the token (shown once — copy it now):

```bash
argocd account generate-token --account octopus
```

Verify the permissions took effect:

```bash
argocd account can-i --auth-token <token> get clusters '*'        # yes
argocd account can-i --auth-token <token> get applications '*'    # yes
argocd account can-i --auth-token <token> get logs '*/*'          # yes
argocd account can-i --auth-token <token> delete applications '*' # no
```

## 6. Install the Octopus Argo CD Gateway

In Octopus Cloud, go to **Infrastructure → Argo CD Instances → Add Argo CD Instance** and work through the wizard:

- A unique instance name (used for the Kubernetes namespace and Helm release name)
- The environment(s) this gateway will service
- The in-cluster ArgoCD API server URL — default is usually correct (`https://argocd-server.argocd.svc.cluster.local`)
- The token generated in step 5

At the end, Octopus generates a `helm upgrade --install` command. Run it on the VM against the k3s cluster.

If ArgoCD is using a self-signed certificate (the default on a fresh install like this one), add to the generated command:

```
--set gateway.argocd.insecure="true"
```

The gateway makes only outbound connections (to the Octopus REST API on `443`, the Octopus gRPC endpoint on `8443`, and the ArgoCD API server in-cluster) — no inbound firewall rules are needed.

Leave the installation dialog open in Octopus; it waits for a health check to pass. Once green, the gateway is connected and ready to use.

## 7. Next steps

With the gateway healthy, the remaining work is wiring up [scoping annotations](https://octopus.com/docs/argo-cd/annotations) on your ArgoCD Applications so Octopus knows which projects, environments, and tenants each one maps to.

## References

- [Octopus Argo CD Instances — Overview](https://octopus.com/docs/argo-cd/instances)
- [Argo CD Authentication for Octopus](https://octopus.com/docs/argo-cd/instances/argo-user)
- [Automated Installation (Helm/Terraform/Argo Application)](https://github.com/OctopusDeploy/docs/blob/main/src/pages/docs/argo-cd/instances/automated-installation.md)
