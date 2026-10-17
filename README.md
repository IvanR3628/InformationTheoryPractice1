## 1. Вариант и масштаб

| Поле | Значение |
|---|---|
| ФИО (как в VK Education) | Рубан Иван Александрович |
| Вариант (запрос) | Q18 |
| Масштаб `sfN` | sf10 |
| Почему выбрал такой масштаб | sf10 - это идеальный вариант для локального Lakehouse. На sf1 данные слишком маленькие (~0.7 ГБ), они полностью помещаются в кэш ОС, и разница между кодеками (GZIP vs ZSTD) теряется в погрешности замеров. sf100 будет работать очень долго и займёт много места на диске, которого на ноутбуках и так не хватает. sf10 позволяет увидеть реальную разницу в размерах файлов и скорости чтения с диска, не перегружая систему. |

## 2. Поднятые сервисы — и почему именно они

| Сервис | Роль в стеке | Почему именно он (а не альтернатива) |
|---|---|---|
| Silo | S3-совместимое объектное хранилище (сюда физически падают parquet-файлы) | S3-совместимость означает, что код и настройки легко переносятся в облако (AWS S3, Yandex Object Storage) без переписывания. В отличие от локальной файловой системы, S3 API позволяет организовать Lakehouse с разделением compute и storage. Так как MinIO стал недоступен бесплатно, был выбран бесплатный форк Silo. |
| PostgreSQL | БД, где Lakekeeper хранит метаданные каталога | Сервис выбран из-за надёжности и ACID — метаданные каталога требуют транзакционности. Легковесные альтернативы вроде SQLite не подходят для конкурентного доступа, а альтернативы вроде MySQL не имеют официальной поддержки в Lakekeeper. |
| Lakekeeper | Iceberg REST Catalog — каталог метаданных таблиц | Открытый стандарт (Iceberg REST). В отличие от проприетарных каталогов (AWS Glue), Lakekeeper позволяет любому движку (Trino, Spark, DuckDB) видеть одни и те же таблицы. В отличие от тяжеловесного Hive Metastore, Lakekeeper написан на Rust, легковесен и не требует ZooKeeper. |
| Trino | SQL-движок: ходит в Lakekeeper за метаданными и читает/пишет данные в Silo | Разделение compute и storage. Trino не копирует данные к себе, а читает parquet прямо из S3. В отличие от Spark, Trino - это in-memory движок для интерактивной аналитики (быстрый отклик для BI), а не для долгих batch-процессов. |
| Superset | отчёт/дашборд поверх Gold | Superset (Apache Superset) был выбран из-за open-source и нативной интеграции с Trino. В отличие от платных Tableau/Power BI, Superset бесплатен и легко разворачивается в Docker. По сравнению с Metabase, Superset имеет встроенный SQL-редактор (SQL Lab) и поддерживает Trino «из коробки» (через sqlalchemy-trino), что позволяет быстро строить дашборды поверх Lakehouse без покупки лицензий и сложной настройки драйверов. |

### Скрипты, которыми поднимал стек

```powershell

mkdir "$HOME\lakehouse" # Создание папки lakehouse у активного пользователя.
cd "$HOME\lakehouse" # Переход в эту папку.

docker run -d --name lakehouse-minio `
  -p 9000:9000 -p 9001:9001 `
  -e MINIO_ROOT_USER=minioadmin `
  -e MINIO_ROOT_PASSWORD=minioadmin `
  -v minio-data:/data `
  pgsty/silo:latest server /data --console-address ":9001"
# Поднятие форка Silo для хранения parquet-файлов данных и метаданные Iceberg (на порты 9000 и 9001).

docker run -d --name lakehouse-pg `
  -e POSTGRES_PASSWORD=postgres `
  -p 5432:5432 `
  -v pg-data:/var/lib/postgresql/data `
  postgres:17
# Поднятие Postgres для хранения метаданных самого каталога Lakekeeper (список таблиц, схемы, права) на порт 5432.

