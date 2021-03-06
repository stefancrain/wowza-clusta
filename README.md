# Template for deploying k3s and Flux backed by SOPS secrets

Template for creating a [k3s](https://k3s.io/) cluster with [k3sup](https://github.com/alexellis/k3sup).

The purpose here is to showcase how you can deploy an entire Kubernetes cluster and show it off to the world using the [GitOps](https://www.weave.works/blog/what-is-gitops-really) tool [Flux](https://toolkit.fluxcd.io/). With the [Flux SOPS integration](https://toolkit.fluxcd.io/guides/mozilla-sops/) you'll be able to commit encrypted secrets to your public repo.

The following components are installed in your [k3s](https://k3s.io/) cluster by default. They are only included to get a minimum viable cluster up and running. You are free to add / remove components to your liking but anything outside the scope of the below components are not supported by this template.

- [flannel](https://github.com/flannel-io/flannel)
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- [flux](https://toolkit.fluxcd.io/)
- [metallb](https://metallb.universe.tf/)
- [cert-manager](https://cert-manager.io/) with Cloudflare DNS challenge
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)
- [homer](https://github.com/bastienwirtz/homer)

## Overview

- [Prerequisites](https://github.com/k8s-at-home/template-cluster-k3s#memo-prerequisites)
- [Lets go!](https://github.com/k8s-at-home/template-cluster-k3s#rocket-lets-go)
- [Post installation](https://github.com/k8s-at-home/template-cluster-k3s#mega-post-installation)
- [Debugging](https://github.com/k8s-at-home/template-cluster-k3s#point_right-debugging)
- [Automation](https://github.com/k8s-at-home/template-cluster-k3s#robot-automation)
- [What's next](https://github.com/k8s-at-home/template-cluster-k3s#grey_question-whats-next)
- [Thanks](https://github.com/k8s-at-home/template-cluster-k3s#handshake-thanks)

## :memo:&nbsp; Prerequisites

### :computer:&nbsp; Nodes

Bare metal or VMs with any modern operating system like Ubuntu, Debian or CentOS.

### :wrench:&nbsp; Tools

:round_pushpin: You need to install the required CLI tools listed below on your workstation.

| Tool                                                               | Purpose                                                             | Minimum version | Required |
|--------------------------------------------------------------------|---------------------------------------------------------------------|:---------------:|:--------:|
| [k3sup](https://github.com/alexellis/k3sup)                        | Tool to install k3s on your nodes                                   |    `0.10.2`     |    ???     |
| [kubectl](https://kubernetes.io/docs/tasks/tools/)                 | Allows you to run commands against Kubernetes clusters              |    `1.21.0`     |    ???     |
| [flux](https://toolkit.fluxcd.io/)                                 | Operator that manages your k8s cluster based on your Git repository |    `0.12.3`     |    ???     |
| [SOPS](https://github.com/mozilla/sops)                            | Encrypts k8s secrets with GnuPG                                     |     `3.7.1`     |    ???     |
| [GnuPG](https://gnupg.org/)                                        | Encrypts and signs your data                                        |    `2.2.27`     |    ???     |
| [pinentry](https://gnupg.org/related_software/pinentry/index.html) | Allows GnuPG to read passphrases and PIN numbers                    |     `1.1.1`     |    ???     |
| [direnv](https://github.com/direnv/direnv)                         | Exports env vars based on present working directory                 |    `2.28.0`     |    ???     |
| [pre-commit](https://github.com/pre-commit/pre-commit)             | Runs checks during `git commit`                                     |    `2.12.0`     |    ???     |
| [kustomize](https://kustomize.io/)                                 | Template-free way to customize application configuration            |     `4.1.0`     |    ???     |
| [helm](https://helm.sh/)                                           | Manage Kubernetes applications                                      |     `3.5.4`     |    ???     |

## :warning:&nbsp; Pre-installation

It's very important and I cannot stress enough, make sure you are not pushing your secrets un-encrypted to a public Git repo.

### pre-commit

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) will check to make sure you are not by accident commiting your secrets un-encrypted.

After pre-commit is installed on your machine run:

```sh
pre-commit install-hooks
```

## :rocket:&nbsp; Lets go!

Very first step will be to create a new repository by clicking the **Use this template** button on this page.

:round_pushpin: In these instructions you will be exporting several environment variables to your current shell env. Make sure you stay with in your current shell to not lose any exported variables.

### :closed_lock_with_key:&nbsp; Setting up GnuPG keys

:round_pushpin: Here we will create a personal and a Flux GPG key. Using SOPS with GnuPG allows us to encrypt and decrypt secrets.

1. Create a Personal GPG Key, password protected, and export the fingerprint

```sh
export GPG_TTY=$(tty)
export PERSONAL_KEY_NAME="First name Last name (location) <email>"

gpg --batch --full-generate-key <<EOF
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${PERSONAL_KEY_NAME}
EOF

gpg --list-secret-keys "${PERSONAL_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       772154FFF783DE317KLCA0EC77149AC618D75581
# uid           [ultimate] k8s@home (Macbook) <k8s-at-home@gmail.com>
# sub   rsa4096 2021-03-11 [E]

export PERSONAL_KEY_FP=772154FFF783DE317KLCA0EC77149AC618D75581
```

2. Create a Flux GPG Key and export the fingerprint

```sh
export GPG_TTY=$(tty)
export FLUX_KEY_NAME="Cluster name (Flux) <email>"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${FLUX_KEY_NAME}
EOF

gpg --list-secret-keys "${FLUX_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       AB675CE4CC64251G3S9AE1DAA88ARRTY2C009E2D
# uid           [ultimate] Home cluster (Flux) <k8s-at-home@gmail.com>
# sub   rsa4096 2021-03-11 [E]

export FLUX_KEY_FP=AB675CE4CC64251G3S9AE1DAA88ARRTY2C009E2D
```

### :sailboat:&nbsp; Installing k3s with k3sup

:round_pushpin: Here we will be install [k3s](https://k3s.io/) with [k3sup](https://github.com/alexellis/k3sup).

1. Ensure you are able to SSH into you nodes with using your private ssh key. This is how k3sup is able to connect to your remote node.

2. Install the master node

```sh
k3sup install \
    --host=192.168.1.140 \
    --user=pi \
    --k3s-version=v1.20.5+k3s1 \
    --k3s-extra-args="--disable servicelb --disable traefik"
```

3. Join a worker node(s) (optional)

```sh
k3sup join \
    --host=192.168.1.141 \
    --server-host=192.168.1.140 \
    --k3s-version=v1.20.5+k3s1 \
    --user=pi
```

4. Verify the nodes are online
   
```sh
kubectl --kubeconfig=./kubeconfig get nodes
# NAME           STATUS   ROLES                       AGE     VERSION
# k8s-master-a   Ready    control-plane,master      4d20h   v1.20.5+k3s1
# k8s-worker-a   Ready    worker                    4d20h   v1.20.5+k3s1
```

### :cloud:&nbsp; Cloudflare API Token

:round_pushpin: In order to use cert-manager with the Cloudflare DNS challenge you will need to create a API token.

1. Head over to Cloudflare and create a API token by going [here](https://dash.cloudflare.com/profile/api-tokens).
2. Click the blue `Create Token` button
3. Scroll down and create a Custom Token by choosing `Get started`
4. Give your token a name like `cert-manager`
5. Under `Permissions` give read access to `Zone` : `Zone` and `Zone` : `DNS`
6. Under `Zone Resources` set it to `Include` : `All Zones`
7. Click `Continue to summary` and then `Create Token`
8. Export this token and your Cloudflare email address to an environment variable on your system to be used in the following steps

```sh
export BOOTSTRAP_CLOUDFLARE_EMAIL="k8s-at-home@gmail.com"
export BOOTSTRAP_CLOUDFLARE_TOKEN="kpG6iyg3FS_du_8KRShdFuwfbwu3zMltbvmJV6cD"
```

### :small_blue_diamond:&nbsp; GitOps with Flux

:round_pushpin: Here we will be installing [flux](https://toolkit.fluxcd.io/) after some quick bootstrap steps.

1. Verify Flux can be installed

```sh
flux --kubeconfig=./kubeconfig check --pre
# ??? checking prerequisites
# ??? kubectl 1.21.0 >=1.18.0-0
# ??? Kubernetes 1.20.5+k3s1 >=1.16.0-0
# ??? prerequisites checks passed
```

2. Pre-create the `flux-system` namespace

```sh
kubectl --kubeconfig=./kubeconfig create namespace flux-system --dry-run=client -o yaml | kubectl --kubeconfig=./kubeconfig apply -f -
```

3. Add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
gpg --export-secret-keys --armor "${FLUX_KEY_FP}" |
kubectl --kubeconfig=./kubeconfig create secret generic sops-gpg \
    --namespace=flux-system \
    --from-file=sops.asc=/dev/stdin
```

4. Update files using `envsubst` or by updating the files listed below manually

```sh
export BOOTSTRAP_GITHUB_REPOSITORY="https://github.com/k8s-at-home/home-cluster"
export BOOTSTRAP_METALLB_LB_RANGE="169.254.1.10-169.254.1.20"
export BOOTSTRAP_DOMAIN="k8s-at-home.com"
export BOOTSTRAP_DOMAIN_CERT="k8s-at-home"
export BOOTSTRAP_INGRESS_NGINX_LB="169.254.1.10"

envsubst < ./tmpl/.sops.yaml > ./.sops.yaml
envsubst < ./tmpl/cluster-secrets.yaml > ./cluster/base/cluster-secrets.yaml
envsubst < ./tmpl/cluster-settings.yaml > ./cluster/base/cluster-settings.yaml
envsubst < ./tmpl/gotk-sync.yaml > ./cluster/base/flux-system/gotk-sync.yaml
envsubst < ./tmpl/secret.enc.yaml > ./cluster/core/cert-manager/secret.enc.yaml
```

5. **Verify** all the above files have the correct information present

6. Encrypt `cluster/cluster-secrets.yaml` and `cert-manager/secret.enc.yaml` with SOPS

```sh
export GPG_TTY=$(tty)
sops --encrypt --in-place ./cluster/base/cluster-secrets.yaml
sops --encrypt --in-place ./cluster/core/cert-manager/secret.enc.yaml
```

:round_pushpin: Variables defined in `cluster-secrets.yaml` and `cluster-settings.yaml` will be usable anywhere in your YAML manifests under `./cluster`

7. **Verify** all the above files are **encrypted** with SOPS

8. Push you changes to git

```sh
git add -A
git commit -m "initial commit"
git push
```

9. Install Flux

```sh
kubectl --kubeconfig=./kubeconfig apply --kustomize=./cluster/base/flux-system
```

:round_pushpin: Don't be concerned with any errors that may be outputted to your console, Flux should have eventual consistency soon.

## :mega:&nbsp; Post installation

### Verify Flux

```sh
kubectl --kubeconfig=./kubeconfig get pods -n flux-system
# NAME                                       READY   STATUS    RESTARTS   AGE
# helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
# kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
# notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
# source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
```

### Verify ingress

If your cluster is not accessible to outside world you can update your hosts file to verify the ingress controller is working.

```sh
sudo echo "${BOOTSTRAP_INGRESS_NGINX_LB} ${BOOTSTRAP_DOMAIN} homer.${BOOTSTRAP_DOMAIN}" >> /etc/hosts
```

Head over to your browser and you _should_ be able to access `https://homer.${BOOTSTRAP_DOMAIN}`

### direnv

This is a great tool to export environment variables depending on what your present working directory is, head over to their [installation guide](https://direnv.net/docs/installation.html) and don't forget to hook it into your shell!

When this is done you no longer have to use `--kubeconfig=./kubeconfig` in your `kubectl` or `flux` commands.

### Delete Flux GPG key

Since there is a GPG key specifically for Flux you can remove the secret key from your personal machine.

```sh
gpg --delete-secret-keys "${FLUX_KEY_FP}"
```

### VSCode SOPS extension

[Here](https://marketplace.visualstudio.com/items?itemName=signageos.signageos-vscode-sops)'s a neat little plugin for those using VSCode.
It will automatically decrypt you SOPS secrets when you click on the file in the editor and encrypt them when you save the file.

## :point_right:&nbsp; Debugging

Manually sync Flux with your Git repository

```sh
flux --kubeconfig=./kubeconfig reconcile source git flux-system
# ??? annotating GitRepository flux-system in flux-system namespace
# ??? GitRepository annotated
# ??? waiting for GitRepository reconciliation
# ??? GitRepository reconciliation completed
# ??? fetched revision main/943e4126e74b273ff603aedab89beb7e36be4998
```

Show the health of you kustomizations

```sh
kubectl --kubeconfig=./kubeconfig get kustomization -A
# NAMESPACE     NAME          READY   STATUS                                                             AGE
# flux-system   apps          True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    3d19h
# flux-system   core          True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    4d6h
# flux-system   flux-system   True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    4d6h
```

Show the health of your main Flux `GitRepository`

```sh
flux --kubeconfig=./kubeconfig get sources git
# NAME           READY	MESSAGE                                                            REVISION                                         SUSPENDED
# flux-system    True 	Fetched revision: main/943e4126e74b273ff603aedab89beb7e36be4998    main/943e4126e74b273ff603aedab89beb7e36be4998    False
```

Show the health of your `HelmRelease`s

```sh
flux --kubeconfig=./kubeconfig get helmrelease -A
# NAMESPACE   	NAME                  	READY	MESSAGE                         	REVISION	SUSPENDED
# cert-manager	cert-manager          	True 	Release reconciliation succeeded	v1.3.0  	False
# home        	homer                 	True 	Release reconciliation succeeded	4.2.0   	False
# networking  	ingress-nginx       	True 	Release reconciliation succeeded	3.29.0  	False
```

Show the health of your `HelmRepository`s

```sh
flux --kubeconfig=./kubeconfig get sources helm -A
# NAMESPACE  	NAME                 READY	MESSAGE                                                   	REVISION                                	SUSPENDED
# flux-system	bitnami-charts       True 	Fetched revision: 0ec3a3335ff991c45735866feb1c0830c4ed85cf	0ec3a3335ff991c45735866feb1c0830c4ed85cf	False
# flux-system	ingress-nginx-charts True 	Fetched revision: 45669a3117fc93acc09a00e9fb9b4445e8990722	45669a3117fc93acc09a00e9fb9b4445e8990722	False
# flux-system	jetstack-charts      True 	Fetched revision: 7bad937cc82a012c9ee7d7a472d7bd66b48dc471	7bad937cc82a012c9ee7d7a472d7bd66b48dc471	False
# flux-system	k8s-at-home-charts   True 	Fetched revision: 1b24af9c5a1e3da91618d597f58f46a57c70dc13	1b24af9c5a1e3da91618d597f58f46a57c70dc13	False
```

Flux has a wide range of CLI options available be sure to run `flux --help` to view more!

## :robot:&nbsp; Automation

[Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) is a very useful tool that when configured will start to create PRs in your Github repository when Docker images, Helm charts or anything else that can be tracked has a newer version. The configuration for renovate is located [here](./.github/renovate.json5).

There's a couple Github workflows included in this repository that will help automate some processes.

- [Flux upgrade schedule](./.github/workflows/flux-schedule.yaml) - workflow to upgrade Flux.
- [Renovate schedule](./.github/workflows/renovate-schedule.yaml) - workflow to annotate `HelmRelease`'s which allows [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) to track Helm chart versions.

## :grey_question:&nbsp; What's next

The world is your cluster, try installing another application or if you have a NAS and want storage back by that check out the helm charts for [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) or [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner).

If you plan on exposing your ingress to the world from your home. Checkout [our guide](https://docs.k8s-at-home.com/guides/dyndns/) to run a k8s `CronJob` to update DDNS.

## :handshake:&nbsp; Thanks

Big shout out to all the authors and contributors to the projects that we are using in this repository.
