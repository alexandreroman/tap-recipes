# How to configure TAP with a load balancer dealing with TLS termination

## Problem

Let's say you have this configuration:

- TAP with `autoTLS` disabled (`shared.ingress_issuer: ""` in `tap-values.yaml`)
- a frontend load balancer that deals with TLS termination

Your load balancer would forward requests to your cluster in plain text.

You may run into these issues:

- Backstage error: `Failed to load entity types`
- Accelerators section is empty
- Applications are served with scheme `http`, but you expect to see `https`
- you have errors in your browser developer console: `HTTP mixed content`

The problem is: TAP and its components have no clue about the base URL of your
installation. Since `autoTLS` is disabled, the configuration relies on http when
accessing TAP GUI and your applications.

However, your load balancer deals with TLS termination, and you can only reach TAP
using https from a user perspective.

## Solution

You need to configure TAP so that all endpoints use https instead of http.
With this configuration, your load balancer will be able to forward all requests to your cluster.

Configure Backstage in `tap-values.yaml`:

```yaml
tap_gui:
  app_config:
    app:
      baseUrl: https://tap-gui.INGRESS_DOMAIN
    backend:
      baseUrl: https://tap-gui.INGRESS_DOMAIN
      reading:
        allow:
        - host: *.INGRESS_DOMAIN
```

Apply [this overlay](overlay-cnrs-config-network.yaml) in your cluster:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: overlay-cnrs-config-network
  namespace: tap-install
type: Opaque
stringData:
  overlay-cnrs-config-network.yml: |
    #@ load("@ytt:overlay", "overlay")

    #@overlay/match by=overlay.subset({"metadata":{"name":"config-network"}, "kind": "ConfigMap"})
    ---
    data:
      #@overlay/match missing_ok=True
      default-external-scheme: https
```

Use this overlay in `tap-values.yaml`:

```yaml
package_overlays:
- name: cnrs
  secrets:
  - name: overlay-cnrs-config-network
```

Apply the TAP configuration.

Make sure you clear your browser cache to get the new configuration.
