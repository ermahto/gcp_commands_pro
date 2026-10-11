"# gcp_commands_pro"

# Set environment variables
export REGION=""
export ZONE=""
export PROJECT_ID=””

Task 1. Create multiple web server instances
# Create three VM instances
gcloud compute instances create web1 \
--zone=$ZONE \
--machine-type=e2-small \
--tags=network-lb-tag \
--image-family=debian-12 \
--image-project=debian-cloud \
--metadata startup-script='#!/bin/bash
apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html'
gcloud compute instances create web2 \
--zone=$ZONE \
--machine-type=e2-small \
--tags=network-lb-tag \
--image-family=debian-12 \
--image-project=debian-cloud \
--metadata startup-script='#!/bin/bash
apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: web2</h3>" | tee /var/www/html/index.html'
gcloud compute instances create web3 \
--zone=$ZONE \
--machine-type=e2-small \
--tags=network-lb-tag \
--image-family=debian-12 \
--image-project=debian-cloud \
--metadata startup-script='#!/bin/bash

apt-get update
apt-get install apache2 -y
service apache2 restart
echo "<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'
# Create a firewall rule to allow HTTP traffic
gcloud compute firewall-rules create www-firewall-network-lb \
--allow=tcp:80 \
--target-tags=network-lb-tag
Task 2. Configure the load balancing service
# Create a static external IP
gcloud compute addresses create network-lb-ip-1 --region=$REGION
# Create an HTTP health check
gcloud compute http-health-checks create basic-check
# Create a target pool
gcloud compute target-pools create www-pool \
--region=$REGION \
--http-health-check=basic-check
# Add instances to the target pool
gcloud compute target-pools add-instances www-pool \
--instances=web1,web2,web3 \
--instances-zone=$ZONE \
--region=$REGION
# Create a forwarding rule
gcloud compute forwarding-rules create www-rule \
--region=$REGION \
--ports=80 \
--address=network-lb-ip-1 \
--target-pool=www-pool

Task 3. Create an HTTP load balancer

# Create an instance template
gcloud compute instance-templates create lb-backend-template \
--region=$REGION \
--network=default \
--subnet=default \
--tags=allow-health-check \
--machine-type=e2-medium \
--image-family=debian-12 \
--image-project=debian-cloud \
--metadata startup-script='#!/bin/bash
apt-get update
apt-get install apache2 -y
a2ensite default-ssl
a2enmod ssl
vm_hostname="$(curl -H "Metadata-Flavor:Google" \
<http://169.254.169.254/computeMetadata/v1/instance/name>)"
echo "Page served from: $vm_hostname" | tee /var/www/html/index.html
systemctl restart apache2'
# Create a managed instance group
gcloud compute instance-groups managed create lb-backend-group \
--template=lb-backend-template \
--size=2 \
--zone=$ZONE
# Create a firewall rule to allow health checks
gcloud compute firewall-rules create fw-allow-health-check \
--network=default \
--action=allow \
--direction=ingress \
--source-ranges=130.211.0.0/22,35.191.0.0/16 \
--target-tags=allow-health-check \
--rules=tcp:80
# Create a health check
gcloud compute health-checks create http http-basic-check --port=80
# Set named ports for the instance group
gcloud compute instance-groups managed set-named-ports lb-backend-group \
--named-ports http:80 \
--zone=$ZONE

# Create a backend service
gcloud compute backend-services create web-backend-service \
--protocol=HTTP \
--port-name=http \
--health-checks=http-basic-check \
--global
# Add backend to the backend service
gcloud compute backend-services add-backend web-backend-service \
--instance-group=lb-backend-group \
--instance-group-zone=$ZONE \
--global
# Create a URL map
gcloud compute url-maps create web-map-http \
--default-service web-backend-service
# Create a target HTTP proxy
gcloud compute target-http-proxies create http-lb-proxy \
--url-map web-map-http
# Create an external IP address
gcloud compute addresses create lb-ipv4-1 \
--ip-version=IPV4 \
--global

https://github.com/CloudRik/Challenge-Lab/tree/73e26d74bfd62df1c988dcd1fe6d04fc9ef63e29/Build-Infrastructure-with-Terraform-on-Google-Cloud-Challenge-Lab
# Create a forwarding rule
gcloud compute forwarding-rules create http-content-rule \
--address=lb-ipv4-1 \
--global \\
--target-http-proxy=http-lb-proxy \
--ports=80



curl -sSL https://raw.githubusercontent.com/anu11234/Develop-Serverless-Applications-on-Cloud-Run-Challenge-Lab/main/solution.sh | bash


curl -L -O ttps://raw.githubusercontent.com/pspcps/Arcade/refs/heads/main/GSP318.sh

sudo chmod +x GSP318.sh

./GSP318.sh



ooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo

