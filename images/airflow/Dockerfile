FROM apache/airflow:3.1.7-python3.12

RUN PIP_CONSTRAINT="" pip install --no-cache-dir \
    oracledb \
    requests \
    "dbt-postgres==1.10.0"
