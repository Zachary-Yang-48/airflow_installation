# Airflow Installation

## Docker
Easy way to install docker
```
#uninstall all possible docker version before new installment
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

sudo yum install -y docker
```

Validation
```
sudo service docker start
sudo docker run hello-world
```

## Install postgreSQL on docker
```
#pull postgresql docker image
docker pull postgres

#run postgresql container
docker run --name docker-postgresql -e POSTGRES_PASSWORD=Tigerat437! -d postgres

#optional: assign specific dir for data storage
#docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -v my_local_data_directory:/var/lib/postgresql/data -d postgres

#verify if container is running
docker ps

#connect to postgresql
docker exec -it docker-postgresql psql -U postgres

#NOTE: Do create user and db and execute the following code:
docker exec -it docker-postgresql psql -U script_master -d airflow

# connect to airflow postgres
sudo docker exec -it airflow-postgres-1 psql -U script_master -d airflow 
```

## Install docker compose v2
```
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
```

Validation
```
docker compose version
```

## Install Airflow
<https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html>

### docker-compose.yml
```
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

# Basic Airflow cluster configuration for CeleryExecutor with Redis and PostgreSQL.
#
# WARNING: This configuration is for local development. Do not use it in a production deployment.
#
# This configuration supports basic cfonfiguration using environment variables or an .env file
# The following variables are supported:
#
# AIRFLOW_IMAGE_NAME           - Docker image name used to run Airflow.
#                                Default: apache/airflow:2.8.3
# AIRFLOW_UID                  - User ID in Airflow containers
#                                Default: 50000
# AIRFLOW_PROJ_DIR             - Base path to which all the files will be volumed.
#                                Default: .
# Those configurations are useful mostly in case of standalone testing/running Airflow in test/try-out mode
#
# _AIRFLOW_WWW_USER_USERNAME   - Username for the administrator account (if requested).
#                                Default: airflow
# _AIRFLOW_WWW_USER_PASSWORD   - Password for the administrator account (if requested).
#                                Default: airflow
# _PIP_ADDITIONAL_REQUIREMENTS - Additional PIP requirements to add when starting all containers.
#                                Use this option ONLY for quick checks. Installing requirements at container
#                                startup is done EVERY TIME the service is started.
#                                A better way is to build a custom image or extend the official image
#                                as described in https://airflow.apache.org/docs/docker-stack/build.html.
#                                Default: ''
#
# Feel free to modify this file to suit your needs.
---
x-airflow-common:
  &airflow-common
  # In order to add custom dependencies or upgrade provider packages you can use your extended image.
  # Comment the image line, place your Dockerfile in the directory where you placed the docker-compose.yaml
  # and uncomment the "build" line below, Then run `docker-compose build` to build the images.
  image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.8.3-python3.10}
  # build: .
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://aa:bb@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://aa:bb@postgres/airflow
    # AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
    # yamllint disable rule:line-length
    # Use simple http server on scheduler for health checks
    # See https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/check-health.html#scheduler-health-check-server
    # yamllint enable rule:line-length
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    # WARNING: Use _PIP_ADDITIONAL_REQUIREMENTS option ONLY for a quick checks
    # for other purpose (development, test and especially production usage) build/extend Airflow image.
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
  volumes:
    # - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - /opt/airflow/scripts/python_ec2/dags:/opt/airflow/dags
    - /opt/airflow/task_logs:/opt/airflow/task_logs
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
    - /opt/airflow/scripts:/opt/airflow/scripts
    - /home/admin/.aws:/home/airflow/.aws
    - /home/admin/.aws:/root/.aws
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    &airflow-common-depends-on
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: script_master
      POSTGRES_PASSWORD: Tigerat437!
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "script_master"]
      interval: 10s
      retries: 5
      start_period: 5s
    restart: always

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "11100:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    build:
      context: .
      dockerfile: Dockerfile
    image: python_dependencies:latest
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    image: python_dependencies:latest
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    image: python_dependencies:latest
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    # yamllint disable rule:line-length
    command:
      - -c
      - |
        if [[ -z "${AIRFLOW_UID}" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
          echo "If you are on Linux, you SHOULD follow the instructions below to set "
          echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
          echo "For other operating systems you can get rid of the warning with manually created .env file:"
          echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
          echo
        fi
        one_meg=1048576
        mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
        cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
        disk_available=$$(df / | tail -1 | awk '{print $$4}')
        warning_resources="false"
        if (( mem_available < 4000 )) ; then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
          echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
          echo
          warning_resources="true"
        fi
        if (( cpus_available < 2 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
          echo "At least 2 CPUs recommended. You have $${cpus_available}"
          echo
          warning_resources="true"
        fi
        if (( disk_available < one_meg * 10 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
          echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
          echo
          warning_resources="true"
        fi
        if [[ $${warning_resources} == "true" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
          echo "Please follow the instructions to increase amount of resources available:"
          echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
          echo
        fi
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    # yamllint enable rule:line-length
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
      _PIP_ADDITIONAL_REQUIREMENTS: ''
    user: "0:0"
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}:/sources

  airflow-cli:
    <<: *airflow-common
    profiles:
      - debug
    environment:
      <<: *airflow-common-env
      CONNECTION_CHECK_MAX_COUNT: "0"
    # Workaround for entrypoint issue. See: https://github.com/apache/airflow/issues/16252
    command:
      - bash
      - -c
      - airflow

  # You can enable flower by adding "--profile flower" option e.g. docker-compose --profile flower up
  # or by explicitly targeted on the command line e.g. docker-compose up flower.
  # See: https://docs.docker.com/compose/profiles/
  flower:
    <<: *airflow-common
    command: celery flower
    profiles:
      - flower
    ports:
      - "5555:5555"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

volumes:
  postgres-db-volume:
```

