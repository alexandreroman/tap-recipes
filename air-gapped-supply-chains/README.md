# How to use TAP supply chains in air-gapped environments

## Problem

The default configuration for supply chains in TAP does not work in air-gapped
environments:

- if your project includes a wrapper script for Maven (`mvnw`) or Gradle (`gradle`),
  the script fails to download Maven / Gradle from Internet
- projects cannot be built since you need to download dependencies from Internet
  using a proxy or a mirror

## Solution

Deploy [this overlay](overlay-ootb-templates-airgapped.yaml) to your cluster:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: overlay-ootb-templates-airgapped
  namespace: tap-install
type: Opaque
stringData:
  overlay-supplychain-source-ignore.yml: |
    #@ load("@ytt:overlay", "overlay")
    #@ load("@ytt:yaml", "yaml")

    #@ def ignoreFiles():
    ignore: |
      gradle
      gradlew
      gradlew.bat
      .mvn
      mvnw
      mvnw.cmd
    #@ end

    #@overlay/match by=overlay.subset({"kind":"ClusterSourceTemplate", "metadata": {"name": "source-template"}})
    ---
    spec:
        #@overlay/replace via=lambda left, right: left.replace("ignore: |", yaml.encode(ignoreFiles()).replace("  ", "    "))
        ytt:
  overlay-kpack-bindings-overlay.yml: |
    #@ load("@ytt:overlay", "overlay")
    #@ load("@ytt:yaml", "yaml")

    #@ def serviceBindings():
    services:
    - name: maven-settings
      kind: Secret
      apiVersion: v1
    - name: npmrc
      kind: Secret
      apiVersion: v1
    #@ end

    #@overlay/match by=overlay.subset({"kind":"ClusterImageTemplate", "metadata": {"name": "kpack-template"}})
    ---
    spec:
        #@overlay/replace via=lambda left, right: left.replace("services: #@ data.values.params.buildServiceBindings", yaml.encode(serviceBindings()).replace("  ", "      ").replace("- ", "    - "))
        ytt:
```

Use this overlay in `tap-values.yaml`:

```yaml
package_overlays:
- name: ootb-templates
  secrets:
  - name: overlay-ootb-templates-airgapped
```

This overlay expects to have resources for configuring Maven and Node in your developer namespace.

Apply this [Maven configuration](maven-settings.yaml) in your developer namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: maven-settings
type: service.binding/maven
stringData:
  type: maven
  provider: sample
  settings.xml: |
    <settings>
      <mirrors>
        <mirror>
          <id>my-mirror</id>
          <name>My Mirror Repository</name>
          <url>https://my-mirror.repo.corp/maven2</url>
          <mirrorOf>external:*</mirrorOf>
        </mirror>
      </mirrors>
    </settings>
```

Apply this [Node configuration](npmrc.yaml) in your developer namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: npmrc
type: service.binding/npmrc
stringData:
  type: npmrc
  provider: sample
  .npmrc: |
    registry=https://my-mirror.repo.corp
```

You should rely on [Namespace Provisioner](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-about.html)
to automatically provision these resources in every developer namespace.

Apply the TAP configuration.
