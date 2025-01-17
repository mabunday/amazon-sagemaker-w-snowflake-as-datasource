Use Snowflake as a data source to train ML models with Amazon SageMaker
================

*Amit Arora*, *Divya Muralidharan*, *Sergey Ermolin*

***This blog post has been updated in May 2023 to include a workflow
that does not require building a custom container***.

[Amazon SageMaker](https://aws.amazon.com/sagemaker/) is a fully managed
machine learning (ML) service. With SageMaker, data scientists and
developers can quickly and easily build and train ML models, and then
directly deploy them into a production-ready hosted environment.
Sagemaker provides an integrated Jupyter authoring notebook instance for
easy access to your data sources for exploration and analysis, so you
don’t have to manage servers. It also provides common ML algorithms that
are optimized to run efficiently against extremely large data in a
distributed environment.

SageMaker requires that the training data for an ML model be present
either in [Amazon Simple Storage Service (Amazon S3), Amazon Elastic
File System (Amazon EFS) or Amazon FSx for Lustre (for more information,
refer to Access Training
Data)](https://docs.aws.amazon.com/sagemaker/latest/dg/model-access-training-data.html).
In order to train a model using data stored outside of the three
supported storage services, the data first needs to be ingested into one
of these services (typically Amazon S3). This requires building a data
pipeline (using tools such as [Amazon SageMaker Data
Wrangler](https://aws.amazon.com/sagemaker/data-wrangler/)) to move data
into Amazon S3. However, this approach may create a data management
challenge in terms of managing the lifecycle of this data storage
medium, crafting access controls, data auditing, and so on, all for the
purpose of staging training data for the duration of the training job.
In such situations, it may be desirable to have the data accessible to
SageMaker in the ephemeral storage media attached to the ephemeral
training instances without the intermediate storage of data in Amazon
S3.

This post shows a way to do this using
[Snowflake](https://www.snowflake.com/) as the data source and by
downloading the data directly from Snowflake into a SageMaker Training
job instance.

## Solution overview

We use the [California Housing
Dataset](https://inria.github.io/scikit-learn-mooc/python_scripts/datasets_california_housing.html)
as a training dataset for this post and train an ML model to predict the
median house value for each district. We add this data to Snowflake as a
new table. We create a custom training container that downloads data
directly from the Snowflake table into the training instance rather than
first downloading the data into an S3 bucket. *We also provide an
optional workflow that does not require creating a custom container*.
After the data is downloaded into the training instance, the custom
training script performs data preparation tasks and then trains the ML
model using the [XGBoost
Estimator](https://sagemaker.readthedocs.io/en/stable/frameworks/xgboost/using_xgboost.html).
All code for this post is available in the [GitHub
repo](https://github.com/aws-samples/amazon-sagemaker-w-snowflake-as-datasource).

The following figure represents the high-level architecture of the
proposed solution to use Snowflake as a data source to train ML models
with SageMaker.

<figure>
<img src="img/snowflake-sagemaker-page-1.png" id="fig-architecture"
alt="Figure 1: Architecture" />
<figcaption aria-hidden="true">Figure 1: Architecture</figcaption>
</figure>

The workflow steps are as follows:

1.  Set up a SageMaker notebook and an [AWS Identity and Access
    Management](https://aws.amazon.com/iam/) (IAM) role with appropriate
    permissions to allow SageMaker to access [Amazon Elastic Container
    Registry](https://aws.amazon.com/ecr/) (Amazon ECR), Secrets
    Manager, and other services within your AWS account.

2.  Store your Snowflake account credentials in AWS Secrets Manager.

3.  Ingest the data in a table in your Snowflake account.

4.  Create a custom container image for ML model training and push it to
    Amazon ECR.

5.  Launch a SageMaker Training job for training the ML model. The
    training instance retrieves Snowflake credentials from Secrets
    Manager and then uses these credentials to download the dataset from
    Snowflake directly. This is the step that eliminates the need for
    data to be first downloaded into an S3 bucket.

6.  The trained ML model is stored in an S3 bucket.

### Prerequisites

To implement the solution provided in this post, you should have an [AWS
account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup),
a [Snowflake
account](https://signup.snowflake.com/?utm_cta=trial-en-www-homepage-top-right-nav-ss-evg&_ga=2.36125795.2140702267.1672969035-1338836953.1670007010)
and familiarity with SageMaker.

#### Set up a SageMaker Notebook and IAM role

We use AWS CloudFormation to create a SageMaker notebook called
`aws-aiml-blogpost-sagemaker-snowflake-example` and an IAM role called
`SageMakerSnowFlakeExample`. Choose **Launch Stack** for the Region you
want to deploy resources to.

|        AWS Region         |                                                                                                                                                   Link                                                                                                                                                   |
|:-------------------------:|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|  us-east-1 (N. Virginia)  |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=sagemaker-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-12893/sagemaker-snowflake-template.yml)    |
|     us-east-2 (Ohio)      |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=sagemaker-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-12893/sagemaker-snowflake-template.yml)    |
| us-west-1 (N. California) |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-west-1#/stacks/new?stackName=sagemaker-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-12893/sagemaker-snowflake-template.yml)    |
|    us-west-2 (Oregon)     |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=sagemaker-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-12893/sagemaker-snowflake-template.yml)    |
|    eu-west-1 (Dublin)     |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=sagemaker-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-12893/sagemaker-snowflake-template.yml)    |
|  ap-northeast-1 (Tokyo)   | [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/new?stackName=sagemaker-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-12893/sagemaker-snowflake-template.yml) |

#### Store Snowflake credentials in Secrets Manager

Store your Snowflake credentials as a secret in Secrets Manager. For
instructions on how to create a secret, refer to
[`Create an AWS Secrets Manager secret`](https://docs.aws.amazon.com/secretsmanager/latest/userguide/create_secret.html).

1.  Name the secret `snowflake_credentials`. This is required because
    the code in `snowflake-load-dataset.ipynb` expects the secret to be
    called that.

2.  Create the secret as a key-value pair with two keys:

    - **username** - Your Snowflake user name.
    - **password** - The password associated with your Snowflake user
      name.

#### Ingest the data in a table in your Snowflake account

To ingest the data, complete the following steps:

1.  On the SageMaker console, choose **Notebooks** in the navigation
    pane.

2.  Select the notebook aws-aiml-blogpost-sagemaker-snowflake-example
    and choose **Open JupyterLab**.

    <figure>
    <img src="img/sm-nb-jl.png" id="fig-open-jl"
    alt="Figure 2: Open JupyterLab" />
    <figcaption aria-hidden="true">Figure 2: Open JupyterLab</figcaption>
    </figure>

3.  Choose `snowflake-load-dataset.ipynb` to open it in JupyterLab. This
    notebook will ingest the [California Housing
    Dataset](https://inria.github.io/scikit-learn-mooc/python_scripts/datasets_california_housing.html)
    to a Snowflake table.

4.  In the notebook, edit the contents of the following cell to replace
    the placeholder values with the one matching your snowflake account:

    ``` python
    sf_account_id = "your-snowflake-account-id"
    ```

5.  On the Run menu, choose **Run All Cells** to run the code in this
    notebook. This will download the dataset locally into the notebook
    and then ingest it into the Snowflake table.

    <figure>
    <img src="img/sm-nb-runall.png" id="fig-notebook-run-all-cells"
    alt="Figure 3: Notebook Run All Cells" />
    <figcaption aria-hidden="true">Figure 3: Notebook Run All
    Cells</figcaption>
    </figure>

    The following code snippet in the notebook ingests the dataset into
    Snowflake. See the `snowflake-load-dataset.ipynb` notebook for the
    full code.

    ``` python
    # connect to Snowflake Table schema
    conn.cursor().execute(f"CREATE SCHEMA IF NOT EXISTS {schema}")
    conn.cursor().execute(f"USE SCHEMA {schema}")

    create_table_sql = f"CREATE TABLE IF NOT EXISTS {db}.{schema}.{table}\n ("

    california_housing.rename(columns=str.upper, inplace=True)
    # iterating through the columns
    for col in california_housing.columns:
        column_name = col.upper()

    if (california_housing[col].dtype.name == "int" or california_housing[col].dtype.name == "int64"):
        create_table_sql = create_table_sql + column_name + " int"
    elif california_housing[col].dtype.name == "object":
        create_table_sql = create_table_sql + column_name + " varchar(16777216)"
    elif california_housing[col].dtype.name == "datetime64[ns]":
        create_table_sql = create_table_sql + column_name + " datetime"
    elif california_housing[col].dtype.name == "float64":
        create_table_sql = create_table_sql + column_name + " float8"
    elif california_housing[col].dtype.name == "bool":
        create_table_sql = create_table_sql + column_name + " boolean"
    else:
        create_table_sql = create_table_sql + column_name + " varchar(16777216)"

    # Deciding next steps. Either column is not the last column (add comma) else end create_tbl_statement
    if california_housing[col].name != california_housing.columns[-1]:
        create_table_sql = create_table_sql + ",\n"
    else:
        create_table_sql = create_table_sql + ")"

    # execute the SQL statement to create the table
    print(f"create_table_sql={create_table_sql}")
    conn.cursor().execute(create_table_sql)  
    print(f"snowflake_table={snowflake_table}")
    conn.cursor().execute('TRUNCATE TABLE IF EXISTS ' + snowflake_table)
    ```

6.  Close the notebook after all cells run without any error. Your data
    is now available in Snowflake. The following screenshot shows the
    `california_housing` table created in Snowflake.

    <figure>
    <img src="img/snowflake-table.png" id="fig-snowflake-table"
    alt="Figure 4: Snowflake Table" />
    <figcaption aria-hidden="true">Figure 4: Snowflake Table</figcaption>
    </figure>

#### Run the `sagemaker-snowflake-example.ipynb` notebook

This notebook creates a custom training container with a Snowflake
connection, extracts data from Snowflake into the training instance’s
ephemeral storage without staging it in Amazon S3, and performs
Distributed Data Parallel (DDP) XGBoost model training on the data. DDP
training is not required for model training on such a small dataset; it
is included here for illustration of yet another recently released
SageMaker feature.

<figure>
<img src="img/sm_snowflake_example.png" id="fig-open-blog-post-notebook"
alt="Figure 5: Open SageMaker Snowflake Example Notebook" />
<figcaption aria-hidden="true">Figure 5: Open SageMaker Snowflake
Example Notebook</figcaption>
</figure>

#### Create a custom container for training

We now create a custom container for the ML model training job. Note
that root access is required for creating a Docker container. This
SageMaker notebook was deployed with root access enabled. If your
enterprise organization policies don’t allow root access to cloud
resources, you may want to use the following Docker file and shell
scripts to build a Docker container elsewhere (for example, your laptop)
and then push it to Amazon ECR. We use the container based on the
SageMaker XGBoost container image
`246618743249.dkr.ecr.us-west-2.amazonaws.com/sagemaker-xgboost:1.5-1`
with the following additions:

- The [Snowflake Connector for
  Python](https://docs.snowflake.com/en/user-guide/python-connector.html)
  to download the data from the Snowflake table to the training
  instance.

- A Python script to connect to Secrets Manager to retrieve Snowflake
  credentials.

Using the Snowflake connector and Python script ensures that users who
use this container image for ML model training don’t have to write this
code as part of their training script and can use this functionality
that is already available to them.

The following is the Dockerfile for the training container:

``` python
# Build an image that can be used for training in Amazon SageMaker, we use
# the SageMaker XGBoost as the base image as it contains support for distributed
# training.
FROM 246618743249.dkr.ecr.us-west-2.amazonaws.com/sagemaker-xgboost:1.5-1

MAINTAINER Amazon AI <sage-learner@amazon.com>


RUN apt-get -y update && apt-get install -y --no-install-recommends \
         wget \
         python3-pip \
         python3-setuptools \
         nginx \
         ca-certificates \
   && rm -rf /var/lib/apt/lists/*

RUN ln -s /usr/bin/python3 /usr/bin/python
RUN ln -s /usr/bin/pip3 /usr/bin/pip

# Here we get snowflake-connector python package.
# pip leaves the install caches populated which uses a 
# significant amount of space. These optimizations save a fair 
# amount of space in the image, which reduces start up time.
RUN pip --no-cache-dir install snowflake-connector-python==2.8.3  

# Include python script for retrieving Snowflake credentials 
# from AWS SecretsManager
ADD snowflake_credentials.py /
```

The container image is built and pushed to Amazon ECR. This image is
used for training the ML model.

#### Run the `sagemaker-snowflake-example-1p.ipynb` notebook (Optional)

This notebook does not create a custom training container but rather
uses the “bring your own script” mode of SageMaker Training. We use the
SageMaker XGBoost container and provide additional scripts in the `src`
directory as part of the `source_dir` parameter to the `Estimator`.
These additional scripts were earlier packaged as part of the custom
container. In this approach, the `Estimator` automatically provides all
files present in the `src` directory to the container on startup. The
`requirements.txt` file that lists the `snowflake-connector-python`
Python package as a dependency is also included in the `src` directory
and this package is installed automatically on container startup.

#### Train the ML model using a SageMaker Training job

After we successfully create the container image and push it to Amazon
ECR, we can start using it for model training.

1.  We create a set of Python scripts to download the data from
    Snowflake using the [Snowflake Connector for
    Python](https://docs.snowflake.com/en/user-guide/python-connector.html),
    prepare the data and then use the `XGBoost Regressor` to train the
    ML model. It is the step of downloading the data directly to the
    training instance that avoids having to use Amazon S3 as the
    intermediate storage for training data.

2.  We facilitate Distributed Data Parallel training by having the
    training code download a random subset of the data such that each
    training instance downloads an equal amount of data from Snowflake.
    For example, if there are two training nodes, then each node
    downloads a random sample of 50% of the rows in the Snowflake table.

    See the following code:

    ``` python
    """
    Read the HOUSING table (this is the california housing dataset  used by this example)
    """
    import pandas as pd
    import snowflake.connector

    def data_pull(ctx: snowflake.connector.SnowflakeConnection, table: str, hosts: int) -> pd.DataFrame:

        # Query Snowflake HOUSING table for number of table records
        sql_cnt = f"select count(*) from {table};"
        df_cnt = pd.read_sql(sql_cnt, ctx)

        # Retrieve the total number of table records from dataframe
        for index, row in df_cnt.iterrows():
            num_of_records = row.astype(int)
            list_num_of_rec = num_of_records.tolist()
        tot_num_records = list_num_of_rec[0]

        record_percent = str(round(100/hosts))
        print(f"going to download a random {record_percent}% sample of the data")
        # Query Snowflake HOUSING table
        sql = f"select * from {table} sample ({record_percent});"
        print(f"sql={sql}")

        # Get the dataset into Pandas
        df = pd.read_sql(sql, ctx)
        print(f"read data into a dataframe of shape {df.shape}")
        # Prepare the data for ML
        df.dropna(inplace=True)

        print(f"final shape of dataframe to be used for training {df.shape}")
        return df
    ```

3.  We then provide the training script to the SageMaker SDK
    [`Estimator`](https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html)
    along with the source directory so that all the scripts we create
    can be provided to the training container when the training job is
    run using the
    [`Estimator.fit`](https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html#sagemaker.estimator.EstimatorBase.fit)
    method:

    ``` python
    custom_img_uri = f"{account_id}.dkr.ecr.{region}.amazonaws.com/{custom_img_name}:{custom_img_tag}"

    # Create Sagemaker Estimator
    xgb_script_mode_estimator = sagemaker.estimator.Estimator(
        image_uri = custom_img_uri,
        role=role,
        instance_count=instance_count,
        instance_type=instance_type,
        output_path="s3://{}/{}/output".format(bucket, prefix),
        sagemaker_session=session,
        entry_point="train.py",
        source_dir="./src",
        hyperparameters=hyperparams,
        environment=env,
        subnets = subnet_ids,
    )

    # start the training job
    xgb_script_mode_estimator.fit()
    ```

    For more information, refer to [Prepare a Scikit-Learn Training
    Script](https://sagemaker.readthedocs.io/en/stable/frameworks/sklearn/using_sklearn.html#prepare-a-scikit-learn-training-script).

4.  After the model training is complete, the trained model is available
    as a `model.tar.gz` file in the default SageMaker bucket for the
    Region:

    ``` python
    print(f"the trained model is available in Amazon S3 -> {xgb_script_mode_estimator.model_data}")
    ```

    You can now deploy the trained model for getting inference on new
    data! For instructions, refer to [Create your endpoint and deploy
    your
    model.](https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints-deployment.html)

## Clean up

To avoid incurring future charges, delete the resources. You can do this
by deleting the CloudFormation template used to create the IAM role and
SageMaker notebook.

<figure>
<img src="img/cfn-delete.png" id="fig-cleaning-up-2"
alt="Figure 6: Cleaning Up" />
<figcaption aria-hidden="true">Figure 6: Cleaning Up</figcaption>
</figure>

You will have to delete the Snowflake resources manually from the
Snowflake console.

## Conclusion

In this post, we showed how to download data stored in a Snowflake table
to a SageMaker Training job instance and train an XGBoost model using a
custom training container. This approach allows us to directly integrate
Snowflake as a data source with a SageMaker notebook without having the
data staged in Amazon S3.

We encourage you to learn more by exploring the [Amazon SageMaker Python
SDK](https://sagemaker.readthedocs.io/en/stable/) and building a
solution using the sample implementation provided in this post and a
dataset relevant to your business. If you have questions or suggestions,
leave a comment.

------------------------------------------------------------------------

## Author bio

<img style="float: left; margin: 0 10px 0 0;" src="img/Amit.png">Amit
Arora is an AI and ML specialist architect at Amazon Web Services,
helping enterprise customers use cloud-based machine learning services
to rapidly scale their innovations. He is also an adjunct lecturer in
the MS data science and analytics program at Georgetown University in
Washington D.C.

<br><br>

<img style="float: left; margin: 0 10px 0 0;" src="img/Divya.jfif">Divya
Muralidharan is a Solutions Architect at Amazon Web Services. She is
passionate about helping enterprise customers solve business problems
with technology. She has a Masters in Computer Science from Rochester
Institute of Technology. Outside of office, she spends time cooking,
singing, and growing plants.

<br><br>

<img style="float: left; margin: 0 10px 0 0;" src="img/sergey.jfif">Sergey
Ermolin is a Principal AIML Solutions Architect at AWS. Previously, he
was a software solutions architect for deep learning, analytics, and big
data technologies at Intel. A Silicon Valley veteran with a passion for
machine learning and artificial intelligence, Sergey has been interested
in neural networks since pre-GPU days, when he used them to predict
aging behavior of quartz crystals and cesium atomic clocks at
Hewlett-Packard. Sergey holds an MSEE and a CS certificate from Stanford
and a BS degree in physics and mechanical engineering from California
State University, Sacramento. Outside of work, Sergey enjoys
wine-making, skiing, biking, sailing, and scuba-diving. Sergey is also a
volunteer pilot for [Angel Flight](https://www.angelflightwest.org/).
