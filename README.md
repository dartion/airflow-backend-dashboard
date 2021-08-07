# Airflow: Log Visualise Alert

Logging: AWS S3
Visualisation: Kibana, OR Grafana
Databases: PostgreSQL, InfluxDB,

## S3 Logging

1. Get the logging config template from Airflow Github, and load it in /mnt/airflow/conf/log_config.py


2. Update the airflow.cfg to have

`logging_config_class = log_config.DEFAULT_LOGGING_CONFIG`

3. Run Airflow

```./restart.sh```

4. Trigger the DAG logging_dag on the ui and check if logging is working after modifying log_config.py

`mnt/airflow/logs/logger_dag/t2/2021-08-06T00:00:00+00:00/1.log`

Airflow Logs ![Alt text](screenshots/screenshot1.png?raw=true "Airflow Branching of Tasks")

5. Once the logs are looking good, to store the logs onto AWS S3 bucket.

5.1. Create a S3 Bucket

5.2. Create a user and attach at least List, Read and Write custom Policy.

5.3. Add connection on

Airflow->UI->Admin->Connections->
Add-> (Conn Id: AWSS3LogStorage, Extras: {"aws_access_key_id": "<key_id>", "aws_secret_access_key": "<secret_key>"})

6. In airflow.cfg set,

`[core]`

`remote_logging = TRUE`

`remote_log_conn_id = AWSS3LogStorage`

`remote_base_log_folder = S3://dartion-airflow-logs/airflow-logs`

`encrypt_s3_logs = False`


7. Restart Airflow

`./restart.sh`

Once the DAGs are triggered, the airflow logs should be uploaded to AWS S3 bucket.

AWS S3 Logging ![Alt text](screenshots/screenshot2.png?raw=true "Logs uploaded to S3 bucket")


## ELK (Elasticsearch - LogStash - Kibana)

Architecture ![Alt text](screenshots/elk-architecture.png?raw=true "ELK Architecture")

_FileBeat reads from Airflow Logs_


1. In Airflow.cfg update
```
[core]
remote_logging = TRUE
remote_log_conn_id =
remote_base_log_folder =
encrypt_s3_logs = False```
```
```
[elasticsearch]
host = http://elasticsearch:9200
log_id_template = {dag_id}-{task_id}-{execution_date}-{try_number}
write_stdout = True
json_format = True
json_fields = asctime, filename, lineno, levelname, message
```

2. Check if http://localhost:9200 to confirm if Elastic search is working

Elasticsearch URL ![Alt text](screenshots/screenshot4.png?raw=true "Elasticsearch URL")

3. Check if http://localhost:5601/ to confirm Kibana is working

Kibana URL ![Alt text](screenshots/screenshot5.png?raw=true "Kibana URL")

4. Execute the following commands

- Exec into the worker container

```docker exec -it <Worker Container ID> /bin/bash```

- Run

```curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.5.2-linux-x86_64.tar.gz``` 

```tar xvzf filebeat-7.5.2-linux-x86_64.tar.gz ```

```vim filebeat.yml ```

- in filebeat.inputs  update

```enabled: true```

``` paths:
    - /usr/local/airflow/logs/*/*/*/*.log
```
- output.logstash:

```  # The Logstash hosts
  hosts: ["localhost:5044"]
```

``` #output.elasticsearch:
   # Array of hosts to connect to.
   #hosts: ["localhost:9200"]
```

- And start file beat with latest config

```./filebeat -e -c filebeat.yml -d "publish"```


- Trigger DAG on Airflow UI, and On Kibana

- Management->Index Management->Create Index Pattern
airflow-logs-*

- Click Next, Select @timestamp

- Click on Discover

Kibana Visualisation ![Alt text](screenshots/screenshot6.png?raw=true "Kibana Visualisation")

- Delete all Tasks Instances, Job from Airflow UI, and delete Indexes and Patterns from Kibana

- Start data_dag

- Create an Index pattern as above And Visualisation->Create Gauge Visualisation -> Customise Visualisation

- Metrics->Count
  field -> dag_id.keyword

The final gauges on Kibana looks like

Visualisation Gauges ![Alt text](screenshots/screenshot7.png?raw=true "Visualisation Gauges")

## Telegraph Influx DB and Grafana

Architecture

TIF Architecture ![Alt text](screenshots/tif-architecture.png?raw=true "TIF Architecture")

Telegraph- Opensource agent collects and process information and aggregate metrics
InfluxDB - A time series database to handle high query load (mainly for timestamp data)
Grafana - Data visualisation tool

1. Run the following commands
```docker-compose -f docker-compose-CeleryExecutorTIG.yml ```

```docker exec -it c119a26ca913 /bin/bash```

``` influx```

```create database telegraf```

```create user telegraf with password 'telegrafpass';```

2. Open http://localhost:3000, enter admin and admin for username and password.

- Add a datasource InfluxDB -> URL: http://influxdb:8086

```Database: telegraf
User: telegraf
Password: telegrafpass
```

3. Create a dashboard for number of dags

Setting up Gauges on Grafana ![Alt text](screenshots/tif-2.png?raw=true "Setting up Gauges on Grafana")

Gauges on Grafana ![Alt text](screenshots/tif-1.png?raw=true "Gauges on Grafana")

4. Setting up Alerts on Grafana

Note: Use smpt_username and smtp password (using Google here)

- Alterting-> Notification Channel -> Add Channel
 Name: email-channel, Type: Email, Address: dartion@gmail.com_

5. Set Email alerts from Grafana (sleep task t2 to 10 seconds to receive an alert)

Email alert from Grafana ![Alt text](screenshots/screenshot8.png?raw=true "Email alert from Grafana")





