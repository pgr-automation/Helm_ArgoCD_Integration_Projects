# Helm_ArgoCD_Integration_Projects

# 1. Create Helm Chart 
To create a reusable Helm chart for production deployment, you’ll want to focus on modularity, parameterization, and best practices to allow the chart to be easily customized and managed across multiple environments. Here’s a guide to help you create a production-grade, reusable Helm chart:


#### 1. Define Chart Structure
```graphql
myapp-chart/
├── charts/              # For dependencies if any
├── templates/           # YAML templates for Kubernetes resources
│   ├── deployment.yaml  # Defines the application Deployment
│   ├── service.yaml     # Defines the Service for application access
│   ├── ingress.yaml     # Defines the Ingress for external access (optional)
│   ├── configmap.yaml   # ConfigMap for app configuration
│   └── _helpers.tpl     # Helper templates for reusable values
├── values.yaml          # Default values
├── values-production.yaml # Production-specific values
├── Chart.yaml           # Chart metadata
└── README.md            # Documentation for usage
```
####  2. Define Templates for Resources

Create templates for the resources your app needs, such as Deployment, Service, ConfigMap, and Ingress.
Deployment Template

Here’s an example deployment.yaml template, parameterized for easy configuration:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app: {{ include "myapp.name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "myapp.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "myapp.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}
          resources:
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
```
#### 3. Service Template

Example service.yaml template:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
  selector:
    app: {{ include "myapp.name" . }}
```
#### 4. Ingress Template (optional)

Add an ingress.yaml if your app needs to be accessed externally.

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "myapp.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

#### 5. Parameterize Values in values.yaml

The values.yaml file should contain defaults and expose configurable parameters:
```yaml
replicaCount: 3
image:
  repository: "myrepo/myapp"
  tag: "latest"
  pullPolicy: "IfNotPresent"

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "250m"
    memory: "256Mi"

ingress:
  enabled: true
  host: "myapp.example.com"
```
#### 6. Use Production-Specific Values
```yaml
replicaCount: 5
image:
  tag: "stable"

resources:
  limits:
    cpu: "1000m"
    memory: "1024Mi"
  requests:
    cpu: "500m"
    memory: "512Mi"

ingress:
  host: "prod.myapp.example.com"
```

#### 7. Modularize with _helpers.tpl
Define helper templates in _helpers.tpl for reusable strings:

```yaml
{{- define "myapp.name" -}}
{{- .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "myapp.fullname" -}}
{{- printf "%s-%s" (include "myapp.name" .) .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

```bash
To install with production values:
helm install myapp ./myapp-chart -f values-production.yaml

To Upgrading the Chart
helm upgrade myapp ./myapp-chart -f values-production.yaml
```

### 8. Test and Deploy
1. **Lint the Chart:** `helm lint ./myapp-chart`
2. **Package and Push** to a Helm repository if deploying in a CI/CD pipeline.
3. **Test** your chart in a staging environment with production values before deploying to production.


# 2. GitOps - ArgoCD Integration
To use a Helm chart with GitOps practices in a tool like Argo CD, you can follow these steps to set up an automated deployment pipeline:

#### 1. Prepare Your Git Repository

In GitOps, the desired state of your application and environment is stored in a Git repository. Create a repository (or a folder in an existing repo) for your Helm chart configurations, and structure it to support both configuration and versioning.

The repository might look like this:

```graphql
myapp-gitops/
├── base/
│   ├── values.yaml               # Default values for all environments
│   ├── myapp-chart/               # Optional: Helm chart if not using a registry
│       └── templates/...
├── overlays/
│   ├── production/
│   │   └── values-production.yaml # Production environment values
│   ├── staging/
│       └── values-staging.yaml    # Staging environment values
└── applications/
    └── production-myapp.yaml      # Argo CD Application manifest
```