--------------------------------
adk eval_set create \
  bigquery_agent \
  ledger
  
IF ABOVE CODE WILL NOT WORK, TRY THE BELOW CODE.....THE ABOVE CODE CAN ALSO BE WRITTEN LIKE THIS....

adk eval_set create bigquery_agent ledger

--------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
adk eval_set add_eval_case \
    bigquery_agent \
    ledger \
    --scenarios_file bigquery_agent/evaluations/scenarios.json \
    --session_input_file bigquery_agent/evaluations/session_input.json
    
IF ABOVE CODE WILL NOT WORK, TRY THE BELOW CODE.....THE ABOVE CODE CAN ALSO BE WRITTEN LIKE THIS....

adk eval_set add_eval_case bigquery_agent ledger --scenarios_file bigquery_agent/evaluations/scenarios.json --session_input_file bigquery_agent/evaluations/session_input.json


---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  agent.py  ( BELOW CODE UPDATED )
---------------------------------

import os
import datetime
from zoneinfo import ZoneInfo
from dotenv import load_dotenv
from google.cloud import bigquery
from google.adk.agents import Agent
from google.adk.tools.bigquery import BigQueryToolset, BigQueryCredentialsConfig
from google.adk.tools.bigquery.config import BigQueryToolConfig, WriteMode
from google.adk.models import Gemini
from google.genai import types
import google.auth
from google.auth.transport.requests import Request

import google.cloud.logging
load_dotenv()
cloud_logging_client = google.cloud.logging.Client(project=os.getenv('GOOGLE_CLOUD_PROJECT'))
cloud_logging_client.setup_logging()
from .callback_logging import log_query_to_model, log_model_response
RETRY_OPTIONS = types.HttpRetryOptions(initial_delay=1, attempts=6)

# Uses externally-managed Application Default Credentials (ADC) by default.
# This decouples authentication from the agent / tool lifecycle.
# https://cloud.google.com/docs/authentication/provide-credentials-adc

application_default_credentials, _ = google.auth.default()
if not application_default_credentials.valid:
    application_default_credentials.refresh(Request())
credentials_config = BigQueryCredentialsConfig(
    credentials=application_default_credentials)

# Define a tool configuration to block any write operations
tool_config = BigQueryToolConfig(write_mode=WriteMode.ALLOWED)

# Instantiate a BigQuery toolset
bigquery_toolset = BigQueryToolset(
    credentials_config=credentials_config,
    bigquery_tool_config=tool_config
)

def _serialize_datetime_in_dict(data):
    """Recursively converts datetime objects in a dictionary or list to ISO format strings."""
    if isinstance(data, dict):
        return {k: _serialize_datetime_in_dict(v) for k, v in data.items()}
    elif isinstance(data, list):
        return [_serialize_datetime_in_dict(elem) for elem in data]
    elif isinstance(data, (datetime.date, datetime.datetime)):
        return data.isoformat()
    return data

def read_table_all(table_name: str):
    """Reads all rows from the specified table.
    Args:
        table_name: The name of the table to read from.
    Returns:
        list of dict: A list of all row dictionaries in the table.
    """
    client = bigquery.Client(project=os.getenv('GOOGLE_CLOUD_PROJECT'))
    query = f"""
        SELECT * FROM `pool_data.{table_name}`
    """
    query_job = client.query(query)
    results = query_job.result()

    # Always returns a list (will be [] if the table is empty), which is safe to iterate
    return [_serialize_datetime_in_dict(dict(row)) for row in results]


def read_table(table_name: str, email: str):
    """Reads a single row matching the customer's email from the specified table.
    Args:
        table_name: The name of the table to read from.
        email: The customer's email address to filter by.
    Returns:
        dict: The row data if found, otherwise None.
    """
    client = bigquery.Client(project=os.getenv('GOOGLE_CLOUD_PROJECT'))
    query = f"""
        SELECT * FROM `pool_data.{table_name}`
        WHERE customer_email = @email
        LIMIT 1
    """
    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter("email", "STRING", email)
        ]
    )
    query_job = client.query(query, job_config=job_config)
    results = query_job.result()

    # Safely get the first row, or None if no rows exist
    row = next(results, None)
    if row:
        return _serialize_datetime_in_dict(dict(row))
    return None


