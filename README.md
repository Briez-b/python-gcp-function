

# Task description:

You are tasked with creating a serverless function that performs the following tasks. Additionally, all cloud infrastructure must be provisioned using programming code. Use Python, Go or NodeJS.

#### 1. Set Up Cloud Infrastructure via Code

**- Create a Cloud-Managed NoSQL Database:**
- Use programming libraries or SDKs to set up a cloud-managed NoSQL database (e.g., DynamoDB for AWS, Firestore for Google Cloud, Cosmos DB for Azure) programmatically.
- Ensure the database is properly configured to allow access from the serverless function.
- Configure necessary database credentials and security settings.

**- Create a Serverless Function:**
- Set up a serverless function (e.g., AWS Lambda, Google Cloud Function, or Azure Function) using programming libraries or SDKs.
- Define the function’s runtime environment, code, and role with appropriate permissions.
- Upload and configure the function code, including environment variables for database connection and API endpoints.

**- Set Up Monitoring and Logging:**
- Implement monitoring tools programmatically to track function execution and performance.
- Configure logging to capture detailed information about the function's operations

#### 2. Fetch Data from an External API
- Make an API Call: Fetch data from an external data source using HTTP requests.
- Error Handling: Implement retry logic for the API call with a maximum number of retries.
- Rate Limiting: Handle rate limits imposed by the external API.

#### 3. Process the Data=

- Transform Data: Perform necessary transformations or processing on the fetched data.
- Data Validation: Include validation steps to ensure the data is correct and clean.
- Logging: Implement detailed logging for each step to monitor and debug the data processing.
#### 4.  Store Data in a Cloud NoSQL Database
- Connect to the Database: Use appropriate libraries or SDKs to connect to the cloud-managed NoSQL database.
- Schema Creation: Define the necessary schema or structure for storing data in the NoSQL database.
- Data Insertion: Insert the processed data into the database, ensuring data integrity.

#### 5. Perform Health Checks and Monitoring
- Database Health Checks: Implement health checks programmatically to verify that the NoSQL database is operational.
- Function Monitoring: Set up alerts and monitoring programmatically to notify if issues arise during function execution.
- Execution Logging: Log the execution details and results for auditing and troubleshooting purposes.






# Implementation:

1) I implemented `create_function.py` script that create all needed infrastructure: nosql database, cloud function, monitoring rules:
   
