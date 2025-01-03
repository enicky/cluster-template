# ⛵ Cluster Template

Welcome to my opinionated and extensible template for deploying a single Kubernetes cluster. The goal of this project is to make it easier for people interested in using Kubernetes to deploy a cluster at home on bare-metal or VMs. This template closely mirrors my personal [home-ops](https://github.com/onedr0p/home-ops) repository.

At a high level this project makes use of [makejinja](https://github.com/mirkolenz/makejinja) to read in a [configuration file](./config.sample.yaml) which renders out templates that will allow you to install and manage your Kubernetes cluster with.

## ✨ Features

The features included will depend on the type of configuration you want to use. There are currently **2 different types** of **configurations** available with this template.

1. **"Argo cluster"** - a Kubernetes cluster deployed on-top of [Talos Linux](https://github.com/siderolabs/talos) with an opinionated implementation of [Argo](https://github.com/argoproj/argo-cd) using [GitHub](https://github.com/) as the Git provider and [sops](https://github.com/getsops/sops) to manage secrets.

    - **Required:** Some knowledge of [Containers](https://opencontainers.org/), [YAML](https://yaml.org/), and [Git](https://git-scm.com/).
    - **Components:** [argo](https://github.com/argoproj/argo-cd), [cilium](https://github.com/cilium/cilium), [cert-manager](https://github.com/cert-manager/cert-manager), [spegel](https://github.com/spegel-org/spegel), [reloader](https://github.com/stakater/Reloader), and [openebs](https://github.com/openebs/openebs).

2. **"Argo cluster with Cloudflare"** - An addition to "**Argo cluster**" that provides DNS and SSL with [Cloudflare](https://www.cloudflare.com/). [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) is also included to provide external access to certain applications deployed in your cluster.

    - **Required:** A Cloudflare account with a domain managed in your Cloudflare account.
    - **Components:** [ingress-nginx](https://github.com/kubernetes/ingress-nginx/), [external-dns](https://github.com/kubernetes-sigs/external-dns) and [cloudflared](https://github.com/cloudflare/cloudflared).

**Other features include:**

- Dev env managed w/ [mise](https://mise.jdx.dev/)
- A [Renovate](https://www.mend.io/renovate)-ready repository with pull request for automated dependency updates
- Integrated [GitHub Actions](https://github.com/features/actions) with helpful workflows.

## 💻 Machine Preparation

### System requirements

> [!NOTE]
> 1. The included behaviour of Talos is that all nodes are able to run workloads, **including** the controller nodes. **Worker nodes** are therefore **optional**.
> 2. Do you have 3 or more nodes? It is highly recommended to make 3 of them controller nodes for a highly available control plane.
> 3. Running the cluster on Proxmox VE? My thoughts and recommendations about that are documented [here](https://onedr0p.github.io/home-ops/archive/proxmox-considerations.html).

| Role    | Cores    | Memory        | System Disk               |
|---------|----------|---------------|---------------------------|
| Control | 4 _(6*)_ | 8GB _(24GB*)_ | 120GB _(500GB*)_ SSD/NVMe |
| Worker  | 4 _(6*)_ | 8GB _(24GB*)_ | 120GB _(500GB*)_ SSD/NVMe |
| _\* recommended_ |

1. Head over to the [Talos Linux Image Factory](https://factory.talos.dev) and follow the instructions. Be sure to only choose the **bare-minimum system extensions** as some might require additional configuration and prevent Talos from booting without it. You can always add system extensions after Talos is installed and working.

2. This will eventually lead you to download a Talos Linux iso file (or for SBCs the `.raw.xz`). Make sure to note the schematic ID you will need this later on.
3. Flash the iso or raw file to a USB drive and boot to Talos on your nodes with it.

## 🚀 Getting Started

Once you have installed Talos on your nodes, there are six stages to getting a Argo-managed cluster up and running.

> [!NOTE]
> For all stages below the commands **MUST** be ran on your personal workstation within your repository directory

### 🎉 Stage 1: Create a Git repository

1. Create a new **public** repository by clicking the big green "Use this template" button at the top of this page.

2. Use `git clone` to download **the repo you just created** to your local workstation and `cd` into it.

### 🌱 Stage 2: Setup your local workstation

1. **Install** and **activate** [mise](https://mise.jdx.dev/) following the instructions for your workstation [here](https://mise.jdx.dev/getting-started.html).

2. Use mise to install the **required** CLI tools.

    ```sh
    mise trust
    mise install
    ```

3. Use mise to install the **required** Python dependencies.

    ```sh
    mise run install
    ```

### 🔧 Stage 3: Bootstrap configuration

> [!NOTE]
> The [config.sample.yaml](./config.sample.yaml) file contains config that is **vital** to the bootstrap process.

1. Generate the `config.yaml` from the [config.sample.yaml](./config.sample.yaml) configuration file.

    📍 _If the below command fails `mise` is either not install or configured incorrectly._

    ```sh
    task init
    ```

2. Fill out the `config.yaml` configuration file using the comments in that file as a guide.

3. Run the following command which will generate all the files needed to continue.

    ```sh
    task configure
    ```

4. Push you changes to git

   📍 _**Verify** all the `./kubernetes/**/*.sops.*` files are **encrypted** with SOPS_

    ```sh
    git add -A
    git commit -m "Initial commit :rocket:"
    git push
    ```

### ⛵ Stage 4: Install Kubernetes

1. Deploy your cluster and bootstrap it. This generates secrets, generates the config files for your nodes and applies them. It bootstraps the cluster afterwards, fetches the kubeconfig file and installs Cilium and kubelet-csr-approver. It finishes with some health checks.

    ```sh
    task bootstrap:talos
    ```

2. ⚠️ It might take a while for the cluster to be setup (10+ minutes is normal), during which time you will see a variety of error messages like: "couldn't get current server API group list," "error: no matching resources found", etc. This is a normal. If this step gets interrupted, e.g. by pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, you likely will need to [reset the cluster](#-reset) before trying again.

#### Cluster validation

1. The `kubeconfig` for interacting with your cluster should have been created in the root of your repository.

2. Verify the nodes are online

    📍 _If this command **fails** you likely haven't configured `mise`._

    ```sh
    kubectl get nodes -o wide
    # NAME           STATUS   ROLES                       AGE     VERSION
    # k8s-0          Ready    control-plane,etcd,master   1h      v1.30.1
    # k8s-1          Ready    worker                      1h      v1.30.1
    ```

3. Continue on to 🔹 [**Stage 6**](#-stage-6-install-argo-in-your-cluster)

### 🔹 Stage 6: Install Argo in your cluster

1. Install Argo and sync the cluster to the Git repository

    ```sh
    task bootstrap:argo
    # namespace/argo-system configured
    # ...
    ```

3. Verify Argo components are running in the cluster

    ```sh
    kubectl -n argo-system get pods -o wide
    # NAME                                                     READY   STATUS    RESTARTS   AGE
    # argo-argocd-application-controller-0                      1/1     Running   0          1h
    # argo-argocd-applicationset-controller-744474598c-lnk6t    1/1     Running   0          1h
    # argo-argocd-dex-server-6bc6fb6896-g84rm                   1/1     Running   0          1h
    # argo-argocd-notifications-controller-78d7b99846-7hssj     1/1     Running   0          1h
    # argo-argocd-redis-64c8457865-zhbk2                        1/1     Running   0          1h
    ```

### 🎤 Verification Steps

_Mic check, 1, 2_ - In a few moments applications should be lighting up like Christmas in July 🎄

1. Output all the common resources in your cluster.

    📍 _Feel free to use the provided [kubernetes tasks](.taskfiles/kubernetes/Taskfile.yaml) for validation of cluster resources or continue to get familiar with the `kubectl` and `argocd` CLI tools._

    ```sh
    task kubernetes:resources
    ```

2. ⚠️ It might take `cert-manager` awhile to generate certificates, this is normal so be patient.

3. 🏆 **Congratulations** if all goes smooth you will have a Kubernetes cluster managed by Argo and your Git repository is driving the state of your cluster.

4. 🧠 Now it's time to pause and go get some motel motor oil ☕ and admire you made it this far!

## 📣 Argo w/ Cloudflare post installation

#### 🌐 Public DNS

  📍 _Use the `external` ingress class to make applications public to the internet_

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `argo` are the only subdomains reachable from the public internet. In order to make additional applications public you must set set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

#### 🏠 Home DNS

  📍 _Use the `internal` ingress class to make applications private to your network_

`k8s_gateway` will provide DNS resolution to external Kubernetes resources (i.e. points of entry to the cluster) from any device that uses your home DNS server. For this to work, your home DNS server must be configured to forward DNS queries for `${bootstrap_cloudflare.domain}` to `${bootstrap_cloudflare.gateway_vip}` instead of the upstream DNS server(s) it normally uses. This is a form of **split DNS** (aka split-horizon DNS / conditional forwarding).

> [!TIP]
> Below is how to configure a Pi-hole for split DNS. Other platforms should be similar.
> 1. Apply this file on the Pihole server while substituting the variables
> ```sh
> # /etc/dnsmasq.d/99-k8s-gateway-forward.conf
> server=/${bootstrap_cloudflare.domain}/${bootstrap_cloudflare.gateway_vip}
> ```
> 2. Restart dnsmasq on the server.
> 3. Query an internal-only subdomain from your workstation (any `internal` class ingresses): `dig @${home-dns-server-ip} echo-server-internal.${bootstrap_cloudflare.domain}`. It should resolve to `${bootstrap_cloudflare.ingress_vip}`.

If you're having trouble with DNS be sure to check out these two GitHub discussions: [Internal DNS](https://github.com/onedr0p/cluster-template/discussions/719) and [Pod DNS resolution broken](https://github.com/onedr0p/cluster-template/discussions/635).

... Nothing working? That is expected, this is DNS after all!

#### 📜 Certificates

By default this template will deploy a wildcard certificate using the Let's Encrypt **staging environment**, which prevents you from getting rate-limited by the Let's Encrypt production servers if your cluster doesn't deploy properly (for example due to a misconfiguration). Once you are sure you will keep the cluster up for more than a few hours be sure to switch to the production servers as outlined in `config.yaml`.

📍 _You will need a production certificate to reach internet-exposed applications through `cloudflared`._

#### 🪝 Github Webhook

By default Argo will periodically check your git repository for changes. In order to have Argo sync on `git push` you must configure Github to send `push` events to Argo.

> [!NOTE]
> This will only work after you have switched over certificates to the Let's Encrypt Production servers.

1. Piece together the full URL for webhook
    ```text
    https://argo-webhook.${bootstrap_cloudflare.domain}/api/webhook
    ```

2. Navigate to the settings of your repository on Github, under "Settings/Webhooks" press the "Add webhook" button. Fill in the webhook URL and your `bootstrap_github_webhook_token` secret in `config.yaml`, Content type: `application/json`, Events: Choose Just the push event, and save.

## 💥 Reset

There might be a situation where you want to destroy your Kubernetes cluster. The following command will reset your nodes back to maintenance mode, append `--force` to completely format your the Talos installation. Either way the nodes should reboot after the command has run.

```sh
task talos:reset # --force
```

## 🛠️ Talos and Kubernetes Maintenance

#### ⚙️ Updating Talos node configuration

📍 _Ensure you have updated `talconfig.yaml` and any patches with your updated configuration._

```sh
# (Re)generate the Talos config
task talos:generate-config
# Apply the config to the node
task talos:apply-node HOSTNAME=? MODE=?
# e.g. task talos:apply-config HOSTNAME=k8s-0 MODE=auto
```

#### ⬆️ Updating Talos and Kubernetes versions

📍 _Ensure the `talosVersion` and `kubernetesVersion` in `talhelper.yaml` are up-to-date with the version you wish to upgrade to._

```sh
# Upgrade node to a newer Talos version
task talos:upgrade-node HOSTNAME=?
# e.g. task talos:upgrade HOSTNAME=k8s-0
```

```sh
# Upgrade cluster to a newer Kubernetes version
task talos:upgrade-k8s
# e.g. task talos:upgrade-k8s
```

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Argo itself! Merging a PR will cause Argo to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](./.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state.

1. Start by checking all Argo Applications and verify they are healthy.

    ```sh
      kubectl get applications -n argo-system
    ```

2. Then check the if the pod is present.

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

3. Then check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    # or
    stern -n <namespace> <fuzzy-name>
    ```

6. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

7. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on NFS. If you are unable to figure out your problem see the help section below.

## 👉 Help

- Make a post in this repository's Github [Discussions](https://github.com/onedr0p/cluster-template/discussions).
- Start a thread in the `#support` or `#cluster-template` channels in the [Home Operations](https://discord.gg/home-operations) Discord server.

## ❔ What's next

The cluster is your oyster (or something like that). Below are some optional considerations you might want to review.

### Ship it

To browse or get ideas on applications people are running, community member [@whazor](https://github.com/whazor) created [Kubesearch](https://kubesearch.dev) as a creative way to search Argo Releases across Github and Gitlab.

### DNS

Instead of using [k8s_gateway](https://github.com/ori-edge/k8s_gateway) to provide DNS for your applications you might want to check out [external-dns](https://github.com/kubernetes-sigs/external-dns), it has wide support for many different providers such as [Pi-hole](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/pihole.md), [UniFi](https://github.com/kashalls/external-dns-unifi-webhook), [Adguard Home](https://github.com/muhlba91/external-dns-provider-adguard), [Bind](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/rfc2136.md) and more.

### Storage

The included CSI (openebs in local-hostpath mode) is a great start for storage but soon you might find you need more features like replicated block storage, or to connect to a NFS/SMB/iSCSI server. If you need any of those features be sure to check out the projects like [rook-ceph](https://github.com/rook/rook), [longhorn](https://github.com/longhorn/longhorn), [openebs](https://github.com/openebs/openebs), [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs),
and [synology-csi](https://github.com/SynologyOpenSource/synology-csi).

## 🙌 Related Projects

If this repo is too hot to handle or too cold to hold check out these following projects.

- [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) - _A template for deploying a Talos Kubernetes cluster including Flux for GitOps._
- [khuedoan/homelab](https://github.com/khuedoan/homelab) - _Modern self-hosting framework, fully automated from empty disk to operating services with a single command._
- [danmanners/aws-argo-cluster-template](https://github.com/danmanners/aws-argo-cluster-template) - _A community opinionated template for deploying Kubernetes clusters on-prem and in AWS using Pulumi, SOPS, Sealed Secrets, GitHub Actions, Renovate, Cilium and more!_
- [ricsanfre/pi-cluster](https://github.com/ricsanfre/pi-cluster) - _Pi Kubernetes Cluster. Homelab kubernetes cluster automated with Ansible and ArgoCD_
- [techno-tim/k3s-ansible](https://github.com/techno-tim/k3s-ansible) - _The easiest way to bootstrap a self-hosted High Availability Kubernetes cluster. A fully automated HA k3s etcd install with kube-vip, MetalLB, and more_

## ⭐ Stargazers

<div align="center">

[![Star History Chart](https://api.star-history.com/svg?repos=ajaykumar4/cluster-template&type=Date)](https://star-history.com/#ajaykumar4/cluster-template&Date)

</div>

## 🤝 Thanks

Big shout out to all the contributors, sponsors and everyone else who has helped on this project.
