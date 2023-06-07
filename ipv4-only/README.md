# How to deploy TAP in IPv4-only environments

## Problem

In IPv4-only environments, the TAP Contour package fails to reconcile.
This happens in Kubernetes distributions such as
[TKGI](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid-Integrated-Edition/index.html).

## Solution

Deploy [this overlay](overlay-contour-fix-ipv6.yaml) to your cluster:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: overlay-contour-fix-ipv6
  namespace: tap-install
stringData:
  overlay-contour-fix-ipv6.yml: |
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind": "Deployment"}),expects=1
    ---
    spec:
      template:
        spec:
          containers:
          #@overlay/match by=overlay.map_key("name")
          - name: contour
            #@overlay/replace
            args:
            - serve
            - --incluster
            - '--xds-address=0.0.0.0'
            - --xds-port=8001
            - '--stats-address=0.0.0.0'
            - '--http-address=0.0.0.0'
            - '--envoy-service-http-address=0.0.0.0'
            - '--envoy-service-https-address=0.0.0.0'
            - '--health-address=0.0.0.0'
            - --contour-cafile=/certs/ca.crt
            - --contour-cert-file=/certs/tls.crt
            - --contour-key-file=/certs/tls.key
            - --config-path=/config/contour.yaml
```

Use this overlay in `tap-values.yaml`:

```yaml
package_overlays:
- name: contour
  secrets:
  - name: overlay-fix-contour-ipv6
```

Apply the TAP configuration.
