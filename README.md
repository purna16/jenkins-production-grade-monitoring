# Monitoring Production grade Jenkins using Prometheus, Grafana & InfluxDB

### Introduction

- This document outlines the comprehensive process of establishing and maintaining effective monitoring for a production-grade Jenkins environment utilizing Prometheus, Grafana, and InfluxDB. The primary objective is to ensure the robust performance and health of the Jenkins environment by tracking various critical metrics and parameters.

### **Monitoring Dashboard:**

To monitor the Jenkins environment effectively, a Grafana dashboard is designed with multiple panels, each serving a distinct monitoring purpose

1. **Jenkins Health Check Panel:** Monitors the overall health status of Jenkins.
2. **Node Count Panel:** Displays the number of active Jenkins nodes.
3. **Build Queue and Executor Count Panel:** Provides insights into the build queue and executor count.
4. **Pipeline Statistics Panel:** Presents a pie chart showing statistics on successful, failed, aborted, and unstable jobs.
5. **Number of Pipelines Ran Panel:** Tracks the total number of pipelines executed.
6. **Total Number of Builds Panel:** Monitors the cumulative count of Jenkins builds.
7. **Average Build Time Panel:** Displays the average build execution time.
8. **Latest Build and Build Details Panel:** Shows details of the most recent build.
9. **Plugins Panel:** Offers insights into the number of active, inactive, and outdated Jenkins plugins.

**Prerequisites:**

Before setting up the monitoring environment, ensure that the following prerequisites are met:

- Docker
- Jenkins
- Prometheus
- Grafana
- InfluxDB
- SQL queries

Once Docker is installed we bring up jenkins, prometheus, grafana and influxdb containers

**Setting Up the Environment:**

### **Jenkins:** Start Jenkins in a Docker container:

```bash
docker run -d -p 8080:8080 jenkins/jenkins:lts-jdk11
```
![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2010-43-35.png)

Install the required Jenkins plugins.
![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/jenkinsplugins.png)
### Configuring Prometheus

**Prometheus:** Create a Prometheus configuration file (prometheus.yml) and run Prometheus in a Docker container:

```bash
docker run -d --name prometheus-container -v /home/purna/prometheus.yml:/etc/prometheus/prometheus.yml -e TZ=UTC -p 9090:9090 ubuntu/prometheus:2.33-22.04_beta
```

### Configuring Grafana

```bash
docker run -d --name=grafana -p 3000:3000 grafana/grafana:8.5.5
```

### Configuring InfluxDB:Start InfluxDB in a Docker container

```bash
docker run -d -p 8086:8086 --name influxdb2 influxdb:1.8.6-alpine
```

- To accept the data from Jenkins to InfluxDB we need to do some pre-requisite
- we need to login into to influx db container and create database

```bash
docker exec -it <container-id> /bin/bash
influx
CREATE DATABASE "jenkins" WITH DURATION 1825d REPLICATION 1 NAME "jenkins-retention"
SHOW DATABASES
use jenkins
```
![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2011-41-19.png)
**Jenkins Plugin Installation:**

- Install two InfluxDB plugins in Jenkins—one for storing pipeline information and the other for stage and folder information.
- Additionally, install the Prometheus-related plugin and restart Jenkins.

**Configure InfluxDB in Jenkins:**

- In Jenkins, navigate to the "Configure System" settings.
- Under InfluxDB configuration, specify the InfluxDB IP address and port (8086), database name ("jenkins"), and retention policy ("jenkins-retention").

**Configure AutoStatus in Jenkins:**

- Configure AutoStatus in Jenkins to collect data on how long each pipeline execution takes. Select InfluxDB as the data source.

**Jenkins Configuration in Prometheus:**

- Edit the Prometheus configuration file (prometheus.yml).
- Inside the YAML file, define the job name and metrics path for Jenkins.

![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2015-41-55.png)

**Testing the Configuration:**

- Create two folders in Jenkins and run sample jobs within them.
- If InfluxDB is successfully connected and sending data, you should see the following lines at the end of each pipeline execution:

```bash
[InfluxDB Plugin] Collecting data...
[InfluxDB plugin] Metrics plugin data found. Writing to InfluxDB...
[InfluxDB Plugin] Publishing data to target 'influx' (url='http://ipaddress:8086', database='jenkins')
[InfluxDB Plugin] Completed.
Finished: SUCCESS
```

**Configuring Grafana Datasource:**

- In the settings section of Grafana, it is necessary to configure Prometheus and InfluxDB as datasources.

![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2015-47-33.png)
### Creating Grafana Dashboard:

**Jenkins Status Panel:**

