---
title: "Advanced Topics"
linkTitle: "Advanced Topics"
weight: 20
date: 2010-10-08
description: >
  Advanced topic and settings for KubeCF: Ingress, Diego and Eirini, ...
---

#### Diego vs Eirini

Diego is the standard scheduler used by KubeCF to deploy CF
applications. [Eirini](https://www.cloudfoundry.org/project-eirini/) is an alternative to Diego that follows a more
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
`features.external_database`. Check the [values.yaml](https://github.com/cloudfoundry-incubator/kubecf/blob/master/chart/values.yaml#L374) for more details.

Setting `features.external_database.seed` to `true` will allow KubeCF to
configure the various required databases for you; however, this will require
providing the root database password (via the `var-pxc-root-password` secret),
which may be an issue under some policies.

{{% alert title="Example of providing root database password" %}}

For example, to provide the `root` password of the KubeCF deployment in the `kubecf` namespace we can apply the following secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: var-pxc-root-password
  namespace: kubecf
stringData:
  password: "root"
```

{{% /alert %}}

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