def write_to_table(table_name: str, row: dict):
    """Writes a row to the specified table, automatically formatting date fields.
    Args:
        table_name: The name of the table to write to.
        row: A dictionary representing the row data to insert.
    Returns:
        str: "Success" on success, empty string if row is empty.
    """
    if not row:
        return ""
    client = bigquery.Client(project=os.getenv('GOOGLE_CLOUD_PROJECT'))

    # We use a standard DML parameterized INSERT statement to avoid BigQuery's 
    # streaming buffer limitation. This ensures that any rows written can be
    # deleted or modified immediately afterwards without causing BadRequest errors.

    columns = ", ".join(row.keys())
    param_placeholders = ", ".join(f"@{k}" for k in row.keys())
    query = f"""
        INSERT INTO `pool_data.{table_name}` ({columns})
        VALUES ({param_placeholders})
    """

    query_parameters = []
    for k, v in row.items():
        if v is None:
            param_type = "STRING"
        elif isinstance(v, bool):
            param_type = "BOOL"
        elif isinstance(v, int):
            param_type = "INT64"
        elif isinstance(v, float):
            param_type = "FLOAT64"
        elif isinstance(v, (datetime.date, datetime.datetime)):
            param_type = "DATE" if isinstance(v, datetime.date) else "DATETIME"
            v = v.isoformat()
        else:
            param_type = "STRING"
        query_parameters.append(
            bigquery.ScalarQueryParameter(k, param_type, v)
        )

    job_config = bigquery.QueryJobConfig(query_parameters=query_parameters)
    query_job = client.query(query, job_config=job_config)
    query_job.result()
    return "Success"

def delete_from_table(table_name: str, email: str):
    """Deletes rows from the specified table that match the customer's email.
    Args:
        table_name: The name of the table to delete from.
        email: The customer's email to match for deletion.
    Returns:
        str: "Success" on success.
    """
    client = bigquery.Client(project=os.getenv('GOOGLE_CLOUD_PROJECT'))
    query = f"""
        DELETE FROM `pool_data.{table_name}`
        WHERE customer_email = @email
    """
    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter("email", "STRING", email)
        ]
    )

    delete_job = client.query(query, job_config=job_config)
    delete_job.result()
    return "Success"

# MODIFICATION ADDED: Implemented atomic transaction handling logic

def perform_consistent_transaction(from_table: str, to_table: str, customer_email: str):
    """
    Search for a record from from_table.
    If that record exists, write it to the to_table and delete it from the original table.
    Args:
        from_table: The table to read the record from
        to_table: The table to write the record to
        customer_email: The email of the customer to perform the transaction for
    Returns:
        Whether it could perform the transaction
    """

    # Look up the row from the source table
    row = read_table(from_table, customer_email)
    if row:
        # Step 1: Add row to destination table
        write_status = write_to_table(to_table, row)
        if write_status == "Success":
            # Step 2: Delete row from the original source table
            delete_status = delete_from_table(from_table, customer_email)
            if delete_status == "Success":
                return True
    return False

# MODIFICATION ADDED: Added explicit state machine check dictating allowed status changes

def check_transaction(from_table: str, to_table: str) -> bool:
    """
    Checks if a transition between two tables is valid.
    Args:
        from_table: The source table.
        to_table: The destination table.
    Returns:
        bool: True if the transition is valid, False otherwise.
    """

    valid_transitions = {
        "pool_estimates": ["accepted_with_deposit", "denied_estimates"],
        "accepted_with_deposit": ["scheduled_installations"],
        "scheduled_installations": ["completed_pools"],
        "completed_pools": ["paid_and_closed"]
    }

    if from_table in valid_transitions and to_table in valid_transitions[from_table]:
        return True
    return False

# Agent Definition
root_agent = Agent(
    model=Gemini(model=os.getenv("MODEL"), retry_options=RETRY_OPTIONS),
    name="bigquery_agent",
    description=(
        "Agent to answer questions about BigQuery data and models and execute"
        " SQL queries."
    ),

    # MODIFICATION ADDED: Overhauled system instructions with strict structural constraints
    instruction=f"""
        You are a data science agent with access to several BigQuery tools.
        Make use of those tools to answer the user's questions.
        When querying BigQuery, always use the project
        {os.getenv('GOOGLE_CLOUD_PROJECT')} and the dataset named `pool_data`.
        Do not create new tables.
        Before deleting a record to move it, confirm it exists and can be moved.
        Before adding a record, confirm it is not already present.
        The tables you have available are:
          - pool_estimates: Contains all pool estimates
          - accepted_with_deposit: Contains all pool estimates that have been accepted and have a deposit
          - denied_estimates: Estimates that have been denied by the customer and will not proceed.
          - scheduled_installations: Contains all pool installations that have been scheduled
          - completed_pools: Contains all pool installations that have been completed
          - paid_and_closed: Contains all pool installations that have been paid and closed
        Use read_table_all to read the data from the tables.
        Use check_transaction to check if a transaction is valid before performing any transactions. If not valid, tell the user so.
        Use perform_consistent_transaction when you need to read a table, insert a row into another table and delete the original row.
    """,
    before_model_callback=log_query_to_model,
    after_model_callback=log_model_response,

    # MODIFICATION ADDED: Limited the tool registry strictly to explicitly defined standalone methods
    tools=[read_table, read_table_all, check_transaction, perform_consistent_transaction],
)
    
