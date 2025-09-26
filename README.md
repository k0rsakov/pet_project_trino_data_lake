# Простой DataLake: Trino + S3 Minio + Spark + Iceberg

## О видео

🚀 В этом [видео](https://youtu.be/-fAwvsbSZh0) ты увидишь, как построить настоящий Data Lake с нуля и разберёшься, зачем дата-инженеру Iceberg, Trino,
MinIO, Spark и PostgreSQL!
Показываю всё на живом проекте: подключим аналитику, устроим хранение в S3, заведём метастор, научимся писать и читать
данные через SQL и PySpark.

Ссылки:

- Менторство/консультации по IT – https://korsak0v.notion.site/Data-Engineer-185c62fdf79345eb9da9928356884ea0
- TG канал – https://t.me/DataLikeQWERTY
- Instagram – https://www.instagram.com/i__korsakov/
- Habr – https://habr.com/ru/users/k0rsakov/publications/articles/
- GitHub проекта – https://github.com/k0rsakov/pet_project_trino_data_lake
- Инфраструктура для Data-Engineer Apache Iceberg – https://habr.com/ru/articles/850674/

🔻 Что тебя ждёт:

- Что такое Data Lake и зачем он нужен в 2025 (простыми словами, на пальцах!)
- Чем Data Lake отличается от классического DWH
- Какие задачи решает связка Trino + Iceberg + S3 + Spark + PostgreSQL
- Как выглядит инфраструктура современного дата-инженера (и как всё это быстро поднять у себя)
- Как Trino читает данные из разных источников
- Как создавать таблицы через SQL и видеть их в S3
- Как работает метастор на PostgreSQL и зачем он нужен
- Как наполнять Data Lake внешними данными через Apache Spark
- Практика: запросы, схемы, создание таблиц, чтение через Spark и Trino
- Советы и лайфхаки по работе с Data Lake

Таймкоды:

- 00:00 – Начало
- 00:23 – Что такое Data Lake
- 02:17 – Разбор инфраструктуры
- 04:51 – Настраиваем подключение к Data Lake
- 05:51 – Настраиваем подключение к OLTP
- 08:29 – Первая запись в Data Lake Iceberg через Trino
- 13:29 – Запись данных в Data Lake Iceberg Через Spark (PySpark)
- 16:43 – Чтение данных из Data Lake Iceberg через Trino
- 17:03 – Чтение данных из Data Lake Iceberg через Spark (PySpark)
- 17:22 – Итог

#DataLake #Trino #Iceberg #S3 #MinIO #Spark #PostgreSQL #DataEngineering #BigData #ETL #SQL

🔥 Не забудь поставить лайк, подписаться на канал и включить колокольчик, чтобы не пропустить новые видео!

## О проекте

### Поднятие инфраструктуры

Для запуска инфраструктуры:

```bash
docker compose up -d
```

Для перезапуска инфраструктуры, если что-то пошло не так:

```bash
docker-compose down && docker-compose build && docker-compose up -d  
```

Для перезапуска `trino` чтобы добавить новые коннекты или изменить существующие:

```bash
docker-compose up -d --force-recreate trino
```

### Подключение к Minio

Параметры подключения стандартные:

- `login`: `minioadmin`
- `password`: `minioadmin`

### Подключение к Trino

Проверка работоспобности Trino:

```bash
docker exec -it trino trino --server localhost:8080 --user test     
```

### Ссылки

- [Инфраструктура для Data-Engineer Apache Iceberg](https://habr.com/ru/articles/850674/)
- [Unable to create and push iceberg tables to S3](https://stackoverflow.com/questions/79162751/unable-to-create-and-push-iceberg-tables-to-s3)
- [CREATE TABLE fails with ICEBERG_FILESYSTEM_ERROR using Iceberg, Postgres, and MinIO #21404](https://github.com/trinodb/trino/issues/21404)
- [Apache Iceberg Quickstart](https://iceberg.apache.org/spark-quickstart/)
- [Trino Connectors](https://trino.io/docs/current/connector.html)

### Gists

### Рабочая SparkSession:

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

#### Trino authorization

При работе с Trino могут возникнуть проблемы с подключением через DBevaer.

Есть несколько вариантов как это можно исправить.

##### Изменение `Driver Settings` в DBeaver

1) Создаём подключение к Trino
    - `host`: `localhost`
    - `port`: `8080`
2) В `Driver Settings` добавляем в `Settings`:
    - `user`: `admin` (или любой другой, для Trino без настройки авторизации – не важно)
    - Ставим флаг `No authentication`
    - Ставим флаг `Allow Empty Password`

##### Изменение `Driver Properties` в DBeaver

1) Создаём подключение к Trino
    - `host`: `localhost`
    - `port`: `8080`
2) В DBeaver: `ПКМ` → `Edit Connection` → вкладка `Driver Properties`.
3) Найдите секцию `User Properties` (Обычно она внизу списка параметров)
4) Добавьте параметры:
    - Нажмите на `+` рядом с `User Properties`.
    - В поле `Property Name` введите: `user`
    - В поле `Value` введите: `admin` (или любой другой, для Trino без настройки авторизации – не важно)
    - Аналогично (если потребуется), добавьте `password` и оставьте пустым или укажите любой текст.