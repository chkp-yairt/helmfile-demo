# Streamlining Kubernetes Deployments with Helmfile: A Practical Guide.

Handling numerous Kubernetes clusters in diverse settings and regions can easily grow to be daunting. Many teams struggle with consistent, secure, and scalable management of each cluster’s configuration and secrets. We will learn about the topic of helmfile and how it can bring a central place for kubernetes deployment and demonstrate its usage through an open example repository: [`chkp-yairt/helmfile-demo`](https://github.com/chkp-yairt/helmfile-demo).

---

## What is Helmfile?

Helmfile is a declarative configuration management tool for Helm, Kubernetes' package manager. At its core, Helmfile allows DevOps teams to define, install, and upgrade even the most complex Kubernetes applications through a single YAML file that serves as a source of truth for all deployments.

### Core Concepts

Helmfile operates on several key concepts:

1. **Releases**: These are the individual Helm chart deployments you want to manage. Each release has a name, chart reference, and optional configuration settings.

2. **Environments**: Helmfile lets you define multiple environments (development, staging, production) with environment-specific values, allowing you to reuse the same configuration across different clusters or namespaces.

3. **Values**: Configuration parameters that customize how your Helm charts are deployed. These can be defined inline or in separate files.

4. **Templates**: Helmfile supports Go templating to make your configurations more dynamic and reusable.

5. **State Files**: Helmfile tracks the desired state of your Kubernetes cluster through these declarations.

### Basic Structure

Here's a simplified example of a `helmfile.yaml` that demonstrates the core structure:

```yaml
# Define chart repositories
repositories:
  - name: stable
    url: https://charts.helm.sh/stable
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

# Set global defaults
helmDefaults:
  wait: true
  timeout: 600
  createNamespace: true

# Define different environments
environments:
  default:
    values:
      - defaults.yaml
  production:
    values:
      - environments/production.yaml
  staging:
    values:
      - environments/staging.yaml

# Declare releases to be deployed
releases:
  - name: nginx-ingress
    namespace: ingress
    chart: stable/nginx-ingress
    version: 1.41.3
    values:
      - common/nginx-values.yaml
      
  - name: prometheus
    namespace: monitoring
    chart: stable/prometheus
    version: 11.12.1
    values:
      - common/prometheus-values.yaml
    set:
      - name: alertmanager.persistentVolume.enabled
        value: false
  ```
## How It Works

When you run Helmfile commands like `helmfile apply`, the tool:

  1. Reads your helmfile.yaml declaration
  2. Determines the desired state for each release
  3. Compares it with the current state in the cluster
  4. Makes the necessary changes to align the actual state with the desired state

This declarative method helps make environments uniform and makes it easy to manage complex Kubernetes apps with multiple components. If you need to run many helm install or helm upgrade commands with parameters, you can code everything and version control it, following GitOps good practices.

When you centralize your Helm deployments in Helmfile, you see all the components that make up your Kubernetes. This allows you to understand dependencies better and manage the system as a whole. 

---

## Advantages of Using Helmfile

Helmfile offers numerous benefits that make it an excellent choice for managing Kubernetes deployments at scale:

1. **Declarative Configuration Management**: 
   - Provides a single declarative specification for all your Helm chart deployments
   - Ensures consistent state across environments with version-controlled configurations

2. **Environment Flexibility**:
   - Seamlessly manages multiple environments (dev, staging, production) from a single codebase
   - Allows for environment-specific overrides while maintaining a common base configuration

3. **Improved Organization and Maintainability**:
   - Groups related releases together for easier management
   - Supports modular structure to separate concerns (monitoring, infrastructure, applications)

4. **Enhanced Security**:
   - Integrates with secret management tools like SOPS for secure handling of sensitive information
   - Separates configuration from secrets for improved security practices

5. **Reduced Duplication**:
   - Leverages templating to eliminate repetitive configuration code
   - Enables reuse of common patterns across releases and environments

6. **Powerful Testing Capabilities**:
   - Provides `diff` and `template` commands to preview changes before applying
   - Supports labels for selective testing and deployment of specific components

7. **GitOps Compatibility**:
   - Works with GitOps tools like ArgoCD
   - Enables automated deployment pipelines based on repository changes

8. **Streamlined Release Management**:
   - Manages dependencies between releases with ordered deployment
   - Simplifies rolling back to previous states when issues occur

9. **Reduced Operational Overhead**:
   - Decreases the manual effort required to manage multiple Kubernetes clusters
   - Centralizes configuration management to minimize human error

10. **Scalability**:
    - Efficiently handles deployments across dozens or hundreds of clusters
    - Maintains performance even with large numbers of releases

These advantages make Helmfile particularly valuable for organizations managing complex Kubernetes infrastructures across multiple environments and regions.

---

## Modular Organization for Scalability.

We organize Helmfiles by functionality like infrastructure, monitoring, tools, etc. To make our Helmfile setup organized and scalable. Deployments with the same root can be coordinated through Modules thus making it easy to manage.

### Example Folder Structure
Here’s an example of how we structure our Helmfile repository:

```
.
├── infra
│   ├── helmfile-monitoring.yaml
│   └── values
│       ├── central
│       │   ├── opentelemetry-collector.yaml
|       |   ├── grafana-values.yaml
│       ├── dev
│       │   └── prometheus-values.yaml
│       ├── prod-us-east
│       │   └── prometheus-values.yaml
│       ├── prod-us-west
│       │   └── prometheus-values.yaml
│       └── prod-asia-south
│           └── prometheus-values.yaml
├── tools
│   ├── helmfile-argocd.yaml
│   ├── helmfile-tools.yaml
│   └── values
│       ├── central
│       │   ├── argocd-values.yaml
│       │   └── istio-values.yaml
│       ├── dev
│       │   ├── jenkins-values.yaml
│       │   └── istio-values.yaml
│       ├── prod-us-east
│       │   ├── jenkins-values.yaml
│       │   └── istio-values.yaml
│       ├── prod-us-west
│       │   ├── jenkins-values.yaml
│       │   └── istio-values.yaml
│       └── prod-asia-south
│           ├── jenkins-values.yaml
│           └── istio-values.yaml
```

### Key Notes:
1. **Environment-Specific Configurations**:
   - Each environment folder (`dev`, `prod-us-east`, `prod-us-west`, `prod-asia-south`) includes configurations relevant to that environment, such as `grafana-values.yaml` and `prometheus-values.yaml` in `infra`, and `jenkins-values.yaml` and `istio-values.yaml` in `tools`.
2. **Packing many releases in one Helmfile**
   - Each Helmfile contains many release that are thematically connected. In our example `helmfile-monitoring.yaml` will deploy Grafana, Opentelemetry, and Prometheus according to which environnment uses them.

3. **Centralized Environment for ArgoCD and Monitoring**:
   - The approach with the `central` environment is to create one cluster to rule them all. ArgoCD controlls deployments to all other clusters.

This structure provides a clear separation of responsibilities, ensuring that ArgoCD, Grafana and Thanos are managed centrally while other components like Prometheus and Jenkins are deployed across specific environments.

---

## Testing Helmfile Configurations and Releases

One of the strengths of Helmfile is its ability to test configurations and releases before deploying them to your Kubernetes clusters. This ensures that changes are safe and predictable.

### Using Labels to Test Releases
Helmfile supports labels, which can be added to your releases to control and filter deployments. For example, consider the following release configuration:

```yaml
- name: opentelemetry-collector-central
  namespace: monitoring
  labels:
    release: otel-collector-central
    stack: monitoring
    deployable: yes
  chart: open-telemetry/opentelemetry-collector
  version: 0.80.1
  values:
    - ./values/opentelemetry-collector-central.yaml.gotmpl
```

By adding a `release` label, you can target this specific release during testing.

### Testing with Helmfile Commands
To test this release, use the `template` command to render the Helm templates and verify their output:
```bash
helmfile template -f helmfile-monitoring.yaml -e central -l release=otel-collector-central
```

This command generates the Kubernetes manifests for the `otel-collector-central` release in the `devops` environment. It's a safe way to validate your templates without making any changes to the cluster.

### Using `diff` for Safe Testing
To see the actual changes this release would make to your Kubernetes cluster, use the `diff` command:
```bash
helmfile diff -f helmfile-monitoring.yaml -e central -l release=otel-collector-central 
```

The `diff` output shows the differences between the current state and the desired state, enabling you to verify changes before applying them. Here's a generic example of what the `diff` output might look like:

```
Comparing release=otel-collector-central, chart=open-telemetry/opentelemetry-collector
opentelemetry-collector-central, ConfigMap (v1) has changed:
  # Source: opentelemetry/templates/configmap.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: opentelemetry-collector-central
    namespace: monitoring
    labels:
      helm.sh/chart: opentelemetry-0.80.1
-     app.kubernetes.io/version: "1.2.3"
+     app.kubernetes.io/version: "1.3.0"
      app.kubernetes.io/managed-by: Helm
  data:
    collector-config.yaml: |
      receivers:
        otlp:
          protocols:
            grpc:
            http:
      exporters:
        logging:
-         level: info
+         level: debug
      processors:
        batch:
      service:
        pipelines:
          traces:
            receivers: [otlp]
            processors: [batch]
            exporters: [logging]
```

This type of output gives you a clear view of what will change, such as updates to ConfigMaps, Deployments, or other Kubernetes resources.

---
## Advanced Features

### Leveraging `.gotmpl` Templates for Reusability

One of the most powerful features of Helmfile is its support for Go templates, which can be used to generalize configurations across multiple clusters. Instead of duplicating values for each cluster, we use `.gotmpl` files to define reusable templates.

#### Example `.gotmpl` Template
```yaml
image: {{- toYaml .Values.image | nindent 2 }}
env: {{- toYaml .Values.env | nindent 2 }}
serviceAccount: {{- toYaml .Values.serviceAccount | nindent 2 }}
adminPassword: {{ .Values.secrets.adminPassword }}
```

This template helps manage configurations for components like Grafana or Prometheus, allowing us to define shared settings while accommodating environment-specific overrides.

---

### Secure Secrets Management with SOPS

Managing sensitive information like API keys and passwords in a GitOps workflow can be a challenge. To address this, we use SOPS, a Helmfile addon, to securely manage encrypted secrets.

#### How We Use SOPS
1. **Create a Secret Values File**: We define sensitive values in a YAML file, such as `grafana-secrets.yaml`.
2. **Encrypt the File**: Using a KMS key, we encrypt the file:
   ```bash
   sops --encrypt --kms <kms-key-arn> grafana-secrets.yaml > grafana-secrets.yaml
   ```
3. **Add to Helmfile**: We include the encrypted file in the `secrets` section of our Helmfile for the relevant environment.
4. **Combine Values**: During deployment, Helmfile merges the encrypted secrets with regular values, creating a comprehensive values file.

#### Best Practices for SOPS
To avoid key conflicts, we use distinct root keys for regular and secret values. For example:
- **Values File**:
  ```yaml
  grafana:
    env: production
  ```
- **Secrets File**:
  ```yaml
  grafana_secrets:
    adminPassword: super-secret-password
  ```

In our templates, we reference these keys as follows:
```yaml
env: {{- toYaml .Values.grafana.env | nindent 2 }}
adminPassword: {{ .Values.grafana_secrets.adminPassword }}
```

This approach ensures that secrets are securely managed and that there are no accidental overwrites.

**Note:**  
In the example repo we've created a pseudo secret file encryped with a fake key. 
---

## Integrating Helmfile with ArgoCD

In our setup, we use ArgoCD to deploy Helmfile results to our clusters. By leveraging the Helmfile addon for ArgoCD, we ensure that a single repository manages all our environments. Changes to the repository trigger automatic deployments across clusters, streamlining the CI/CD process.

---

## Disadvantages of Using Helmfile

While Helmfile is a powerful tool, there are some challenges and limitations to be aware of:

1. **Complexity of Templating**:
   - Templating with `.gotmpl` can be tricky and cumbersome, particularly for large-scale configurations.

2. **Steep Learning Curve**:
   - The Helmfile learning curve can be steep, particularly for teams new to Helm or templating systems.

3. **Debugging Challenges**:
   - Debugging issues in Helmfile configurations often requires a deep understanding of Helm and Kubernetes, making troubleshooting harder for less experienced teams.

---

## Lessons Learned and Best Practices

Over time, we’ve refined our use of Helmfile and learned several valuable lessons:
1. **Modularity is Key**: Grouping Helmfiles by functional areas improves maintainability.
2. **Embrace Templating**: Use `.gotmpl` templates to reduce duplication and manage complexity.
3. **Secure Secrets with SOPS**: Encrypt sensitive information and use distinct root keys to avoid conflicts.
4. **Test Releases with Labels**: Use Helmfile’s labeling feature to control deployments and test changes before rolling them out to production.



---

## End-to-End Example: Deploying with Helmfile

This section walks through a complete example of using Helmfile to manage and deploy multiple releases across several environments. The example demonstrates how to define repositories, environments, and releases, along with dynamic values and secrets.

### Example Folder Structure

See the [demo repository](https://github.com/chkp-yairt/helmfile-demo) for a working example:

```
.
├── charts
│   └── custom-chart
│       ├── templates
│       │   └── deployment.yaml
│       └── Chart.yaml
├── infra
│   ├── helmfile.yaml
│   └── values
│       ├── dev
│       │   ├── custom-values.yaml
│       │   ├── grafana-values.yaml
│       │   └── custom-secrets.yaml
│       ├── prod
│       │   ├── custom-values.yaml
│       │   ├── grafana-values.yaml
│       │   └── grafana-secrets.yaml
|       |── grafana-values.yaml.gotmpl
|       |──custom-chart-values.yaml.gotmpl
```
Here there are two environment `dev` and `prod`, each of the environments is using the same two helm charts. One of the helm charts is a community chart (in our example it is `grafana`), the other is a custom chart we hold in our `charts` directory so we can change the templates if needed.

Each environment’s value files hold its specific config.

---

### Helmfile Configuration

Here is the `helmfile.yaml` configuration for multiple releases and environments:

```yaml
helmDefaults:
  createNamespace: true

repositories:
  - name: grafana
    url: https://grafana.github.io/helm-charts

environments:
  dev:
    values:
      - ./values/{{ .Environment.Name  }}/custom-values.yaml
      - ./values/{{ .Environment.Name  }}/grafana-values.yaml
    secrets:
      - ./values/{{ .Environment.Name  }}/custom-secrets.yaml

  prod:
    values:
      - ./values/{{ .Environment.Name  }}/custom-values.yaml
      - ./values/{{ .Environment.Name  }}/grafana-values.yaml
    secrets:
      - ./values/{{ .Environment.Name  }}/grafana-secrets.yaml
---
releases:
  - name: custom-release
    namespace: custom-namespace
    chart: ../charts/custom-chart
    version: 1.0.0
    labels:
      release: custom-release
      environment: dev
    values:
      - ./values/custom-chart-values.yaml.gotmpl

  - name: grafana
    namespace: community-namespace
    chart: grafana/grafana
    version: 5.0.0
    labels:
      release: grafana
      environment: prod
    values:
      - ./values/community-values.yaml.gotmpl
```

---

### Templates and Values

#### Custom Values Template (`grafana-values.yaml.gotmpl`)

```yaml
image: {{- toYaml .Values.grafana.image | nindent 2 }}
tolerations:  {{- toYaml .Values.grafana.tolerations | nindent 2 }}
global:  
  pullSecrets: {{ .Values.grafana.global.pullSecrets }}
grafana.ini: {{- toYaml .Values.grafana.grafana_ini | nindent 2 }}
nodeSelector: {{- toYaml .Values.grafana.nodeSelector | nindent 2 }}
serviceAccount:  {{- toYaml .Values.grafana.serviceAccount | nindent 2 }}
adminPassword: {{ .Values.grafana_secrets.adminPassword }}
```
#### Custom Values File  (`grafana-values.yaml`)
```yaml
grafana:
  image:
    registry: docker.io
    repository: docker-hub/grafana/grafana
    tag: "10.2.1"
    pullPolicy: IfNotPresent

  tolerations:
    - key: dedicated
      operator: Equal
      value:monitoring

  global:
    pullSecrets: []
  grafana_ini:
    database:
      wal: true
    analytics:
      check_for_updates: true
    auth:
      sigv4_auth_enabled: true
    auth.azuread:
      name: CP Azure AD
      enabled: true
      allow_sign_up: true
      auto_login: true
      scopes: openid email profile
      auth_url: https://login.microsoftonline.com/<tenant_id>/oauth2/v2.0/authorize
      token_url: https://login.microsoftonline.com/<tenant_id>/oauth2/v2.0/token
    server:
      root_url: https://<grafana_url>
    users:
      viewers_can_edit: true
    dataproxy:
      timeout: 300
    security:
      allow_embedding: true

  nodeSelector:
    app: monitoring

  serviceAccount:
    create: true
    annotations:

```
#### Custom Secret Values File  (`grafana-secrets.yaml`)
```yaml
grafana_secrets:
  env:
    adminPassword: adminpassword
```
this is a non encrypted version of the grafana-secrets.yaml we will of course encrypt it using sops before commiting it to our repository.

---

### Testing the Releases

Labels make it easy to target specific releases for testing. Here’s how to test the `custom-release` in the dev environment:

1. **Render the Template**:
   ```bash
   helmfile template -f helmfile.yaml -e dev -l release=grafana
   ```

2. **View the Differences with Current Deployment**:
   ```bash
   helmfile diff -f helmfile.yaml -e dev -l release=grafana
   ```

---

## Conclusion

Helmfile has proven invaluable for managing Kubernetes deployments. Its modular structure, templating support, SOPS integration, and ArgoCD compatibility allow us to maintain a centralized, secure, and scalable deployment process.

If you want to see almost everything in action, check out the [chkp-yairt/helmfile-demo](https://github.com/chkp-yairt/helmfile-demo) repository. Whether you’re managing a single cluster or dozens across multiple regions, Helmfile can simplify your workflow and ensure consistency. Adopting the practices outlined here will help streamline your deployments and let you focus on delivering value.
