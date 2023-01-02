## Rainforest Setup

<p class="warn">
    ⛷️ <b>NOTE</b> ⛷️ - You need an OpenShift 4.11+ cluster with cluster-admin privilege.
</p>

### SNO for 100

My favourite development environment has become SNO on SPOT in AWS. With this setup, if you choose your region and zones wisely, i can have pretty much an environment running all day without interruption.

For me that is *Ohio* ! To get a 16 core, 64 GB machine for less than $4 a day i use the following to install the cluster.

```bash
export AWS_PROFILE=rhpds
export AWS_DEFAULT_REGION=us-east-2
export AWS_DEFAULT_ZONES=["us-east-2c"]
export CLUSTER_NAME=foo-sno
export BASE_DOMAIN=sandbox.opentlc.com
export PULL_SECRET=$(cat ~/tmp/pull-secret)
export SSH_KEY=$(cat ~/.ssh/id_rsa.pub)
export INSTANCE_TYPE=m6a.4xlarge
export ROOT_VOLUME_SIZE=200

mkdir -p ~/tmp/sno-foo && cd ~/tmp/sno-foo
curl -Ls https://raw.githubusercontent.com/eformat/sno-for-100/main/sno-for-100.sh | bash -s -- -d
```

Once done, you should have a login to your OpenShift Cluster. There are a couple of infra-ops steps i always do, so login as _kubeadmin_ now to your cluster.

```bash
oc login -u kubeadmin -p <kubeadmin password> https://api.$CLUSTER_NAME.$BASE_DOMAIN:6443
```

### Add HTPassword

My goto IDP is htpassword rather than the _kubeadmin_ user. Don't worry we will configure IPA for users as well.

```bash
htpasswd -bBc /tmp/htpasswd admin <admin password>
```

```bash
oc adm policy add-cluster-role-to-user cluster-admin admin
oc delete secret htpasswdidp-secret -n openshift-config
oc create secret generic htpasswdidp-secret -n openshift-config --from-file=/tmp/htpasswd
```

```bash
cat << EOF > /tmp/htpasswd.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswdidp-secret
EOF
oc apply -f /tmp/htpasswd.yaml -n openshift-config
```

```bash
watch oc get co/authentication
```

```bash
oc login -u admin -p <admin password> https://api.$CLUSTER_NAME.$BASE_DOMAIN:6443
```

```bash
oc delete secret kubeadmin -n kube-system
```

### Configure Lets Encrypt

I use acme.sh all the time for generating Lets Encrypt certs, its easy and fast. The only downside is its also manual labour.

Create _caa.your-domain_ entries in your two Route53 hosted zones.   
```bash
CAA
0 issuewild "letsencrypt.org;"
```

```bash
export AWS_ACCESS_KEY_ID=<aws key id>
export AWS_SECRET_ACCESS_KEY=<aws secret access key>
```

```bash
export LE_API=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
export LE_WILDCARD=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}')
```

```bash
cd ~/git
git clone https://github.com/Neilpang/acme.sh.git
```

```bash
~/git/acme.sh/acme.sh --issue --dns dns_aws -d ${LE_API} -d *.${LE_WILDCARD} --dnssleep 100 --force --insecure
```

```bash
oc -n openshift-ingress delete secret router-certs
oc -n openshift-ingress create secret tls router-certs --cert=/home/mike/.acme.sh/${LE_API}/fullchain.cer --key=/home/mike/.acme.sh/${LE_API}/${LE_API}.key
oc -n openshift-ingress-operator patch ingresscontroller default --patch '{"spec": { "defaultCertificate": { "name": "router-certs"}}}' --type=merge
```

```bash
watch oc get co
```

### Configure Extra Disk

To keep things cheap, I use a 200GB gp3 volume and configure the OpenShift LVM Operator to use it as the default dynamic Storage Class for my SNO instance. Choose the size to meet your PVC needs, change profile, region, az and instance_id to suit.

```bash
export AWS_PROFILE=rhpds
region=us-east-2
az=us-east-2c
instance_id=i-078a2c00fbb90e28b
```

```bash
vol=$(aws ec2 create-volume \
--availability-zone ${az} \
--volume-type gp3 \
--size 200 \
--region=${region})
```

```bash
aws ec2 attach-volume \
--volume-id $(echo ${vol} | jq -r '.VolumeId') \
--instance-id $instance_id \
--device /dev/sdf
```

```bash
cat <<EOF | oc apply -f-
kind: Namespace
apiVersion: v1
metadata:
  name: openshift-storage
EOF
```

```bash
cat <<'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operator-storage
  namespace: openshift-storage
spec:
  targetNamespaces:
  - openshift-storage
EOF
```

