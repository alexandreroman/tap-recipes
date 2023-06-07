# How to trigger TAP package reconciliation

## Problem

TAP packages are managed by Carvel [kapp-controller](https://carvel.dev/kapp-controller/).
Those packages are periodically synchronized with the current state of your cluster
(30 min by default).

As you update package configuration, you may want to manually trigger package
reconciliation.

## Solution

Install the [kctrl CLI](https://carvel.dev/kapp-controller/docs/latest/install/#installing-kapp-controller-cli-kctrl).

Get the list of installed packages:

```shell
kctrl package installed list -A
```

Get the package name and its namespace.

Trigger package reconciliation with this command:

```shell
kctrl package installed kick -n tap-install -i tap
```

This also works with kapp-controller `Application` instances.

For example, use this command to trigger reconcilation for the Namespace Provisioner app:

```shell
kctrl app kick -n tap-namespace-provisioning -a provisioner
```
