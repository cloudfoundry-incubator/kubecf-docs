---
title: "Deploy KubeCF on K3s"
date: 2020-10-01
weight: 3
description: >
  Step-by-step guide to deploy KubeCF on top of k3s
---

## Before you begin

This guideline will provide a quick way to deploy KubeCF with K3S and should be used only for evaluation purposes.

## Prerequisites

- [K3s](https://github.com/rancher/k3s/)
- [Helm](https://github.com/helm/helm/releases)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [KubeCF release](https://github.com/cloudfoundry-incubator/kubecf/releases)

Before you begin you need a k3s cluster running and a [KubeCF release](https://github.com/cloudfoundry-incubator/kubecf/releases) copy extracted in a folder which will be used in the tutorial, you will find notes/tips to deploy on a VM based environment.

### K3s installation notes
`curl -sfL https://raw.githubusercontent.com/rancher/k3s/master/install.sh | INSTALL_K3S_EXEC="server --no-deploy traefik --node-external-ip $EXT_IP" sh`

>
  - `--node-external-ip` Replace `$EXT_IP` with the node external IP. For example, the IP which can be reached from over the network.
  - `--no-deploy traefik` It's to disable traefik deploy by default

Depending on your Linux Distribution, you might want to install relevant packages for `k3s` to work properly, remember to check [Installation requirements](https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/) and the appropriate section for your OS. For example in OpenSUSE Tumbleweed, `zypper in -y kernel-default which curl wget open-iscsi` or switch to iptable-legacy for [Debian](https://rancher.com/docs/k3s/latest/en/advanced/#enabling-legacy-iptables-on-raspbian-buster).

- agent token to join new nodes in `/var/lib/rancher/k3s/server/node-token`
- joining a node in the k3s cluster: `curl -sfL https://raw.githubusercontent.com/rancher/k3s/master/install.sh | INSTALL_K3S_EXEC="agent --node-external-ip ext-ip" K3S_URL=https://externalip:6443 K3S_TOKEN=agent_token sh -`
- Take the `kubeconfig` from `/etc/rancher/k3s/k3s.yaml` on the master node and edit `server` with the node ip reachable from outside
- Debian hosts might need specific kernels, e.g. `apt install linux-image-4.19-cloud-amd64`

### Longhorn installation notes

Make sure to have installed host required dependencies (iscsi) ( for e.g. in Tumbleweed: `zypper in -y kernel-default which curl wget open-iscsi` )
and enable them on boot `systemctl enable --now iscsid`.


```bash

$ git clone https://github.com/longhorn/longhorn

 # Tweak depending on your number of nodes
$ cat <<EOF >>longhorn_values.yaml
persistence:  
  defaultClass: true
  defaultClassReplicaCount: 1
EOF
$ kubectl create namespace longhorn-system
$ helm install longhorn ./longhorn/chart/ --namespace longhorn-system --values longhorn_values.yaml
```

### SUSE charts

Add the SUSE helm repository, where we will install nginx-ingress from:
```bash
$ helm repo add suse https://kubernetes-charts.suse.com/
"suse" has been added to your repositories
$ helm repo update
```

## Deploy Nginx-ingress

We will use the nginx-ingress with KubeCF in the following example.

Let's create a `nginx_ingress.yaml` values file with the following content:
```bash
$ cat <<EOF >>nginx_ingress.yaml
tcp:
  2222: "kubecf/scheduler:2222"
  20000: "kubecf/tcp-router:20000"
  20001: "kubecf/tcp-router:20001"
  20002: "kubecf/tcp-router:20002"
  20003: "kubecf/tcp-router:20003"
  20004: "kubecf/tcp-router:20004"
  20005: "kubecf/tcp-router:20005"
  20006: "kubecf/tcp-router:20006"
  20007: "kubecf/tcp-router:20007"
  20008: "kubecf/tcp-router:20008"
EOF
```
we will use it to install the Ingress and support tcp routing in CF.


{{% alert title="ExternalIPs" %}}
> If you want to expose the ingress over specific externalIPs in your cluster, you can while installing the ingress with helm

> we will find th node external IPs which we are interested in:
```bash
$ kubectl get node
NAME     STATUS   ROLES    EXTERNAL-IP
host-1   Ready    master   203.0.113.1
host-2   Ready    node     203.0.113.2
host-3   Ready    node     203.0.113.3
```

> and update the `nginx_ingress` values file:
```bash
$ cat <<EOF >>nginx_ingress.yaml
service:
  externalIPs:
  - 203.0.113.2
  - 203.0.113.3
EOF
```

{{% /alert %}}

Let's install `nginx-ingress` with helm:

```bash
$ kubectl create namespace nginx-ingress
namespace/nginx-ingress created

$ helm install nginx-ingress suse/nginx-ingress \
--namespace nginx-ingress \
--values nginx_ingress.yaml
```

## Deploy Quarks-Operator
```bash

$ kubectl create namespace cf-operator
namespace/cf-operator created

$ helm install cf-operator ./cf-operator.tgz --namespace cf-operator --set "global.singleNamespace.name=kubecf"
```

## Deploy KubeCF
We will use a `nip.io` domain, for whose those aren't familiar, it's a free redirection service. Meaning that we can assign to kubecf an entire `*.domain` like `myip.nip.io` to make our deployment reachable from outside. If you aren't interested in the ingress option, just skip the instructions in the `features` block:

```bash
cat <<EOF >> kubecf-config-values.yaml
system_domain: $EXT_IP.nip.io

credentials:
  cf_admin_password: testcluster
  uaa_admin_client_secret: testcluster

features:
  ingress:
    enabled: true
    tls:
      crt: |
        -----BEGIN CERTIFICATE-----
        MIIE8jCCAtqgAwIBAgIUT/Yu/Sv8AUl5zHXXEKCy5RKJqmYwDQYJKoZIhvcMOQMM
        [...]
        xC8x/+zB7XlvcRJRio6kk670+25ABP==
        -----END CERTIFICATE-----
      key: |
        -----BEGIN RSA PRIVATE KEY-----
        MIIE8jCCAtqgAwIBAgIUSI02lj2b2ImLy/zMrjNgW5d8EygwQSVJKoZIhvcYEGAW
        [...]
        to2WV7rPMb9W9fd2vVUXKKHTc+PiNg==
        -----END RSA PRIVATE KEY-----
EOF
```

if you don't have a certificate, and if you are setting up a staging environment, you can generate one with quarks-secret and avoid to provide it in the values, for e.g. by applying (mind to replace `$EXT_IP` with your ip ):

```yaml
apiVersion: quarks.cloudfoundry.org/v1alpha1
kind: QuarksSecret
metadata:
  name: nip.quarks.ca
  namespace: kubecf
spec:
  request:
    certificate:
      alternativeNames: null
      commonName: $EXT_IP.nip.io
      isCA: true
      signerType: local
  secretName: nip.secret.ca
  type: certificate
---
apiVersion: quarks.cloudfoundry.org/v1alpha1
kind: QuarksSecret
metadata:
  name: nip.quarks.tls
  namespace: kubecf
spec:
  request:
    certificate:
      CAKeyRef:
        key: private_key
        name: nip.secret.ca
      CARef:
        key: certificate
        name: nip.secret.ca
      alternativeNames:
      - "*.$EXT_IP.nip.io"
      commonName: kubeTlsTypeCert
      isCA: false
      signerType: local
  secretName: kubecf-ingress-tls
  type: tls
```

or by providing your own certificate in `kubecf/kubecf-ingress-tls` a tls certificate type.

{{% alert title="Storage Class" %}}

> Note if you have a custom Storage class, set the relevant section in the KubeCF config values:


```bash

$ cat <<EOF >> kubecf-config-values.yaml

kube:
  storage_class: longhorn
EOF
```

{{% /alert %}}


Finally let's install KubeCF:

```bash
$ helm install kubecf --namespace kubecf ./kubecf_release.tgz --values kubecf-config-values.yaml

NAME: kubecf
LAST DEPLOYED: Thu Oct  1 11:34:06 2020
NAMESPACE: kubecf
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Welcome to your new deployment of KubeCF.

    The endpoint for use by the `cf` client is
        https://api.$EXT_IP.nip.io

    To target this endpoint and login, run
        cf login --skip-ssl-validation -a https://api.$EXT_IP.nip.io -u admin

    Please remember, it may take some time for everything to come online.

    You can use
        kubectl get pods --namespace kubecf

    to spot-check if everything is up and running, or
        watch -c 'kubectl get pods --namespace kubecf'

    to monitor continuously.

    You will be running 1 diego cell(s), with 40960Mi of disk each.
    The default app quota is set to 1024Mi, which means you'll have enough disk
    space to run about 40 apps.

    The online documentation (release notes, deployment guide) can be found at
        https://kubecf.io/docs
```

Now you are ready to [deploy Stratos](/docs/tutorials/deploy-stratos/)

{{% alert title="Note on eirini" %}}

After deployment if Eirini is enabled, it's necessary to trust the CA used by eirini to pull images from internal registry on the node.

On each node, you can do with (needs yq on the nodes):
 
```bash
$ k3s kubectl get secret bits-service-ssl -n kubecf -o yaml | yq r - 'data.ca' | base64 -d > eirini-ca.crt
$ cp -rfv eirini-ca.crt /etc/ssl/certs/ && systemctl restart k3s
```
{{% /alert %}}
