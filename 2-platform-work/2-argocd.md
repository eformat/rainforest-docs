## 🐙 ArgoCD
## ArgoCD GitOps setup

1. Check env vars

   ```bash
   echo $TEAM_NAME
   echo $GIT_SERVER
   ```
   
   ```bash
   export SERVICE_ACCOUNT=vault
   ```
   
2. Helm setup

   ```bash
   helm repo add redhat-cop https://redhat-cop.github.io/helm-charts
   ```

3. ArgoCD Operator

    ```bash
    run()
    {
      NS=$(oc get subscriptions.operators.coreos.com/openshift-gitops-operator -n openshift-operators \
        -o jsonpath='{.spec.config.env[?(@.name=="ARGOCD_CLUSTER_CONFIG_NAMESPACES")].value}')
      opp=
      if [ -z $NS ]; then
        NS="${TEAM_NAME}-ci-cd"
        opp=add
      elif [[ "$NS" =~ .*"${TEAM_NAME}-ci-cd".* ]]; then
        echo "${TEAM_NAME}-ci-cd already added."
        return
      else
        NS="${TEAM_NAME}-ci-cd,${NS}"
        opp=replace
      fi
      oc -n openshift-operators patch subscriptions.operators.coreos.com/openshift-gitops-operator --type=json \
        -p '[{"op":"'$opp'","path":"/spec/config/env/1","value":{"name": "ARGOCD_CLUSTER_CONFIG_NAMESPACES", "value":"'${NS}'"}}]'
      echo "EnvVar set to: $(oc get subscriptions.operators.coreos.com/openshift-gitops-operator -n openshift-operators \
        -o jsonpath='{.spec.config.env[?(@.name=="ARGOCD_CLUSTER_CONFIG_NAMESPACES")].value}')"
    }
    run
    ```
 
4. Setup ArgoCD

   ```yaml
   cat << EOF > /tmp/argocd-values.yaml
   ignoreHelmHooks: true
   operator: []
   namespaces:
     - ${TEAM_NAME}-ci-cd
   argocd_cr:
     statusBadgeEnabled: true
     repo:
       mountsatoken: true
       serviceaccount: ${SERVICE_ACCOUNT}
       volumes:
       - name: custom-tools
         emptyDir: {}
       initContainers:
       - name: download-tools
         image: registry.access.redhat.com/ubi8/ubi-minimal:latest
         command: [sh, -c]
         env:
           - name: AVP_VERSION
             value: "1.11.0"
         args:
           - >-
             curl -Lo /tmp/argocd-vault-plugin https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v\${AVP_VERSION}/argocd-vault-plugin_\${AVP_VERSION}_linux_amd64 && chmod +x /tmp/argocd-vault-plugin && mv /tmp/argocd-vault-plugin /custom-tools/
         volumeMounts:
         - mountPath: /custom-tools
           name: custom-tools
       volumeMounts:
       - mountPath: /usr/local/bin/argocd-vault-plugin
         name: custom-tools        
         subPath: argocd-vault-plugin    
     initialRepositories: |
       - name: rainforest
         url: https://${GIT_SERVER}/${TEAM_NAME}/rainforest.git
     repositoryCredentials: |
       - url: https://${GIT_SERVER}
         type: git
         passwordSecret:
           key: password
           name: git-auth
         usernameSecret:
           key: username
           name: git-auth
     configManagementPlugins: |
       - name: argocd-vault-plugin
         generate:
           command: ["sh", "-c"]
           args: ["argocd-vault-plugin -s ${TEAM_NAME}-ci-cd:team-avp-credentials generate ./"]
       - name: argocd-vault-plugin-helm
         init:
           command: [sh, -c]
           args: ["helm dependency build"]
         generate:
           command: ["bash", "-c"]
           args: ['helm template "\$ARGOCD_APP_NAME" -n "\$ARGOCD_APP_NAMESPACE" -f <(echo "\$ARGOCD_ENV_HELM_VALUES") . | argocd-vault-plugin generate -s ${TEAM_NAME}-ci-cd:team-avp-credentials -']
       - name: argocd-vault-plugin-kustomize
         generate:
           command: ["sh", "-c"]
           args: ["kustomize build . | argocd-vault-plugin -s ${TEAM_NAME}-ci-cd:team-avp-credentials generate -"]
EOF
   ```

5. Install

   ```bash
   helm upgrade --install argocd \
     --namespace ${TEAM_NAME}-ci-cd \
     -f /tmp/argocd-values.yaml \
     --create-namespace \
     redhat-cop/gitops-operator
   ```

6. Login

    ```bash
    echo https://$(oc get route argocd-server --template='{{ .spec.host }}' -n ${TEAM_NAME}-ci-cd)
    ```

7. Argocd Projects setup

   ```bash
   oc apply -n ${TEAM_NAME}-ci-cd -k /projects/rainforest/tenant-argocd/overlay/rainforest
   ```

8. Gitlab, Group, PAT, check in code

   Login to Gitlab and create an Internal Group called <TEAM_NAME>. Create an Internal Project called <TEAM_NAME>.

9. Push our code to Gitlab. 
 
    ```bash
    export GITLAB_USER=<USER_NAME>
    ```
   
    ```bash
    export GITLAB_PASSWORD=<PASSWORD>
    ```

    ```bash
    gitlab_pat
    ```

     ```bash
     cd /projects/rainforest
     git remote set-url origin https://${GITLAB_USER}:${GITLAB_PAT}@${GIT_SERVER}/${TEAM_NAME}/rainforest.git
     ```

     Use the `GITLAB_PAT` from above when you are prompted for the password (this will be cached)

     ```bash
     cd /projects/rainforest
     git add .
     git commit -am "🐙 ADD - rainforest 🐙"
     git push -u origin --all
     ```


10. Create GitLab - ArgoCD webhook

   ```bash
   echo https://$(oc -n <TEAM_NAME>-ci-cd get route argocd-server --template='{{ .spec.host}}'/api/webhook)
   ```
