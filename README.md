# pet_project_trino_data_lake


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