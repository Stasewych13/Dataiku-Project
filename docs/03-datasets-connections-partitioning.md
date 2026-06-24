# 03. Datasets, Connections та Partitioning

> Частина бази знань про Dataiku DSS для аналітика. Версії DSS 13.x/14.x (2025–2026).
> Технічні, UI- та код-терміни наведені в оригіналі (English).

## Вступ

У Dataiku DSS уся робота з даними починається з трьох фундаментальних понять:

- **Dataset** — логічний об'єкт, що описує таблицеподібний набір даних (схема + посилання на місце зберігання). Це «вузли» вашого Flow.
- **Connection** — налаштоване з'єднання з системою зберігання (база даних, cloud storage, файлова система, API). Саме connection визначає, *де* фізично лежать дані.
- **Partitioning** — поділ датасету на незалежні частини (partitions) уздовж осмислених вимірів (час, категорія), що дає інкрементальну та ефективну обробку.

Розуміння цих трьох понять — основа для побудови масштабованих, надійних і недорогих data-пайплайнів. Цей документ покриває їх від базового рівня до просунутого.

Перехресні посилання: загальні концепції Flow — `02-projects-flow-core-concepts.md`; visual recipes, що читають/пишуть датасети — `04-visual-recipes.md`; metrics/checks/scenarios — `09-scenarios-automation-metrics-checks.md`.

---

## 1. Datasets (Базово → Середній)

### 1.1 Що таке dataset

**Dataset** — це центральний об'єкт DSS, аналог SQL-таблиці: «a series of records with the same schema» (набір записів з однаковою схемою). Важливо: dataset — **логічний** об'єкт. Він не обов'язково *містить* дані; здебільшого він зберігає **інформацію про connection/location** + **schema** і вказує на фактичні дані в зовнішній системі.

Dataset може вказувати на:

- SQL-таблицю або кастомний SQL-запит;
- колекцію MongoDB (та інші NoSQL);
- теку з файлами на локальній файловій системі сервера;
- теку з файлами на Hadoop HDFS;
- cloud object storage (Amazon S3, Azure Blob, GCS), Cassandra тощо.

### 1.2 Managed vs External datasets (ключова відмінність)

| | **External dataset** | **Managed dataset** |
|---|---|---|
| Хто володіє даними | Зовнішня система (DSS лише читає) | DSS «бере ownership» |
| Як створюється | Import / підключення до наявних даних (наявна SQL-таблиця, uploaded file) | Як **output** рецепту (recipe) |
| Що DSS може робити | Тільки читати | Створювати/видаляти таблиці, змінювати схему (для SQL — drop/create), перезаписувати дані |
| Розташування | Будь-де, куди є connection | local filesystem, HDFS, SQL, S3, Azure, GCS тощо |

- **Uploaded / imported files** — це різновид external dataset: ви завантажуєте файл, а DSS читає його через відповідний format.
- **Managed datasets** — типовий результат роботи рецептів; DSS повністю керує їх життєвим циклом (див. розділ 5 про storage).

> (Базово) Правило великого пальця: усе, що ви *приносите* в DSS — external; усе, що DSS *створює* у Flow — managed.

### 1.3 Налаштування датасету (Dataset settings tabs)

Відкривши dataset, ви бачите вкладки:

- **Explore** — попередній перегляд **sample** даних (не всіх даних!).
- **Settings** — connection/location, format config, parsing-опції.
- **Schema** — колонки, їх **storage type** і **meaning**.
- **Sampling** (підвкладка Explore/Settings) — налаштування семпла.
- Для партиціонованих датасетів — додатково вибір/керування partitions.

---

## 2. Schema: Storage Types vs Meanings (Середній)

Schema датасету = список колонок, де в кожної є **name**, **storage type** (технічний) і **meaning** (семантичний). Це **дві різні, взаємодоповнювальні** системи типів — одна з найважливіших концепцій DSS.

### 2.1 Storage type (технічний, «strict»)

Визначає, як дані *фізично* зберігаються в бекенді (SQL/Hadoop/Spark). Він **strict**: не можна зберегти «невалідне» для типу значення (наприклад, decimal у int-колонку).

Повний перелік storage types (точні назви):

