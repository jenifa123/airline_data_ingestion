# Airline Data Ingestion Pipeline

## Overview

An end to end data pipeline which automates the ingestion of daily flights data into Redshift data warehouse using various AWS Services. AWS Services used in this project are:

1. AWS S3
2. AWS EventBridge
3. AWS Step Function
4. AWS Glue
5. AWS Redshift
6. AWS SNS


## Architecture Diagram:


## AWS S3 Bucket Setup:

Create a S3 bucket: airline-data-landing-zone
Add two new folders inside the bucket: 1. daily_flights (stores the flight data based on the hive style partitioning) 2. dim (stores the airport data as it is SCD)
Upload airports.csv file to the dim folder


## AWS Redshift Data Setup:

Create a Redshift cluster with proper username and password. Test the connection.
Redshift serves as a central warehouse to store the facts and the dimension tables.
Create the airlines schema, and then airports_dim dimension and  daily_flights_fact tables inside the airlines schema in redshift
Copy the airports.csv file from S3 bucket into airports_dim table


## AWS Glue Setup:

### Crawlers Configuration:

Create a JDBC connection to Redshift.
Make sure to create S3 Endpoint inside the VPC attached to the cluster so as to setup the connection between S3 and Redshift.
Add Inbound Rule for Redshift with a port of 5439 inside the security group attached to the Redshift cluster.
Attach the proper Role and test the connection.
Create a datamart named as airline_data_mart inside the databases in Glue


### Creating Crawlers

Create 3 crawlers:
1. airline_dim_crawler  : To crawl the dimension table in Redshift and load the metadata in dev_airlines_airports_dim in tables in GLUE
2. airline_fact_crawler : To crawl the fact table in Redshift and load the metadata in dev_airlines_daily_flights_fact in tables in GLUE
3. flights_data_crawler : To crawl the csv file present in S3 and load the data in raw_daily_flights in tables in GLUE



### Creating GLUE Job using Visual ETL:

1. Add a data source which will take data from raw_daily_flights table created from crawler in glue.
2. Add a Filter condition to filter out the departure delay >= 60 mins i.e to filter out the flights delayed by atleast 1 hr 
3. Add another data source which will take the dev_airlines_airports_dim as the source
4. Join the filtered data with the aiports_dim and select the columns which are required and update the schema as required.
5. Join the output from step 4 with aiports_dim to add the arrival airport details and update the schema as required.
6. Add target to load the data from all the above steps into Redshift dev_airlines_daily_flights_fact table
7. Add the job parameter named as --JOB_NAME as key and Visual ETL file name as value in the Glue 
8. Enable Job Bookmarking.

### Process Orchestration:

Orchestration is done as follows:

As soon as the file lands in S3 bucket, it will send a trigger to EventBridge pattern rule and it will in turn trigger the Step Function.
Step Function will trigger crawler,glue job and SNS.

Enable EventBridge Notification inside the S3 bucket.

##### Creation of State Machine inside Step Functions

1. Start the crawler as soon as the file lands in S3, pass the name of the crawler as {"Name": "flights_data_crawler"} in API Parameters .
2. Keep checking the status of the crawler as soon as the crawler starts using GetCrawler.
3. Add a choice state to check the status , similar to if else condition
    1. If the crawler is still running then it will go to wait state and wait for some seconds and then again check for the status in GetCrawler
    2. If the crawler is completed then start the Glue job using GlueStartJobRun by passing {"JobName":"flight_data_ingestion"}
4. Again add a choice state for GlueStartJobRun to check the status.
5. If successfully completed then trigger SNS to send a success message.
6. If failed then trigger SNS to send a failure message

### SNS Configuration:

1. Create a EventBridge Rule for S3 Event Notification
2. Add proper email address to get the notifications

## Conclusion:

The end to end pipeline ensures that the process is automated to handle the daya to dday flight data and uses various AWS Services for high processing, efficient storage, and notifications.
