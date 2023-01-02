## 🔐 Secrets
## Vault and App Secrets Setup

1. Create sa

    ```bash
    oc -n ${TEAM_NAME}-ci-cd create sa ${SERVICE_ACCOUNT}
    ```

2. OCP 4.11+ no default token with sa

    ```yaml
    cat <<EOF | oc -n ${TEAM_NAME}-ci-cd apply -f -
    apiVersion: v1
    kind: Secret
    metadata:
      name: vault-token
      annotations:
        kubernetes.io/service-account.name: "vault" 
    type: kubernetes.io/service-account-token 
    EOF
    ```

3. Link secret vault-token to sa

    ```bash
    oc -n ${TEAM_NAME}-ci-cd secrets link vault vault-token
    ```

4. Read secrets and bind to vault sa

    ```bash
    oc adm policy add-cluster-role-to-user edit -z vault -n ${TEAM_NAME}-ci-cd
    oc adm policy add-cluster-role-to-user system:auth-delegator -z vault -n ${TEAM_NAME}-ci-cd
    ```

5. Gitlab - FIXME, not GitOps yet

    ```yaml
    cat <<EOF | oc -n ${TEAM_NAME}-ci-cd apply -f -
    apiVersion: v1
    data:
      password: "$(echo -n ${GITLAB_PAT} | base64)"
      username: "$(echo -n ${GITLAB_USER} | base64)"
    kind: Secret
    metadata:
      annotations:
        tekton.dev/git-0: https://${GIT_SERVER}
      name: git-auth
    type: kubernetes.io/basic-auth
    EOF
    ```

6. Hashi Vault unseal

   ```bash
   oc -n rainforest exec -ti platform-base-vault-0 -- vault operator init -key-threshold=1 -key-shares=1
   ```

   You should see the following sort of debug, copy these somewhere safe!

   ```bash
   Unseal Key 1: <unseal key>
   Initial Root Token: <root token>
   ```

   ```bash
   export UNSEAL_KEY=<unseal key>
   export ROOT_TOKEN=<root token>
   ```
   
   ```bash
   oc -n rainforest exec -ti platform-base-vault-0 -- vault operator unseal $UNSEAL_KEY
   ```

7. Vault setup ldap auth

   ```bash
   export LDAP_BIND_PASSWORD=<ldap_admin password>
   ```
   
   ```bash
   export VAULT_ROUTE=vault.apps.${CLUSTER_DOMAIN}
   export VAULT_ADDR=https://${VAULT_ROUTE}
   export VAULT_SKIP_VERIFY=true
   ```

   ```bash
   vault login token=${ROOT_TOKEN}
   ```
   
   ```bash
   vault auth enable ldap
   ```

   ```bash
   vault write auth/ldap/config \
     url="ldap://ipa.ipa.svc.cluster.local:389" \
     binddn="uid=ldap_admin,cn=users,cn=accounts,dc=redhatlabs,dc=dev" \
     bindpass="$LDAP_BIND_PASSWORD" \
     userdn="cn=users,cn=accounts,dc=redhatlabs,dc=dev" \
     userattr="uid" \
     groupdn="cn=student,cn=groups,cn=accounts,dc=redhatlabs,dc=dev" \
     groupattr="cn"
   ```

8. Vault setup app policy for <TEAM_NAME>-ci-cd
 
   ```bash
   export APP_NAME=vault
   export TEAM_GROUP=student
   export PROJECT_NAME=<TEAM_NAME>-ci-cd
   ```

   ```bash
   vault policy write $TEAM_GROUP-$PROJECT_NAME -<<EOF
   path "kv/data/{{identity.groups.names.$TEAM_GROUP.name}}/$PROJECT_NAME/*" {
       capabilities = [ "create", "update", "read", "delete", "list" ]
   }
   path "auth/$CLUSTER_DOMAIN-$PROJECT_NAME/*" {
       capabilities = [ "create", "update", "read", "delete", "list" ]
   }
   EOF
   ```

9. Test ldap login and add entity id mapping for <USER_NAME>

   ```bash
   vault login -method=ldap username=<USER_NAME>
   ```
   
   ```bash
   vault login token=${ROOT_TOKEN}
   vault list identity/entity/id
   ```
   
   ```bash
   export ENTITY_ID=<id from identity/entity/id>
   ```

   ```bash
   vault write identity/group name="$TEAM_GROUP" \
   policies="$TEAM_GROUP-$PROJECT_NAME" \
   member_entity_ids=$ENTITY_ID \
   metadata=team="$TEAM_GROUP"
   ```

10. Enable Kubernetes auth in vault

   ```bash
   vault auth enable -path=$CLUSTER_DOMAIN-${PROJECT_NAME} kubernetes
   ```
   
   ```bash
   export MOUNT_ACCESSOR=$(vault auth list -format=json | jq -r ".\"$CLUSTER_DOMAIN-$PROJECT_NAME/\".accessor")
   ```
   
   ```bash
   vault policy write $CLUSTER_DOMAIN-$PROJECT_NAME-kv-read -<< EOF
   path "kv/data/$TEAM_GROUP/{{identity.entity.aliases.$MOUNT_ACCESSOR.metadata.service_account_namespace}}/*" {
   capabilities=["read","list"]
   }
   EOF
   ```

