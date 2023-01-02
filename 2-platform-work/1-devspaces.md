## 🌲 DevSpaces
## DevSpaces Workspace setup

1. Login to your DevSpaces Editor. The link to this will be provided by your instructor.

2. FIXME - We are using a private git repository at the moment. DevSpaces supports this however we need to first create the credential secret. The token will be provided by your instructor.

   ```bash
   oc -n <USER_NAME>-devspaces apply -f - <<EOF
   kind: Secret
   apiVersion: v1
   metadata:
     name: devspaces-git-creds
     annotations:
       controller.devfile.io/mount-path: /tmp/.git-credentials/
     labels:
       controller.devfile.io/git-credential: "true"
       controller.devfile.io/watch-secret: "true"
       controller.devfile.io/mount-to-devworkspace: "true"
   stringData:
     credentials: https://eformat:<GITHUB_TOKEN>@github.com
   EOF
   ```

3. Create your workspace. On DevSpaces Workspaces, "Add Workspace > Import from Git". The token url will be provided by your instructor.

   <p> 
   For OpenShift 4.11+ - Enter this URL to load the dev stack:</br>
   <span style="color:blue;"><a id=crw_dev_filelocation_4.11 href=""></a></span>
   </p>

4. Login to Terminal in DevSpaces. Export env vars
 
    ```bash
    echo export TEAM_NAME="<TEAM_NAME>" | tee -a ~/.bashrc -a ~/.zshrc 
    echo export CLUSTER_DOMAIN="<CLUSTER_DOMAIN>" | tee -a ~/.bashrc -a ~/.zshrc
    echo export GIT_SERVER="<GIT_SERVER>" | tee -a ~/.bashrc -a ~/.zshrc
    ```
   
   ```bash
   source ~/.zshrc
   ```

5. Check if you can connect to OpenShift. Run the command below.

    ```bash
    oc login --server=https://api.${CLUSTER_DOMAIN##apps.}:6443 -u <USER_NAME> -p <PASSWORD>
    ```
