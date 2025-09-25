# Простой DataLake: Trino + S3 Minio + Spark + Iceberg


### Поднятие инфраструктуры

```bash
docker compose up -d
```

```bash
docker-compose down && docker-compose build && docker-compose up -d  
```

```bash
docker-compose up -d --force-recreate trino
```

### Подключение к Minio

Параметры подключения стандартные:

- `login`: `minioadmin`
- `password`: `minioadmin`


```bash
docker exec -it trino trino --server localhost:8080 --user test     
```

### Ссылки

- [Инфраструктура для Data-Engineer Apache Iceberg](https://habr.com/ru/articles/850674/)
- [Unable to create and push iceberg tables to S3](https://stackoverflow.com/questions/79162751/unable-to-create-and-push-iceberg-tables-to-s3)
- [CREATE TABLE fails with ICEBERG_FILESYSTEM_ERROR using Iceberg, Postgres, and MinIO #21404](https://github.com/trinodb/trino/issues/21404)
- [Apache Iceberg Quickstart](https://iceberg.apache.org/spark-quickstart/)

### Gists

Рабочая SparkSession:

```python
from pyspark.sql import SparkSession

# --- ШАГ 1: Принудительно останавливаем любую существующую сессию ---
try:
    SparkSession.builder.getOrCreate().stop()
    print("Существующая Spark-сессия остановлена.")
except Exception as e:
    print(f"Не было активной сессии для остановки, что хорошо: {e}")


# --- ШАГ 2: Создаем новую сессию с исправленной конфигурацией чтения ---

print("Создание новой, правильно сконфигурированной Spark-сессии...")

spark = (
    SparkSession.builder
    .appName("Iceberg-Read-Write-Final")
    
    # === КЛЮЧЕВОЕ ИЗМЕНЕНИЕ: Настройка чтения из S3 ===
    # Говорим Spark, что для адресов s3:// нужно использовать драйвер s3a
    .config("spark.hadoop.fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    # Отключаем кеширование файловой системы, чтобы гарантировать применение настроек
    .config("spark.hadoop.fs.s3.impl.disable.cache", "true")
    
    # --- Остальные настройки из предыдущей рабочей версии ---
    
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    
    .config("spark.sql.catalog.datalake", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.datalake.catalog-impl", "org.apache.iceberg.rest.RESTCatalog")
    .config("spark.sql.catalog.datalake.uri", "http://rest:8181")
    
    .config("spark.sql.catalog.datalake.io-impl", "org.apache.iceberg.aws.s3.S3FileIO")
    .config("spark.sql.catalog.datalake.s3.endpoint", "http://minio:9000")
    .config("spark.sql.catalog.datalake.s3.path-style-access", "true")

    .config("spark.hadoop.fs.s3a.endpoint", "http://minio:9000")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")
    .config("spark.hadoop.fs.s3a.access.key", "minioadmin")
    .config("spark.hadoop.fs.s3a.secret.key", "minioadmin")
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    
    .getOrCreate()
)

print("✅ Spark-сессия успешно создана и готова к чтению и записи!")
spark
```

PySpark DML:

```python
# Записываем данные, используя полное имя: каталог.схема.таблица
df.writeTo("datalake.nyc.taxis").createOrReplace()
```