docker run --rm `
  -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  quay.io/lakekeeper/catalog:latest-main migrate
# Одноразовая команда (--rm), которая создает необходимые таблицы внутри PostgreSQL (миграция).

docker run -d --name lakekeeper `
  -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -p 8181:8181 `
  quay.io/lakekeeper/catalog:latest-main serve
# Поднятие Lakekeeper на порт 8181.

curl.exe http://localhost:8181/health # Проверка, что всё удалось (вывод - JSON).

@'
{
  "warehouse-name": "demo",
  "project-id": "00000000-0000-0000-0000-000000000000",
  "storage-profile": {
    "type": "s3",
    "bucket": "warehouse",
    "key-prefix": "",
    "endpoint": "http://host.docker.internal:9000",
    "region": "us-east-1",
    "path-style-access": true,
    "flavor": "s3-compat",
    "sts-enabled": false
  },
  "storage-credential": {
    "type": "s3",
    "credential-type": "access-key",
    "access-key-id": "minioadmin",
    "secret-access-key": "minioadmin"
  }
}
'@ | Set-Content -Path create-warehouse.json -Encoding ascii
# Создание JSON-файла с настройками через powershell.

Invoke-RestMethod -Uri "http://localhost:8181/management/v1/warehouse" `
  -Method Post `
  -ContentType "application/json" `
  -InFile "create-warehouse.json"
# Отправление JSON в Lakekeeper через RestApi запрос. Результат - создание хранилища.

mkdir catalog -Force
# Создание папки catalog, куда Trino будет смотреть при старте

@'
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://host.docker.internal:8181/catalog
iceberg.rest-catalog.warehouse=demo
iceberg.rest-catalog.nested-namespace-enabled=true
iceberg.unique-table-location=true
fs.native-s3.enabled=true
s3.endpoint=http://host.docker.internal:9000
s3.path-style-access=true
s3.region=us-east-1
s3.aws-access-key=minioadmin
s3.aws-secret-key=minioadmin
'@ | Set-Content -Path catalog\lakekeeper.properties -Encoding ascii
# Создание файла коннектора Iceberg. 

docker run -d --name trino `
  -p 8080:8080 `
  -v "${PWD}\catalog:/etc/trino/catalog" `
  trinodb/trino:476
# Запуск Trino, монтирование папки catalog, чтобы он прочитал настройки Lakekeeper и MinIO (порт - 8080).

```

## 3. Разбор запроса — какие данные нужны

| Таблица TPC-H | Нужные колонки | Что запрос с ними делает (фильтр / джойн / агрегат / вывод) |
|---|---|---|
| customer | custkey, name | Джойн с таблицей orders по ключу custkey. Колонка name попадает в итоговый вывод (имя клиента). |
| orders | orderkey, custkey, orderdate, totalprice | Джойн с customer (по custkey) и lineitem (по orderkey). Фильтр по orderkey (оставляем только заказы из подзапроса). Остальные колонки идут в вывод и сортировку. |
| lineitem | orderkey, quantity | Агрегат: sum(quantity) с группировкой по orderkey и фильтром HAVING > 300 (в подзапросе). Также джойн с orders по orderkey. |

## 4. Медальон — решения по каждому слою

### Bronze (сырой)

| Таблица | Колонки (что оставил) | Строк | Вес, МБ | Формат | Сжатие | Партиционирование | Обоснование |
|---|---|---|---|---|---|---|---|
| customer | custkey, name, address, nationkey, phone, acctbal, mktsegment, comment | 1500000 | 72,4 | ORC | GZIP | нет | Слой Bronze хранит сырые данные «как есть». Выбран ORC+GZIP, так как на сыром слое приоритет - максимальная экономия дискового пространства. |
| orders | orderkey, custkey, orderstatus, totalprice, orderdate, orderpriority, clerk, shippriority, comment | 15000000 | 330,2 | ORC | GZIP | нет | Слой Bronze хранит сырые данные «как есть». Выбран ORC+GZIP, так как на сыром слое приоритет - максимальная экономия дискового пространства. |
| lineitem | orderkey, partkey, suppkey, linenumber, quantity, extendedprice, discount, tax, returnflag, linestatus, shipdate, commitdate, receiptdate, shipinstruct, shipmode, comment | 59986052 | 1541,3 | ORC | GZIP | нет | Самый большой объем данных. ORC+GZIP дает наилучшее сжатие для экономии места, скорость чтения на сыром слое не критична. |

