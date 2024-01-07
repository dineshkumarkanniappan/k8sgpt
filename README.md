# Installing Local-AI in Kubernetes Cluster

This set of commands is related to Helm, a package manager for Kubernetes.these commands allow you to add a Helm chart repository, update the repository information, create a custom values.yaml file, let you potentially modify those values if needed, and finally install the Helm chart into a Kubernetes cluster using the modified or default configurations.


1. This command adds a Helm chart repository named go-skynet with the URL https://go-skynet.github.io/helm-charts/. Helm repositories are collections of charts (packages of pre-configured Kubernetes resources).

    ```
    helm repo add go-skynet https://go-skynet.github.io/helm-charts/
    ```

2. This updates the Helm repositories to ensure that you have the latest information about available charts and their versions from the added repositories.

    ``` 
    helm repo update
    ```

3. Create a values.yaml file for the LocalAI chart and customize as needed:

```yml
replicaCount: 1

deployment:
  image: quay.io/go-skynet/local-ai:latest
  env:
    threads: 4
    context_size: 512
  modelsPath: "/models"

resources:  {}

promptTemplates:  {}

# Models to download at runtime
models:
  forceDownload: false

  # The list of URLs to download models from
  # Note: the name of the file will be the name of the loaded model
  list:
    - url: "https://gpt4all.io/models/ggml-gpt4all-j.bin"
      # basicAuth: base64EncodedCredentials

  # Persistent storage for models and prompt templates.
  # PVC and HostPath are mutually exclusive. If both are enabled,
  # PVC configuration takes precedence. If neither are enabled, ephemeral
  # storage is used.
  persistence:
    pvc:
      enabled: false
      size: 6Gi
      accessModes:
        - ReadWriteOnce

      annotations: {}

      # Optional
      storageClass: ~

    hostPath:
      enabled: false
      path: "/models"

service:
  type: ClusterIP
  port: 80
  annotations: {}
  # If using an AWS load balancer, you'll need to override the default 60s load balancer idle timeout
  # service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "1200"

ingress:
  enabled: false
  className: ""
  annotations:
    {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

nodeSelector: {}

tolerations: []

affinity: {}
```
    
4. This command installs the Helm chart named local-ai from the go-skynet repository. It uses the values.yaml file as a source for customized configurations (-f flag). The chart will be installed in your Kubernetes cluster according to the specified configurations.

    ```
    helm install local-ai go-skynet/local-ai -f values.yaml
    ```
---
# Installing k8sGpt with localAI backend in Kubernetes

These commands add a Helm repository, update the local cache with the repository's information, and then install a specific Helm chart (k8sgpt-operator) from the added repository into a specified namespace (k8sgpt-operator-system).

1. This command adds a Helm repository named k8sgpt with the URL https://charts.k8sgpt.ai/. It allows Helm to access charts stored in this repository.

    ```
    helm repo add k8sgpt https://charts.k8sgpt.ai/
    ```

2. This command updates the local Helm repository cache, ensuring that the latest information about available charts and their versions is fetched from all added repositories. This is important to have the most up-to-date chart information before installation.

    ```
    helm repo update
    ```

3. This command installs a Helm chart named k8sgpt-operator from the k8sgpt repository.
    ```
    helm install release k8sgpt/k8sgpt-operator -n k8sgpt-operator-system --create-namespace
    ```

 Here's the breakdown:

+ **release** is the name given to the release/installation of this Helm chart. You can replace it with a name of your choice for this particular installation.

+ **k8sgpt/k8sgpt-operator** specifies the chart to be installed. It includes the repository (k8sgpt) and the chart name (k8sgpt-operator) within that repository.

+ **-n k8sgpt-operator-system** sets the namespace where the chart will be installed. In this case, it specifies a namespace named k8sgpt-operator-system. If the namespace doesn't exist, the **--create-namespace** flag creates it during installation.

4. Apply the K8sGPT configuration object:

```yml
kubectl apply -f - << EOF
apiVersion: core.k8sgpt.ai/v1alpha1
kind: K8sGPT
metadata:
  name: k8sgpt-local-ai
  namespace: default
spec:
  ai:
    enabled: true
    model: ggml-gpt4all-j
    backend: localai
    baseUrl: http://local-ai.local-ai.svc.cluster.local:8080/v1
  noCache: false
  repository: ghcr.io/k8sgpt-ai/k8sgpt
  version: v0.3.8
EOF
```

Note: ensure that the value of baseUrl is a properly constructed DNS name for the LocalAI Service. It should take the form: http://local-ai.<namespace_local_ai_was_installed_in>.svc.cluster.local:8080/v1.

5. Once the custom resource has been applied the K8sGPT-deployment will be installed and you will be able to see the Results objects of the analysis after some minutes (if there are any issues in your cluster):

```yml
kubectl get results -o json | jq .
{
  "apiVersion": "v1",
  "items": [
    {
      "apiVersion": "core.k8sgpt.ai/v1alpha1",
      "kind": "Result",
      "spec": {
        "details": "The error message means that the service in Kubernetes doesn't have any associated endpoints, which should have been labeled with \"control-plane=controller-manager\". \n\nTo solve this issue, you need to add the \"control-plane=controller-manager\" label to the endpoint that matches the service. Once the endpoint is labeled correctly, Kubernetes can associate it with the service, and the error should be resolved.",
      }
    }
}
```
