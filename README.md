# Подключение dbt к Clickhouse в Я.Облаке

## 1. Разворачиваем кластер clickhouse
```
yc managed-clickhouse cluster create /  
    --name <my-cluster-name> /  
    --environment production /  
    --network-name <my-net-name> /  
    --host type=clickhouse,zone-id=ru-central1-a,subnet-id=<my_subnet_id> /  
    --clickhouse-resource-preset  m2.micro /  
    --clickhouse-disk-type network-ssd /  
    --clickhouse-disk-size 10 /  
    --user name=user1,password=<my_password> /  
    --database name=<my_db_name>
```


## 2.Устанавливаем dbt и расширение

Cоздаем виртуальное окружение   
```
python3 -m venv dbt-env\
```            

Активируем виртуальное окружение  
```
source dbt-env/bin/activate
``` 

Клонируем репо с dbt  
```
git clone https://github.com/dbt-labs/dbt.git  
cd dbt
```

Устанавливаем dbt по списку из файла  
```
pip install -r requirements.txt
```

Ставим расширение dbt для работы с CH  
```
pip install dbt==0.20.0 dbt-clickhouse==0.20.0
```

Создаём проект dbt  
```
dbt init clickhouse_starschema
```

## 3. Генерим данные в виде csv-файлов

Клонируем репо с генератором  
```
git clone git@github.com:vadimtk/ssb-dbgen.git   
cd ssb-dbgen
```

Инициируем генератор и при помощи команд генерим файлы csv  
```
make  
./dbgen -s 1 -T d  
./dbgen -s 1 -T s  
./dbgen -s 1 -T l  
./dbgen -s 1 -T p
```  

## 4. Подготовка сервисного аккаунта для загрузки файлов в Object Storage

Проверяем список уже созданных сервисных аккаунтов  
```
yc iam service-account list
```

И создаём новый сервисный аккаунт если нужно  
```
yc iam service-account create --name <my-account-name>
```

Проверяем его данные  
```
yc iam service-account get <my-account-name>
```

Смотрим список ролей и выбираем нужную. Нас интересует storage.admin  
```
yc iam role list
```

Присваиваем роль аккаунту  
```
yc iam service-account add-access-binding <my-account-name> /  
  --role storage.admin /   
  --subject serviceAccount:<your_service-account_id>
```

Генерируем credentials для дальнейшего указания в AWS CLI config. Нужно где-то записать  
```
yc iam access-key create --service-account-name <my-account-name>
```


## 5. Для загрузки локальных CSV в S3-bucket устанавливаем AWS CLI 

Скачиваем нужную версию CLI  
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

Распаковываем архив  
```
unzip awscliv2.zip
```

Устанавливаем CLI  
```
sudo ./aws/install
```

Проверяем установку  
```
aws --version
```

Устанавливаем конфигурацию файла, где указываем значения сгенерированные ранее.  
```
aws configure  
--AWS Access Key ID [None]: <my-service-account-access-key-id>  
--AWS Secret Access Key [None]: <my-service-account-access-secret-key>  
--Default region name [None]: ru-central1  
--Default output format [None]: json
```

## 6. Загружаем файлы в S3

Создаём S3-bucket и грузим в него файлы  
```
aws --endpoint-url=https://storage.yandexcloud.net s3 mb s3://<my-bucket-name> --acl=public-read
```

