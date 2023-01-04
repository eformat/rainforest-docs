## 𝌭️ Object Detection
## Create a model to recognize dogs and cats 
> learn howto to recognize fraud detection from a synthetic dataset

1. We can use the instructions here in our environment - https://redhat-scholars.github.io/rhods-od-workshop
2. Use the **Elyra TensorFlow Notebook Image**
3. There are some minor changes required when running the first notebook - **1-explore.ipynb**.

   Download data

   ```bash
   wget https://github.com/eformat/object-detection-rest/releases/download/0.0.1/twodogs.jpg
   ```

   Copy to s3

   ```bash
   mc cp twodogs.jpg dev/data
   ```

   For completeness, use boto3 client in the notebook even though we already have the data!

   ```python
   import os
   #boto3.set_stream_logger(name='botocore')
   
   s3conn = boto3.Session(aws_access_key_id=f"{os.environ['AWS_ACCESS_KEY_ID']}",
   aws_secret_access_key=f"{os.environ['AWS_SECRET_ACCESS_KEY']}")
   client = s3conn.client(
   "s3",
   endpoint_url="http://minio.rainforest-ci-cd.svc.cluster.local:9000",
   verify=False
   )
   
   client.download_file('data', 'twodogs.jpg', 'twodogs.jpg')
   ```

4. When you get to the app part using Flask in **2_predict.ipynb**, install these deps only, edit requirements.txt to contain:

   ```bash
   $ cat requirements.txt 
   Flask
   gunicorn
   ```
