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

#### 2. Set Up the Argo CD Application

Argo CD can deploy Helm charts directly by referencing your Git repository. You can configure an Argo CD Application manifest to point to your Helm chart in this repository.

Here’s an example Argo CD Application manifest for deploying the Helm chart to production:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-myapp
  namespace: argocd         # Typically, Argo CD runs in its own namespace
spec:
  project: default          # Use an existing Argo CD project or create a new one
  source:
    repoURL: 'https://github.com/yourorg/myapp-gitops'
    targetRevision: HEAD    # Or a specific branch, tag, or commit hash
    path: base              # Path to the Helm chart or configurations
    helm:
      valueFiles:           # List of value files to use for this deployment
        - values.yaml       # Base values
        - overlays/production/values-production.yaml # Environment-specific values
  destination:
    server: https://kubernetes.default.svc # Kubernetes API endpoint
    namespace: myapp-prod    # Namespace to deploy the app in
  syncPolicy:
    automated:
      prune: true           # Automatically remove resources no longer in Git
      selfHeal: true        # Automatically apply changes in Git
    syncOptions:
      - CreateNamespace=true # Ensure the namespace is created if it doesn't exist
```

#### 3. Customize with Helm Parameters in Argo CD (If required)

Argo CD supports setting Helm values directly in the Application manifest. For example, you could override parameters without changing the values.yaml file:
```yaml
source:
  helm:
    parameters:
      - name: replicaCount
        value: "5"
      - name: image.tag
        value: "v1.2.3"
```
This is useful if you want to apply quick, environment-specific overrides directly in Argo CD without modifying Git files.

#### 4. Commit and Push Changes

Push your Helm chart and Application manifest to your Git repository. Argo CD will automatically detect changes and apply them based on its sync policy.

#### 5. Set Up Sync Policies in Argo CD

In Argo CD:

    - Manual Sync lets you control when changes are applied by requiring manual approval.
    - Automatic Sync (used above) ensures that Argo CD continuously monitors the Git repository and syncs any changes to the live environment.

You can view your Argo CD application in the Argo CD dashboard to track deployments, monitor sync status, and review logs for troubleshooting.

#### 6. Deploy Using Argo CD CLI or UI

With your setup in place, you can trigger deployments manually or let Argo CD manage them automatically. To trigger a manual sync:
```bash
argocd app sync production-myapp
```

#### 7 1. Update the Version in Git

When a new release is available for your application, you’ll need to update the Helm values or chart version in your Git repository.
Example 1: Update the Image Tag

If the new release is a change in the Docker image version, update the image.tag value in your environment-specific values file (e.g., values-production.yaml):

```yaml
# overlays/production/values-production.yaml
image:
  repository: "myrepo/myapp"
  tag: "v2.0.0"  # Update to the new version here
```
#### 7. 2: Update the Helm Chart Version

If you’re using a Helm chart from a chart repository (like an official chart or a custom Helm repository), update the source section in your Argo CD Application manifest to specify the new chart version.
```yaml
# applications/production-myapp.yaml
spec:
  source:
    chart: myapp-chart
    targetRevision: "2.0.0"  # Specify the new Helm chart version here
    helm:
      valueFiles:
        - values.yaml
        - overlays/production/values-production.yaml
```
- Commit and Push Changes to Git

## Jenkins CICD

```groovy
pipeline {
    agent any

    environment {
        // Set up environment variables
        GIT_REPO = 'https://github.com/yourorg/myapp-gitops.git' // Your GitOps repo
        GIT_BRANCH = 'main'                                      // Git branch to push updates
        CHART_PATH = 'base/'                                     // Path to the Helm chart in the Git repo
        VALUES_FILE = 'overlays/production/values-production.yaml'
        IMAGE_TAG = ''                                           // This will be populated dynamically
        GIT_CREDENTIALS_ID = 'your-git-credentials-id'           // Jenkins credential ID for Git
    }

    stages {
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    // Build and tag the Docker image
                    IMAGE_TAG = "v${env.BUILD_ID}" // Use a unique tag (e.g., build number)
                    docker.build("myrepo/myapp:${IMAGE_TAG}").push()
                }
            }
        }

        stage('Update Helm Chart Values') {
            steps {
                script {
                    // Clone the GitOps repository
                    checkout([$class: 'GitSCM', 
                        branches: [[name: GIT_BRANCH]],
                        userRemoteConfigs: [[url: GIT_REPO, credentialsId: GIT_CREDENTIALS_ID]]
                    ])

                    // Update the image tag in the values file
                    def valuesFile = readYaml file: VALUES_FILE
                    valuesFile.image.tag = IMAGE_TAG
                    writeYaml file: VALUES_FILE, data: valuesFile

                    // Commit and push changes
                    sh """
                    git config user.name "jenkins"
                    git config user.email "jenkins@example.com"
                    git add ${VALUES_FILE}
                    git commit -m "Update image tag to ${IMAGE_TAG}"
                    git push origin ${GIT_BRANCH}
                    """
                }
            }
        }

        stage('Trigger Argo CD Sync') {
            steps {
                script {
                    // Optionally trigger Argo CD sync via CLI or API (if Argo CD doesn't have automatic sync enabled)
                    // This step requires the Argo CD CLI and an API token
                    withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                        sh """
                        argocd app sync production-myapp \
                            --grpc-web \
                            --server argocd-server.yourdomain.com \
                            --auth-token $ARGOCD_TOKEN
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    // Optional: Verify the new image is running
                    sh "kubectl rollout status deployment/myapp-deployment -n myapp-prod"
                }
            }
        }
    }

    post {
        always {
            // Clean up the workspace
            cleanWs()
        }
        success {
            echo "Deployment to production successful!"
        }
        failure {
            echo "Deployment failed. Please check the logs."
        }
    }
}
```