Загружаем все файлы из папки с постфиксом tbl  
```
aws --endpoint-url=https://storage.yandexcloud.net s3 sync . s3://<my-bucket-name>/<my-folder-name>/`   --exclude=* --include=*.tbl --acl=public-read
```

Проверяем список загруженных файлов  
```sh
aws --endpoint-url=https://storage.yandexcloud.net s3 ls --recursive s3://<my-bucket-name>
```

## 7. Созданием таблицы в CH

Создаём таблицы в нашей базе db1 в CH  

```sql
CREATE TABLE <my_db_name>.src_customer  
        (  
                C_CUSTKEY       UInt32,  
                C_NAME          String,  
                C_ADDRESS       String,  
                C_CITY          LowCardinality(String),  
                C_NATION        LowCardinality(String),  
                C_REGION        LowCardinality(String),  
                C_PHONE         String,  
                C_MKTSEGMENT    LowCardinality(String)  
        )  
        ENGINE = S3('https://storage.yandexcloud.net/<my-bucket-name>/<my-folder-name>/customer.tbl', 'CSV')  
        ;

        CREATE TABLE <my_db_name>.src_lineorder
        (
            LO_ORDERKEY             UInt32,
            LO_LINENUMBER           UInt8,
            LO_CUSTKEY              UInt32,
            LO_PARTKEY              UInt32,
            LO_SUPPKEY              UInt32,
            LO_ORDERDATE            Date,
            LO_ORDERPRIORITY        LowCardinality(String),
            LO_SHIPPRIORITY         UInt8,
            LO_QUANTITY             UInt8,
            LO_EXTENDEDPRICE        UInt32,
            LO_ORDTOTALPRICE        UInt32,
            LO_DISCOUNT             UInt8,
            LO_REVENUE              UInt32,
            LO_SUPPLYCOST           UInt32,
            LO_TAX                  UInt8,
            LO_COMMITDATE           Date,
            LO_SHIPMODE             LowCardinality(String)
        )
        ENGINE = S3('https://storage.yandexcloud.net/<my-bucket-name>/<my-folder-name>/lineorder.tbl', 'CSV')
        ;

        CREATE TABLE <my_db_name>.src_part
        (
                P_PARTKEY       UInt32,
                P_NAME          String,
                P_MFGR          LowCardinality(String),
                P_CATEGORY      LowCardinality(String),
                P_BRAND         LowCardinality(String),
                P_COLOR         LowCardinality(String),
                P_TYPE          LowCardinality(String),
                P_SIZE          UInt8,
                P_CONTAINER     LowCardinality(String)
        )
        ENGINE = S3('https://storage.yandexcloud.net/<my-bucket-name>/<my-folder-name>/part.tbl', 'CSV')
        ;

        CREATE TABLE <my_db_name>.src_supplier
        (
                S_SUPPKEY       UInt32,
                S_NAME          String,
                S_ADDRESS       String,
                S_CITY          LowCardinality(String),
                S_NATION        LowCardinality(String),
                S_REGION        LowCardinality(String),
                S_PHONE         String
        )
        ENGINE = S3('https://storage.yandexcloud.net/<my-bucket-name>/<my-folder-name>/supplier.tbl', 'CSV')
        ;
```

## 8. Конфигурация подключения dbt к базе данных внутри кластера clickhouse

По умолчанию dbt ищет параметры подключения в файле /home/usr/.dbt/profiles.yml  
```yaml
<example-profile-name>:  
  target: dev  
  outputs:  
    dev:  
      type: clickhouse --тип расширения что мы скачивали ранее  
      schema: <my_db_name>  
      host: <my-host.mdb.yandexcloud.net>  
      port: 9440  
      user: <my-user>  
      password: <my-password>  
      secure: True
 ```     

В файле dbt_project.yml в рабочем проекте dbt указываем название для profile-параметра  
```yaml
name: 'clickhouse'  
version: '1.0.0'  
config-version: 2`  
profile: '<example-profile-name>'
```
Проверяем подключение с помощью команды  
```bash
dbt debug
```

В этом же файле указываем расположение моделей и типы таблиц в них. Такая же иерархия будет в папке models в проекте. Переходим туда и создаём подпапку clickhouse, а внутри неё остальные - sources, staging, star
```yaml
models:  
  clickhouse:  
    sources:  
      +materialized: table  
    staging:  
      +materialized: view  
    star:  
      +materialized: table
```

## 9. Создание моделей

Сперва указываем создаём файл sources.yml в папке models/clickhouse/sources

```yaml
version: 2

sources:
    - name: dbgen
      database: db1
      schema: db1
      tags: ["db1"]      
      description: "External tables"

      tables:
        - name: customer
          description: "Customer source in S3 bucket"
          identifier: src_customer
        - name: lineorder
          description: "Lineorder source in S3 bucket"
          identifier: src_lineorder
        - name: supplier
          description: "Supplier source in S3 bucket"
          identifier: src_supplier
        - name: part
          description: "Part source in S3 bucket"
          identifier: src_part
```

Затем создаём модели. Переходим в папку /staging и создаём 4 файла

Таблица stg_customer
```SQL
{{ config(materialized='view') }}

Select *
FROM {{ source('dbgen', 'customer')}}
```

Таблица stg_lineorder
```SQL
{{ config(materialized='view') }}

Select *
FROM {{ source('dbgen', 'lineorder')}}
```

Таблица stg_part
```SQL
{{ config(materialized='view') }}

Select *
FROM {{ source('dbgen', 'part')}}
```

Таблица stg_supplier
```SQL
{{ config(materialized='view') }}

Select *
FROM {{ source('dbgen', 'supplier')}}
```

