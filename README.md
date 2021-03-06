# Real-time anomaly detection in Industrial IoT sensors
### Yacine Benzerga - Insight Data Engineering (New York 2020)

The goal of this project is to leverage IoT sensors to allow industrial companies the identification of malfunctioning assets by detecting usage anomalies in real time and enable predictive maintenance by detecting global usage anomalies in 24 hours window.


## Approach
To simulate an Industrial IoT environement. A kafka producer is streaming temperature recordings of 90 sensors at a rate of 27000 events/sec. the generated data follows a normal distribution for the assets functioning properly, and a combined normal, exponential distribution as noise for the malfunctioning assets.   

![approach](docs/frontend.png)


## Architecure

![architecture](docs/final_pipeline.jpeg)


***Streaming***: real-time sensor observations are streamed by Kafka into Spark Streaming, which handles two tasks: 

 1.Detect anomalies in each sensor every 5 seconds using an offline High-low pass filter algorithm and write the results to  anomaly_window_tbl in Timescale.
 
2.Preprocess & downsample generated data and writes to downsampled_table in Timescale.


***Batch***: downsampled data is then ingested every 24 hours using Airflow from Timescale to Spark, where global anomlies are detected using a python [implementation](https://github.com/nachonavarro/seasonal-esd-anomaly-detection) of Twitter hybrid Seasonal ESD, and results are saved back to Timescale in global_anomalies_table

## Instructions 

Pegasus is a tool that allows you to quickly deploy a number of distributed technologies.

Install and configure [AWS CLI](https://aws.amazon.com/cli/) and [Pegasus](https://github.com/InsightDataScience/pegasus) on your local machine, and clone this repository using
`git clone https://github.com/YacineBenzerga/IoT-real-time-anomaly-detection`.

#### SET UP CLUSTER:
- (4 nodes) Spark-Cluster, instance type: m4.large
- (3 nodes) Kafka-Cluster , instance type: m4.large
- (1 node) PostgreSQL(Timescale) , instance type: m4.large
- (1 node) Dash , instance type: t2.micro

-Follow the instructions in `docs/pegasus.txt` to create required clusters, install and start technologies

>Required technologies per cluster
-(Spark): aws, hadoop, spark
-(Kafka): aws, zookeeper, kafka

-Install Postgres with Timescale extension using this [link](https://docs.timescale.com/latest/getting-started/installation/ubuntu/installation-apt-ubuntu) then follow instructions in `docs/postgres.txt`

-Airflow scheduler can be installed on the master node of Spark-Cluster.Follow the instructions in `docs/airflow.txt` to install and launch the Airflow server.



## Running the platform

Log into EC2 instances containing the master nodes of spark-cluster and kafka-cluster using
the command `peg ssh cluster-name 1`

### Setting up Kafka
Run`peg ssh kafka-cluster 1`

-Create a kafka topic with 9 partitions and a replication factor of 2 using the following command:

`/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 2 --partitions 9 --topic temp_topic`

### Required jar files and dependancies
-Download spark-streaming-kafka-0-8_2.11:2.2.0 and postgresql-42.2.2.jar files to your spark-cluster master node

-The batch processing job in Spark depends on numpy, pandas and stldecompose libraries that have to be shared accross Spark workers.

-Run `/bash_scripts/dist_spark_dependancies.sh` to get the required (mypkg.zip) file that will be included in the spark-submit command

### Start streaming
-Message streaming

`peg ssh kafka-cluster 1`

`./bash_scripts/kafka_run.sh` 

-Message processing

`peg ssh spark-cluster 1`

`./bash_scripts/spark_streaming_run.sh` 

### Start batch processing
Airflow scheduler is installed on the master node of Spark-cluster:

`peg ssh spark-cluster 1`

`airflow/schedule.sh`

### Frontend-Dash
`peg ssh dash 1`

`nohup python ./frontend/app.py`
