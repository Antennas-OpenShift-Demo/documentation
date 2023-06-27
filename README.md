# Demo Scenario

## Setup

- Install the **Shipwright.io** operator
- Instanciate Shipwright

```yaml
apiVersion: operator.shipwright.io/v1alpha1
kind: ShipwrightBuild
metadata:
  name: shipwright-operator
spec:
  targetNamespace: shipwright-build
```

- Give **privileged** SCC to the **pipeline** ServiceAccount

```sh
oc adm policy add-scc-to-user privileged -z pipeline -n antennas-dev
```

- Install the **buildpacks-v3** ClusterBuildStrategy

```sh
oc apply -f https://raw.githubusercontent.com/shipwright-io/build/v0.11.0/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml
```

## Cloud Native Buildpack

- Create a new project named `antennas-dev`
- Create the Shipwright Build:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: Build
metadata:
  name: antennas-front
  namespace: antennas-dev
spec:
  output:
    image: image-registry.openshift-image-registry.svc:5000/antennas-dev/antennas-front:latest
  source:
    url: 'https://github.com/nmasse-itix/antennas-front.git'
  strategy:
    kind: ClusterBuildStrategy
    name: buildpacks-v3
```

- **Developer** > **+ Add** > **Container images**
- **Image stream tag from internal registry**
  - Project: `antennas-dev`  
    Image Stream: `antennas-front`  
    Tag: `latest`  
    Application Name: `antennas-front`
  - Click **Create**

**TODO**: fix missing command