```bash
cat <<EOF | oc apply -f-
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/odf-lvm-operator.openshift-storage: ''
  name: odf-lvm-operator
  namespace: openshift-storage
spec:
  channel: stable-4.11
  installPlanApproval: Automatic
  name: odf-lvm-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: odf-lvm-operator.v4.11.4
EOF
```

```bash
cat <<EOF | oc apply -f-
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: sno-lvm
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: vgsno
        thinPoolConfig:
          name: thin-pool-1
          overprovisionRatio: 10
          sizePercent: 90
EOF
```

```bash
oc annotate sc/odf-lvm-vgsno storageclass.kubernetes.io/is-default-class=true
oc annotate sc/gp2 storageclass.kubernetes.io/is-default-class-
```

### Deploy IPA

Deploy IPA

```bash
helm repo add redhat-cop https://redhat-cop.github.io/helm-charts
helm repo up
```

```bash
helm upgrade --install ipa redhat-cop/ipa --namespace=ipa \
--create-namespace --set app_domain=apps.<CLUSTER_DOMAIN> \
--set admin_password="<admin password>" \
--set ocp_auth.enabled=true \
--set ocp_auth.bind_password="<ldap_admin password>" \
--set ocp_auth.bind_dn="uid=ldap_admin\\,cn=users\\,cn=accounts\\,dc=redhatlabs\\,dc=dev" --timeout=15m
```

Wait for ipa to install, then create objects:

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "echo <admin password> | /usr/bin/kinit admin"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "echo <ldap_admin password> | \
ipa user-add ldap_admin --first=ldap \
--last=admin --email=ldap_admin@redhatlabs.dev --password"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "ipa group-add-member admins --users=ldap_admin"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "ipa group-add-member editors --users=ldap_admin"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "ipa group-add-member 'trust admins' --users=ldap_admin"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "ipa group-add student --desc 'Student Group'"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "echo <user password> | ipa user-add <USER_NAME> --first=user \
--last=user1 --email=user1@redhatlabs.dev --password"
```

```bash
oc exec -it dc/ipa -n ipa -- \
sh -c "ipa group-add-member student --users=<USER_NAME>"
```

```bash
cd ~/git
git clone https://github.com/eformat/openshift-management
```

```bash
helm upgrade cronjob-ldap-group-sync \
--install charts/cronjob-ldap-group-sync \
--set image="quay.io/openshift/origin-cli" \
--set image_tag="latest" \
--set ldap_bind_dn="uid=ldap_admin\\,cn=users\\,cn=accounts\\,dc=redhatlabs\\,dc=dev" \
--set ldap_bind_password="<ldap_admin password>" \
--set ldap_bind_password_secret="ldap-bind-password" \
--set ldap_group_membership_attributes='["member"]' \
--set ldap_group_name_attributes='["cn"]' \
--set ldap_group_uid_attribute=ipaUniqueID \
--set ldap_groups_filter='(&(objectclass=ipausergroup)(cn=student))' \
--set ldap_groups_search_base="cn=groups\\,cn=accounts\\,dc=redhatlabs\\,dc=dev" \
--set ldap_url='ldap://ipa.ipa.svc.cluster.local:389' \
--set ldap_user_name_attributes='["uid"]' \
--set ldap_user_uid_attribute=dn \
--set ldap_users_search_base="cn=users\\,cn=accounts\\,dc=redhatlabs\\,dc=dev" \
--set ldap_groups_whitelist="" \
--set schedule="*/10 * * * *" \
--set namespace=cluster-ops \
--namespace=cluster-ops \
--create-namespace
```

### Install Rainforest Base using Helm

The base tooling and operators are configured using the helm chart in the `platform/` directory. This configures

- gitlab
- devspaces
- hashicorp vault and certs
- user workload monitoring
- gitops operator
- pipelines operator

1. Helm setup

   ```bash
   helm repo add redhat-cop https://redhat-cop.github.io/helm-charts
   helm repo add hashicorp https://helm.releases.hashicorp.com
   helm dep up
   ```

2. Vault Route

   ```bash
   export VAULT_ROUTE=vault.<CLUSTER_DOMAIN>
   ```

3. Install as cluster-admin

   FIXME - devspaces, chicken and egg, move this to step.1, devspaces separate.

   ```bash
   cd platform/base
   ```

   ```bash
   helm upgrade --install platform-base . \
     --set vault.server.route.host=$VAULT_ROUTE \
     --set vault.server.extraEnvironmentVars.VAULT_TLS_SERVER_NAME=$VAULT_ROUTE \
     --namespace rainforest \
     --create-namespace \
     --timeout=15m \
     --debug
   ```

   It will take a little time to download and install and configure all the bits. Once done, we are ready to start deploying our *AIMLOPS Platform as Product*.

6. FIXME rbac

   ```bash
   oc adm policy add-cluster-role-to-user cluster-admin <USER_NAME>
   ```

🪄🪄 Now, let's continue with even more exciting tools... !🪄🪄
