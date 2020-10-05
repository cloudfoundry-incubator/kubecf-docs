
---
title: "Upgrading KubeCF deployments"
linkTitle: "KubeCF Upgrades"
weight: 6
date: 2020-06-19
description: >
  How to upgrade KubeCF
---

## Upgrading from a previous deployment

Upgrading is roughly the same as doing an [initial deployment]; however, please
use `helm upgrade` intead of `helm install`.

If you are providing a configuration file that was originally from a previous
deployment, please take care to review the configuration to ensure you are not
maintaining default values from a previous version unintentionally.  Where
possible, not specifying configuration that maintains the default values will
prevent accidentally changing them if the defaults have changed in the new
version.

[initial deployment]: https://kubecf.io/docs/deployment/kubernetes-deploy/

## Upgrading from SCF

Please refer to the [SUSE documentation].  Exporting data from the SCF
insallation and importing into KubeCF is required.

[SUSE documentation]: https://documentation.suse.com/suse-cap/2.0/html/cap-guides/cha-cap-upgrade.html
