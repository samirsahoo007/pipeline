
![airflow](airflowlogo.png)

# Yet Another Scalable Apache Airflow With Docker Example Setup
This repository contains the code for article https://medium.com/@tomaszdudek/yet-another-scalable-apache-airflow-with-docker-example-setup-84775af5c451. It features a scaffold of Apache Airflow, whose workers instead of executing the job themselves, run Docker containers and wait until they complete.

# Requirements
* Linux or MacOS
* Docker and docker-compose
* Python 3.6 (with `docker-py` installed)

# How to run example
* clone the project
* go to `./airflow/` and run `docker build . -t puckel-airflow-with-docker-inside:latest`
* make sure you have [docker-py](https://github.com/docker/docker-py) (tested with version `docker==3.7.0` - **careful because using `docker-py==1.10.x` won't work**) and run `python build_images.py`
* go to `./airflow` and run `docker-compose up -f docker-compose-linux.yml` (or just `up` when using OSX)
* launch http://localhost:8080/admin/, ensure that DAG is ON:

![on](on.png)

* trigger the DAG:

![trigger](trigger.png)

* watch the pipeline as it launches Docker containers:

![pip](result.png)

# Modify
* feel free to make PR's