После этого  создаём файл staging.yml
```sql

version: 2

models:  
  - name: stg_customer  
    description: "Customer model"  

  - name: stg_lineorder  
    description: "Lineorder dbt model"  
  
  - name: stg_supplier  
    description: "Supplier dbt model"  
  
  - name: stg_part  
    description: "Part dbt model"  
``` 

Переходим в папку /star и создаём файл star.sql в котором соединяем все предыдущие таблицы из staging

```sql
{{ config(materialized='table') }}

SELECT
    l.LO_ORDERKEY AS LO_ORDERKEY,
    l.LO_LINENUMBER AS LO_LINENUMBER,
    l.LO_CUSTKEY AS LO_CUSTKEY,
    l.LO_PARTKEY AS LO_PARTKEY,
    l.LO_SUPPKEY AS LO_SUPPKEY,
    l.LO_ORDERDATE AS LO_ORDERDATE,
    l.LO_ORDERPRIORITY AS LO_ORDERPRIORITY,
    l.LO_SHIPPRIORITY AS LO_SHIPPRIORITY,
    l.LO_QUANTITY AS LO_QUANTITY,
    l.LO_EXTENDEDPRICE AS LO_EXTENDEDPRICE,
    l.LO_ORDTOTALPRICE AS LO_ORDTOTALPRICE,
    l.LO_DISCOUNT AS LO_DISCOUNT,
    l.LO_REVENUE AS LO_REVENUE,
    l.LO_SUPPLYCOST AS LO_SUPPLYCOST,
    l.LO_TAX AS LO_TAX,
    l.LO_COMMITDATE AS LO_COMMITDATE,
    l.LO_SHIPMODE AS LO_SHIPMODE,
    c.C_NAME AS C_NAME,
    c.C_ADDRESS AS C_ADDRESS,
    c.C_CITY AS C_CITY,
    c.C_NATION AS C_NATION,
    c.C_REGION AS C_REGION,
    c.C_PHONE AS C_PHONE,
    c.C_MKTSEGMENT AS C_MKTSEGMENT,
    s.S_NAME AS S_NAME,
    s.S_ADDRESS AS S_ADDRESS,
    s.S_CITY AS S_CITY,
    s.S_NATION AS S_NATION,
    s.S_REGION AS S_REGION,
    s.S_PHONE AS S_PHONE,
    p.P_NAME AS P_NAME,
    p.P_MFGR AS P_MFGR,
    p.P_CATEGORY AS P_CATEGORY,
    p.P_BRAND AS P_BRAND,
    p.P_COLOR AS P_COLOR,
    p.P_TYPE AS P_TYPE,
    p.P_SIZE AS P_SIZE,
    p.P_CONTAINER AS P_CONTAINER
FROM {{ ref('stg_lineorder') }} AS l
INNER JOIN {{ ref('stg_customers') }} AS c ON c.C_CUSTKEY = l.LO_CUSTKEY
INNER JOIN {{ ref('stg_supplier') }} AS s ON s.S_SUPPKEY = l.LO_SUPPKEY
INNER JOIN {{ ref('stg_part') }} AS p ON p.P_PARTKEY = l.LO_PARTKEY
```

Создаём файл star.yml
```yaml
version: 2

models:
  - name: star
    description: "Star model"
```

B последним действием собираем модели, выполнив команду 
```
dbt run
```

## 10. Выполняем запросы к сформированной таблице star

Q1.1.
```
SELECT sum(LO_EXTENDEDPRICE * LO_DISCOUNT) AS revenue
FROM star
WHERE toYear(LO_ORDERDATE) = 1993 AND LO_DISCOUNT BETWEEN 1 AND 3 AND LO_QUANTITY < 25;
```
Ответ: 446031203850


Q1.2
```
SELECT sum(LO_EXTENDEDPRICE * LO_DISCOUNT) AS revenue
FROM star
WHERE toYYYYMM(LO_ORDERDATE) = 199401 AND LO_DISCOUNT BETWEEN 4 AND 6 AND LO_QUANTITY BETWEEN 26 AND 35;
```
Ответ: 98714004603


Q1.3
```
SELECT sum(LO_EXTENDEDPRICE * LO_DISCOUNT) AS revenue
FROM star
WHERE toISOWeek(LO_ORDERDATE) = 6 AND toYear(LO_ORDERDATE) = 1994
  AND LO_DISCOUNT BETWEEN 5 AND 7 AND LO_QUANTITY BETWEEN 26 AND 35;
```
Ответ: 26110729246

P.s. Ещё можно всё покрыть тестами, автоматизировать загрузку из хранилища и т.д.