---
title: "Deployment Walkthrough"
linkTitle: "Deployment Walkthrough"
weight: 10
date: 2017-01-05
description: >
  Explain steps to deploy KubeCF in any Kubernetes cluster
---

The intended audience of this document are developers wishing to
contribute to the Kubecf project.

Here we explain how to deploy Kubecf using:

  - A generic kubernetes cluster.
  - A released cf-operator helm chart.
  - A released kubecf helm chart.

## Kubernetes

In contrast to other recipes, we are not set on using a local
cluster. Any Kubernetes cluster will do, assuming that the following
requirements are met:

  - Presence of a default storage class (provisioner).

  - For use with a diego-based kubecf (default), a node OS with XFS
    support.

      - For GKE, using the option `--image-type UBUNTU` with the
        `gcloud beta container` command selects such an OS.

This can be any of, but not restricted to:

  - GKE ([Notes](../provider/gke.md))
  - AKS
  - EKS

Note that how to deploy and tear-down such a cluster is outside of the
scope of this recipe.

## cf-operator

The [cf-operator] is the underlying generic tool to deploy a (modified)
BOSH deployment like Kubecf for use.

[cf-operator]: https://github.com/cloudfoundry-incubator/cf-operator

It has to be installed in the same Kubernetes cluster that Kubecf will
be deployed to.

Here we are not using development-specific dependencies like bazel,
but only generic tools, i.e. `kubectl` and `helm`.

[Installing and configuring Helm](helm.md) is the same regardless of
the chosen foundation, and assuming that the cluster does not come
with Helm Tiller pre-installed.

### Deployment and Tear-down

```shell
helm install cf-operator \
     --namespace cfo \
     --set "global.singleNamespace.name=kubecf" \
     https://cf-operators.s3.amazonaws.com/helm-charts/cf-operator-5.0.0%2B0.gd7ac12bc.tgz
```

In the example above, version 5.0.0 of the operator was used. Look
into the `cf_operator` [section](https://github.com/cloudfoundry-incubator/kubecf/blob/13ffb01dff8ab5eba16da54539b36e3b3b5f758e/dependencies.yaml#L108) of the top-level `dependencies.yaml` file to find
the version of the operator validated against the current kubecf
master.

**Note:**
> The above `helm install` will generate many controllers spread over multiple pods inside the `cfo` namespace.
> Most of these controllers run inside the `cf-operator` pod.
>
> The `global.singleNamespace.name=kubecf` path tells the
controllers to watch for CRD´s instances into the `kubecf` namespace.
>
> The cf-operator helm chart will generate the `kubecf` namespace during installation, and eventually one of the
controllers will use a webhook to label this namespace with the `cf-operator-ns` key.
>
> If the `kubecf` namespace is deleted, but the operators are still running, they will no longer
know which namespace to watch. This can lead to problems, so make sure you also delete the pods
inside the `cfo` namespace, after deleting the `kubecf` namespace.

Note how the namespace the operator is installed into (`cfo`) differs
from the namespace the operator is watching for deployments (`kubecf`).

This form of deployment enables restarting the operator because it is
not affected by webhooks. It further enables the deletion of the
Kubecf deployment namespace to start from scratch, without redeploying
the operator itself.

Tear-down is done with a standard `helm delete ...` command.

## Kubecf

With all the prerequisites handled by the preceding sections it is now
possible to build and deploy kubecf itself.

This again uses helm and a released helm chart.

### Deployment and Tear-down

```shell
helm install kubecf \
     --namespace kubecf \
     https://kubecf.s3.amazonaws.com/kubecf-v2.3.0.tgz \
     --set "system_domain=kubecf.suse.dev"
```

In this default deployment, kubecf is launched without Ingress, and it
uses the Diego scheduler.

Tear-down is done with a standard `helm delete ...` command.

### Access

To access the cluster after the cf-operator has completed the
deployment and all pods are active invoke:

```sh
cf api --skip-ssl-validation "https://api.<domain>"

# Copy the admin cluster password.
admin_pass=$(kubectl get secret \
        --namespace kubecf var-cf-admin-password \
        -o jsonpath='{.data.password}' \
        | base64 --decode)

# Use the password from the previous step when requested.
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

By default, kubecf includes a single-availability database provided by the
cf-mysql-release. Kubecf also exposes a way to use an external database via the
Helm property `features.external_database`. Check the [values.yaml] for more
details.

[values.yaml]: ../../deploy/helm/kubecf/values.yaml

For local development with an external database, the command
`bash  ./scripts/deploy_mysql.sh` will bring a mysql database up and running
ready to be consumed by kubecf.

An example for the additional values to be provided to `make kubecf:apply`:

```yaml
features:
  external_database:
    enabled: true
    type: mysql
    host: kubecf-mysql.kubecf-mysql.svc
    port: 3306
    databases:
      uaa:
        name: uaa
        password: <root_password>
        username: root
      cc:
        name: cloud_controller
        password: <root_password>
        username: root
      bbs:
        name: diego
        password: <root_password>
        username: root
      routing_api:
        name: routing-api
        password: <root_password>
        username: root
      policy_server:
        name: network_policy
        password: <root_password>
        username: root
      silk_controller:
        name: network_connectivity
        password: <root_password>
        username: root
      locket:
        name: locket
        password: <root_password>
        username: root
      credhub:
        name: credhub
        password: <root_password>
        username: root
```