- The first panel displays whether Jenkins is running or stopped.
- Query: **`up{instance="localhost:8080", job="jenkins"}`**
- Value mapping is applied to display "UP" when the instance is one (indicating Jenkins is running) and "DOWN" when the value is 0 (indicating Jenkins is stopped).

**Node Executor Count Panel:**

- This panel tracks the number of concurrent jobs Jenkins can run.
- Query: **`jenkins_executor_count_value{instance="localhost:8080", job="jenkins"}`**

![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2017-48-23.png)

**Node Count Panel:**

- Monitors the total number of Jenkins nodes.
- Query: **`jenkins_node_count_value{instance="localhost:8080", job="jenkins"}`**

**Queue Size Panel:**

- Displays the size of the build queue.
- Query: **`jenkins_queue_size_value{instance="localhost:8080", job="jenkins"}`**

![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2018-08-44.png)

**Folder Structure and Variables:**

- To make the dashboard responsive to folder selection, folders similar to those created in Jenkins should be set up in Grafana dashboard settings and select variables.
- The query to obtain folder details is: **`SHOW TAG VALUES FROM job WITH KEY = "owner"`**
- Folder names correspond to the "owner" tag in the InfluxDB database.

**Displaying Jobs for Selected Folder:**

- To display jobs under a specified folder, a query is used to make the panel reactive to folder selection.
- Query Example: **`SHOW TAG VALUES FROM job WITH KEY = repo WHERE "owner" =~ /^($folder)$/`**

![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2018-19-48.png)

**Build Statistics Panel:**

- This panel provides counts for successful, failed, unstable, and aborted builds.
- Queries:
    - Successful Build Count: **`SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'SUCCESS' OR "build_result" = 'CompletedSuccess' ) AND $timeFilter`**
    - Failed Build Count: **`SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'FAILURE' OR "build_result" = 'CompletedError' ) AND $timeFilter`**
    - Aborted Build Count: **`SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'ABORTED' OR "build_result" = 'Aborted' ) AND $timeFilter`**
    - Unstable Build Count: **`SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'UNSTABLE' OR "build_result" = 'Unstable' ) AND $timeFilter`**
    

**Pipeline Statistics Panel:**

- Pie charts are utilized to visualize build statistics.
- Legends are configured in the table field, and appropriate colors are selected.

**Number of Pipelines Ran Panel:**

- Counts the distinct project names to monitor the number of pipelines executed.
- Query: **`select count(DISTINCT project_name) FROM jenkins_data WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter`**

**Total Number of Builds Panel:**

- Monitors the total number of Jenkins builds.
- Query: **`SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter`**

**Average Build Time Panel:**

- Displays the average build execution time.
- Query: **`select build_time/1000 FROM jenkins_data WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter`**

**Latest Build Status Panel:**

- Shows the status of the latest build.
- Query: **`SELECT build_result FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter ORDER BY time DESC LIMIT 1`**

![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2018-20-43.png)

**Build Details Panel:**

- Displays details such as project path, build causer, build number, build time, and build result.
- Query: **`SELECT "build_exec_time","project_path","build_number","build_causer","build_time","build_result" FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter`**
- Panel customization includes hiding the time table, modifying field names, date format adjustment, and linking pipeline paths.
- I hide the table which indicates time using override property and changed names using field names to standard options display name using overrides property
- under standard options changed date format using unit date iso
- and in pipeline path i included links using data links option
- i added colors to success, failure, aborted and unstable using overrides and value mapping options

  ![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2019-56-42.png)

**Plugin Status Panel:**

- Provides information on active, inactive, and update-required Jenkins plugins.
- Queries:
    - Active Plugins: **`jenkins_plugins_active{instance="localhost:8080", job="jenkins"}`**
    - Inactive Plugins: **`jenkins_plugins_inactive{instance="localhost:8080", job="jenkins"}`**
    - Update Required Plugins: **`jenkins_plugins_withUpdate{instance="localhost:8080", job="jenkins"}`**

 ![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2020-07-37.png)



**Project Completion:**

- With the completion of the aforementioned steps, the Grafana dashboard is ready for use, providing comprehensive insights into the Jenkins environment.

  ![Image Alt Text](https://github.com/purna16/jenkins-production-grade-monitoring/blob/main/images/Screenshot%20from%202023-09-04%2022-25-20.png)


If you are interested, Notion link: https://bittersweet-gondola-35c.notion.site/Monitoring-Production-grade-Jenkins-using-Prometheus-Grafana-InfluxDB-3b96be3f771b41c4a265bbb4aee907d1?pvs=4
