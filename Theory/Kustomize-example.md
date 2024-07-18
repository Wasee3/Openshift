# Kustomize Tutorials

Kustomize is an open-source tool that allows users to customize Kubernetes resource configurations. It is designed to manage and apply changes to Kubernetes manifests without the need for templating. Kustomize works by layering patches and overlays on top of base configurations, enabling reusable and maintainable configuration management across different environments.

# Theory of Kustomize

<details>

<summary> Feature of Kustomize (Click to expand)</summary> 

# Key Features of Kustomize
- **Base and Overlays**: Kustomize uses a base configuration that represents the common setup for all environments. Overlays are then applied on top of the base to make environment-specific changes. This approach promotes reusability and reduces duplication of configuration files.

- **No Templating**: Unlike other configuration tools, Kustomize does not use templating. Instead, it relies on a patching mechanism that applies changes directly to the YAML files. This makes the configurations easier to read, validate, and debug.

- **Declarative Approach**: Kustomize follows a purely declarative approach to configuration management. It allows users to specify the desired state of their resources in YAML files, which are then used to generate the final configuration.

- **Resource Generators**: Kustomize includes built-in generators for common resources like ConfigMaps and Secrets. This simplifies the creation and management of these resources.

- **Built-in Support in kubectl**: Kustomize is integrated into kubectl, the Kubernetes command-line tool. This integration makes it easy to use Kustomize alongside other Kubernetes operations.

# Why is Kustomize Required?
Kustomize addresses several challenges associated with managing Kubernetes configurations:

- **Environment-Specific Customizations**: In a typical deployment pipeline, applications need to be configured differently for development, staging, and production environments. Kustomize makes it easy to manage these customizations without duplicating the entire configuration.

- **Configuration Drift**: As applications evolve, maintaining consistency across multiple environments can become challenging. Kustomize helps prevent configuration drift by allowing users to manage all changes centrally and apply them consistently across environments.

- **Avoiding Template Complexity**: Traditional templating solutions like Helm use complex templating languages to achieve customization. Kustomize's patch-based approach is simpler and more intuitive, reducing the cognitive load on developers and operators.

- **Promoting Best Practices**: Kustomize encourages best practices in configuration management by promoting the separation of base and overlay configurations. This leads to cleaner, more maintainable configuration files.
</details>

# Kustomize Demonstration

Kustomize uses a directory structure to manage and apply different configurations for different environments. Here's a typical directory structure for Kustomize, organized to manage multiple environments such as base, dev, staging, and prod.

```
.
├── base
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
├── overlays
│   ├── dev
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   ├── staging
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   └── prod
│       ├── kustomization.yaml
│       └── patch.yaml
```

### Explanation
1. **base/**: Contains the base configuration files that are common across all environments.
   - **kustomization.yaml**: Specifies the resources to include and any common customizations.
   - **deployment.yaml, service.yaml, etc.**: Base YAML files for Kubernetes resources.

2. **overlays/**: Contains environment-specific customizations.
   - Each subdirectory (e.g., dev, staging, prod) represents a different environment.
   - Each environment directory has its own kustomization.yaml and patch files (e.g., patch.yaml) to override or extend the base configuration.

The patches field in kustomization.yaml contains a list of patches to be applied in the order they are specified.

Each patch may:
- be either a strategic merge patch, or a JSON6902 patch
- be either a file, or an inline string
- target a single resource or multiple resources

The patch target selects resources by group, version, kind, name, namespace, labelSelector and annotationSelector. Any resource which matches all the specified fields has the patch applied to it (regular expressions).

For a full list of operation that can be performed on bases are listed at https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/

### Example Files

**base/deployment.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        ports:
        - containerPort: 80
        env:
        - name: ENV
          value: "base"
```

**base/service.yaml**
```
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

**base/kustomization.yaml**
```
resources:
  - deployment.yaml
  - service.yaml
```

**overlays/dev/kustomization.yaml**
```
bases:
  - ../../base

patches:
  - patch.yaml
```

**overlays/dev/patch.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:dev
          env:
            - name: ENV
              value: "development"
```

**overlays/staging/kustomization.yaml**
```
bases:
  - ../../base

patches:
  - patch.yaml
```

**overlays/staging/patch.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:staging
          env:
            - name: ENV
              value: "staging"
```

**overlays/prod/kustomization.yaml**
```
bases:
  - ../../base

patches:
  - patch.yaml
```

**overlays/prod/patch.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:prod
          env:
            - name: ENV
              value: "production"
```
With this structure, you can easily apply configurations for different environments by running Kustomize in the appropriate directory:

**dev overlay **(`kustomize build overlays/dev | kubectl apply -f -`)
Output:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:dev
        ports:
        - containerPort: 80
        env:
        - name: ENV
          value: "development"
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

**staging overlay** (`kustomize build overlays/staging | kubectl apply -f -`)
Output:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:staging
        ports:
        - containerPort: 80
        env:
        - name: ENV
          value: "staging"
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

**prod overlay** (`kustomize build overlays/prod | kubectl apply -f - `)
Output:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:prod
        ports:
        - containerPort: 80
        env:
        - name: ENV
          value: "production"
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