- Цілі: `tinyint` (8-bit), `smallint` (16-bit), `int` (32-bit), `bigint` (64-bit)
- Дробові: `float` (32-bit), `double` (64-bit)
- `string` (текст змінної довжини)
- `boolean`
- Часові: `datetime with zone`, `datetime no zone`, `date only`
- Геопросторові: `geopoint`, `geometry`
- Складені: `array`, `map`, `object`

> (Середній) Чому це важливо: storage type керує тим, як рецепти генерують jobs/SQL/Spark-код. Помилковий тип («mistyped column») зазвичай призводить до **failures**, коли дані переходять між SQL/Hadoop/Spark.

### 2.2 Meaning (семантичний, «rich, flexible»)

Високорівнева семантична інтерпретація поверх storage type. На відміну від storage type, meanings **не strict**: клітинки лише позначаються як **valid**/**invalid** щодо meaning, але це не блокує зберігання. **Meanings автоматично визначаються** з вмісту колонки (pattern recognition).

Вбудовані (recognized) meanings — точні назви за категоріями:

*Базові:* **Text**, **Decimal**, **Integer** (використовує `bigint`, якщо поза 32-bit), **Boolean** (розпізнає true/false/yes/no/1/0…), **Datetime with zone** (лише ISO-8601), **Datetime no zone** (`yyyy-MM-dd HH:mm:ss`), **Date only** (`yyyy-MM-dd`), **Object / Array** (JSON), **Natural language**.

*Геопросторові:* **Latitude / Longitude**, **Geopoint** (вкл. WKT), **Geometry** (WKT), **Country** (англ. назви + ISO-коди), **US State**.

*Web-специфічні:* **IP Address** (IPv4/IPv6), **URL**, **HTTP Query String**, **User Agent**, **E-Mail address**. Плюс **Currency code**, **Gender** та інші.

User-defined meanings — чотири типи:

- **Declarative** — лише документація, без валідації, без auto-detect.
- **Values list** — валідація за списком дозволених значень (exact / case-insensitive / accent-insensitive).
- **Values mapping** — пари ключ→мітка (напр. `0`→`no`, `1`→`yes`); має окремий prep-processor.
- **Pattern** — значення мають збігатися з Java-regex.

> (Просунуто) Auto-detection для user-defined meanings технічно доступний, але **не рекомендований**: він «can cause built-in meanings not to be recognized anymore» і помітні **slowdowns**.

### 2.3 Як DSS виводить (infers) схему

- **SQL & Cassandra** — схема читається прямо зі структури БД.
- **Text-based files** (CSV/TSV…) — інференс аналізом вмісту файлу (і storage types, і meanings).
- **Typed files** (Parquet/Avro/ORC) — схема береться з вбудованої у файл схеми.

---

## 3. Explore, Preview та Sampling (Базово → Середній)

### 3.1 Sample, а не всі дані

Вкладка **Explore** показує **sample** датасету, а не повні дані. Sample **повністю завантажується в RAM** для швидкої інтерактивної роботи (explore, visual prep, charts), тож конфігурація семпла прямо впливає на продуктивність.

- **Default sample:** перші **10,000 records**.
- **Рекомендований максимум:** до **200,000 records** для гарної інтерактивності.
- Налаштовується на вкладці **Sampling** (метод + розмір).

### 3.2 Sampling methods (точні назви)

1. **First records** — перші N рядків; найшвидше, але може бути зміщеним (biased).
2. **Random sampling (fixed number of records)** — N випадкових записів, один прохід.
3. **Random sampling (approximate ratio)** — ~X% записів, один прохід.
4. **Random sampling (approximate number of records)** — ~N записів, точніше на великих даних, два проходи.
5. **Column values subset** — усі рядки для випадкового підмножини значень колонки; зберігає повні послідовності (напр. усі дії юзера).
6. **Stratified (fixed number of records)** — N рядків зі збереженням розподілу значень колонки.
7. **Stratified (approximate ratio)** — X% зі збереженням розподілу.
8. **Class rebalancing (approximate number of records)** — ~N рядків, ребаланс по модальностях (лише undersampling).
9. **Class rebalancing (approximate ratio)** — X% через undersampling.
10. **Last records** — останні N рядків (потрібен повний прохід).

> (Середній) Чому sampling важливий: уся інтерактивна робота йде по семплу. Biased sample (напр. «First records» на відсортованих даних) може ввести в оману трансформації та статистику. Random/stratified дають репрезентативніший вигляд ціною повних проходів по даних.

---

## 4. File Formats (Середній → Просунуто)

### 4.1 Підтримувані формати

Точні назви з formats index: **CSV / TSV** (Delimiter-separated values), **Excel**, **Fixed width**, **Parquet**, **Avro**, **Hive ORCFile** (ORC), **XML**, **JSON**, **ESRI Shapefiles**, **Delta Lake**, **YXDB** (Alteryx), **SPSS**, **SAS Format**, **KML**.

### 4.2 Row-oriented vs Columnar

- **Row-oriented:** CSV/TSV, JSON, XML, Excel, **Avro**. Людиночитані (CSV/JSON) або компактні бінарні (Avro). Добрі для обміну даними та транзакційного/стрімінгового запису.
- **Columnar:** **Parquet**, **Hive ORCFile (ORC)**, **Delta Lake** (на базі Parquet). Оптимізовані для analytics / big data — читають лише потрібні колонки, краще стискаються.

### 4.3 Особливості форматів

**CSV/TSV** — чотири стилі quoting: **Excel-style (RFC4180)**, **Unix-style**, **Escaping only**, **No escaping, no quoting**. Для Hive/Spark-рецептів підтримуються лише «Escaping only» та «no escaping, no quoting»; newlines у полях не працюють з Hadoop. Autodetection quoting може помилятися — перевіряйте вручну. Стиснення — на рівні файлу (напр. gzip), а не опція формату.

**Parquet** — Dataiku **наполегливо рекомендує Parquet замість CSV при роботі з Hadoop**. Column-oriented (навіть для вкладених складених типів), **block-based compression**, **predicate pushdown** (фільтри проштовхуються вниз). «Speedups can reach up to x100 on select queries». Працює на HDFS, S3, GCS, Azure Blob. Застереження: **використовуйте lowercase identifiers** (case-sensitivity Hive/Parquet); за замовчуванням DSS використовує схему датасету, а не вбудовану в файл.

**Avro** — «compact, fast, binary data format» з багатими структурами (`map`, `union`, `array`, `record`, `enum`); «very well suited for data exchange since the schema is stored along with the data» (row-oriented, self-describing). На запис Avro-схему виводять зі схеми датасету; всі типи обгортаються в nullable union.

**Hive ORCFile (ORC)** — columnar для екосистеми Hadoop/Hive; як Parquet, добрий для великомасштабної аналітики зі стисненням.

**JSON / XML** — добрі для вкладених/напівструктурованих даних та обміну; неефективні для великих аналітичних сканів.

**Excel (xlsx)** — зручний для бізнес-користувачів і малих даних; не для big data.

### 4.4 Коли який формат

- **CSV** — людиночитані, портативні малі/середні дані, обмін.
- **Parquet/ORC** — analytics та big-data пайплайни на Hadoop/cloud (стиснення + columnar pushdown).
- **Avro** — schema-carrying бінарний обмін / стрімінг.
- **JSON/XML** — вкладений/напівструктурований обмін.
- **Excel** — малі файли для бізнесу.

---

## 5. Connections (Базово → Просунуто)

### 5.1 Що таке connection і де налаштовується

**Connection** — це іменоване, налаштоване визначення того, як DSS досягає джерела даних (host, paths, credentials, security). Datasets **посилаються** на connections: connection тримає location + credentials, а dataset тримає schema + конкретну таблицю/шлях усередині цієї connection.

- Налаштовується в **Administration > Connections**.
- Більшість типів (Filesystem, HDFS, FTP, SQL…) «can only be created and managed by DSS administrators».
- Виняток — **personal connections**: non-admin користувач із відповідним дозволом може створити їх для типів, що потребують credentials. Застереження: «Personal connections run with the permissions of the DSS service account and may allow actions beyond simple data access».
- Довідкова сторінка-матриця: `.../connecting/connections.html` («Supported connections») — список усіх конекторів + read/write formats.

### 5.2 Типи connections (точні назви з UI)

**Cloud Data Warehouses:** Snowflake, Databricks, Google BigQuery, Amazon Redshift, Azure Synapse, Microsoft Fabric Warehouse.

**SQL Databases:** PostgreSQL, MySQL, Microsoft SQL Server, Oracle, Teradata, Pivotal Greenplum, Google AlloyDB, Google Cloud SQL, AWS Athena, Trino/Starburst, Treasure Data, Vertica, SAP HANA, Dremio, IBM Netezza, Exasol, IBM DB2, Denodo.

**Cloud Storage:** Amazon S3, Azure Blob Storage, Google Cloud Storage, Iceberg.

**NoSQL & Search:** MongoDB, Elasticsearch and OpenSearch, Cassandra, Amazon DynamoDB.

**File Transfer & Access:** FTP, SCP/SFTP (SSH), HDFS, Server filesystem.

**Application connectors:** Airtable, ServiceNow, Stripe, Salesforce, Zendesk, SharePoint Online, Google Drive, Google Sheets, Google Analytics, SAP OData, OneDrive, Jira, LinkedIn Marketing, YouTube та інші.

**Other / Streaming:** HTTP, OData, kdb+, File uploads, Sample datasets, Synthetic data generation; Kafka, AWS SQS, HTTP Server-Sent Events (streaming).

### 5.3 Особливості ключових типів

**Server filesystem** — на базі **Filesystem connection** з **root path**: «All datasets within a connection must reside within this defined root path». Admin-only. **Недоступний на Dataiku Cloud**.

**Uploaded files** — файли, завантажені прямо в DSS; зберігаються в managed location, а не в зовнішній системі.

**SQL** (спільна поведінка — див. 5.5). Особливості:

- **Snowflake** — поля Host (account id), Database, Warehouse, Schema, Role. **Auth Type:** User/Password; **OAuth** (External/Snowflake OAuth, per-user або global/client-credentials); **Key Pair** (private key, з/без passphrase). Advanced: **Extended push-down** (Java UDF), Spark native integration, **Snowpark**.
- **Google BigQuery** — auth: **Private key** (service-account JSON), **Environment** (Application Default Credentials), або **OAuth**. **Automatic fast-write** через GCS staging (Auto fast write connection + Path in connection). Опції: Job labels, Require partition filter.
- **PostgreSQL, MySQL, Oracle, SQL Server, Redshift, Greenplum, Teradata, Vertica, SAP HANA, Synapse, Databricks** — стандартна JDBC-конфігурація (host/port/db/user/password). Redshift підтримує IAM-auth.

**Cloud object storage:**

- **Amazon S3** — auth: **AWS Keypair** (access/secret key); **Environment** (instance/IAM profile, коли DSS на AWS); **AssumeRole** (рекомендовано для cross-account, через STS). Режими шляхів: **Free selection** vs **Path restrictions**. **Default bucket** + **Default path** для managed datasets. **Encryption Mode:** None / SSE-S3 / SSE-KMS.
- **Azure Blob Storage / ADLS Gen2** — нативна підтримка **ADLS Gen2**, швидкі read/write зі Spark + Parquet. Auth: Account/Shared key; **OAuth from App** (service principal: Tenant id, App id, App secret). Container, Default bucket/path, Free selection vs Path restrictions.
- **Google Cloud Storage** — auth: **Service Account** (JSON + project id) або **OAuth2**. Default container + Default path, Path restrictions.

**HDFS** — через Hadoop Filesystem API; покриває схеми `hdfs://`, `maprfs://`, `s3a://`, `wasb://`, `adl://`, `gs://`. Kerberos/keytab — на рівні Hadoop security.

**NoSQL:** MongoDB, Cassandra, Elasticsearch/OpenSearch, Amazon DynamoDB — read/write конектори. MongoDB і більшість DB підтримують **per-user credentials**.

**File transfer / network:**

- **SCP/SFTP (SSH)** — Host, User, **Use public key authentication**, Password (password mode) або Key passphrase (key mode).
- **FTP** — Host, User (empty = anonymous), Password, Path, Use passive mode, **Writable**, **Allow managed datasets**, **Allow managed folders**.
- **HTTP** — files-based read-конектор.

**Google Sheets** — application-конектор; auth через service account JSON або OAuth.

**Kafka** (streaming endpoint) — список **bootstrap servers** + security: PLAINTEXT, SASL (вкл. Kerberos), опційно SSL. *Twitter як окремий конектор у поточних версіях — legacy/не first-class.*

### 5.4 Credentials та Security

**Global vs per-user credentials:**

- **Global credentials** — один сервісний акаунт для всього доступу через connection. Типи: shared key/keypair (cloud), username/password (SQL), **AWS AssumeRole**. Плюси: централізована авторизація, паролі лишаються секретними якщо **Details readable by = None**, контроль на рівні груп, audit на боці DSS. Мінуси: немає per-user audit на віддаленій системі; усі юзери мають однакові віддалені права.
- **Per-user credentials** (credential mapping) — кожен юзер дає свої credentials; кожна віддалена дія виконується від його імені. Налаштування: **Administration > Connections** → **Connections credentials** → **Per-user**; юзери вводять свої в **Profile and settings > Credentials > Connection credentials**. Типи: username/password, shared key/keypair, **OAuth2**. **Потребує User Isolation Framework (UIF)** ліцензії. Плюси: гранулярний доступ і audit на віддаленій системі.

**Як DSS зберігає credentials безпечно:**

- Паролі в config-файлах шифруються **symmetric authenticated AES в CTR mode** (AES-128/192/256).
- **Encryption key** лежить у DSS data directory, ніколи не доступний юзерам; опційно в **cloud secret manager** (AWS Secrets Manager, Azure Key Vault, Google Cloud Secret Manager).
- **Local user** паролі логіну — **one-way hash** (необоротний), окремо від оборотного шифрування connection-credentials (паролі connection мають бути оборотними, бо DSS шле raw-пароль віддаленій системі).

**Connection-level permissions** (надаються лише **групам**):

- **Freely use / Usable by** — хто може створювати датасети, змінювати settings, browse connection і **слати код (напр. SQL)** проти неї. **Не** обмежує читання вже визначених на connection датасетів.
- **Details readable by** — доступ до **незашифрованих деталей** (filesystem paths, HDFS properties, SQL hostnames/databases і **credentials**). Попередження: «Granting 'Details readable by' on a connection to a group gives users access to the unencrypted credential». Для **HDFS і S3** надання цього **наполегливо рекомендоване для Spark-продуктивності** (без нього Spark падає на повільний шлях).

**OAuth / IAM / Kerberos:**

- **OAuth2** — Snowflake, BigQuery, GCS, Azure Blob (per-user або global/client-credentials); redirect URI `DSS_BASE_URL/dip/api/oauth2-callback`.
- **IAM roles** — AWS: Environment/instance profile + **AssumeRole** (STS); GCP: Application Default Credentials; Azure: service principal (OAuth from App) / managed identity.
- **Kerberos/keytabs** — на рівні Hadoop (keytab) і Kafka (SASL-Kerberos).

### 5.5 SQL specifics

- **SQL pushdown / in-database computation** — на більшості БД DSS може «Execute Visual recipes directly in-database» (DB-to-DB visual recipe ніколи не виносить дані з БД) і «Execute Charts directly in-database». Snowflake додає **Extended push-down** через Java UDFs.
- **Allow managed datasets** — налаштування `allowManagedDatasets` визначає, чи можна писати **managed datasets** на цю connection (паралельні тумблери для managed folders).
- **Schema / naming / table prefixes** — managed SQL datasets створюються у налаштованому **Schema** connection із застосуванням **naming rules** (DSS шаблонізує імена таблиць, зазвичай вмикаючи project key + dataset name, щоб уникнути колізій). Для object storage аналог — **Default path** + naming rules у **Default bucket/container**.

---

## 6. Managed Datasets та Storage (Середній)

- Managed datasets зберігаються в **location тієї connection**, на яку вони налаштовані; розкладку файлів/папок **керує DSS** (ви не керуєте шляхами вручну).
- Вбудовані connections за замовчуванням:
  - **`filesystem_managed`** — дані managed datasets;
  - **`filesystem_folders`** — дані managed folders;
  - вони мапляться на каталог **`managed_datasets`** усередині DSS **Data Directory** (де встановлено DSS).
- Connection для managed folder за замовчуванням — **`managed_folders`** → каталог `managed_folders` у data dir. Location managed folder має бути filesystem-подібною connection (local FS, HDFS, S3, Azure, GCS, FTP, SSH).
- **Рекомендація:** налаштуйте **naming rules** і дефолтний path/bucket на рівні connection, щоб managed datasets/folders з різних проєктів **не перекривалися**. Існують relocation-фічі, щоб робити managed datasets relocatable.

> (Просунуто) Для продакшну зазвичай не зберігають managed-дані в локальній FS DSS-сервера: їх кладуть у SQL/cloud storage заради масштабованості та pushdown.

---

## 7. Metrics на датасетах (коротко) (Середній)

> Детально — у `09-scenarios-automation-metrics-checks.md`. Тут — лише огляд.

- Metrics застосовні до трьох елементів Flow: **datasets, managed folders, saved models**; конфігуруються у вкладці **Status** (підвкладка Metrics/Edit). Обчислюються кнопкою **Compute** або через scenarios.
- Доступні **probes** для датасетів:
  - **Basic info** — розмір і кількість файлів;
  - **Records** — кількість рядків (metric id `records:COUNT_RECORDS`);
  - **Column statistics** — MIN, MAX, AVG, most-frequent, top-N;
  - **Data validity** — кількість/частка невалідних значень за meaning;
  - **Custom probes** — SQL- та Python-based.
- Дефолтні метрики — **column count** і **record count**. Metrics **автоматично історизуються**; є окремий «Metrics dataset» конектор для downstream-аналізу історії.

---

## 8. Partitioning (Просунуто)

### 8.1 Концепція

> «Partitioning refers to the splitting of a dataset along meaningful dimensions. Each partition contains a subset of the dataset that can be built independently.»

**Навіщо:**

- **Incremental processing** — рецепт «only processes the involved partitions and does not access the full datasets». Ви перебудовуєте лише ті partitions, що змінилися (напр. дані за сьогодні), а не весь датасет.
- **Performance** — уникнення повторної обробки історичних даних щоразу.

**Дві моделі партиціонування:**

- **Files-based** — партиціонування задається **розкладкою файлів на диску**.
- **Column-based** — партиціонування виводиться з **однієї або кількох колонок** даних (для SQL/NoSQL).

### 8.2 Dimensions (виміри)

**Time-based** (гранулярності `year`, `month`, `day`, `hour`). Формати ідентифікаторів:

- Year: `YYYY` → `2020`
- Month: `YYYY-MM` → `2020-01`
- Day: `YYYY-MM-DD` → `2020-01-17`
- Hour: `YYYY-MM-DD-HH` → `2020-01-17-13`
- Спецключові слова (для динамічних таргетів у scenarios): `CURRENT_DAY`, `PREVIOUS_MONTH`, `PREVIOUS_DAY` тощо.

**Discrete** (value-based, напр. country, category): «the identifier of a discrete dimension value is the value itself» — лише літери й цифри.

**Multiple dimensions** — об'єднуються в один ідентифікатор через `|` (pipe). Приклад (DAY + country): `2020-01-01|France`.

**Range / partition-spec syntax:**

- Часовий діапазон: `2020-01-25/2020-01-28` (3 дні).
- Комбіновані: `2020-02/2020-03|FR/IT` → 4 partitions (cross-product 2 місяці × 2 країни).

### 8.3 File-based partitioning

Partitions мапляться на структуру тек на диску:

```
/folder/2020/02/03/file0.csv
/folder/2020/02/04/file0.csv
```

**Pattern syntax (placeholders):**

- `%Y` — рік (4 цифри), `%M` — місяць (2), `%D` — день (2), `%H` — година (2)
- `%{dimension_name}` — значення discrete-виміру

Приклад: `/%Y/%M/%D/.*`. Правила:

- Початковий `/` обов'язковий (усі шляхи anchored до root).
- Фінальний `.*` обов'язковий (ловить усі файли з префіксом).
- Часові ієрархії мають бути валідними: day-level потребує `%Y`, `%M` і `%D`.

**Конфігурація:** вкладка Partitioning → **Activate partitioning** (DSS може автозапропонувати pattern) → визначити dimensions → задати pattern → **List partitions** для перевірки (показує, які partitions згенеруються, і які файли не зматчилися).

**Ключове обмеження:** «Partitioning a files-based dataset cannot be defined by the content of a column within this dataset» — лише розкладка файлів.

Підтримувані file-connections: Filesystem, HDFS, S3, Azure Blob, GCS, Network.

### 8.4 SQL partitioning

> «Datasets based on SQL tables support partitioning based on the values of specified columns: the partitioning columns must be part of the schema of the table.»

- Значення partition лежить **у колонці** (частина даних/схеми), а не у шляху теки.
- **Time-based partitioning columns мають бути strings, не Date-типи**; значення у DSS-синтаксисі (`2020-02-28` для day).
- DSS може використовувати **native partitioning** БД (якщо є) — ефективніше listing/removing partitions.
- Коли **non-SQL recipe** пише partition, DSS автоматично видаляє наявні записи цього partition, заповнює partitioning-колонку значенням partition і гарантує її наявність у вихідній схемі.
- Ліміти engine: напр. **Vertica — лише 1024 partitions на таблицю** (може унеможливити hour-level).

### 8.5 Partition dependencies

DSS мапить запитані **output**-partitions на потрібні **input**-partitions через dependency functions:

| Функція | Поведінка |
|---|---|
| **Equals** | «Use the same value» — input = output (той самий тип/період). Для time і discrete. |
| **Time range** | Усі періоди у відносному до output діапазоні (FROM/TO). |
| **Since beginning of week / month** | Усі дні/години від початку тижня/місяця до output-дати. |
| **Explicit values** | Прямо задані значення; підтримує range-синтаксис. |
| **Latest available** | Один partition = найсвіжіший доступний (для не-time: «last by alphabetical ordering»). |
| **All available** | Усі partitions input, доступні на момент запуску. Не генерує нові. |
| **Custom (Python)** | Python-функція повертає список потрібних ідентифікаторів. |

Модель виконання рецепту: «Read one or several partitions of the input datasets» → «Write exactly one partition for each output dataset». За кількох виходів — усі output datasets мають **однакове партиціонування**, обчислюється **той самий** partition.

### 8.6 Partitioned builds

Ви задаєте *що* будувати (target partition identifier або spec); DSS визначає *що запустити* через dependency functions (проштовхуючи запити вгору по Flow).

**Build modes (UI):**

- **Build only this dataset** — будує задані partitions без upstream-залежностей.
- **Build required datasets** (recursive) — автоматично проштовхує запити вгору, будуючи лише потрібні upstream-partitions.
- **Force-rebuild** — перебудовує незалежно від наявних/up-to-date даних.

API-константи build-mode: `RECURSIVE_BUILD`, `NON_RECURSIVE_FORCED_BUILD`, `RECURSIVE_FORCED_BUILD`. У scenarios можна вживати динамічні ключові слова (`PREVIOUS_DAY` тощо) як target.

### 8.7 Partition redispatch

**Точна назва опції:** **Redispatch partitioning according to input columns**.

- **Що робить:** перетворює **non-partitioned** input на **partitioned** output, розкидаючи кожен рядок у partition за значеннями колонок. «Dataiku reads all input data simultaneously and routes each row to precisely one partition» — будує **всі** partitions одразу.
- **Де є:** у рецепті **Sync** (вкладка Configuration) і **Prepare** (вкладка Advanced). Опція з'являється лише коли **output** партиціонований.
- **Use case:** пласкі дані (напр. транзакції з колонкою `merchant_subsector`) розкласти на partitions. Workflow: Sync recipe → зробити output партиціонованим → увімкнути redispatch → build (partition у Run options — «dummy», redispatch будує всі partitions попри це).
- **Caveat (files-based):** «the partition column(s) are removed from the schema» — після партиціонування partitioning-колонки недоступні в рецептах. Обхід: дублювати колонку перед партиціонуванням або збагатити після.
- **Inverse — Collecting:** зведення партиціонованого датасету назад у єдиний non-partitioned output.

### 8.8 Обмеження partitioning

- Files-based не керується вмістом колонки — лише розкладкою файлів (обхід: redispatch).
- SQL time-partition columns мають бути **strings**, не Date; діють ліміти engine (Vertica 1024/таблиця).
- Рецепти з кількома виходами потребують ідентичного партиціонування всіх виходів.
- **Partitioned ML models:** лише Visual ML з **Python backend** і **prediction**-моделями; Deep Learning і computer vision — **не підтримуються**.
- Column-based партиціонування потребує SQL- або підтримуваної NoSQL- (MongoDB, Cassandra, Elasticsearch) connection.

---

## 9. Best Practices

### Вибір connection

- (Базово) Для прототипів — uploaded files / filesystem; для продакшну — SQL або cloud storage.
- (Середній) Тримайте дані якомога ближче до обчислень: SQL-connection дозволяє **SQL pushdown** (in-database computation) → DSS не тягне дані на сервер.
- Розділяйте connections для різних оточень (dev/test/prod) і керуйте доступом через groups.

### Формати для великих даних

- (Середній) Для аналітики на Hadoop/cloud — **Parquet** (або ORC): columnar + стиснення + predicate pushdown, прискорення до x100.
- Уникайте CSV для великих managed-датасетів усередині Flow — лише для import/export та обміну.
- Дотримуйтесь **lowercase identifiers** для Parquet/Hive.

### Partitioning-стратегії

- (Просунуто) Партиціонуйте за **time** (day/month) для логів/подій з інкрементальним щоденним білдом — це класичний кейс економії.
- Discrete-партиціонування — коли обробка природно розбивається по категоріях (country, tenant).
- Не over-партиціонуйте: дрібні partitions (напр. hour на роки) → тисячі дрібних файлів і повільний listing; зважайте на engine-ліміти.
- Використовуйте **Build required datasets** + **Time range**/**Equals** залежності для коректної інкрементальної обробки.