### Docker File
<https://airflow.apache.org/docs/docker-stack/build.html#adding-packages-from-requirements-txt>
Customize python environment, pre-set permissions, prerequisite applications etc.
```
sudo nano python_dependencies
```

```
# Use the official Airflow base image
FROM apache/airflow:2.8.3-python3.10
ENV AIRFLOW__CORE__LOAD_EXAMPLES=True
# Set the working directory in the container
WORKDIR /opt/airflow/

# Install any needed packages specified in requirements.txt
COPY requirements.txt /opt/airflow/
RUN pip install --no-cache-dir -r requirements.txt
RUN chown airflow /home/airflow/.aws/credentials
RUN chown airflow /home/airflow/.aws/config
RUN chmod -R 777 /opt/airflow/task_logs/
RUN chown airflow /opt/airflow/task_logs
```

Then build the image before install/restart airflow service
```
sudo docker build -t python_dependencies:latest -f python_dependencies .
```

In order to get rid of warning, create a .env file in the same dir as docker-compose.yaml file
```
AIRFLOW_UID=50000
```

Initialize db
```
docker compose up airflow-init
```

Run airflow
```
docker compose up
```

Create airflow user
```
#check container name and id
docker ps

docker exec -it <container_name_or_id> airflow users create \
    --username <username> \
    --firstname <first name> \
    --lastname <last name> \
    --role <role> \
    --email <email>
    --password your_password
    
# Valid roles are: [Admin, Viewer, User, Op, Public]
```

## Daemon
### Direct install on server
#### .service file config
Save .service under '/etc/systemd/system/'
```
[Unit]
Description=Airflow webserver daemon
After=network.target

[Service]
Environment="AIRFLOW_HOME=~/airflow"
User=script.master
# Group=TradeUP_BA
Type=simple
ExecStart=/home/script.master/pyenv/PROD-Legacy/bin/airflow webserver -p 8080
Restart=always
RestartSec=5s
KillMode=process

[Install]
WantedBy=multi-user.target
```

### Install on docker
#### .service file config
```
[Unit]
Description=Apache Airflow with Docker Compose
Requires=docker.service
After=docker.service

[Service]
User=root
WorkingDirectory=/opt/airflow
ExecStart=/usr/bin/docker compose up
ExecStop=/usr/bin/docker compose down
Restart=always

[Install]
WantedBy=multi-user.target
```

Once .service file is configured, reload daemon
```
sudo systemctl daemon-reload
```

### Start, stop, restart service, and check service status
```
sudo systemctl start airflow-webserver.service
sudo systemctl stop airflow-webserver.service
sudo systemctl restart airflow-webserver.service

sudo systemctl status airflow-webserver.service
```

### Start on boot/reboot, or disable
```
sudo systemctl enable airflow-webserver.service
sudo systemctl disable airflow-webserver.service
```

## Operation
### Mount
```
docker run -d -p 8080:8080 \
    -v /path/to/your/dags/on/host:/opt/airflow/dags \
    --name airflow your-airflow-image

--docker image name can be found by means of sudo docker ls
```

### Enter container
```
sudo docker exec -it airflow-airflow-scheduler-1 bash
```
