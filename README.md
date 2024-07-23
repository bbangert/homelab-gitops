<div align="center">

<img src="https://raw.githubusercontent.com/bbangert/homelab-gitops/main/icons/logo.png" align="center" width="144px" height="144px"/>

### My Home Operations repository :octocat:

_... managed with Flux, Renovate and GitHub Actions_ 🤖


</div>

<div align="center">

[![Discord](https://img.shields.io/discord/673534664354430999?style=for-the-badge&label&logo=discord&logoColor=white&color=blue)](https://discord.gg/k8s-at-home)&nbsp;&nbsp;
[![GitHub last commit](https://img.shields.io/github/last-commit/bbangert/homelab-gitops?color=purple&style=for-the-badge)](https://github.com/bbangert/homelab-gitops/commits/main 'Commit History')\
[![GitHub stars](https://img.shields.io/github/stars/bbangert/homelab-gitops?color=green&style=for-the-badge)](https://github.com/bbangert/homelab-gitops/stargazers 'This repo star count')

</div>

---

## 📖 Overview

This is a mono repository for my home infrastructure and Kubernetes cluster. I try to adhere to Infrastructure as Code (IaC) and GitOps practices using tools like [Kubernetes](https://kubernetes.io/), [Flux](https://github.com/fluxcd/flux2), [Renovate](https://github.com/renovatebot/renovate) and [GitHub Actions](https://github.com/features/actions).

---

## ⛵ Kubernetes

There is a template over at [onedr0p/flux-cluster-template](https://github.com/onedr0p/flux-cluster-template) if you want to try and follow along with some of the practices I use here.

### Installation

My Kubernetes cluster is deployed with [Talos](https://www.talos.dev/). This is a hyper-converged cluster, workloads and block storage are sharing the same available resources on my nodes with some larger volumes mounted on specific nodes with off-site backups.

### Core Components

- [cert-manager](https://cert-manager.io/docs/): Creates SSL certificates for services in my Kubernetes cluster.
- [cilium](https://cilium.io/): Internal Kubernetes networking plugin.
- [cloudflared](https://github.com/cloudflare/cloudflared): Enables Cloudflare secure access to certain ingresses.
- [external-dns](https://github.com/kubernetes-sigs/external-dns): Automatically manages DNS records from my cluster in a cloud DNS provider.
- [external-secrets](https://github.com/external-secrets/external-secrets/): Managed Kubernetes secrets using [1Password Connect](https://github.com/1Password/connect).
- [ingress-nginx](https://github.com/kubernetes/ingress-nginx/): Ingress controller to expose HTTP traffic to pods over DNS.
- [longhorn](https://github.com/longhorn/longhorn): Distributed block storage for peristent storage.
- [sops](https://toolkit.fluxcd.io/guides/mozilla-sops/): Managed secrets for Kubernetes, Ansible.
- [spegel](https://github.com/spegel-org/spegel): Stateless cluster local OCI registry mirror.

### GitOps

[Flux](https://github.com/fluxcd/flux2) watches my [kubernetes](./kubernetes/) folder (see Directories below) and makes the changes to my cluster based on the YAML manifests.

The way Flux works for me here is it will recursively search the [kubernetes/apps](./kubernetes/apps) folder until it finds the most top level `kustomization.yaml` per directory and then apply all the resources listed in it. That aforementioned `kustomization.yaml` will generally only have a namespace resource and one or many Flux kustomizations. Those Flux kustomizations will generally have a `HelmRelease` or other resources related to the application underneath it which will be applied.

[Renovate](https://github.com/renovatebot/renovate) watches my **entire** repository looking for dependency updates, when they are found a PR is automatically created. When some PRs are merged [Flux](https://github.com/fluxcd/flux2) applies the changes to my cluster.

### Directories

This Git repository contains the following directories under [kubernetes](./kubernetes/).

```sh
📁 kubernetes      # Kubernetes cluster defined as code
├─📁 bootstrap     # Flux installation
├─📁 flux          # Main Flux configuration of repository
└─📁 apps          # Apps deployed into my cluster grouped by namespace (see below)
```

### Cluster layout

Below is a a high level look at the layout of how my directory structure with Flux works. In this brief example you are able to see that `authelia` will not be able to run until `glauth` and  `cloudnative-pg` are running. It also shows that the `Cluster` custom resource depends on the `cloudnative-pg` Helm chart. This is needed because `cloudnative-pg` installs the `Cluster` custom resource definition in the Helm chart.

```python
# Key: <kind> :: <metadata.name>
GitRepository :: home-ops-kubernetes
    Kustomization :: cluster
        Kustomization :: cluster-apps
            Kustomization :: cluster-apps-authentik
                DependsOn:
                    Kustomization :: cluster-apps-cloudnative-pg-cluster
                HelmRelease :: authelia
            Kustomization :: cluster-apps-cloudnative-pg
                HelmRelease :: cloudnative-pg
            Kustomization :: cluster-apps-cloudnative-pg-cluster
                DependsOn:
                    Kustomization :: cluster-apps-cloudnative-pg
                Cluster :: postgres
```

---

## ☁️ Cloud Dependencies

While most of my infrastructure and workloads are self-hosted I do rely upon the cloud for certain key parts of my setup. This saves me from having to worry about two things. (1) Dealing with chicken/egg scenarios and (2) services I critically need whether my cluster is online or not.

| Service                                                 | Use                                                            | Cost           |
|---------------------------------------------------------|----------------------------------------------------------------|----------------|
| [1Password](https://1password.com/)                     | Secrets with [External Secrets](https://external-secrets.io/)  | ~$65/yr        |
| [Cloudflare](https://www.cloudflare.com/)               | Domain, DNS and proxy management, R2 backups                   | ~$30/yr        |
| [Backblaze B2](https://www.backblaze.com/cloud-storage) | B2 backups for disaster recovery                               | ~$72/yr        |
| [GitHub](https://github.com/)                           | Hosting this repository and continuous integration/deployments | Free           |
|                                                         |                                                                | Total: ~$14/mo |

---

### Ingress Controller

External access to my cluster is done using a [Cloudflare](https://www.cloudflare.com/) tunnel. This works to prevent me from having to open ports in my router / firewall, as you would normally have to do to allow access to internal services. I also have some additional web services for my other domains that route directly to the service off the tunnel.

### Internal DNS

My `opnSense` router serves as my Internal DNS server and is listening on `:53`. All DNS queries for _**my**_ domains are forwarded to [k8s_gateway](https://github.com/ori-edge/k8s_gateway) that is running in my cluster. With this setup `k8s_gateway` has direct access to my clusters ingresses and services and serves DNS for them in my internal network.

### Ad Blocking

My `opnSense` router is utilizing blocklists with `Unbound` which allows me to filter out known ad-serving sites & domains.

### External DNS

[external-dns](https://github.com/kubernetes-sigs/external-dns) is deployed in my cluster and configure to sync DNS records to [Cloudflare](https://www.cloudflare.com/). The only ingresses `external-dns` looks at to gather DNS records to put in `Cloudflare` are ones that have an annotation of `external-dns.alpha.kubernetes.io/target` or that I explicitly allow for other hosted sites.

---

## 🔧 Hardware

<details>
  <summary>Click to see da rack!</summary>

  <img src="https://raw.githubusercontent.com/bbangert/homelab-gitops/main/icons/rack.jpg" align="center" width="200px" alt="rack"/>
</details>

| Device                   | Count | OS Disk Size        | Data Disk Size | Ram   | Operating System | Purpose                     |
|--------------------------|-------|---------------------|----------------|-------|------------------|-----------------------------|
| Proctli VP2420           | 1     | 128GB NVMe          |                | 32GB  | opnSense         | Router                      |
| Supermicro 5019D-FTN4    | 1     | 1TB NVMe            | 2 TB SATA SSD  | 128GB | Talos            | Kubernetes Master           |
| ODroid H4                | 2     | 2TB NVMe (longhorn) |                | 32GB  | Talos            | Kubernetes Masters          |
| Beelink N5095            | 1     | 2TB SATA SSD        |                | 8GB   | Talos            | Kubernetes Worker (Frigate) |
| CyberPower  OL1500RTXL2U | 1     | -                   | -              | -     | -                | UPS                         |
| QNAP QSW-M3216R-8S8T     | 1     | -                   | -              | -     | -                | Core 10Gb Switch            |
| Unifi USW-16-POE         | 1     | -                   | -              | -     | -                | Service Switch (PoE)        |

---

## ⭐ Stargazers

<div align="center">

[![Star History Chart](https://api.star-history.com/svg?repos=bbangert/homelab-gitops&type=Date)](https://star-history.com/#bbangert/homelab-gitops&Date)

</div>

---

## 🤝 Gratitude and Thanks

Thanks to all the people who donate their time to the [Kubernetes @Home](https://discord.gg/k8s-at-home) Discord community. A lot of inspiration for my cluster comes from the people that have shared their clusters using the [k8s-at-home](https://github.com/topics/k8s-at-home) GitHub topic. Be sure to check out the [Kubernetes @Home search](https://nanne.dev/k8s-at-home-search/) for ideas on how to deploy applications or get ideas on what you can deploy. Also a massive thanks to [onedr0p](https://github.com/onedr0p/) specifically for spending so much time cultivating this entire project, and helping people with questions along the way.

---

## 📜 Changelog

See my _realllllly bad_ [commit history](https://github.com/bbangert/homelab-gitops/commits/main)

---

## 🔏 License

See [LICENSE](./LICENSE)