``` python
import zipfile
import os
import subprocess
from google.cloud import storage, functions_v2, firestore_admin_v1, monitoring_v3
from google.cloud.firestore_admin_v1.types import database
from google.cloud.firestore_admin_v1.types import firestore_admin
from google.api_core.exceptions import AlreadyExists

# Configuration
project_id = "gd-gcp-gridu-devops-t1-t2"
location = "us-central1"
bucket_name = "bryshten-bucket"
project_path = f"projects/{project_id}"

function_name = "bryshten-test3"
db_id = "bryshten-test3"

service_account = f"{project_id}@appspot.gserviceaccount.com"


##################################################################
# 1. Create function
##################################################################

# Zip the existing files
with zipfile.ZipFile("function_code.zip", "w") as z:
    z.write("main.py")
    z.write("requirements.txt")

# Upload the ZIP to Google Cloud Storage
storage_client = storage.Client(project=project_id)
bucket = storage_client.bucket(bucket_name)
blob = bucket.blob("function_code.zip")
blob.upload_from_filename("function_code.zip")

print("Zip uploaded. Instructing GCP to build the function...")

# Tell Cloud Functions to deploy using that ZIP
client = functions_v2.FunctionServiceClient()
parent = f"projects/{project_id}/locations/{location}"

function_config = {
    "name": f"{parent}/functions/{function_name}",
    "build_config": {
        "runtime": "python310",
        "entry_point": "main_pipeline",
        "source": {
            "storage_source": {
                "bucket": bucket_name,
                "object": "function_code.zip"
            }
        }
    },
    "service_config": {
        "service_account_email": service_account, 
        "environment_variables": {
            "EXTERNAL_API_URL": "https://jsonplaceholder.typicode.com/posts/1",
            "FIRESTORE_DB_NAME": db_id
        }
    }
    
}

request = functions_v2.CreateFunctionRequest(
    parent=parent,
    function=function_config,
    function_id=function_name
)

operation = client.create_function(request=request)

print("Deployment started (v2)...")
response = operation.result()

print(f"v2 Function is live! URL: {response.service_config.uri}")

# Adding allUsers policy to make URL public
try:
    subprocess.run([
        "gcloud", "functions", "add-iam-policy-binding", function_name,
        "--region", location,
        "--member", "allUsers",
        "--role", "roles/run.invoker"
    ], check=True)
    print("Success: Function is now publicly accessible!")
except subprocess.CalledProcessError as e:
    print(f"Failed to set permissions automatically. Error: {e}")




####################################################################################
# 2. Setup Firestore
####################################################################################

# Initialize Firestore Admin Client
admin_client = firestore_admin_v1.FirestoreAdminClient()

# Define the database configuration
db_config = database.Database(
    location_id=location,
    type_=database.Database.DatabaseType.FIRESTORE_NATIVE,
)

create_db_request = firestore_admin.CreateDatabaseRequest(
    parent=project_path,
    database=db_config,
    database_id=db_id
)

print(f"Checking/Creating Firestore database '{db_id}' in {location}...")

try:
    # Attempt to create the database
    db_operation = admin_client.create_database(request=create_db_request)
    print("Waiting for database creation to complete (this can take a few minutes)...")
    db_operation.result()
    print("Firestore database created successfully.")
except AlreadyExists:
    print("Firestore database already exists. Proceeding with existing instance.")
except Exception as e:
    print(f"An error occurred while setting up Firestore: {e}")




######################################################################################
# 3. Setup Monitoring and Alerts
#######################################################################################
print("Setting up Cloud Monitoring alerts...")

try:
    monitoring_client = monitoring_v3.AlertPolicyServiceClient()
    project_path = f"projects/{project_id}"

    # Define the rule: Trigger if there is an error in execution
    # This filter looks specifically for your function where the status is NOT 'ok'
    metric_filter = (
        f'resource.type = "cloud_function" AND '
        f'resource.labels.function_name = "{function_name}" AND '
        f'metric.type = "cloudfunctions.googleapis.com/function/execution_count" AND '
        f'metric.labels.status != "ok"'
    )

    condition = monitoring_v3.AlertPolicy.Condition(
        display_name="Function Execution Failure Condition",
        condition_threshold=monitoring_v3.AlertPolicy.Condition.MetricThreshold(
            filter=metric_filter,
            comparison=monitoring_v3.ComparisonType.COMPARISON_GT,
            threshold_value=0.0,
            duration={"seconds": 60}, 
            aggregations=[
                monitoring_v3.Aggregation(
                    alignment_period={"seconds": 60},
                    per_series_aligner=monitoring_v3.Aggregation.Aligner.ALIGN_COUNT,
                )
            ],
        ),
    )

    alert_policy = monitoring_v3.AlertPolicy(
        display_name=f"Execution Alert: {function_name} Failure",
        conditions=[condition],
        combiner=monitoring_v3.AlertPolicy.ConditionCombinerType.OR,
        enabled=True
    )

    # Check if a policy with this name already exists to avoid duplicates
    policies = monitoring_client.list_alert_policies(name=project_path)
    exists = any(p.display_name == alert_policy.display_name for p in policies)

    if not exists:
        created_policy = monitoring_client.create_alert_policy(
            name=project_path, alert_policy=alert_policy
        )
        print(f"Monitoring Alert Policy successfully created: {created_policy.name}")
    else:
        print("Monitoring Alert Policy already exists. Skipping creation.")

except Exception as e:
    print(f"Note: Could not set up monitoring policy: {e}")
    print("Check if the Monitoring API is enabled in your project.")
```


And this is the `main.py` I use for cloud function

