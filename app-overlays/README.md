# How to apply overlays to deployed apps

## Problem

When deploying TAP apps, you may want to customize some resources depending
on the target environment, infrastructure or namespace.

For example: you want to override the `initialDelaySeconds` for an app which
is deployed in a production environment, but you want to stick with the default
configuration with other environments.

## Solution

Deploy [this overlay](overlay-ootb-templates-app-overlays.yaml) to your cluster:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: overlay-ootb-templates-app-overlays
  namespace: tap-install
  annotations:
    kapp.k14s.io/change-group: pkgi
stringData:
  overlay-ootb-templates-app-overlays.yaml: |
    #@ load("@ytt:overlay", "overlay")
    #@ load("@ytt:yaml", "yaml")

    #@ def appOverlays():
    - inline:
        pathsFrom:
        - secretRef:
            name: app-overlays
    #@ end

    #@ def indent(str, size):
    #@   strLines = []
    #@   indentStr = "  " * size
    #@   for line in str.splitlines():
    #@     strLines.append(indentStr + line)
    #@   end
    #@   return "\n".join(strLines)
    #@ end

    #@overlay/match by=overlay.subset({"kind":"ClusterDeploymentTemplate", "metadata": {"name": "app-deploy"}})
    ---
    spec:
        #@overlay/replace via=lambda left, right: left.replace("fetch:\n", "fetch:\n" + indent(yaml.encode(appOverlays()), 2) + "\n")
        ytt:
```

Use this overlay in `tap-values.yaml`:

```yaml
package_overlays:
- name: ootb-templates
  secrets:
  - name: overlay-ootb-templates-app-overlays
```

Apply the TAP configuration.

The goal of this overlay is to update the `ClusterDeploymentTemplate` used when
deploying apps. This resource is responsible for creating a `kapp-controller` `App`
which takes care of deploying your app resources.

As you deploy your next app, the patched `App` will rely on a local overlay
(stored as a Kubernetes `Secret` named `app-overlays` in the same namespace) which
will be applied against the app resources.

For example, let's say you create [this app overlay](app-overlays.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-overlays
stringData:
  overlay-app-delayed-probes.yaml: |
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind":"Deployment"}), expects="0+"
    ---
    spec:
      template:
        spec:
          containers:
          #@overlay/match by=overlay.index(0)
          - name: workload
            #@overlay/match-child-defaults missing_ok=True
            livenessProbe:
              initialDelaySeconds: 30
            #@overlay/match-child-defaults missing_ok=True
            readinessProbe:
              initialDelaySeconds: 30
            #@overlay/match-child-defaults missing_ok=True
            startupProbe:
              initialDelaySeconds: 30
```

In this case, all apps deployed through a `Deployment` (workload type set to `server`)
in the namespace will have additional configuration for Kubernetes probes.

You can add as many overlays as you need in this resource.

The Kubernetes `Secret` `app-overlays` should be deployed to all namespaces by leveraging
Namespace Provisioner.
