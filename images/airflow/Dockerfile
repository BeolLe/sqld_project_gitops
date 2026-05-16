FROM apache/airflow:3.1.7-python3.12

RUN python -m pip uninstall -y dbt-core dbt-postgres || true && \
    env -u PIP_CONSTRAINT python -m pip install --no-cache-dir --upgrade --force-reinstall \
        oracledb \
        requests \
        "dbt-core==1.10.0" \
        "dbt-postgres==1.10.0"