### Silver (очищенный)

| Таблица | Колонки (что оставил) | Строк | Вес, МБ | Формат | Сжатие | Партиционирование | Обоснование |
|---|---|---|---|---|---|---|---|
| customer | custkey, name | 1500000 | 2,6 | PARQUET | ZSTD | нет | Очистка (trim пробелов). Parquet+ZSTD выбран для Silver, чтобы обеспечить быстрое чтение при последующих джойнах. |
| orders | orderkey, custkey, orderdate, totalprice | 15000000 | 144,5 | PARQUET | ZSTD | нет | Убрали лишние колонки, оставили только нужные для Q18. Формат оптимизирован под быстрый джойн. |
| lineitem | orderkey, quantity | 59986052 | 66,9 | PARQUET | ZSTD | нет | Оставили только ключ и количество для агрегации. ZSTD дает отличную скорость декомпрессии. |
| large_orders | orderkey, total_quantity | 624 | 0,003 | PARQUET | ZSTD | нет | Материализованный подзапрос из Q18 (заказы с qty > 300). Выделен в отдельную таблицу, чтобы в Gold не делать тяжелый GROUP BY повторно. |

### Gold (витрина)

| Таблица | Колонки | Строк | Вес, МБ | Формат | Сжатие | Партиционирование | Обоснование |
|---|---|---|---|---|---|---|---|
| q18_result | name, custkey, orderkey, orderdate, totalprice, total_quantity | 100 | 0,003 | PARQUET | ZSTD | нет | Готовая витрина для BI. Данных мало, но формат Parquet+ZSTD позволяет отдавать их в Superset с максимальной скоростью. |

## 5. Тесты производительности

| Сжатие на Bronze | Время запроса | Размер Bronze, МБ | Твой вывод (что лучше и почему) |
|---|---|---|---|
| GZIP | 8,285 с | 1994,8 | Показал более медленное время выполнения запроса (8,285 с), так как алгоритм GZIP требует больше ресурсов CPU для декомпрессии. Размер данных сопоставим с ZSTD, но время чтения выше. |
| ZSTD | 7,37 с | 1943,9 | Показал лучший результат по обоим параметрам: и меньший размер (экономия места), и более высокую скорость выполнения запроса (7,37 с). Это доказывает, что современный алгоритм ZSTD оптимально подходит для слоев Lakehouse, где важна скорость аналитики. |

## 6. Часть 2 — осмысление и работа с инструментами

### 6.1. Понять и объяснить

