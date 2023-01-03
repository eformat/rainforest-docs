## 𝌭️ Model Pipeline Promotion
## Model pipeline promotion with MLFlow
> mlflow - model lifecycle using a simple regression model, airflow pipeline promotion

1. Once your notebook has opened

2. Open **aimlops-demos/mlflow** and Click on the notebook **mlflow.ipynb**

3. Click **Terminal**, Install python deps

    ```bash
    pip install category_encoders matplotlib boto3 mlflow==1.27.0
    ```

4. Follow the notebook. Open MLFlow UI

   ```bash
   oc login --server=https://api.${CLUSTER_DOMAIN##apps.}:6443 -u <USER_NAME> -p <PASSWORD>
   ```

   ```bash
   echo -e https://$(oc get route mlflow --template='{{ .spec.host }}' -n ${PROJECT_NAME})
   ```

5. Observe Model params
6. Register Model as **TestModel** **Version1**
7. Open **mlflow.pipeline** browse Properties
8. Trigger Airflow Pipeline
9. Watch build steps in Airflow

   ```bash
   echo -e https://$(oc get route airflow --template='{{ .spec.host }}' -n ${PROJECT_NAME})
   ```

10. Check dag in git, logs in Airflow, and S3 logs

    ```bash
    AWS_ACCESS_KEY_ID=$(oc get secret s3-auth -n ${PROJECT_NAME} -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
    AWS_SECRET_ACCESS_KEY=$(oc get secret s3-auth -n ${PROJECT_NAME} -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
    ```

    ```bash
    mc alias set dev http://minio.${TEAM_NAME}-ci-cd.svc.cluster.local:9000 ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} 
    ```

    ```bash
    mc ls dev
    ```

11. Once Seldon model deployed we can check the inference end point

   ```bash
   HOST=$(echo -n https://$(oc -n ${PROJECT_NAME} get $(oc -n ${PROJECT_NAME} get route -l app.kubernetes.io/managed-by=seldon-core -o name) --template='{{ .spec.host }}'))
   ```
   
   ```bash
   curl -H 'Content-Type: application/json' -H 'Accept: application/json' -X POST $HOST/api/v1.0/predictions -d '{"data": {"ndarray": [[1.23]]}}'
   ```
