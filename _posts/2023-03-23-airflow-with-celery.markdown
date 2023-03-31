---
layout: post
title:  "airflow 구축(with celery executor)"
date:   2023-03-23 21:03:36 +0530
categories: airflow Docker
---

![Untitled](/assets/posts/2023-03-23-airflow-with-celery/0.png){: .center}

# airflow 구축(with celery executor)

### version

- airflow version 2.5.1
- python 3.7.7
- postgresql

## 구성

- master node 1
- worker node 2

# 구축 과정

- airflow 공식 문서에서 제공하는 docker-compose.yml 파일

[Running Airflow in Docker - Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#)

```jsx
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.5.1/docker-compose.yaml'
```

docker-compose up을 하기전에 실행 경로에 `./dags ./logs ./plugins` 세개의 디렉토리를 생성한다.

만약 다른 경로에 만들고 싶다면 `AIRFLOW_PROJ_DIR` 를 수정해주면 된다.

데이터 마이그레이션을 실행하고 첫 번째 사용자 계정을 생성하기 위해 아래와 같은 명령어를 입력한다.

```jsx
docker compose up airflow-init
```

초기화가 끝난면  다음과 같은 메시지를 볼 수 있다.

![Untitled](/assets/posts/2023-03-23-airflow-with-celery/1.png)

이후 `docker-compose up` 을 하면 celery executor를 airflow 환경을 확인 할 수 있다.

> 만약 docker log를 보고 싶지 않다면 `-d` 옵션을 주면 된다.
> 

- airflow webui
    
    ![Untitled](/assets/posts/2023-03-23-airflow-with-celery/2.png)
    
    ![Untitled](/assets/posts/2023-03-23-airflow-with-celery/3.png)
    

flower를 실행하기 위해 `docker-compose up -d flower` 를 한다.

- flower
    
    ![Untitled](/assets/posts/2023-03-23-airflow-with-celery/4.png)
    

**만약 뭔가 꼬여서 처음부터 다시 하고 싶다면..!**

```jsx
docker-compose down --volumes --remove-orphans
```

추가로 airflow cluster를 구성해 다른 서버에 worker를 띄우는 방법은 다음과 같다.

먼저 fernet_key를 생성한다.

![Untitled](/assets/posts/2023-03-23-airflow-with-celery/5.png)

그리고 이 fernet_key를 docker-compose.yml의 `AIRFLOW__CORE__FERNET_KEY` 에 넣어준다.

그리고 docker-compose를 `--forece-recreate` 옵션을 사용해 다시 실행한다.

다음으로 worker node용 Dockerfile을 작성한다.

```jsx
FROM python:3.7.16-slim

# install airflow
ENV AIRFLOW_HOME="/root/airflow"
ENV AIRFLOW__WORKER__NAME="worker_node"

ENV AIRFLOW__MASTER__IP="HOST_IP"
ENV AIRFLOW__POSTGRESQL__HOST=${AIRFLOW__MASTER__IP}:5432
ENV AIRFLOW__REDIS__HOST=${AIRFLOW__MASTER__IP}:6379
ENV AIRFLOW__CORE__EXECUTOR="CeleryExecutor"
ENV AIRFLOW__DATABASE__SQL_ALCHEMY_CONN="postgresql+psycopg2://airflow:airflow@${AIRFLOW__POSTGRESQL__HOST}/airflow"
ENV AIRFLOW__CORE__SQL_ALCHEMY_CONN="postgresql+psycopg2://airflow:airflow@${AIRFLOW__POSTGRESQL__HOST}/airflow"
ENV AIRFLOW__CELERY__RESULT_BACKEND="db+postgresql://airflow:airflow@${AIRFLOW__POSTGRESQL__HOST}/airflow"
ENV AIRFLOW__CELERY__BROKER_URL="redis://:@${AIRFLOW__REDIS__HOST}/0"
ENV AIRFLOW__LOGGING__BASE_LOG_FOLDER="${AIRFLOW_HOME}/log"
ENV AIRFLOW__SCHEDULER__CHILD_PROCESS_LOG_DIRECTORY="${AIRFLOW_HOME}/log/scheduler"
ENV AIRFLOW__LOGGING__DAG_PROCESSOR_MANAGER_LOG_LOCATION="${AIRFLOW_HOME}/log/dag_processor_manager/dag_processor_manager.log"
ENV AIRFLOW__CORE__HOSTNAME_CALLABLE="airflow.utils.net.get_host_ip_address"
ENV AIRFLOW__CORE__FERNET_KEY=""
ENV AIRFLOW__WEBSERVER__SECRET_KEY=""
ENV AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION="true"
ENV AIRFLOW__CORE__LOAD_EXAMPLES="false"
ENV AIRFLOW__API__AUTH_BACKENDS="airflow.api.auth.backend.basic_auth"
ENV _pip_ADDITIONAL_REQUIREMENTS="${_pip_ADDITIONAL_REQUIREMENTS:-apache-airflow-providers-google}"

# pip update
RUN apt-get update && apt-get install -y python3-distutils python3-setuptools
RUN python3 -m pip install pip --upgrade pip
# RUN apt-get install python3.7-dev libmysqlclient-dev gcc -y
RUN python3 -m pip install cffi

# install airflow
RUN pip install apache-airflow[celery]==2.5.0
RUN pip install psycopg2-binary
RUN pip install Redis
RUN airflow db init
RUN mkdir -p ./airflow/dags
RUN mkdir ./airflow/plugins

# healthcheck
HEALTHCHECK --interval=10s --timeout=10s --retries=5 CMD airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"
```

`AIRFLOW__CORE__FERNET_KEY`에 앞서 발급한 fernet_key를 넣어준다.

Dockerfile을 빌드하고 실행을 하면

```jsx
docker build . worker:test
docker run -it worker:test airflow celery 
```

flower에서 worker node가 하나 더 생겼음을 확인할 수 있다.

![Untitled](/assets/posts/2023-03-23-airflow-with-celery/6.png)

worker node를 실행시키는 명령어 `-H` 와 `-q` 옵션을 통해 worker와 queue의 이름을 정할 수 있다. 

![Untitled](/assets/posts/2023-03-23-airflow-with-celery/7.png)

## 참고 문서
<https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html>
<https://jaeyung1001.tistory.com/286>