``` python
import os
import logging
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import functions_framework
from google.cloud import firestore

# Configure standard Python logging (this automatically flows into GCP Cloud Logging)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize firestore
custom_db_name = os.environ.get("FIRESTORE_DB_NAME")

if not custom_db_name:
    raise ValueError("CRITICAL ERROR: FIRESTORE_DB_NAME environment variable is missing!")

db = firestore.Client(database=custom_db_name)



############################################################
# Helper functions
############################################################

# Fetch Data, Error Handling, Rate Limiting
def fetch_data_with_retry(api_url):
    logger.info(f"Attempting to fetch data from {api_url}")
    
    # Setup retry strategy (e.g., retry 3 times, backoff for rate limits)
    retry_strategy = Retry(
        total=3,
        status_forcelist=[429, 500, 502, 503, 504],
        backoff_factor=1 
    )
    adapter = HTTPAdapter(max_retries=retry_strategy)
    http = requests.Session()
    http.mount("https://", adapter)
    http.mount("http://", adapter)

    response = http.get(api_url, timeout=10)
    response.raise_for_status() # Raises an error if the fetch failed
    return response.json()

# Transform Data, Validation, Logging
def process_data(raw_data):
    logger.info("Validating and transforming data...")
    
    # Example Validation: Check if expected keys exist
    if "userId" not in raw_data or "title" not in raw_data:
        logger.error("Data validation failed: Missing required fields.")
        raise ValueError("Invalid data format received from API.")
        
    # Example Transformation: Add a new field or clean up existing ones
    transformed_data = {
        "user_id": raw_data["userId"],
        "post_title": raw_data["title"].upper(), # Transforming text
        "processed_status": "Success"
    }
    
    logger.info("Data processed successfully.")
    return transformed_data

# Database Health Checks
def perform_health_check():
    try:
        # A simple read operation to verify the DB is responsive
        db.collection("health_checks").document("ping").get()
        return True
    except Exception as e:
        logger.error(f"Database health check failed: {e}")
        return False



##############################################################
# MAIN ENTRY POINT
##############################################################

@functions_framework.http
def main_pipeline(request):
    logger.info("Function execution started.")
    
    # Route for health check
    if request.path == '/health':
        is_healthy = perform_health_check()
        if is_healthy:
            return "Database is healthy and operational.", 200
        return "Database health check failed.", 500

    try:
        # 1. Get the API URL from environment variables
        api_url = os.environ.get("EXTERNAL_API_URL")
        if not api_url:
            raise ValueError("EXTERNAL_API_URL environment variable is missing.")

        # 2. Fetch
        raw_data = fetch_data_with_retry(api_url)
        
        # 3. Process
        clean_data = process_data(raw_data)
        
        # 4. Store in Firestore (Schema creation is automatic in NoSQL!)
        logger.info("Inserting data into Firestore...")
        doc_ref = db.collection("api_records").document() # Auto-generates an ID
        doc_ref.set(clean_data)
        
        logger.info("Function execution completed successfully.")
        return f"Pipeline succeeded! Data saved with ID: {doc_ref.id}", 200

    except Exception as e:
        # Requirement 5: Execution Logging for troubleshooting
        logger.error(f"Pipeline failed: {str(e)}")
        return f"Error processing request: {str(e)}", 500
```

And of course `requirements.txt`
``` 
google-cloud-storage
functions-framework
google-cloud-firestore
requests
```
And we also need to install all needed libraries with `pip install` and configure GCP environment on our machine.
### The execution:

1) Execute `create_function.py`. It will create all needed infrastructure and deploy our serverless function.
2) 
![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%2020260330060502.png)

1) Execute url and see if new entry is in firestore database

![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%20260330060624.png)

Here we can see the database really created and new entry was written to database `bryshten-test3:
![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%20260330060716.png)

 Also check if alerts and function really work:

![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%20260330060948.png)
 
And let's create incident by deleting database `bryshten-test3.py` and executing function again to be sure alert works

![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%20260330061216.png)

![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%20260330061228.png)

![](https://github.com/Briez-b/python-gcp-function/blob/main/Pasted%20image%20260330061629.png)


Everything works. Task completed.
