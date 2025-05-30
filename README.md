# helm-resources

This repository contains the Helm charts and resource templates for deploying the chatbot system and basic monitoring stack onto a Kubernetes cluster.  
The project supports multiple stages (dev, staging, prod) and follows best practices for scalable deployments.

## Monitoring

To install or upgrade the monitoring stack use the following command from within the monitoring directory:
```
helm  upgrade --install prometheus prometheus-community/kube-prometheus-stack  --namespace monitoring  --values .\values-devops.yaml  --set crds.install=true
```

## Continuous Deployment with Argo CD
The project uses Argo CD for GitOps-based continuous deployment. 

### Argo CD Installation
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace --set ha.enabled=false
```

### Accessing the Argo CD GUI
1. Port-forward the Argo CD server:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
2. Open the GUI:  
Navigate to https://localhost:8080 in your browser.
3. Login credentials:
   - Username: admin
   - Password: Retrieve it with:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### ArgoCD Applications
The repository includes ArgoCD application definitions for each environment in the `argocd/applications`-directory. To deploy using ArgoCD:
1. Create the required namespaces:
   ```bash
   kubectl create ns dev
   kubectl create ns staging
   kubectl create ns prod
   ```
2. Apply the ArgoCD application definitions:
   ```bash
   kubectl apply -f argocd/applications/
   ```

## Continuous Deployment with Keel
The project uses [Keel](https://keel.sh/) for automatic image updates in dev and staging environments.

### Keel Installation
```bash
helm repo add keel https://charts.keel.sh 
helm upgrade --install keel keel/keel -n keel --create-namespace --set service.type=ClusterIP
```

### How it works
- **Dev environment**: Automatically pulls and deploys `latest` tags every 5 minutes
- **Staging environment**: Automatically pulls and deploys `stable` tags every 10 minutes  
- **Prod environment**: Manual deployment with fixed version tags

### Monitoring Keel
```bash
# Check Keel logs
kubectl logs -n keel deployment/keel


## Secret Management
Secrets are securely managed using Kubernetes SealedSecrets. All credentials are encrypted and safe to store in version control.

### Creating/Updating Secrets
1. Install [kubeseal](https://github.com/bitnami-labs/sealed-secrets#installation)
2. Create/update secrets using:
```bash
# For ACR credentials
kubeseal --scope cluster-wide --cert=crt.pem -f acr-auth.yaml -o yaml > charts/chatbots/templates/acr-auth-sealedsecret.yaml

# For chatbot API-Keys
kubeseal --scope cluster-wide --cert=crt.pem -f chatbot-api-keys.yaml -o yaml > charts/chatbots/templates/chatbot-api-keys-sealedsecret.yaml
```

## Deployment Strategy
The project uses **Rolling Deployment** for zero-downtime updates:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # max 1 additional pod during update
    maxUnavailable: 0  # zero downtime --> no pods can be unavailable
```

### Benefits
- **Zero Downtime**: service remains available during updates
- **Resource Efficient**: only temporary +1 pod overhead
- **Built-in Rollback**: easy rollback with `kubectl rollout undo`
- **Perfect for stateless services**: ideal for our microservices architecture

## Applying Secrets
SealedSecrets are automatically applied with Helm deployments. Manual application:
```bash
kubectl apply -f charts/chatbots/templates/*-sealedsecret.yaml
```

## Chatbots

The chatbots and connecting world are always deployed together. 

The value files are separated into dev, staging and prod stages. 

Each bot can be configured individually. 

### Dev

The dev stage contains the latest image. It is the most unstable one.

command
```
helm upgrade --install chatbot-dev ./chatbots -n dev -f ./chatbots/values-dev.yaml
```

### Staging

The staging stage contains more stable images. The stable images are created whenever a commit is tagged with a version.

```
helm upgrade --install chatbot-staging ./chatbots -n staging -f ./chatbots/values-staging.yaml
```

### Prod

The prod stage has always fixed versions. This is done manually at the moment. But this is the most stable one.

```
helm upgrade --install chatbot-prod ./chatbots --namespace prod -f ./chatbots/values-prod.yaml
```

### Verification
To check the Quality of Service (QoS) class of the pods after deployment:
```bash
kubectl get pods -n dev --output=custom-columns=NAME:.metadata.name,QOS:.status.qosClass
kubectl get pods -n staging --output=custom-columns=NAME:.metadata.name,QOS:.status.qosClass
kubectl get pods -n prod --output=custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```
This will show the QoS class (Guaranteed/**Burstable**/BestEffort) for all pods in each namespace.
## Scaling


You can temporarily turn off a stage by scaling all Deployments in its namespace to zero replicas.

Turn off all deployments in a stage:
```bash
kubectl scale deployment --all --replicas=0 -n dev
```
Turn on all deployments in a stage:
```bash
kubectl scale deployment --all --replicas=1 -n dev
```