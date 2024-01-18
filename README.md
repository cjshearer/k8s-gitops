# k8s-gitops

## Stack

### Workstation

- [age](https://age-encryption.org/): simple, modern and secure encryption tool
- [kubectl](https://kubernetes.io/docs/reference/kubectl/): CLI for communicating with k8s control plane
- [SOPS](https://getsops.io): simple and flexible tool for managing secrets
- [talhelper](https://budimanjojo.github.io/talhelper/latest/): CLI for declaratively creating Talos configuration files
- [talosctl](https://www.talos.dev/v1.6/learn-more/talosctl/): CLI for managing Talos machines
- [Taskfile](https://taskfile.dev): task runner / build tool

### Cluster

- [Flux](https://fluxcd.io): reconciles remote k8s manifest changes with the cluster
- [Tailscale](https://github.com/siderolabs/extensions/tree/main/network/tailscale): zero-config mesh VPN
- [Talos Linux](https://talos.dev): container optimized Linux distro

## Workstation setup

```bash
# Install Taskfile and other dependencies
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin

task workstation:install
```

> [!NOTE]
> We rely on Taskfile to set environment variables required by some CLI tools (e.g. talosctl). While the integrated vscode terminal has been configured to load these on startup, other shell environments can load these manually with something like `source <(task env)`.

## Bootstrapping Cluster

This guide assumes you are starting from scratch and will run a single node on bare-metal. It also assumes you have already created a remote copy of this repo and cloned it.

### Talos

This guide is intended to complement the [talhelper guide](https://budimanjojo.github.io/talhelper/latest/getting-started/).

1. Select your [system extensions](https://github.com/siderolabs/extensions) in `talos/talconfig.yaml`.

   > [!NOTE]
   > My machine has an Intel CPU and AMD GPU, so I included `intel-ucode` and `amdgpu-firmware`. I also use [`tailscale`](https://github.com/siderolabs/extensions/tree/main/network/tailscale) to handle networking and provide secure access to the node.

2. Flash the ISO from `task talos:genurl-iso` on a USB drive and boot into that on your new node.

   > [!IMPORTANT]
   > During bootstrapping, your node will need to be on the same network as your workstation.

3. Choose a `hostname` for your new node and substitute it for `endpoint`, `ipAddress`, and `hostname` in `talconfig.yaml`. This only works if you have tailscale's "MagicDNS" feature enabled [here](https://login.tailscale.com/admin/dns).

4. To designate the disk where Talos will be installed, update `installDisk` in `talconfig.yaml`.

   > [!TIP]
   > You can get a list of disks on your new node by running `talosctl disks -e <IP_ADDR> -n <IP_ADDR> --insecure` from your workstation, where `<IP_ADDR>` is your node's local IP address.

5. To later encrypt secrets stored in git, generate an age key in the root directory with `age -o age.key`.

   > [!WARNING]
   > Don't add age.key to git. Back it up in a secure location.

6. Run `task talos:gensecret --force` to generate your own Talos cluster secrets.

7. To authenticate the new node with tailscale, [generate an auth key](https://login.tailscale.com/admin/settings/keys) with a tag (e.g. `tag:server`) to disable key expiry. If you want the auth key to be reusable, it should be stored in `talenv.sops.yaml`. Otherwise, delete `talenv.sops.yaml` and `export TAILSCALE_AUTHKEY=<your authkey>` in your shell.

8. Using your new node's local IP address, generate the talos config and initiate the bootstrapping process with `node=<new node IP address> task talos:bootstrap`. It may take several minutes for this to finish (watch the node's logs for errors). You should eventually see the machine has [joined the tailnet](https://login.tailscale.com/admin/machines).

## Flux

With our node operational, we can bootstrap flux...

> [!IMPORTANT]
> Your ssh-agent must be configured with access to your repo.

```sh
task flux:bootstrap
# namespace/flux-system created
# ...
# ✔ source-controller: deployment ready
# ✔ all components are healthy
```

...and verify its controllers are running

```sh
kubectl -n flux-system get pods
# NAME                                       READY   STATUS    RESTARTS   AGE
# helm-controller-58d5cc6f5b-ssbtx           1/1     Running   0          84m
# kustomize-controller-d84b877fb-zp67f       1/1     Running   0          84m
# notification-controller-77fd94d6d4-4sbrc   1/1     Running   0          84m
# source-controller-7657cffdb-5zdwc          1/1     Running   0          84m
```
