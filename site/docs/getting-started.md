# Getting Started

There are multiple ways to deploy Lakekeeper. You can use one of our [self-contained examples](#self-contained-examples), deploy on [Kubernetes](#kubernetes) or deploy the [Binary](#binary) directly.

## Deployment

### Self-Contained Examples
!!! note

    Our docker compose examples are not designed to be used with compute outside of the docker network (e.g. external Spark).

All docker expose examples come with batteries included (Identity Provider, Storage (S3), Query Engines, Jupyter) but are not accessible (by default) for compute outside of the docker network.

=== "🐳 With Authentication & Authorization"

    ```bash
    git clone https://github.com/lakekeeper/lakekeeper
    cd docker-compose/access-control
    docker-compose up -d
    ```

=== "🐳 Without Authentication"

    ```bash
    git clone https://github.com/lakekeeper/lakekeeper
    cd docker-compose/minimal
    docker-compose up -d
    ```

Then open your browser and head to `localhost:8888` to load the example Jupyter notebooks or head to `localhost:8080` for the Lakekeeper UI.

### Kubernetes
We recommend deploying the catalog on Kubernetes using our [Helm Chart](https://github.com/lakekeeper/lakekeeper-charts/tree/main/charts/lakekeeper). Please check the Helm Chart's documentation for possible values. To enable Authentication and Authorization, and external identity provider is required.

A community driven [Kubernetes Operator](https://github.com/lakekeeper/lakekeeper-operator) is currently in development.

### Binary

For single node deployments, you can also download the Binary for your architecture from [Github Releases](https://github.com/lakekeeper/lakekeeper/releases). A basic configuration via environment variables would look something like this:

```bash
export LAKEKEEPER__BASE_URI=http://localhost:8080
export LAKEKEEPER__LISTEN_PORT=8080
export LAKEKEEPER__PG_DATABASE_URL_READ="postgres://postgres_user:postgres_urlencoded_password@hostname:5432/catalog_database"
export LAKEKEEPER__PG_DATABASE_URL_WRITE="postgres://postgres_user:postgres_urlencoded_password@hostname:5432/catalog_database"
export LAKEKEEPER__PG_ENCRYPTION_KEY="MySecretEncryptionKeyThatIBetterNotLoose"

./lakekeeper migrate
./lakekeeper serve
```

## First Steps

Now that the catalog is up-and-running, the following endpoints are available:

1. `<LAKEKEEPER__BASE_URI>` - the UI
1. `<LAKEKEEPER__BASE_URI>/catalog` is the Iceberg REST API
1. `<LAKEKEEPER__BASE_URI>/management` contains the management API
1. `<LAKEKEEPER__>/swagger-ui` hosts Swagger to inspect the API specifications

### Bootstrapping
Our self-contained docker compose examples are already bootstrapped and require no further actions.

After the initial deployment, Lakekeeper needs to be bootstrapped. This can be done via the UI or the bootstrap endpoint. Among others, bootstrapping sets the initial administrator of lakekeeper and creates the first project. Please find more information on bootstrapping in the [Bootstrap Docs](../docs/nightly/bootstrap/).

### Creating a Warehouse
Now that the server is running, we need to create a new warehouse. We recommend to do this via the UI. Alternatively, we can use the REST-API directly.

For an S3 backed warehouse, create a file called `create-warehouse-request.json`:

```json
{
  "warehouse-name": "my-warehouse",
  "storage-profile": {
    "type": "s3",
    "bucket": "my-example-bucket",
    "key-prefix": "optional/path/in/bucket",
    "region": "us-east-1",
    "sts-role-arn": "arn:aws:iam::....:role/....",
    "sts-enabled": true,
    "flavor": "aws"
  },
  "storage-credential": {
    "type": "s3",
    "credential-type": "access-key",
    "aws-access-key-id": "...",
    "aws-secret-access-key": "..."
  }
}
```

We now create a new Warehouse by POSTing the request to the management API:

```sh
curl -X POST http://localhost:8080/management/v1/warehouse -H "Content-Type: application/json" -d @create-warehouse-request.json
```

If you want to use a different storage backend, see the [Storage Guide](../docs/nightly/storage/) for example configurations.

### Connect Compute

That's it - we can now use the catalog:

```python
import pandas as pd
import pyspark

SPARK_VERSION = pyspark.__version__
SPARK_MINOR_VERSION = '.'.join(SPARK_VERSION.split('.')[:2])
ICEBERG_VERSION = "1.6.1"

# if you use adls as storage backend, you need iceberg-azure instead of iceberg-aws-bundle
configuration = {
    "spark.jars.packages": f"org.apache.iceberg:iceberg-spark-runtime-{SPARK_MINOR_VERSION}_2.12:{ICEBERG_VERSION},org.apache.iceberg:iceberg-aws-bundle:{ICEBERG_VERSION}",
    "spark.sql.extensions": "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions",
    "spark.sql.defaultCatalog": "demo",
    "spark.sql.catalog.demo": "org.apache.iceberg.spark.SparkCatalog",
    "spark.sql.catalog.demo.catalog-impl": "org.apache.iceberg.rest.RESTCatalog",
    "spark.sql.catalog.demo.uri": "http://localhost:8080/catalog/",
    "spark.sql.catalog.demo.token": "dummy",
    "spark.sql.catalog.demo.warehouse": "my-warehouse",
}
spark_conf = pyspark.SparkConf()
for k, v in configuration.items():
    spark_conf = spark_conf.set(k, v)

spark = pyspark.sql.SparkSession.builder.config(conf=spark_conf).getOrCreate()

spark.sql("USE demo")

spark.sql("CREATE NAMESPACE IF NOT EXISTS my_namespace")
print(f"\n\nCurrently the following namespace exist:")
print(spark.sql("SHOW NAMESPACES").toPandas())
print("\n\n")

sdf = spark.createDataFrame(
    pd.DataFrame(
        [[1, 1.2, "foo"], [2, 2.2, "bar"]], columns=["my_ints", "my_floats", "strings"]
    )
)

spark.sql("DROP TABLE IF EXISTS demo.my_namespace.my_table")
spark.sql(
    "CREATE TABLE demo.my_namespace.my_table (my_ints INT, my_floats DOUBLE, strings STRING) USING iceberg"
)
sdf.writeTo("demo.my_namespace.my_table").append()
spark.table("demo.my_namespace.my_table").show()
```