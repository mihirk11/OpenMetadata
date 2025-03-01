---
title: Run BigQuery Connector using Airflow SDK
slug: /connectors/database/bigquery/airflow
---

# Run BigQuery using the Airflow SDK 

In this section, we provide guides and references to use the BigQuery connector.

Configure and schedule BigQuery metadata and profiler workflows from the OpenMetadata UI:
- [Requirements](#requirements)
- [Metadata Ingestion](#metadata-ingestion)
- [Query Usage and Lineage Ingestion](#query-usage-and-lineage-ingestion)
- [Data Profiler](#data-profiler)
- [DBT Integration](#dbt-integration)

## Requirements

<InlineCallout color="violet-70" icon="description" bold="OpenMetadata 0.12 or later" href="/deployment">
To deploy OpenMetadata, check the <a href="/deployment">Deployment</a> guides.
</InlineCallout>

To run the Ingestion via the UI you'll need to use the OpenMetadata Ingestion Container, which comes shipped with
custom Airflow plugins to handle the workflow deployment.

### Python Requirements

To run the BigQuery ingestion, you will need to install:

```bash
pip3 install "openmetadata-ingestion[bigquery]"
```

If you want to run the Usage Connector, you'll also need to install:

```bash
pip3 install "openmetadata-ingestion[bigquery-usage]"
```

<h4>GCP Permissions</h4>

<p> To execute metadata extraction and usage workflow successfully the user or the service account should have enough access to fetch required data. Following table describes the minimum required permissions </p>

| # | GCP Permission | GCP Role | Required For |
| :---------- | :---------- | :---------- | :---------- |
| 1 | bigquery.datasets.get | BigQuery Data Viewer | Metadata Ingestion |
| 2 | bigquery.tables.get | BigQuery Data Viewer | Metadata Ingestion |
| 3 | bigquery.tables.getData | BigQuery Data Viewer | Metadata Ingestion |
| 4 | bigquery.tables.list | BigQuery Data Viewer | Metadata Ingestion |
| 5 | resourcemanager.projects.get | BigQuery Data Viewer | Metadata Ingestion |
| 6 | bigquery.jobs.create | BigQuery Job User | Metadata Ingestion |
| 7 | bigquery.jobs.listAll | BigQuery Job User | Metadata Ingestion |
| 8 | datacatalog.taxonomies.get | BigQuery Policy Admin | Fetch Policy Tags |
| 9 | datacatalog.taxonomies.list | BigQuery Policy Admin | Fetch Policy Tags |
| 10 | bigquery.readsessions.create | BigQuery Admin | Bigquery Usage Workflow |
| 11 | bigquery.readsessions.getData | BigQuery Admin | Bigquery Usage Workflow |


## Metadata Ingestion

All connectors are defined as JSON Schemas.
[Here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/entity/services/connections/database/bigQueryConnection.json)
you can find the structure to create a connection to BigQuery.

In order to create and run a Metadata Ingestion workflow, we will follow
the steps to create a YAML configuration able to connect to the source,
process the Entities if needed, and reach the OpenMetadata server.

The workflow is modeled around the following
[JSON Schema](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/metadataIngestion/workflow.json)

### 1. Define the YAML Config

This is a sample config for BigQuery:

```yaml
source:
  type: bigquery
  serviceName: "<service name>"
  serviceConnection:
    config:
      type: BigQuery
      credentials:
        gcsConfig:
          type: My Type
          projectId: project ID
          privateKeyId: us-east-2
          privateKey: |
            -----BEGIN PRIVATE KEY-----
            Super secret key
            -----END PRIVATE KEY-----
          clientEmail: client@mail.com
          clientId: 1234
          # authUri: https://accounts.google.com/o/oauth2/auth (default)
          # tokenUri: https://oauth2.googleapis.com/token (default)
          # authProviderX509CertUrl: https://www.googleapis.com/oauth2/v1/certs (default)
          clientX509CertUrl: https://cert.url
  sourceConfig:
    config:
      markDeletedTables: true
      includeTables: true
      includeViews: true
      # includeTags: true
      # databaseFilterPattern:
      #   includes:
      #     - database1
      #     - database2
      #   excludes:
      #     - database3
      #     - database4
      # schemaFilterPattern:
      #   includes:
      #     - schema1
      #     - schema2
      #   excludes:
      #     - schema3
      #     - schema4
      # tableFilterPattern:
      #   includes:
      #     - table1
      #     - table2
      #   excludes:
      #     - table3
      #     - table4
      # For DBT, choose one of Cloud, Local, HTTP, S3 or GCS configurations
      # dbtConfigSource:
      # # For cloud
      #   dbtCloudAuthToken: token
      #   dbtCloudAccountId: ID
      # # For Local
      #   dbtCatalogFilePath: path-to-catalog.json
      #   dbtManifestFilePath: path-to-manifest.json
      # # For HTTP
      #   dbtCatalogHttpPath: http://path-to-catalog.json
      #   dbtManifestHttpPath: http://path-to-manifest.json
      # # For S3
      #   dbtSecurityConfig:  # These are modeled after all AWS credentials
      #     awsAccessKeyId: KEY
      #     awsSecretAccessKey: SECRET
      #     awsRegion: us-east-2
      #   dbtPrefixConfig:
      #     dbtBucketName: bucket
      #     dbtObjectPrefix: "dbt/"
      # # For GCS
      #   dbtSecurityConfig:  # These are modeled after all GCS credentials
      #     type: My Type
      #     projectId: project ID
      #     privateKeyId: us-east-2
      #     privateKey: |
      #       -----BEGIN PRIVATE KEY-----
      #       Super secret key
      #       -----END PRIVATE KEY-----
      #     clientEmail: client@mail.com
      #     clientId: 1234
      #     authUri: https://accounts.google.com/o/oauth2/auth (default)
      #     tokenUri: https://oauth2.googleapis.com/token (default)
      #     authProviderX509CertUrl: https://www.googleapis.com/oauth2/v1/certs (default)
      #     clientX509CertUrl: https://cert.url (URI)
      #   dbtPrefixConfig:
      #     dbtBucketName: bucket
      #     dbtObjectPrefix: "dbt/"
sink:
  type: metadata-rest
  config: {}
workflowConfig:
  # loggerLevel: DEBUG  # DEBUG, INFO, WARN or ERROR
  openMetadataServerConfig:
    hostPort: "<OpenMetadata host and port>"
    authProvider: "<OpenMetadata auth provider>"

```

#### Source Configuration - Service Connection

- **hostPort**: This is the BigQuery APIs URL.
- **username**: (Optional) Specify the User to connect to BigQuery. It should have enough privileges to read all the metadata.
- **projectID**: (Optional) The BigQuery Project ID is required only if the credentials path is being used instead of values.
- **credentials**: We support two ways of authenticating to BigQuery inside **gcsConfig**
    1. Passing the raw credential values provided by BigQuery. This requires us to provide the following information, all provided by BigQuery:
        - **type**, e.g., `service_account`
        - **projectId**
        - **privateKey**
        - **privateKeyId**
        - **clientEmail**
        - **clientId**
        - **authUri**, https://accounts.google.com/o/oauth2/auth by defaul
        - **tokenUri**, https://oauth2.googleapis.com/token by default
        - **authProviderX509CertUrl**, https://www.googleapis.com/oauth2/v1/certs by default
        - **clientX509CertUrl**
    2. Passing a local file path that contains the credentials:
        - **gcsCredentialsPath**

If you prefer to pass the credentials file, you can do so as follows:
```yaml
credentials:
  gcsConfig: <path to file>
```

- **Enable Policy Tag Import (Optional)**: Mark as 'True' to enable importing policy tags from BigQuery to OpenMetadata.
- **Tag Category Name (Optional)**: If the Tag import is enabled, the name of the Tag Category will be created at OpenMetadata.
- **Database (Optional)**: The database of the data source is an optional parameter, if you would like to restrict the metadata reading to a single database. If left blank, OpenMetadata ingestion attempts to scan all the databases.
- **Connection Options (Optional)**: Enter the details for any additional connection options that can be sent to BigQuery during the connection. These details must be added as Key-Value pairs.
- **Connection Arguments (Optional)**: Enter the details for any additional connection arguments such as security or protocol configs that can be sent to BigQuery during the connection. These details must be added as Key-Value pairs.
    - In case you are using Single-Sign-On (SSO) for authentication, add the `authenticator` details in the Connection Arguments as a Key-Value pair as follows: `"authenticator" : "sso_login_url"`
    - In case you authenticate with SSO using an external browser popup, then add the `authenticator` details in the Connection Arguments as a Key-Value pair as follows: `"authenticator" : "externalbrowser"`

If you want to use [ADC authentication](https://cloud.google.com/docs/authentication#adc) for BigQuery you can just leave
the GCS credentials empty. This is why they are not marked as required.

```yaml
...
  config:
    type: BigQuery
    credentials:
      gcsConfig: {}
...
```

#### Source Configuration - Source Config

The `sourceConfig` is defined [here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/metadataIngestion/databaseServiceMetadataPipeline.json):

- `markDeletedTables`: To flag tables as soft-deleted if they are not present anymore in the source system.
- `includeTables`: true or false, to ingest table data. Default is true.
- `includeViews`: true or false, to ingest views definitions.
- `databaseFilterPattern`, `schemaFilterPattern`, `tableFilternPattern`: Note that the they support regex as include or exclude. E.g.,

```yaml
tableFilterPattern:
  includes:
    - users
    - type_test
```

#### Sink Configuration

To send the metadata to OpenMetadata, it needs to be specified as `type: metadata-rest`.

#### Workflow Configuration

The main property here is the `openMetadataServerConfig`, where you can define the host and security provider of your OpenMetadata installation.

For a simple, local installation using our docker containers, this looks like:

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: openmetadata
    securityConfig:
      jwtToken: '{bot_jwt_token}'
```

We support different security providers. You can find their definitions [here](https://github.com/open-metadata/OpenMetadata/tree/main/openmetadata-spec/src/main/resources/json/schema/security/client).
You can find the different implementation of the ingestion below.

<Collapse title="Configure SSO in the Ingestion Workflows">

### Openmetadata JWT Auth

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: openmetadata
    securityConfig:
      jwtToken: '{bot_jwt_token}'
```

### Auth0 SSO

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: auth0
    securityConfig:
      clientId: '{your_client_id}'
      secretKey: '{your_client_secret}'
      domain: '{your_domain}'
```

### Azure SSO

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: azure
    securityConfig:
      clientSecret: '{your_client_secret}'
      authority: '{your_authority_url}'
      clientId: '{your_client_id}'
      scopes:
        - your_scopes
```

### Custom OIDC SSO

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: custom-oidc
    securityConfig:
      clientId: '{your_client_id}'
      secretKey: '{your_client_secret}'
      domain: '{your_domain}'
```

### Google SSO

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: google
    securityConfig:
      secretKey: '{path-to-json-creds}'
```

### Okta SSO

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: http://localhost:8585/api
    authProvider: okta
    securityConfig:
      clientId: "{CLIENT_ID - SPA APP}"
      orgURL: "{ISSUER_URL}/v1/token"
      privateKey: "{public/private keypair}"
      email: "{email}"
      scopes:
        - token
```

### Amazon Cognito SSO

The ingestion can be configured by [Enabling JWT Tokens](https://docs.open-metadata.org/deployment/security/enable-jwt-tokens)

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: auth0
    securityConfig:
      clientId: '{your_client_id}'
      secretKey: '{your_client_secret}'
      domain: '{your_domain}'
```

### OneLogin SSO

Which uses Custom OIDC for the ingestion

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: custom-oidc
    securityConfig:
      clientId: '{your_client_id}'
      secretKey: '{your_client_secret}'
      domain: '{your_domain}'
```

### KeyCloak SSO

Which uses Custom OIDC for the ingestion

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: custom-oidc
    securityConfig:
      clientId: '{your_client_id}'
      secretKey: '{your_client_secret}'
      domain: '{your_domain}'
```

</Collapse>

### 2. Prepare the Ingestion DAG

Create a Python file in your Airflow DAGs directory with the following contents:

```python
import pathlib
import yaml
from datetime import timedelta
from airflow import DAG

try:
    from airflow.operators.python import PythonOperator
except ModuleNotFoundError:
    from airflow.operators.python_operator import PythonOperator

from metadata.config.common import load_config_file
from metadata.ingestion.api.workflow import Workflow
from airflow.utils.dates import days_ago

default_args = {
    "owner": "user_name",
    "email": ["username@org.com"],
    "email_on_failure": False,
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "execution_timeout": timedelta(minutes=60)
}

config = """
<your YAML configuration>
"""

def metadata_ingestion_workflow():
    workflow_config = yaml.safe_load(config)
    workflow = Workflow.create(workflow_config)
    workflow.execute()
    workflow.raise_from_status()
    workflow.print_status()
    workflow.stop()

with DAG(
    "sample_data",
    default_args=default_args,
    description="An example DAG which runs a OpenMetadata ingestion workflow",
    start_date=days_ago(1),
    is_paused_upon_creation=False,
    schedule_interval='*/5 * * * *',
    catchup=False,
) as dag:
    ingest_task = PythonOperator(
        task_id="ingest_using_recipe",
        python_callable=metadata_ingestion_workflow,
    )
```

Note that from connector to connector, this recipe will always be the same.
By updating the YAML configuration, you will be able to extract metadata from different sources.

## Query Usage and Lineage Ingestion

To ingest the Query Usage and Lineage information, the `serviceConnection` configuration will remain the same.
However, the `sourceConfig` is now modeled after this JSON Schema.

### 1. Define the YAML Config

This is a sample config for BigQuery Usage:

```yaml
source:
  type: bigquery-usage
  serviceName: <service name>
  serviceConnection:
    config:
      type: BigQuery
      credentials:
        gcsConfig:
          type: My Type
          projectId: project ID
          privateKeyId: us-east-2
          privateKey: |
            -----BEGIN PRIVATE KEY-----
            Super secret key
            -----END PRIVATE KEY-----
          clientEmail: client@mail.com
          clientId: 1234
          # authUri: https://accounts.google.com/o/oauth2/auth (default)
          # tokenUri: https://oauth2.googleapis.com/token (default)
          # authProviderX509CertUrl: https://www.googleapis.com/oauth2/v1/certs (default)
          clientX509CertUrl: https://cert.url
  sourceConfig:
    config:
      # Number of days to look back
      queryLogDuration: 7
      # This is a directory that will be DELETED after the usage runs
      stageFileLocation: <path to store the stage file>
      # resultLimit: 1000
      # If instead of getting the query logs from the database we want to pass a file with the queries
      # queryLogFilePath: path-to-file
processor:
  type: query-parser
  config: {}
stage:
  type: table-usage
  config:
    filename: /tmp/bigquery_usage
bulkSink:
  type: metadata-usage
  config:
    filename: /tmp/bigquery_usage
workflowConfig:
  # loggerLevel: DEBUG  # DEBUG, INFO, WARN or ERROR
  openMetadataServerConfig:
    hostPort: <OpenMetadata host and port>
    authProvider: <OpenMetadata auth provider>
```

#### Source Configuration - Service Connection

You can find all the definitions and types for the `serviceConnection` [here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/entity/services/connections/database/bigQueryConnection.json).
They are the same as metadata ingestion.

#### Source Configuration - Source Config

The `sourceConfig` is defined [here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/metadataIngestion/databaseServiceQueryUsagePipeline.json).

- `queryLogDuration`: Configuration to tune how far we want to look back in query logs to process usage data.
- `resultLimit`: Configuration to set the limit for query logs

#### Processor, Stage and Bulk Sink

To specify where the staging files will be located.

Note that the location is a directory that will be cleaned at the end of the ingestion.

#### Workflow Configuration

The same as the metadata ingestion.

### 2. Run with the CLI

There is an extra requirement to run the Usage pipelines. You will need to install:

```bash
pip3 install --upgrade 'openmetadata-ingestion[bigquery-usage]'
```

For the usage workflow creation, the Airflow file will look the same as for the metadata ingestion. Updating the YAML configuration will be enough.

## Data Profiler

The Data Profiler workflow will be using the `orm-profiler` processor.
While the `serviceConnection` will still be the same to reach the source system, the `sourceConfig` will be
updated from previous configurations.

### 1. Define the YAML Config

This is a sample config for the profiler:

```yaml
source:
  type: bigquery
  serviceName: <service name>
  serviceConnection:
    config:
      type: BigQuery
      credentials:
        gcsConfig:
          type: My Type
          projectId: project ID
          privateKeyId: us-east-2
          privateKey: |
            -----BEGIN PRIVATE KEY-----
            Super secret key
            -----END PRIVATE KEY-----
          clientEmail: client@mail.com
          clientId: 1234
          # authUri: https://accounts.google.com/o/oauth2/auth (default)
          # tokenUri: https://oauth2.googleapis.com/token (default)
          # authProviderX509CertUrl: https://www.googleapis.com/oauth2/v1/certs (default)
          clientX509CertUrl: https://cert.url
  sourceConfig:
    config:
      type: Profiler
      # generateSampleData: true
      # profileSample: 85
      # threadCount: 5 (default)
      # databaseFilterPattern:
      #   includes:
      #     - database1
      #     - database2
      #   excludes:
      #     - database3
      #     - database4
      # schemaFilterPattern:
      #   includes:
      #     - schema1
      #     - schema2
      #   excludes:
      #     - schema3
      #     - schema4
      # tableFilterPattern:
      #   includes:
      #     - table1
      #     - table2
      #   excludes:
      #     - table3
      #     - table4
processor:
  type: orm-profiler
  config: {}  # Remove braces if adding properties
  # tableConfig:
  #   - fullyQualifiedName: <table fqn>
  #     profileSample: <number between 0 and 99>
  #     columnConfig:
  #       profileQuery: <query to use for sampling data for the profiler>
  #       excludeColumns:
  #         - <column name>
  #       includeColumns:
  #         - columnName: <column name>
  #         - metrics:
  #           - MEAN
  #           - MEDIAN
  #           - ...
sink:
  type: metadata-rest
  config: {}
workflowConfig:
  # loggerLevel: DEBUG  # DEBUG, INFO, WARN or ERROR
  openMetadataServerConfig:
    hostPort: <OpenMetadata host and port>
    authProvider: <OpenMetadata auth provider>
```

#### Source Configuration

- You can find all the definitions and types for the `serviceConnection` [here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/entity/services/connections/database/bigQueryConnection.json).
- The `sourceConfig` is defined [here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/metadataIngestion/databaseServiceProfilerPipeline.json).

Note that the filter patterns support regex as includes or excludes. E.g.,

```yaml
tableFilterPattern:
  includes:
  - *users$
```

#### Processor

Choose the `orm-profiler`. Its config can also be updated to define tests from the YAML itself instead of the UI:

```yaml
processor:
  type: orm-profiler
  config:
    tableConfig:
      - fullyQualifiedName: <table fqn>
        profileSample: <number between 0 and 99>
        columnConfig:
          partitionConfig:
            partitionField: <field to use as a partition field>
            partitionQueryDuration: <for date/datetime partitioning based set the offset from today>
            partitionValues: <values to uses as a predicate for the query>
          profileQuery: <query to use for sampling data for the profiler>
          excludeColumns:
            - <column name>
          includeColumns:
            - columnName: <column name>
            - metrics:
                - MEAN
                - MEDIAN
                - ...
```

`tableConfig` allows you to set up some configuration at the table level.
All the properties are optional. `metrics` should be one of the metrics listed [here](https://docs.open-metadata.org/openmetadata/ingestion/workflows/profiler/metrics)

#### Workflow Configuration

The same as the metadata ingestion.

### 2. Prepare the Profiler DAG

Here, we follow a similar approach as with the metadata and usage pipelines, although we will use a different Workflow class:

```python
import yaml
from datetime import timedelta

from airflow import DAG

try:
   from airflow.operators.python import PythonOperator
except ModuleNotFoundError:
   from airflow.operators.python_operator import PythonOperator

from airflow.utils.dates import days_ago

from metadata.orm_profiler.api.workflow import ProfilerWorkflow


default_args = {
   "owner": "user_name",
   "email_on_failure": False,
   "retries": 3,
   "retry_delay": timedelta(seconds=10),
   "execution_timeout": timedelta(minutes=60),
}

config = """
<your YAML configuration>
"""

def metadata_ingestion_workflow():
   workflow_config = yaml.safe_load(config)
   workflow = ProfilerWorkflow.create(workflow_config)
   workflow.execute()
   workflow.raise_from_status()
   workflow.print_status()
   workflow.stop()

with DAG(
   "profiler_example",
   default_args=default_args,
   description="An example DAG which runs a OpenMetadata ingestion workflow",
   start_date=days_ago(1),
   is_paused_upon_creation=False,
   catchup=False,
) as dag:
   ingest_task = PythonOperator(
       task_id="profile_and_test_using_recipe",
       python_callable=metadata_ingestion_workflow,
   )
```

## DBT Integration

You can learn more about how to ingest DBT models' definitions and their lineage [here](https://docs.open-metadata.org/openmetadata/ingestion/workflows/metadata/dbt).
