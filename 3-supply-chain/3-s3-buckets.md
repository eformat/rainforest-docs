## 🪣 S3 buckets
## Configure s3 buckets

1. Configure s3 buckets for our apps. We keep this separate in case we start using other s3 instead of minio.

   ```bash
   export PROJECT_NAME=daintree-dev
   ```
   
   ```bash
   oc -n <TEAM_NAME>-ci-cd port-forward svc/minio 9000:9000
   ```

   ```bash
   AWS_ACCESS_KEY_ID=$(oc get secret s3-auth -n ${PROJECT_NAME} -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
   AWS_SECRET_ACCESS_KEY=$(oc get secret s3-auth -n ${PROJECT_NAME} -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
   ```
   
   ```bash
   mc alias set dev http://localhost:9000 ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} 
   ```
   
   ```bash
   mc mb dev/mlflow-${PROJECT_NAME}
   mc mb dev/airflow-${PROJECT_NAME}
   mc mb dev/spark-history-${PROJECT_NAME}/spark-data
   mc mb dev/hive-${PROJECT_NAME}
   mc mb dev/data
   ```
