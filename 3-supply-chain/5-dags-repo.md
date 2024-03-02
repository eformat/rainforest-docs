## 🐑️ Initialize DAG Git Repositories
## Initialize a DAG repository for Airflow

We need to set up our Data Science JupyterHub environment so we can run the Airflow demo's. Let's do that now.

1. Login to Gitlab and under your group <TEAM_NAME> create a dag project repo called **daintree-dev-dags**

   ![1-dags-repo](./images/dags-repo.png)

2. In DevSpaces initialize the dags repo.

    ```bash
    cd /projects
    git clone https://<GIT_SERVER>/<TEAM_NAME>/daintree-dev-dags.git
    cd daintree-dev-dags
    echo "# rainforest/daintree-dev-dags" > README.md
    git add README.md
    git commit -m "🦩 initial commit 🦩"
    git branch -M main
    git push -u origin main
    ```

3. Copy airflowignore file to root of dags directory
 
   ```bash
   cd /projects/daintree-dev-dags
   echo "# ignore the symlinked directory" > .airflowignore
   echo "daintree-dev-dags.git" >> .airflowignore   
   podname=$(oc -n daintree-dev get pod -lapp.kubernetes.io/name=airflow-scheduler -o name)
   oc -n daintree-dev -c airflow-scheduler cp .airflowignore ${podname##pod/}:/opt/app-root/dags
   ``` 
