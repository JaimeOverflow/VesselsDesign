# Vessels pipeline design

In this exercise we want to design a data pipeline in AWS for two operations:
- Ingest data to feed an algorithm that creates an AI model to predict vessel consumptions.
- Visualise data aggregates (count, mean, etc) in analytics dashboards.

## Aditional suggestions

- Reduce to 0 the risk that analytical queries would impact the online transactional database.
- Guarantee that the analytics takes into considerations, data cleanup and enrichment that happens in spark jobs.
- To reduce the amount of weather to store, please consider that we only need to request the web API for weather in the location of the vessels that we have obtained by kafka at the specified vessel position time.

##  Initial approach for input data

First of all we should talk about the input data, where our sources are Kafka, a postgres database and a web API. 

My initial approach would be that we are a vessels company so our data sources are in our AWS infrastructure except the external weather web API. 

In this case, for the Kafka source we would be using AWS MSK (Managed Streaming for Kafka), for our postgres database we are using AWS RDS (Relational Database Service).

The advantages of using MSK and RDS are that AWS will take care of a lot of operations including patching the OS, manage all the monitoring, scale our services in case we need it (high availability), backups, replicas, etc.

### External input sources

In case our input data sources are external to our infrastructure but we have access to them, my first approach would be to replicate the data in our infrastructure as we don't have control over the external systems.

For our postgres database, we could use the AWS DMS (Database Migration Service) which allows us to migrate external databases to an internal AWS RDS.

For our Kafka data source, we could use MSK Connect which supports connectors that can take the events from the Kafka cluster to our AWS MSK.

## Data warehose

As part of the additional suggestions section, we are going to have analytical queries and we don't want them to affect our AWS RDS postgres database.

My suggestion is to use AWS Redshift which is an online analytical database that works very efficiently for analytical/aggregated queries. The reason is because the data is stored in columns into the memory so our aggregated queries have a speed up compared to storing data as rows like our postgres database.

## AI model to predict vessel consumptions

We could be using a forescasting service to predict the vessel consumptions like Amazon Forecast or we could use our own AI model which would be running in a Spark cluster (AWS EMR). I'm going to follow the second approach for the design.

We need the data from the different input sources to be able to predict the vessel computions. Firstly, we need the vessel data locations (Kafka) to know the current route. Secondly, we need the vessel performance (postgres) to know engine power and other characteristics. Finally, we need the weather conditions which it has certain influence. We can predict vessel consumption for a route once we have all data together. 

## Visualisation dashboard

We need to create dashboard to visualisise aggregated data. In this case, we could use different visualisation tools like Tableau, Power BI or AWS Quicksight.

In this case, I would suggest using AWS Quicksight because most of our infrastructure is in AWS and It provides all the tools to connect our data warehose.

## Design:

First of all, our AI model is a Spark application running on the AWS EMR cluster. Using AWS EMR, we need to provide an input AWS S3 bucket and an output bucket for the results. Apart from that, we need to use the AWS Glue Data Catalog to define the data schema and format in our S3 buckets.

We know that we are going to need further processing over the initial data sources which will allow our AI model to process the data and finally building dashboard using Quicksight.

In case our Kafka cluster in MSK, we need to have Kafka application that is able to manipulate our cluster (create topics, produce and consume events from topics) which could use two approaches. The first approach in case we are Java developers is to use a Kafka Streams application. The second approach would be using KsqlDB so we use SQL language. I will explain more about it later.

For our Kafka cluster (MSK), we are going to use MSK Connect which allows to store the events into an S3 bucket.

In case our postgres database, we would need further processing for analytical queries which means I need to transfer all the needed data to our data warehose which is Redshift in this case.

I'm going to use AWS DMS (Database Migration Service) to transfer the data from our AWS RDS postgres to Redshift.

We are going to need a workflow orchestration tool like Airflow which will take care of most of the tasks in our system and everything will be scheduled.

Airflow will take care of additional tasks including:
- Run the analytical queries in our data warehose, store the results into the AWS S3 input bucket for our AWS EMR Spark jobs.
- Take the last vessels locations from our Kafka cluster and send a request to the weather web API with the weather location we are interested and finally store it into our data warehose which will could by the analytical queries that mentioned previously and finally use it for the AWS EMR jobs.
- In case we need to aggregate data between our Kafka cluster and our data warehose, we could store the Kafka events into an S3 bucket using MSK Connect, and use Airflow to load the data from this bucket to our data warehose.
- Schedule and run the AWS EMR jobs when we have stored the processed data into the AWS S3 input bucket.

Once the AWS EMR jobs are done, we can decide to create the Quicksight dashboards using the AWS S3 output bucket from EMR or we could use Airflow to ingest the data from the bucket to Redshift and build the dashboards from the Redshift data (It's the approach I took in the design image).

We are going to need a Kubernetes cluster to manage all infrastructure for our Airflow and our Kafka application. Airflow works perfectly fine with K8S and every time an Airflow task is running, i will be processed in a pod. In case our Kafka application, it would run into a pods as well. We could use the AWS EKS service.

### Extras: Container image repository

I didn't include it into the design but our K8S cluster will need to access to the docker images through a repository that has access. We need to have an image for our Airflow application and another for our Kafka application. We could use the AWS ECR (Elastic Container Registry).

