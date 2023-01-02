## 🏘️ Supply Chain Builds
## Build application images from source code

1. Create argocd app of apps

   ```bash
   cd /projects/rainforest
   oc -n <TEAM_NAME>-ci-cd apply -f gitops/argocd/rainforest-ci-cd-dev-app-of-apps.yaml
   ```
