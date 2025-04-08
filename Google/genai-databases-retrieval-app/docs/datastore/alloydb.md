# Setup and configure AlloyDB (with Public IP)

## Before you begin

1. Make sure you have a Google Cloud project and billing is enabled.

1. Set your `PROJECT_ID` environment variable:

    ```bash
    export PROJECT_ID=<YOUR_PROJECT_ID>
    ```

1. [Install](https://cloud.google.com/sdk/docs/install) the gcloud CLI.

1. Set gcloud project:

    ```bash
    gcloud config set project $PROJECT_ID
    ```

1. Enable APIs:

    ```bash
    gcloud services enable alloydb.googleapis.com \
                           compute.googleapis.com \
                           cloudresourcemanager.googleapis.com \
                           servicenetworking.googleapis.com \
                           vpcaccess.googleapis.com \
                           aiplatform.googleapis.com
    ```

1. [Install python][install-python] and set up a python [virtual environment][venv].

1. Make sure you have python version 3.11+ installed.

    ```bash
    python -V
    ```

1. Download and install [postgres-client cli (`psql`)][install-psql].

1. Install the [AlloyDB Auth Proxy][install-alloydb-auth-proxy].

[install-python]: https://cloud.google.com/python/docs/setup#installing_python
[venv]: https://cloud.google.com/python/docs/setup#installing_and_using_virtualenv
[install-psql]: https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/
[install-alloydb-auth-proxy]: https://cloud.google.com/alloydb/docs/auth-proxy/connect#install


## Enable private services access

In this step, we will enable Private Services Access so that AlloyDB is able to
connect to your VPC. You should only need to do this once per VPC (per project).

1. Set environment variables:

    ```bash
    export RANGE_NAME=my-allocated-range-default
    export DESCRIPTION="peering range for alloydb-service"
    ```

1. Create an allocated IP address range:

    ```bash
    gcloud compute addresses create $RANGE_NAME \
        --global \
        --purpose=VPC_PEERING \
        --prefix-length=16 \
        --description="$DESCRIPTION" \
        --network=default
    ```

1. Create a private connection:

    ```bash
    gcloud services vpc-peerings connect \
        --service=servicenetworking.googleapis.com \
        --ranges="$RANGE_NAME" \
        --network=default
    ```


## Create an AlloyDB cluster

1. Set environment variables. For security reasons, use a different password for
   `$DB_PASS` and note it for future use:

    ```bash
    export CLUSTER=my-alloydb-cluster
    export INSTANCE=my-alloydb-instance
    export REGION=us-central1
    export DB_USER=postgres
    export DB_PASS=my-alloydb-pass
    ```

1. Create an AlloyDB cluster:

    ```bash
    gcloud alloydb clusters create $CLUSTER \
        --password=$DB_PASS\
        --network=default \
        --region=$REGION \
        --project=$PROJECT_ID
    ```

1. Create a primary instance:

    ```bash
    gcloud alloydb instances create $INSTANCE \
        --instance-type=PRIMARY \
        --cpu-count=8 \
        --region=$REGION \
        --cluster=$CLUSTER \
        --project=$PROJECT_ID \
        --ssl-mode=ALLOW_UNENCRYPTED_AND_ENCRYPTED \
        --database-flags=password.enforce_complexity=on
    ```

1. Enable public IP on instance:

    ```bash
    gcloud alloydb instances update $INSTANCE \
        --cluster=$CLUSTER  \
        --region=$REGION  \
        --assign-inbound-public-ip=ASSIGN_IPV4
    ```


## Connect to the AlloyDB instance

1. Connect to instance using AlloyDB auth proxy:

    ```bash
    ./alloydb-auth-proxy --public-ip \
        "projects/$PROJECT_ID/locations/$REGION/clusters/$CLUSTER/instances/$INSTANCE"
    ```

1. Verify you can connect to your instance with the `psql` tool. Enter
   password for AlloyDB (`$DB_PASS` environment variable set above) when prompted:

    ```bash
    psql -h 127.0.0.1 -p 5432 -U $DB_USER
    ```

## Update config

1. Change into the retrieval service directory:

    ```bash
    cd ./retrieval_service
    ```

1. Install requirements:

    ```bash
    pip install -r requirements.txt
    ```

1. Make a copy of `example-config.yml` and name it `config.yml`.

    ```bash
    cp example-config-alloydb.yml config.yml
    ```

1. Update `config.yml` with your database information.

```bash
host: 0.0.0.0
datastore:
    # Example for alloydb.py provider
    kind: "alloydb-postgres"
    # Update this with your project ID
    project: <PROJECT_ID>
    region: us-central1
    cluster: my-alloydb-cluster
    instance: my-alloydb-instance
    # Update this with the database name
    database: "assistantdemo"
    # Update with database user, the default is `postgres`
    user: "postgres"
    # Update with database user password
    password: "my-alloydb-pass"
```

## Initialize data

1. While connected using `psql`, create a database and switch to it:

    ```bash
    CREATE DATABASE assistantdemo;
    \c assistantdemo
    ```

1. Install [`pgvector`][pgvector] extension in the database:

    ```bash
    CREATE EXTENSION vector;
    ```

1. Populate data into database:

    ```bash
    python run_database_init.py
    ```

[pgvector]: https://github.com/pgvector/pgvector

## Clean up resources

Clean up after completing the demo.

1. Set environment variables:

    ```bash
    export CLUSTER=my-alloydb-cluster
    export REGION=us-central1
    ```

1. Delete AlloyDB cluster that contains instances:

    ```bash
    gcloud alloydb clusters delete $CLUSTER \
        --force \
        --region=$REGION \
        --project=$PROJECT_ID
    ```

## Developer information

This section is for developers that want to develop and run the app locally.

### Test Environment Variables

#### 1. Create Secrets
Follow the steps [here](https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets) to create two secrets with the following Secret names:
1. `alloy_db_user`
1. `alloy_db_pass`

Set the Secret values as your database username and password respectively.

#### 2. Set environment variables:

```bash
export DB_NAME=""
export DB_USER=""
export DB_PASS=""
export DB_PROJECT=""
export DB_REGION=""
export DB_CLUSTER=""
export DB_INSTANCE=""
```

#### 3. Run tests

Run retrieval service unit tests:

```bash
gcloud builds submit --config retrieval_service/alloydb.tests.cloudbuild.yaml \
    --substitutions _DATABASE_NAME=$DB_NAME,_DATABASE_USER=$DB_USER,_ALLOYDB_REGION=$DB_REGION,_ALLOYDB_CLUSTER=$DB_CLUSTER,_ALLOYDB_INSTANCE=$DB_INSTANCE
```

Where `$DB_NAME`,`$DB_USER`,`$DB_REGION`,`$DB_CLUSTER`,`$DB_INSTANCE` are environment variables with your database values.

### Troubleshooting
If you get the following error:
```
failed to access secret version for secret projects/<PROJECT_NUMBER>/secrets/alloy_db_user/versions/1: rpc error: code = PermissionDenied desc = Permission 'secretmanager.versions.access' denied for resource 'projects/<PROJECT_NUMBER>/secrets/alloy_db_user/versions/1' (or it may not exist).
```
Go to `Cloud Build > Settings`, and make sure that the GCP Service `Secret Manager` is enabled for your Service Account.

You can find the Service Account under `History > <BUILD VERSION> > Execution Details > Service Account`.

Eg. `<PROJECT_NUMBER>-compute@developer.gserviceaccount.com`