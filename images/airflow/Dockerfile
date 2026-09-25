# Preserve the verified running Airflow/dbt stack; extend its immutable image.
FROM ghcr.io/beolle/sqld-airflow@sha256:8d78773a4c30e2cddf36b66b8f54978314ac6a9586094201a46bffcd0f63ebbe

RUN env -u PIP_CONSTRAINT python -m pip install --no-cache-dir --no-deps \
        "pyarrow==18.1.0" \
        "duckdb==1.4.3" && \
    python -c 'import sys, duckdb, pyarrow; assert sys.version_info[:2] == (3, 12); assert duckdb.__version__ == "1.4.3"; assert pyarrow.__version__ == "18.1.0"; assert duckdb.sql("select 1").fetchone() == (1,)'
