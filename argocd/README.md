# ArgoCD Configuration
This directory contains ArgoCD application definitions for deploying the chatbot system to different environments.

## Applications
The following applications are defined:
- `chatbot-dev`: Deploys to the `dev` namespace using `values-dev.yaml`
- `chatbot-staging`: Deploys to the `staging` namespace using `values-staging.yaml`
- `chatbot-prod`: Deploys to the `prod` namespace using `values-prod.yaml`

## Setup Instructions
1. Create namespaces for each environment:
   ```bash
   kubectl create ns dev
   kubectl create ns staging
   kubectl create ns prod
   ```
2. Apply the ArgoCD application manifests:
   ```bash
   kubectl apply -f applications/
   ```
3. ArgoCD will automatically sync the applications and deploy the chatbot system to each namespace.