| Вопрос | Твой ответ  |
|---|---|
| Зачем нужны большие данные и lakehouse (а не обычная БД/хранилище)? | Обычные реляционные базы данных (PostgreSQL, Oracle) дорого масштабировать на терабайты и петабайты данных, а также они неэффективны для хранения сырых файлов (логов, медиа). Классические Data Lake (S3) дёшевы, но не поддерживают ACID-транзакции и SQL. Lakehouse объединяет лучшее из двух миров: дешевое объектное хранилище (S3/MinIO) + табличный формат (Iceberg), который добавляет ACID, схему и возможность делать SQL-запросы. Это позволяет разделять вычисления и хранение (Compute-Storage Separation) и не переплачивать за дорогое железо. |
| Пример schema evolution на Silver: добавил/переименовал колонку — что произошло? | Если выполнить "ALTER TABLE silver.lakehouse.orders ADD COLUMN order_year INT", формат Iceberg не переписывает старые Parquet/ORC файлы. Изменения вносятся только в метаданные (JSON-манифесты). Старые данные остаются нетронутыми, а при чтении Trino просто подставляет NULL для новой колонки в старых строках. Новые данные будут записываться уже с этой колонкой. Это делает операцию мгновенной, вместо того, чтобы долго заполнять все строки значениями NULL. |
| Пример скрипта восстановления данных из снапшота (time travel) | Сначала нужно узнать ID нужного снапшота (SELECT snapshot_id, committed_at FROM silver.lakehouse."orders$snapshots";), затем можно либо просто прочитать историческую версию не меняя таблицу (SELECT * FROM silver.lakehouse.orders FOR VERSION AS OF 123456789;), или откатить таблицу назад (CALL silver.system.rollback_to_snapshot('lakehouse', 'orders', 123456789);) |

### 6.2. Подключить инструмент к lakehouse

| Инструмент | Почему выбрал именно его |
|---|---|
| Superset | Open-source BI-инструмент с богатыми возможностями визуализации. Имеет встроенный коннектор к Trino, что позволяет подключаться напрямую к Lakehouse (каталог bronze) без дублирования данных. Отлично подходит для построения дашбордов поверх сырых данных и витрин. |

```powershell

docker run -d -p 8088:8088 --name superset -e SUPERSET_SECRET_KEY="(удалил из соображений безопасности)" apache/superset
# Поднятие Superset. Запустить получилось со второй попытки с добавлением флага -e SUPERSET_SECRET_KEY, без него Superset отказывался запускаться из соображений безопасности.

docker exec -it --user root superset uv pip install sqlalchemy-trino trino
# Установка пакетов sqlalchemy-trino и trino. Получилось тоже со второй попытки, потому что понадобились права администратора.

docker exec -it superset superset fab create-admin --username admin --firstname Admin --lastname User --email admin@fab.org --password admin
# Создание учетной записи администратора (admin/admin) для входа в веб-интерфейс.

docker exec -it superset superset db upgrade
# Применение миграций внутренней базы данных Superset (хранит настройки, дашборды, юзеров).

docker exec -it superset superset init
# Инициализация ролей и прав доступа по умолчанию.

# После этого я перешёл на http://localhost:8088, ввёл логин и пароль (admin, admin).
# В настройках я смог создать подключение к базе данных Trino с SQLAlchemy URI trino://admin@host.docker.internal:8080/bronze.
# Подключение произошло успешно и мне открылись возможности создания графиков, основываясь на таблицах bronze.
# Я попробовал добавить новое подключение Trino2 с SQLAlchemy URI trino://admin@host.docker.internal:8080/gold, чтобы увидеть таблицу q18_result.
# Ожидаемо, я увидел новую базу данных с другими таблицами.

```

## 7. Выводы

В ходе выполнения работы я на практике убедился в эффективности медальонной архитектуры: разделение данных на сырой (Bronze), очищенный (Silver) и агрегированный (Gold) слои позволяет найти баланс между производительностью и хранением данных за счёт отсечения лишних колонок и предварительных агрегаций. Бенчмарк показал, что выбор кодека - это компромисс между размером и скоростью: GZIP лучше экономит место на диске (что идеально для архивного слоя Bronze), однако ZSTD обеспечивает более высокую скорость чтения (что критически важно для слоев Silver и Gold, к которым обращается BI). Выбранный стек Lakehouse (MinIO + Lakekeeper + Trino) превосходит традиционные монолитные СУБД за счет разделения вычислений и хранения (Compute-Storage Separation), низкой стоимости масштабирования и использования открытого формата Iceberg без привязки к конкретному вендору (vendor lock-in). Интеграция с Apache Superset подтвердила, что благодаря открытым стандартам любой BI-инструмент может быть легко и бесшовно подключен напрямую к озеру данных.