### Уникнення дублювання даних

- (Середній) Не створюйте зайвих managed-копій: якщо рецепт лише читає — лишайте дані в джерелі.
- Налаштуйте **naming rules** на connections, щоб managed datasets різних проєктів не конфліктували в одному bucket/schema.
- Для незмінних reference-таблиць використовуйте external dataset замість копіювання.

---

## 10. Типові помилки

- **Плутати sample з повними даними.** Усе в Explore/prep — це sample. Метрики на семплі ≠ метрики на повному датасеті.
- **Biased sample.** «First records» на відсортованих даних спотворює статистику — беріть random/stratified.
- **Неправильний storage type.** Mistyped column ламає jobs при переході SQL↔Spark↔Hadoop. Перевіряйте схему після import.
- **Покладатися на auto-detect user-defined meanings.** Вимикає розпізнавання built-in meanings і гальмує.
- **CSV для big data у Flow.** Повільно, без pushdown — використовуйте Parquet/ORC.
- **Uppercase identifiers з Parquet/Hive.** Призводить до помилок case-sensitivity.
- **Over-partitioning.** Надто дрібні partitions → багато дрібних файлів, повільний listing, удар по engine-лімітах.
- **Очікувати доступ до partitioning-колонок після files-based redispatch.** Їх видаляють зі схеми — дублюйте заздалегідь.
- **Date-тип для SQL time-partition column.** Має бути string у DSS-синтаксисі.
- **Managed datasets різних проєктів в одній локації без naming rules** → перекриття та втрата даних.

---

## 11. Перехресні посилання

- `02-projects-flow-core-concepts.md` — Flow, datasets як вузли, recipes.
- `04-visual-recipes.md` — Sync, Prepare, інші рецепти, що читають/пишуть датасети та партиції.
- `05-data-preparation-prepare-recipe.md` — робота зі схемою/meanings у Prepare.
- `06-coding-recipes-code-envs-notebooks.md` — SQL/Python-рецепти, in-database computation.
- `09-scenarios-automation-metrics-checks.md` — детально metrics, checks, partitioned builds у scenarios.
- `10-mlops-deployment-api-node.md` — partitioned ML, продакшн storage.
- `13-governance-security-collaboration.md` — connection-level security, групи, доступи.
- `16-glossary.md` — терміни: dataset, connection, partition, storage type, meaning тощо.

---

> Джерела: doc.dataiku.com/dss/latest, knowledge.dataiku.com (DSS 14, перевірено червень 2026).