11. Enable kv2 in vault
 
   ```bash
   vault secrets enable -path=kv/ -version=2 kv
   ```

12. Add ArgoCD Service Account token for k8s auth in vault 

   ```bash
   oc login --server=https://api.${CLUSTER_DOMAIN##apps.}:6443 -u <USER_NAME> -p <PASSWORD>
   ```
   
   ```bash
   vault login -method=ldap username=<USER_NAME>
   ```

   ```bash
   vault write auth/$CLUSTER_DOMAIN-$PROJECT_NAME/role/$APP_NAME \
   bound_service_account_names=$APP_NAME \
   bound_service_account_namespaces=$PROJECT_NAME \
   policies=$CLUSTER_DOMAIN-$PROJECT_NAME-kv-read \
   period=120s
   ```

   ```bash
   export SA_TOKEN=$(oc -n ${PROJECT_NAME} get sa/${APP_NAME} -o yaml | grep ${APP_NAME}-token | awk '{print $3}')
   export SA_JWT_TOKEN=$(oc -n ${PROJECT_NAME} get secret $SA_TOKEN -o jsonpath="{.data.token}" | base64 --decode; echo)
   export SA_CA_CRT=$(oc -n ${PROJECT_NAME} get secret $SA_TOKEN -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
   ```

   ```bash
   vault write auth/$CLUSTER_DOMAIN-${PROJECT_NAME}/config \
   token_reviewer_jwt="$SA_JWT_TOKEN" \
   kubernetes_host="$(oc whoami --show-server)" \
   kubernetes_ca_cert="$SA_CA_CRT"
   ```

13. Create Team ArgoCD Vault Plugin Secret

   ```bash
   export AVP_TYPE=vault
   export VAULT_ADDR=https://platform-base-vault.rainforest.svc:8200   # vault url
   export AVP_AUTH_TYPE=k8s                                            # kubernetes auth
   export AVP_K8S_ROLE=vault                                           # vault role/sa
   export VAULT_SKIP_VERIFY=true
   export AVP_MOUNT_PATH=auth/$CLUSTER_DOMAIN-$PROJECT_NAME
   ```

   ```yaml
   cat <<EOF | oc apply -n ${PROJECT_NAME} -f-
   apiVersion: v1
   stringData:
     VAULT_ADDR: "${VAULT_ADDR}"
     VAULT_SKIP_VERIFY: "${VAULT_SKIP_VERIFY}"
     AVP_AUTH_TYPE: "${AVP_AUTH_TYPE}"
     AVP_K8S_ROLE: "${AVP_K8S_ROLE}"
     AVP_TYPE: "${AVP_TYPE}"
     AVP_K8S_MOUNT_PATH: "${AVP_MOUNT_PATH}"
   kind: Secret
   metadata:
     name: team-avp-credentials
     namespace: ${PROJECT_NAME}
   type: Opaque
   EOF
   ```

14. Create an example kv2 secret

   ```bash
   export VAULT_HELM_RELEASE=vault
   export VAULT_ROUTE=${VAULT_HELM_RELEASE}.$CLUSTER_DOMAIN
   export VAULT_ADDR=https://${VAULT_ROUTE}
   export VAULT_SKIP_VERIFY=true
   ```
   
   ```bash
   export APP_NAME=secret-test
   export TEAM_GROUP=student
   export PROJECT_NAME=rainforest-ci-cd
   ```
   
   ```bash
   vault login -method=ldap username=<USER_NAME>
   ```
   
   ```bash
   vault kv put kv/$TEAM_GROUP/$PROJECT_NAME/$APP_NAME \
   app=$APP_NAME \
   username=foo \
   password=bar 
   ```

15. Unencrypt rainforest vault-secrets file.

   ```bash
   ansible-vault decrypt /projects/rainforest/gitops/secrets/vault-rainforest
   ```

16. In your IDE, Globally replace across ALL files

   ```bash
   foo-sno.sandbox1965.opentlc.com ->  echo ${CLUSTER_DOMAIN##apps.}
   ```

   ```bash
   github.com/eformat ->  <GIT_SERVER>/<TEAM_NAME>
   ```

   💥 DO NOT CHECK IN the gitops/secrets/vault-rainforest file just yet !! 💥

17. Create all the application secrets in vault.

   ```bash
   ./gitops/secrets/vault-rainforest
   ```

18. Encrypt rainforest vault-secrets file and check it in.

   ```bash
   ansible-vault encrypt /projects/rainforest/gitops/secrets/vault-rainforest
   ```

   ```bash#test
   cd /projects/rainforest
   git add .
   git commit -am "🐙 ADD - vault secrets file 🐙"
   git push -u origin --all
   ```
