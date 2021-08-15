<p align="center">
  <a href="https://dev.to/vumdao">
    <img alt="Airflow Quick Start With docker-compose on AWS EC2" src="https://github.com/vumdao/airflow-docker/blob/master/cover.jpg?raw=true" width="700" />
  </a>
</p>
<h1 align="center">
  <div><b>Airflow Quick Start With docker-compose on AWS EC2</b></div>
</h1>

## Abstract
For quick set up and start learning [Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/index.html), we will deploy airflow using docker-compose and running on AWS EC2

## Table Of Contents
 * [Introduction](#Introduction)
 * [Additional PIP requirements](#Additional-PIP-requirements)
 * [How to build customize airflow image](#How-to-build-customize-airflow-image)
 * [Persistent airflow log, dags, and plugins](#Persistent-airflow-log,-dags,-and-plugins)
 * [Using git-sync to up-to-date DAGs](#Using-git-sync-to-up-to-date-DAGs)
 * [How to run](#How-to-run)
 * [Add airflow connectors](#Add-airflow-connectors)
 * [Understand airflow parameters in airflow.models](#Understand-airflow-parameters-in-airflow.models)


---

## 🚀 **Introduction** <a name="Introduction"></a>
The docker-compose.yaml  contains several service definitions:
    - airflow-scheduler - The scheduler monitors all tasks and DAGs, then triggers the task instances once their dependencies are complete.
    - airflow-webserver - The webserver available at http://localhost:8080.
    - airflow-worker - The worker that executes the tasks given by the scheduler.
    - airflow-init - The initialization service.
    - flower - The flower app for monitoring the environment. It is available at http://localhost:5555.
    - postgres - The database.
    - redis - The redis - broker that forwards messages from scheduler to worker.

Some directories in the container are mounted, which means that their contents are synchronized between the services and persistent.
- ./dags - you can put your DAG files here.
- ./logs - contains logs from task execution and scheduler.
- ./plugins - you can put your custom plugins here.

## 🚀 **Additional PIP requirements** <a name="Additional-PIP-requirements"></a>
- airflow image contains almost enough PIP packages for operating, but we still need to install extra packages such as clickhouse-driver, pandahouse and apache-airflow-providers-slack.

- Airflow from 2.1.1 supports ENV _PIP_ADDITIONAL_REQUIREMENTS to add additional requirements when starting all containers

```
_PIP_ADDITIONAL_REQUIREMENTS: 'pandahouse==0.2.7 clickhouse-driver==0.2.1 apache-airflow-providers-slack'
```

- It's not recommended to use this way on production, we should build the own image which contain all necessary pip packages then push to AWS ECR

## 🚀 How to build customize airflow image <a name="How-to-build-customize-airflow-image"></a>
- Build own image.

- `requirements.txt`
```
pandahouse
clickhouse-driver
```

- `Dockerfile`
```
FROM apache/airflow:2.1.2-python3.9
COPY requirements.txt .
RUN pip install -r requirements.txt
```

- `docker build -t my-airlfow .`

## 🚀 Persistent airflow log, dags, and plugins <a name="Persistent-airflow-log,-dags,-and-plugins"></a>
Not only persistent the folders but also share these ones between scheduler, worker and web-server 
```
  volumes:
    - /mnt/airflow/dags:/opt/airflow/dags
    - /mnt/airflow/logs:/opt/airflow/logs
    - /mnt/airflow/plugins:/opt/airflow/plugins
    - /mnt/airflow/data:/opt/airflow/data
```

## 🚀 Using git-sync to up-to-date DAGs <a name="Using-git-sync-to-up-to-date-DAGs"></a>
- The git-sync service will polling the registered project each 10s to clone the new commit to /dags
- We use HTTP method and Access key token to provide permission for the container
```
  af-gitsync:
    container_name: af-gitsync
    image: k8s.gcr.io/git-sync/git-sync:v3.2.2
    environment:
      - GIT_SYNC_REV=HEAD
      - GIT_SYNC_DEPTH=1
      - GIT_SYNC_USERNAME=airflow
      - GIT_SYNC_MAX_FAILURES=0
      - GIT_KNOWN_HOSTS=false
      - GIT_SYNC_DEST=repo
      - GIT_SYNC_REPO=https://cloudopz.co/devops/airflow-dags.git
      - GIT_SYNC_WAIT=60
      - GIT_SYNC_TIMEOUT=120
      - GIT_SYNC_ADD_USER=true
      - GIT_SYNC_PASSWORD=
      - GIT_SYNC_ROOT=/dags
      - GIT_SYNC_BRANCH=master
    volumes:
      - /mnt/airflow/dags:/dags
```

## 🚀 How to run <a name="How-to-run"></a>
1. **Initializing Environment**
```
mkdir ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_GID=0" > .env
```

2. **Prepare `docker-compose.yaml`**
<details>
  <summary>docker-compose.yaml</summary>

```
version: '3.5'
x-airflow-common:
  &airflow-common
  image: apache/airflow:2.1.2-python3.9
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@af-pg/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@af-pg/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@af-redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'true'
    AIRFLOW__API__AUTH_BACKEND: 'airflow.api.auth.backend.basic_auth'
    AIRFLOW_CONN_RDB_CONN: 'postgresql://dbapplication_user:dbapplication_user@rdb:5432/postgres'
    _PIP_ADDITIONAL_REQUIREMENTS: 'pandahouse==0.2.7 clickhouse-driver==0.2.1 apache-airflow-providers-slack'
  volumes:
    - /mnt/airflow/dags:/opt/airflow/dags
    - /mnt/airflow/logs:/opt/airflow/logs
    - /mnt/airflow/plugins:/opt/airflow/plugins
    - /mnt/airflow/data:/opt/airflow/data
  user: "${AIRFLOW_UID:-50000}:${AIRFLOW_GID:-50000}"
  depends_on:
    - af-redis
    - af-pg

services:
  af-pg:
    image: postgres:13
    container_name: af-pg
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 5s
      retries: 5
    restart: always

  af-redis:
    container_name: af-redis
    image: redis:latest
    ports:
      - 6379:6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 30s
      retries: 50
    restart: always

  af-websrv:
    container_name: af-websrv
    <<: *airflow-common
    command: webserver
    ports:
      - 28080:8080
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always

  af-sch:
    container_name: af-sch
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"']
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always

  af-w:
    container_name: af-w
    <<: *airflow-common
    command: celery worker
    healthcheck:
      test:
        - "CMD-SHELL"
        - 'celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always

  af-int:
    container_name: af-int
    <<: *airflow-common
    command: version
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_UPGRADE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'

  af-flower:
    container_name: af-flower
    <<: *airflow-common
    command: celery flower
    ports:
      - 5555:5555
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always
 
  af-gitsync:
    container_name: af-gitsync
    image: k8s.gcr.io/git-sync/git-sync:v3.2.2
    environment:
      - GIT_SYNC_REV=HEAD
      - GIT_SYNC_DEPTH=1
      - GIT_SYNC_USERNAME=airflow
      - GIT_SYNC_MAX_FAILURES=0
      - GIT_KNOWN_HOSTS=false
      - GIT_SYNC_DEST=repo
      - GIT_SYNC_REPO=https://cloudopz.co/devops/airflow-dags.git
      - GIT_SYNC_WAIT=60
      - GIT_SYNC_TIMEOUT=120
      - GIT_SYNC_ADD_USER=true
      - GIT_SYNC_PASSWORD=
      - GIT_SYNC_ROOT=/dags
      - GIT_SYNC_BRANCH=master
    volumes:
      - /mnt/airflow/dags:/dags

volumes:
  postgres-db-volume:

```

</details>

3. **Run airflow-init to setup airflow database**
```
docker-compose up airflow-init
```

- Running airflow - Up

```
docker-compose up -d
```

- Read more [Extension fields](https://docs.docker.com/compose/compose-file/compose-file-v3/#extension-fields) to understand docker-compose.yaml contents

## 🚀 Add airflow connectors <a name="Add-airflow-connectors"></a>
- Add slack connection
```
airflow connections add 'airflow-slack' \
    --conn-type 'http'
    --conn-password '/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX' \
    --conn-host 'https://hooks.slack.com/services'
```

- We can add connector in UI
<p align="left">
  <a href="https://dev.to/vumdao">
    <img alt="Airflow Quick Start With docker-compose on AWS EC2" src="https://github.com/vumdao/airflow-docker/blob/master/slack-connection.png?raw=true" width="800" />
  </a>
</p>

- We can add connection through env file eg. `.env` in docker-compose
```
AIRFLOW_CONN_MY_POSTGRES_CONN: 'postgresql://airflow:airflow@postgres-pg:5432/airflow'
AIRFLOW_CONN_AIRFLOW_SLACK_CONN: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
```

## 🚀 Understand airflow parameters in airflow.models <a name="Understand-airflow-parameters-in-airflow.models"></a>
- [airflow.models](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/index.html)

- `Context`
    - `on_failure_callback` (TaskStateChangeCallback) – a function to be called when a task instance of this task fails. a context dictionary is passed as a single parameter to this function. Context contains references to related objects to the task instance and is documented under the macros section of the API.
    - `on_execute_callback` (TaskStateChangeCallback) – much like the on_failure_callback except that it is executed right before the task is executed.
    - `on_retry_callback` (TaskStateChangeCallback) – much like the on_failure_callback except that it is executed when retries occur.
    - `on_success_callback` (TaskStateChangeCallback) – much like the on_failure_callback except that it is executed when the task succeeds.
    - `trigger_rule` (str) – defines the rule by which dependencies are applied for the task to get triggered. Options are: { all_success | all_failed | all_done | one_success | one_failed | none_failed | none_failed_or_skipped | none_skipped | dummy} default is all_success. Options can be set as string or using the constants defined in the static class airflow.utils.TriggerRule

---

<h3 align="center">
  <a href="https://dev.to/vumdao">:stars: Blog</a>
  <span> · </span>
  <a href="https://github.com/vumdao/aws-eks-the-hard-way">Github</a>
  <span> · </span>
  <a href="https://stackoverflow.com/users/11430272/vumdao">stackoverflow</a>
  <span> · </span>
  <a href="https://www.linkedin.com/in/vu-dao-9280ab43/">Linkedin</a>
  <span> · </span>
  <a href="https://www.linkedin.com/groups/12488649/">Group</a>
  <span> · </span>
  <a href="https://www.facebook.com/CloudOpz-104917804863956">Page</a>
  <span> · </span>
  <a href="https://twitter.com/VuDao81124667">Twitter :stars:</a>
</h3>
