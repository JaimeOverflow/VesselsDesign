
# Vessels pipeline design

In this exercise we want to design a data pipeline in AWS for two operations:
- Ingest data to feed an algorithm that creates an AI model to predict vessel consumptions.
- Visualise data aggregates (count, mean, etc) in analytics dashboards.


##  Initial approach for input data

First of all we should talk about the input data, where our sources are Kafka, a postgres database and a web API. 

My initial approach would be that we are a vessels company so our data sources are in our AWS infrastructure except the external weather web API. 

In this case, for the Kafka source we would be using AWS MSK (Managed Streaming for Kafka), for our postgres database we are using AWS RDS (Relational Database Service).

The advantages of using MSK and RDS are that AWS will take care of a lot of operations including patching the OS, manage all the monitoring, scale our services in case we need it (high availability), backups, replicas, etc.

### External input sources

In case our input data sources are external to our infrastructure but we have access to them, my first approach would be to replicate the data in our infrastructure as we don't have control over the external systems.

For our postgres database, we could use the AWS DMS (Database Migration Service) which allows us to migrate external databases to an internal AWS RDS.

For our Kafka data source, we could use MSK Connect which supports connectors that can take the events from the Kafka cluster to our AWS MSK.



## AI model to predict vessel consumptions

We could be using a forescasting service to predict the vessel consumptions like Amazon Forecast or we could use our own AI model which would be running in a Spark cluster (AWS EMR). I'm going to follow the second approach for the design.

We need the data from the different input sources to be able to predict the vessel computions. Firstly, we need the vessel data locations (Kafka) to know the current route. Secondly, we need the vessel performance (postgres) to know engine power and other characteristics. Finally, we need the weather conditions which it has certain influence. We can predict vessel consumption for a route once we have all data together.

## Visualisation dashboard

We need to create dashboard to visualisise aggregated data. In this case, we could use different visualisation tools like Tableau, Power BI or AWS Quicksight.

In this case, I would suggest using AWS Quicksight because most of our infrastructure is in AWS and It provides all the tools to connect to our AWS RDS database and create dashboards.

## First design: Use directly our input data sources

![alt first design](https://github.com/JaimeOverflow/VesselsDesign/blob/main/first_design.png)

This approach would only work if we have all the data we need in our input data sources and minimising the costs in terms of infrastructure.

Our AI model is a Spark application running on the AWS EMR cluster. Using AWS EMR, we need to provide an input AWS S3 bucket and an output bucket for the results so we need to transfer all the data (Kafka, RDS and weather data) into an S3 bucket for the Spark processing.

Apart from that, we need to use the AWS Glue Data Catalog to define the data schema and format in our S3 buckets.

For our Kafka cluster (MSK), we are going to use MSK Connect which allows to store the events into an S3 bucket.

For our AWS RDS database (postgres), we can use AWS DMS (Database Migration Service) to store the database data into an S3 bucket.

For our external web API, we could use a scheduled AWS lambda function ([link](https://aws.amazon.com/releasenotes/release-aws-lambda-on-2015-10-08/)) which sends a request to the API and stores the result into our AWS RDS database which will be used for the dashboards or the vessel consumption AI model. Using a Lambda function will take the minimum effort and reduce the cost to obtain the weather data.

The AWS EMR service will be able to schedule our EMR jobs and provide all the monitoring needed (CloudWatch).

For our dashboards, we would be able to use AWS Quicksight which can connect to your AWS RDS database. We will be able to run multiple functions which can aggregate data ([link](https://docs.aws.amazon.com/quicksight/latest/user/functions-by-category.html)).

### Extras

These extras are not included in the design but we should take into account.

#### Dashboards for our Kafka cluster

In this case, we would need to use MSK Connect (Kafka Connect) to transfer all the events to our RDS database which will be used by Quicksight.

## Second design: Treated data sources

![alt second design](https://github.com/JaimeOverflow/VesselsDesign/blob/main/second_design.png)

In this approach, we are going to use the same approach than the fist design but we are going to add a layer of data processing for our AWS RDS and Kafka cluster so we can have more treated data for our AI model and Quicksight dashboards.

In case our Kafka cluster in MSK, we need to have Kafka application that is able to manipulate our cluster (create topics, produce and consume events from topics) which could use two approaches. The first approach in case we are Java developers is to use a Kafka Streams application. The second approach would be using KsqlDB so we use SQL language.

In case our postgres database, we would need further processing so we should use a workflow orchestration tool like Airflow. In case of AWS, we could use the MWAA service which is a managed Airflow.

Using Airflow, we will be able to orchestrate all the tasks needed for all postgres data (creating table, manipulate data, etc), running the EMR jobs and sending requests to the weather web API and storing the weather data into our RDS database.

My recommendation in this case would be using a K8S cluster which will manage all infrastructure for our Airflow and our Kafka application. Airflow works perfectly fine with K8S and every time an Airflow task is running, i will be processed in a pod. In case our Kafka application, it would run into a pods as well. We could use the AWS EKS service.

### Extras

These extras are not included in the design but we should take into account.

#### Huge amount of data

Our postgres RDS is a row-based database which means that is less efficient for aggregated queries than a columnar database. 
In case we would have huge amount of data in our database, it would be worth taking into account to use a columnar database such as AWS Redshift.

#### Container image repository

I didn't include it into the design but our K8S cluster will need to access to the docker images through a repository that has access. We need to have an image for our Airflow application and another for our Kafka application. We could use the AWS ECR (Elastic Container Registry).

