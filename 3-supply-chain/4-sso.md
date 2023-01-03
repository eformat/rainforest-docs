## 👔 Single Sign On
## Configure SSO

1. We are using a gitops pattern to manage single sign on for our apps.

   Update the gitops/iam/daintree-dev/values.yaml file with UID and your name.

   ```bash
   oc get user <USER_NAME> -o jsonpath='{.metadata.uid}'
   ```
   
   ```yaml
   users:
     user1:
       username: "user1"
       firstName: "Joe"      <------- UPDATE NAME
       lastName: "Blogs"     <------- UPDATE NAME
       email: "user1@redhatlabs.dev"
       federatedIdentities:
         create: True
         userId: "<uid>"     <------- UPDATE UID
       clientRole: adminRole
       clusterRole: edit
   ```

    ```bash
    cd /projects/rainforest
    git add gitops/iam/daintree-dev/values.yaml
    git commit -am "🐙 UPDATE - iam values file 🐙"
    git push
    ```

2. Login to SSO as admin

   ```bash
   echo -e https://$(oc get route keycloak --template='{{ .spec.host }}' -n ${TEAM_NAME}-ci-cd)
   ```

   ```bash
   echo -e $(oc get secret credential-keycloak -n ${TEAM_NAME}-ci-cd -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d)
   ```

3. Delete Users -> user1, let argocd resync the user. Check Role mappings.
 
