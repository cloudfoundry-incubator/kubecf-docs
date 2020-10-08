---
title: "Deployment Walkthrough"
linkTitle: "Deployment Walkthrough"
weight: 10
date: 2017-01-05
description: >
  Explain steps to deploy KubeCF in any Kubernetes cluster
---

The intended audience of this document are developers wishing to contribute to the Kubecf project.

Here we explain how to deploy Kubecf using:

- A generic kubernetes cluster.
- A released cf-operator helm chart.
- A released kubecf helm chart.

## Before

To assure that `kubecf` is installed with the correct `cf-operator` version, it's highly recommended
to use the kubecf bundle artifact from the GitHub [releases](https://github.com/cloudfoundry-incubator/kubecf/releases)
page.

## Kubernetes

In contrast to other instructions, we are not set on using a local cluster. Any Kubernetes cluster will
do, assuming that the following requirements are met:

- Presence of a default storage class (provisioner).

- For use with a diego-based kubecf (default), a node OS with XFS
    support.

  - For GKE, using the option `--image-type UBUNTU` with the `gcloud beta container` command selects
    such an OS.

This can be any of, but is not restricted to:

- GKE
- AKS
- EKS

Note that how to deploy and tear-down such a cluster is outside of the
scope of instructions.

## cf-operator

The [cf-operator] is the underlying generic tool to deploy a (modified)
BOSH deployment like Kubecf for use.

[cf-operator]: https://github.com/cloudfoundry-incubator/cf-operator

It has to be installed in the same Kubernetes cluster that Kubecf will
be deployed to.

Here we are not using development-specific dependencies like bazel,
but only generic tools, i.e. `kubectl` and `helm`.

Installing and configuring Helm is the same regardless of
the chosen foundation, and assuming that the cluster does not come
with Helm Tiller pre-installed.

### Deployment

First, let's create the `cf-operator` namespace manually

```shell
$ kubectl create namespace cf-operator

namespace/cf-operator
```

then we can proceed with the `cf-operator` deployment

```shell
$ helm install cf-operator \
--namespace cf-operator \
--set "global.singleNamespace.name=kubecf" \
./cf-operator.tgz

NAME: cf-operator
LAST DEPLOYED: Mon Sep 28 11:30:32 2020
NAMESPACE: cf-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Running the operator will install the following CRD´s:

- boshdeployments.quarks.cloudfoundry.org
- quarksjobs.quarks.cloudfoundry.org
- quarksecrets.quarks.cloudfoundry.org
- quarkstatefulsets.quarks.cloudfoundry.org

You can always verify if the CRD´s are installed, by running:
 $ kubectl get crds

You can check the charts README: `helm show readme quarks/cf-operator` for more information about configuration options.

Interacting with the cf-operator pod

1. Check the cf-operator pod status
  kubectl -n cf-operator get pods

2. Tail the cf-operator pod logs
  export OPERATOR_POD=$(kubectl get pods -l name=cf-operator --namespace cf-operator --output name)
  kubectl -n cf-operator logs $OPERATOR_POD -f

3. Apply one of the BOSH deployment manifest examples
  kubectl -n kubecf apply -f docs/examples/bosh-deployment/boshdeployment-with-custom-variable.yaml

4. See the cf-operator in action!
  watch -c "kubectl -n kubecf get pods"
```

and wait

```shell
$ watch -c kubectl -n cf-operator get pods

Every 2.0s: kubectl -n cf-operator get pods                                                                 Jaimes-MacBook-Pro.local: Mon Sep 28 11:31:04 2020

NAME                                         READY   STATUS    RESTARTS   AGE
cf-operator-c89644498-q75dh                  1/1     Running   0          30s
cf-operator-quarks-job-85665697bb-72vsp      1/1     Running   0          31s
cf-operator-quarks-secret-844844556b-xl9jq   1/1     Running   0          31s
```

until all the pods are up & running.

**Note:**
> The above `helm install` will generate many controllers spread over multiple pods inside the `cfo`
> namespace. Most of these controllers run inside the `cf-operator` pod.
>
> The `global.singleNamespace.name=kubecf` path tells the
controllers to watch for CRD´s instances into the `kubecf` namespace.
>
> The cf-operator helm chart will generate the `kubecf` namespace during installation, and
> eventually one of the controllers will use a webhook to label this namespace with the
> `cf-operator-ns` key.
>
> If the `kubecf` namespace is deleted, but the operators are still running, they will no longer
> know which namespace to watch. This can lead to problems, so make sure you also delete the pods
> inside the `cfo` namespace, after deleting the `kubecf` namespace.

**Note:** 
> how the namespace the operator is installed into (`cfo`) differs from the namespace the operator
> is watching for deployments (`kubecf`).
> This form of deployment enables restarting the operator because it is not affected by webhooks.
> It further enables the deletion of the Kubecf deployment namespace to start from scratch, without
> redeploying the operator itself.

### Tear-down

```shell
$ helm uninstall cf-operator --namespace cf-operator

release "cf-operator" uninstalled
```

## KubeCF

With all the prerequisites handled by the preceding sections it is now possible to build and deploy
kubecf itself.

This again uses helm and a released helm chart.

### Deployment

```shell
$ helm install kubecf \
--namespace kubecf \
--set "system_domain=kubecf.yourdomain.net" \
./kubecf_release.tgz

NAME: kubecf
LAST DEPLOYED: Mon Sep 28 11:39:01 2020
NAMESPACE: kubecf
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Welcome to your new deployment of KubeCF.

    The endpoint for use by the `cf` client is
        https://api.kubecf.suse.dev

    To target this endpoint and login, run
        cf login --skip-ssl-validation -a https://api.kubecf.suse.dev -u admin

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

and wait until all the pods are up & running (which can take around 20 minutes)

```shell
$  watch -c 'kubectl get pods --namespace kubecf'

Every 2.0s: kubectl get pods --namespace kubecf                                                             Jaimes-MacBook-Pro.local: Mon Sep 28 12:03:23 2020

NAME                                     READY   STATUS      RESTARTS   AGE
api-0                                    17/17   Running     1          16m
apps-dns-77b46d5657-5rm24                1/1     Running     0          24m
auctioneer-0                             6/6     Running     1          16m
bosh-dns-8679ff8bd4-5hb95                1/1     Running     0          22m
bosh-dns-8679ff8bd4-fzkhq                1/1     Running     0          22m
cc-worker-0                              6/6     Running     0          16m
credhub-0                                8/8     Running     2          16m
database-0                               2/2     Running     0          22m
database-seeder-25b239b49d162e7c-xhrmr   0/2     Completed   0          23m
diego-api-0                              9/9     Running     2          16m
diego-cell-0                             12/12   Running     2          16m
doppler-0                                6/6     Running     0          16m
log-api-0                                9/9     Running     0          16m
log-cache-0                              10/10   Running     0          16m
nats-0                                   7/7     Running     0          16m
router-0                                 7/7     Running     0          16m
routing-api-0                            6/6     Running     0          16m
scheduler-0                              13/13   Running     1          16m
singleton-blobstore-0                    8/8     Running     0          16m
tcp-router-0                             7/7     Running     0          16m
uaa-0                                    9/9     Running     4          16m
```

In this default deployment, kubecf is launched without Ingress, and it
uses the Diego scheduler.

### Tear-down

```shell
$ helm uninstall kubecf --namespace kubecf

release "kubecf" uninstalled
```

### Access

Run the following commands to access the cluster after the cf-operator and KubeCF has completed
sucessfully the deployment and all pods are up and running correctly

```shell
cf api --skip-ssl-validation "https://api.<domain>"

```

Copy the admin cluster password.

```shell

$ admin_pass=$(kubectl get secret \
        --namespace kubecf var-cf-admin-password \
        -o jsonpath='{.data.password}' \
        | base64 --decode)
```

Use the password from the previous step when requested.

```shell
cf auth admin "${admin_pass}"


```

### Advanced Topics

#### Diego vs Eirini

Diego is the standard scheduler used by kubecf to deploy CF
applications. Eirini is an alternative to Diego that follows a more
Kubernetes native approach, deploying the CF apps directly to a
Kubernetes namespace.

To activate this alternative, use the option
`--set features.eirini.enabled=true` when deploying kubecf from its chart.

#### Diego Cell Affinities & Tainted Nodes

Note that the `diego-cell` pods used by the Diego standard scheduler
are

  - privileged,
  - use large local emptyDir volumes (i.e. require node disk storage),
  - and set kernel parameters on the node.

These things all mean that these pods should not live next to other
Kubernetes workloads. They should all be placed on their own
__dedicated nodes__ instead where possible.

This can be done by setting affinities and tolerations, as explained in
the associated [tutorial].

[tutorial]: {{<ref affinities-and-tolerations>}}

#### Ingress

By default, the cluster is exposed through its Kubernetes services.

To use the NGINX ingress instead, it is necessary to:

- Install and configure the NGINX Ingress Controller.
- Configure Kubecf to use the ingress controller.

This has to happen before deploying kubecf.

##### Installation of the NGINX Ingress Controller

```sh
helm install stable/nginx-ingress \
  --name ingress \
  --namespace ingress \
  --set "tcp.2222=kubecf/scheduler:2222" \
  --set "tcp.<services.tcp-router.port_range.start>=kubecf/tcp-router:<services.tcp-router.port_range.start>" \
  ...
  --set "tcp.<services.tcp-router.port_range.end>=kubecf/tcp-router:<services.tcp-router.port_range.end>"
```

The `tcp.<port>` option uses the NGINX TCP pass-through.

In the case of the `tcp-router` ports, one `--set` for each port is required, starting with
`services.tcp-router.port_range.start` and ending with `services.tcp-router.port_range.end`. Those
values are defined on the `values.yaml` file with default values.

##### Configure kubecf

Use the Helm option `--set features.ingress.enabled=true` when
deploying kubecf.

#### External Database

By default, KubeCF includes an internal single-availability database. KubeCF
also exposes a way to use an external database via the Helm property
`features.external_database`. Check the [values.yaml] for more details.

[values.yaml]: ../../deploy/helm/kubecf/values.yaml

Setting `features.external_database.seed` to `true` will allow KubeCF to
configure the various required databases for you; however, this will require
providing the root database password (via the `var-pxc-root-password` secret),
which may be an issue under some policies.

It is also possible to manually create all necessary databases externally, and
then provide the individual credentials to KubeCF.  When deploying KubeCF with
Helm, it will error out if any of the required database credentials are missing.
An example configuration is below:

```yaml
features:
  external_database:
    enabled: true
    type: mysql
    host: mariadb-server.corp.example.com
    port: 3306
    seed: false
    databases:
      uaa:
        name: uaa
        username: uaa-database-user
        password: PAWjxQst5l16J3w3vJdfX7fXY8HyHtyb
      cc:
        name: cloud_controller
        username: cc-database-user
        password: qj5KI3qwVe+XVFENAuEU9AfSkpUK/nzb
      bbs:
        name: diego
        username: diego-database-user
        password: z7gvkjWNUgX+xcn3Ia0loMEnAD7MXBgE
      routing_api:
        name: routing-api
        username: routing-api-database-user
        password: bTL6DhK89F+G05OZtHbvEdR1uwkyRMJt
      policy_server:
        name: network_policy
        username: network-policy-database-user
        password: EYviLFS/F4dAyVry5Stm8wrpOI64Xmnz
      silk_controller:
        name: network_connectivity
        username: silk-database-user
        password: ah7nbU1wsHuZ4BtmcdU1vV37KgVV2lLf
      locket:
        name: locket
        username: locket-database-user
        password: rWicBxhg8mIrhkuSR/aDlqljOMORuyL
      credhub:
        name: credhub
        username: credhub-database-user
        password: 5tJcIWHCR1QTLdfQDiN1Mz8K8jB+clD
      autoscaler:
        name: autoscaler
        username: autoscaler-database-user
        password: cYo2Tdi61N39mUjoEksXHuu3F7Dk58F
```
