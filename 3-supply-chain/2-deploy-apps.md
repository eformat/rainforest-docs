## 🎸 Deploy Apps
## Deploy AIML apps

1. Create argocd app of apps

   ```bash
   cd /projects/rainforest
   oc -n <TEAM_NAME>-ci-cd apply -f gitops/argocd/daintree-dev-app-of-apps.yaml
   ```
