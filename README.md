# Demo Scenario

## Setup

- Install the **EDB** operator
- Install the **Red Hat OpenShift Serverless** operator
- Instanciante the KNative Serving

```yaml
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
    name: knative-serving
    namespace: knative-serving
```

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

- Install the **buildpacks-quarkus** ClusterBuildStrategy

```yaml
apiVersion: shipwright.io/v1alpha1
kind: ClusterBuildStrategy
metadata:
  name: buildpacks-quarkus
spec:
  parameters:
    - name: platform-api-version
      description: The referenced version is the minimum version that all relevant buildpack implementations support.
      default: "0.4"
  buildSteps:
    - name: prepare
      image: docker.io/codejive/buildpacks-quarkus-builder:jvm
      imagePullPolicy: Always
      securityContext:
        runAsUser: 0
        capabilities:
          add:
            - CHOWN
      command:
        - chown
      args:
        - -R
        - "185:185"
        - /workspace/source
      resources:
        limits:
          cpu: 500m
          memory: 1Gi
        requests:
          cpu: 250m
          memory: 65Mi
    - name: build-and-push
      image: docker.io/codejive/buildpacks-quarkus-builder:jvm
      imagePullPolicy: Always
      securityContext:
        runAsUser: 185
        runAsGroup: 185
      env: 
        - name: CNB_PLATFORM_API
          value: $(params.platform-api-version)
        - name: PARAM_SOURCE_CONTEXT
          value: $(params.shp-source-context)
        - name: PARAM_OUTPUT_IMAGE
          value: $(params.shp-output-image)
      command:
        - /bin/bash
      args:
        - -c
        - |
          set -euo pipefail
          ls -ln /workspace/source
          id

          echo "> Processing environment variables..."
          ENV_DIR="/platform/env"

          envs=($(env))

          # Denying the creation of non required files from system environments.
          # The creation of a file named PATH (corresponding to PATH system environment)
          # caused failure for python source during pip install (https://github.com/Azure-Samples/python-docs-hello-world)
          block_list=("PATH" "HOSTNAME" "PWD" "_" "SHLVL" "HOME" "")

          for env in "${envs[@]}"; do
            blocked=false

            IFS='=' read -r key value string <<< "$env"

            for str in "${block_list[@]}"; do
              if [[ "$key" == "$str" ]]; then
                blocked=true
                break
              fi
            done

            if [ "$blocked" == "false" ]; then
              path="${ENV_DIR}/${key}"
              echo -n "$value" > "$path"
            fi
          done

          LAYERS_DIR=/tmp/layers
          CACHE_DIR=/tmp/cache

          mkdir "$CACHE_DIR" "$LAYERS_DIR"

          function anounce_phase {
            printf "===> %s\n" "$1" 
          }

          anounce_phase "DETECTING"
          /cnb/lifecycle/detector -app="${PARAM_SOURCE_CONTEXT}" -layers="$LAYERS_DIR"

          anounce_phase "ANALYZING"
          /cnb/lifecycle/analyzer -layers="$LAYERS_DIR" -cache-dir="$CACHE_DIR" "${PARAM_OUTPUT_IMAGE}"

          anounce_phase "RESTORING"
          /cnb/lifecycle/restorer -cache-dir="$CACHE_DIR"

          anounce_phase "BUILDING"
          /cnb/lifecycle/builder -app="${PARAM_SOURCE_CONTEXT}" -layers="$LAYERS_DIR"

          exporter_args=( -layers="$LAYERS_DIR" -report=/tmp/report.toml -cache-dir="$CACHE_DIR" -app="${PARAM_SOURCE_CONTEXT}")
          grep -q "buildpack-default-process-type" "$LAYERS_DIR/config/metadata.toml" || exporter_args+=( -process-type web ) 

          anounce_phase "EXPORTING"
          /cnb/lifecycle/exporter "${exporter_args[@]}" "${PARAM_OUTPUT_IMAGE}"

          # Store the image digest
          grep digest /tmp/report.toml | tr -d ' \"\n' | sed s/digest=// > "$(results.shp-image-digest.path)"
      volumeMounts:
        - mountPath: /platform/env
          name: platform-env
      resources:
        limits:
          cpu: 2000m
          memory: 1Gi
        requests:
          cpu: 250m
          memory: 65Mi
```

- Create the Helm chart repository:

```yaml
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: antennas-charts
  namespace: antennas-dev
spec:
  name: antennas-charts
  connectionConfig:
    url: https://demo-orange-charts.s3.eu-west-3.amazonaws.com
```

## Cloud Native Buildpack + Helm + Operators

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
    name: buildpacks-quarkus
```

- **Developper** > **+ Add** > **Import YAML**

```yaml
apiVersion: v1
stringData:
  username: postgres
  password: secret
kind: Secret
metadata:
  name: antennas-superuser
type: kubernetes.io/basic-auth
---
apiVersion: v1
stringData:
  username: antennas
  password: antennas
kind: Secret
metadata:
  name: antennas-user
type: kubernetes.io/basic-auth
---
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: antennas-db
spec:
  instances: 1
  superuserSecret:
    name: antennas-superuser
  bootstrap:
    initdb:
      database: antennas
      owner: antennas
      secret:
        name: antennas-user
  storage:
    size: 1Gi
```

- **Developer** > **+ Add** > **Helm chart**
  - On the left pane, check **antennas-charts**
  - Click **antennas-incident**
  - Click **Create**
  - Fill-in the following values.yaml:

```yaml
apikey: 'secret'
db:
  dbname: 'antennas'
  hostname: 'antennas-db-rw'
  kind: postgresql
  password: 'antennas'
  username: 'antennas'
image:
  pullPolicy: Always
  repository: quay.io/redhat_sa_france/antennas-incident
  tag: latest
```

  - Click **Create**

- **Developer** > **+ Add** > **Container images**
- **Image stream tag from internal registry**
  - Fill in the form with the following information:
    - Project: `antennas-dev`  
    - Image Stream: `antennas-front`  
    - Tag: `latest`  
    - Application Name: `antennas-front`  
    - Resource type: **Serverless Deployment**
  - Click **Deployment** and add an **Environment variable**:
    - Name: `APIKEY`  
      Value: `secret`
    - Name: `QUARKUS_REST_CLIENT_INCIDENT_SERVICE_URL`  
      Value: `http://antennas-incident:8080`
  - Click **Create**
