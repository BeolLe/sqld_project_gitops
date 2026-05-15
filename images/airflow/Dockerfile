FROM apache/airflow:3.1.7-python3.12

RUN pip install --no-cache-dir \
    "apache-airflow==${AIRFLOW_VERSION}" \
    oracledb \
    requests \
    "dbt-postgres==1.10.0"
