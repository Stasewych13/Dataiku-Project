# 06. Coding recipes, code environments, notebooks

> Рівень: новачок → досвідчений
> Версії DSS: 13.x / 14.x
> Джерела: [doc.dataiku.com/dss/latest](https://doc.dataiku.com/dss/latest/), [developer.dataiku.com](https://developer.dataiku.com/), [knowledge.dataiku.com](https://knowledge.dataiku.com/)

---

## Вступ

Dataiku сповідує **white-box** підхід: окрім візуальних рецептів (див. [04-visual-recipes.md](04-visual-recipes.md) та [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md)) у Flow можна вставляти **code recipes** — рецепти, що виконують довільний код користувача (Python, R, SQL, Shell, Spark, Hive, Impala). Код інтегрується у той самий конвеєр: читає вхідні датасети/folders, пише вихідні, керується scenarios, бере участь у lineage.

Цей документ покриває чотири взаємоповʼязані теми:

1. **Code recipes** — типи, призначення, де виконуються, обмеження inputs/outputs.
2. **Dataiku Python API** всередині рецептів — пакет `dataiku` (`Dataset`, `Folder`, читання/запис, схема, змінні, секрети, `SQLExecutor2`).
3. **Code environments** — ізольовані Python/R-середовища з власними пакетами, containerized execution, відтворюваність.
4. **Notebooks, project libraries, Code Studios** — інтерактивна розробка та повторне використання коду.

> Передумови: бажано знати поняття Flow і recipe ([02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md)) та datasets/connections ([03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md)).

---

## Частина 1. Code recipes

### Спільні принципи (Базово)

Будь-який code recipe:

- читає **один або декілька** входів (datasets та/або folders) і пише **один або декілька** виходів;
- **outputs мають існувати як обʼєкти Flow до запуску**, але створюються під час налаштування рецепта. У діалозі створення ви «Select or create the output datasets». Коли вводите нове імʼя — DSS одразу створює **managed dataset**-placeholder (порожній), а запуск рецепта його наповнює;
- створюється з вкладки **Actions** датасета або через меню **+ Recipe**. Spark/Hadoop-рецепти живуть під **+ Recipe > Hadoop & Spark**;
- має вкладки **Input/Output**, редактор коду, **Advanced** (engine, code env, Spark config), кнопку **Validate** (перевірка коду/схеми без запуску) і **Run**;
- може мати **schema propagation** — оголошену вихідну схему, що поширюється вниз по Flow без виконання.

### Огляд типів і де вони виконуються

| Recipe | Де виконується | Outputs | In-database / distributed |
|---|---|---|---|
| **Python** | DSS server / container (одна машина, in-memory або streaming) | багато | ні |
| **R** | DSS server (одна машина, in-memory) | багато | ні |
| **SQL query** | у БД-джерелі (in-DB) **або** streaming через DSS | **1** | так, якщо той самий connection |
| **SQL script** | завжди повністю у БД | багато | завжди |
| **Shell** | shell-процес на DSS server | 1 (stdout) | ні |
| **PySpark** | Spark cluster (local / YARN / K8s) | багато | ні (Spark) |
| **Spark/R** (SparkR, sparklyr) | Spark cluster (Tier 2) | багато | ні (Spark) |
| **Spark-Scala** | Spark cluster | багато | ні (Spark) |
| **SparkSQL** | Spark engine над даними будь-якого backend | **1** | ні (Spark) |
| **Hive** | Hadoop cluster (HiveServer2 / Tez) | HDFS+Hive | так (cluster) |
| **Impala** | `impalad`-демони (MPP) | **1** | так (cluster) |

---

### Python recipe (Базово)

**Призначення:** найгнучкіший рецепт. Читає гетерогенні джерела (SQL-датасет + S3-датасет одночасно), застосовує довільну Python-логіку (pandas, scikit-learn, будь-який пакет із code env), пише будь-куди.

**Inputs/Outputs:** багато входів і виходів; підтримує partitioned datasets, folders, code environments, containerized execution.

**Де виконується:** локальний Python-процес на DSS server **або** у Kubernetes-контейнері (якщо налаштовано containerized execution). Це **одна машина**, не розподілено.

**Ключове обмеження:** in-memory підхід (`get_dataframe()`) вимагає, щоб дані влізли в RAM DSS server. Для більших обсягів — chunked read/write (див. Частину 2) або pushdown через `SQLExecutor2`.

```python
import dataiku

# read -> transform -> write
df = dataiku.Dataset("input_orders").get_dataframe()
df = df[df["amount"] > 0]
dataiku.Dataset("clean_orders").write_with_schema(df)
```

---

### R recipe (Середній)

**Призначення:** статистичні обчислення та маніпуляція даними в R; читає/пише DSS-датасети незалежно від backend.

**Де виконується:** локальний R-процес на DSS server (одна машина, in-memory). Потребує R code environment.

```r
library(dataiku)

df <- dkuReadDataset("input_orders", samplingMethod = "head", nbRows = 100000)
df <- df[df$amount > 0, ]
dkuWriteDataset(df, "clean_orders")
```

> Примітка щодо API імен: у R-рецептах класичні функції — `dkuReadDataset()` / `dkuWriteDataset()` (для динамічної схеми `dkuWriteDatasetSchema()` / `dkuWriteDatasetWithSchema()`). У старіших templates трапляються синоніми `read.dataset()` / `write.dataset()`. Завжди беріть готовий шаблон, який DSS вставляє в новий рецепт.

---

### SQL recipes (Середній)

Тут найбільше плутанини, тож розберемо детально. Є **два code-рецепти** — **SQL query** і **SQL script** — і окреме поняття **in-database execution** (як, а не що).

#### SQL query recipe

**Призначення:** рекомендований за замовчуванням. Ви пишете **один `SELECT`**, а DSS бере на себе «сантехніку» — CREATE/DROP таблиці, `INSERT INTO ... SELECT`.

**Inputs/Outputs:**
- входи — «one or several SQL datasets»; **усі входи мають бути SQL table datasets** (external або managed) і **зазвичай — в одному database connection**;
- вихід — **рівно один** dataset.

**Де виконується:**
- **повністю у БД (in-database)**, якщо вихід — SQL-таблиця **на тому самому connection**, що й входи: DSS переписує запит у `INSERT INTO ... SELECT`, нуль руху даних;
- **streaming через DSS server**, якщо вихід на іншому connection або non-SQL backend: DSS стрімить результати `SELECT` із джерела на DSS server і пише у target.

**Validate** лише просить БД розпарсити запит (не виконує) — дешево; також може інферити вихідну схему без запуску.

**Обмеження:** не підтримує CTE (`WITH`) — DSS не вміє коректно вставити `INSERT` у такий запит; не всі geometry-типи; деякі складні конструкції не переписуються. У таких випадках — SQL script.

```sql
-- SQL query recipe: лише SELECT, решту робить DSS
SELECT
    customer_id,
    COUNT(*)          AS n_orders,
    SUM(amount)       AS total_amount
FROM   orders
WHERE  amount > 0
GROUP  BY customer_id
```

#### SQL script recipe

**Призначення:** коли DSS не може переписати з одного `SELECT` — CTE/`WITH`, non-native типи (PostGIS), **кілька виходів**, або `UPDATE`/`MERGE`. Ви пишете **повний скрипт** самі.

**Inputs/Outputs:** входи — SQL table datasets в одному connection; **виходи — багато** (виходи лише у БД).

**Де виконується:** **завжди повністю у БД**; дані ніколи не йдуть через DSS.

**Ви відповідаєте за все DDL/DML:** CREATE/DROP, INSERT. Для ідемпотентності завжди майте `TRUNCATE`/`DELETE` перед вставкою.

```sql
-- SQL script recipe: повний контроль, кілька statements
DROP TABLE IF EXISTS "${DKU_DST_summary}";
CREATE TABLE "${DKU_DST_summary}" AS
WITH ranked AS (
    SELECT customer_id, amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
    FROM   "${DKU_SRC_orders}"
)
SELECT customer_id, amount FROM ranked WHERE rn = 1;
```

> `${DKU_DST_xxx}` / `${DKU_SRC_xxx}` — змінні, якими DSS підставляє фактичні імена таблиць виходів/входів. Не хардкодьте імена таблиць.

#### In-database execution (концепція)

Це **спосіб** виконання, не окремий рецепт:
- **in-database** — обчислення відбувається всередині рушія БД (`INSERT INTO ... SELECT`); найефективніше, без руху даних. Досягається SQL query/script (один connection) і **візуальними рецептами**, чий engine резолвиться у SQL;
- **streaming / in-memory** — SQL query стрімить результати на DSS server; Python/R вантажать дані в памʼять DSS server.

> Cross-connection: обидва SQL-рецепти можуть обійти правило одного connection через **Advanced > Allow SQL across connections** (потрібні fully-qualified імена; можливий streaming).

---

### Shell recipe (Середній)

**Призначення:** запуск довільних shell-скриптів у Flow (виклик зовнішніх утиліт, file ops, оркестрація).

**Inputs/Outputs:** до **одного входу** (подається у stdin як tab-separated CSV) і **одного виходу** (читається зі stdout, tab-separated CSV з авто-інференсом схеми). Решта датасетів доступні через env-змінні.

**Де виконується:** shell-процес, запущений DSS (інтерпретатор `sh` за замовчуванням; override через shebang або **Advanced**).

**Доступ до датасетів через env-змінні:** `DKU_INPUT_X_DATASET_ID`, `DKU_OUTPUT_X_DATASET_ID` (індекс із нуля) тощо — повний перелік у вкладці **Variables** поруч зі скриптом.

**Обмеження:** параметри не передаються через командний рядок; I/O — лише stdin/stdout (tab-separated CSV).

---

### Spark recipes (Просунуто)

**Загальне:** Spark-рецепти виконуються на **Spark cluster** — local Spark на DSS server, **YARN** (Hadoop), або **Kubernetes** (Elastic AI / Spark-on-K8s). Адміни визначають **named Spark configurations** (Administration > General Settings), а кожен рецепт обирає базову конфігурацію у **Advanced**.

- **HDFS і S3** датасети повністю використовують розподіленість Spark (native partition splitting). Інші — «спрощений reader»: одна Spark-partition в одному потоці, ліміт 2 ГБ на partition; робіть `df.repartition(X)`.
- Overhead Spark **немалий** — використовуйте Spark лише коли дані перевищують памʼять однієї машини.

**PySpark recipe** — Spark у Python (DataFrame API):

```python
import dataiku
from dataiku import spark as dkuspark
from pyspark import SparkContext
from pyspark.sql import SQLContext

sc = SparkContext()
sqlContext = SQLContext(sc)

df = dkuspark.get_dataframe(sqlContext, dataiku.Dataset("input_ds"))
result = df.groupBy("country").count()
dkuspark.write_with_schema(dataiku.Dataset("output_ds"), result)  # перезаписує схему
```

**SparkSQL recipe** — SQL-запит, що виконується на Spark engine над даними **будь-якого backend** (кожен вхід — SparkSQL-таблиця з імʼям датасета); **один вихід**. Відрізняється від SQL recipe тим, що SQL recipe пушить роботу у справжню БД, а SparkSQL — у Spark.

**Spark/R recipe** (SparkR + sparklyr) — **Tier 2** support; **не працює** на Cloudera та Elastic AI / Spark-on-K8s.

**Spark-Scala recipe** — Spark у Scala (native), через `DataikuSparkContext`. Підтримує append/overwrite та явні partitions.

---

### Hive та Impala recipes (Просунуто)

**Hive recipe** — обчислення HDFS-датасетів через **HiveQL**; великомасштабний SQL на Hadoop. Входи/виходи — HDFS-датасети, синхронізовані з Hive metastore (формати: CSV, Parquet, ORC, Avro, RC/Sequence). Виконується на Hadoop через **HiveServer2** (рекомендовано; Hive CLI режими deprecated), рушій Tez/MapReduce. DSS створює **EXTERNAL** Hive-таблиці над HDFS-фолдерами і авто-оновлює metastore при зміні схеми.

**Impala recipe** — інтерактивний аналітичний SQL над HDFS; виконується на `impalad`-демонах (MPP-рушій, не MapReduce). **Один вихід** з авто-керованою схемою. Швидший за Hive, але вужча підтримка форматів і **не підтримує complex types**. Без user impersonation — пише як system-користувач `impala`.

---

## Частина 2. Dataiku Python API всередині recipes

Пакет `dataiku` автоматично доступний у Python-рецептах і notebooks (прив'язаний до контексту проєкту). Нижче — найважливіші виклики з коректним синтаксисом.

### Dataset: handle, читання, запис (Базово)

```python
import dataiku

ds    = dataiku.Dataset("my_dataset")                  # у тому самому проєкті
other = dataiku.Dataset("OTHERPROJECT.their_dataset")  # cross-project
```

**Читання у pandas — `get_dataframe()`** (вантажить весь/семпльований датасет у памʼять):

```python
df = ds.get_dataframe()                                       # все
df = ds.get_dataframe(columns=["id", "amount"],               # підмножина колонок
                      sampling="head", limit=100000)          # семплінг
# infer_with_pandas=False -> типи беруться зі схеми DSS (надійніше для дат/типів)
df = ds.get_dataframe(infer_with_pandas=False)
```

**Запис із pandas:**

```python
out = dataiku.Dataset("output_ds")
out.write_with_schema(df)   # пише дані І перезаписує схему під DataFrame (звичайний шлях)
out.write_dataframe(df)     # пише дані, НЕ змінюючи наявну схему (стабільна схема)
out.write_dataframe(df, infer_schema=True)  # інферить і встановлює схему з df
```

> **Gotcha:** `write_with_schema(df)` **перезаписує схему щоразу при запуску** — downstream-схема залежить від того, який вигляд має df у рантаймі. Якщо потрібна фіксована/контрольована схема — задайте її явно (`write_schema`) і використовуйте `write_dataframe`.

### Великі дані: chunked read/write (Просунуто)

`get_dataframe()` **не має** `chunksize`. Для читання великих даних — `iter_dataframes()`:

```python
ds = dataiku.Dataset("big_dataset")
for chunk_df in ds.iter_dataframes(chunksize=10000):   # ≤ 10k рядків на ітерацію
    process(chunk_df)
```

**Row-by-row читання — `iter_rows()`** (мінімум памʼяті; рядок — dict-like):

```python
for row in dataiku.Dataset("my_dataset").iter_rows():
    print(row["customer_id"], row["amount"])
```

**Streaming-запис — `get_writer()`** (для великих виходів). Схему **треба задати ДО** відкриття writer:

```python
out = dataiku.Dataset("output_ds")
out.write_schema([
    {"name": "origin", "type": "string"},
    {"name": "count",  "type": "int"},
])
with out.get_writer() as writer:               # with -> авто-close (flush)
    for origin, count in compute_rows():
        writer.write_row_array([origin, count])  # значення у порядку колонок схеми
```

Методи writer: `write_row_array(values)`, `write_tuple(values)`, `write_row_dict({...})`, `write_dataframe(df)`, `close()`.

> **Два головні gotchas:** (1) `get_writer()` вимагає **попередньо встановленої схеми** — не змінює її по ходу; (2) завжди **`close()`** (або `with`) — інакше останній буфер рядків **не запишеться**.

Memory-bounded copy/transform-ідіома:

```python
inp = dataiku.Dataset("input_ds")
out = dataiku.Dataset("output_ds")
out.write_schema(inp.read_schema())            # копіюємо схему
with out.get_writer() as writer:
    for df in inp.iter_dataframes(chunksize=10000):
        writer.write_dataframe(transform(df))
```

### Робота зі схемою (Середній)

```python
schema = ds.read_schema()
# -> [{"name": "id", "type": "bigint"}, {"name": "name", "type": "string"}, ...]

out.write_schema([                              # задати схему явно (перед get_writer)
    {"name": "id",    "type": "int"},
    {"name": "label", "type": "string"},
])
out.write_schema_from_dataframe(df)             # схема з df без запису даних
```

Типи DSS: `string`, `int`, `bigint`, `double`, `float`, `boolean`, `date` та ін.

### dataiku.Folder — managed folders (Середній)

```python
folder = dataiku.Folder("my_folder")           # за імʼям або ID
```

| Метод | Призначення |
|---|---|
| `get_path()` | локальний FS-шлях — **лише для filesystem-folders**; падає на S3/GCS/Azure/HDFS |
| `list_paths_in_partition()` | список відносних шляхів (усі backends) |
| `get_download_stream(path)` | file-like для **читання** файлу (cloud-safe) |
| `upload_stream(path, f)` / `upload_data(path, data)` | **запис** файлу |
| `get_writer(path)` | інкрементальний запис одного файлу |
| `read_json` / `write_json` | JSON-хелпери |

```python
import io, pickle

# читання (backend-agnostic):
with folder.get_download_stream("models/model.pkl") as s:
    model = pickle.load(s)

# запис:
buf = io.BytesIO(); pickle.dump(model, buf)
folder.upload_data("models/model.pkl", buf.getvalue())
```

> **Gotcha:** не використовуйте `get_path()` для cloud-folders — лише `get_download_stream`/`upload_stream`/`list_paths_in_partition`.

### Custom variables (Середній)

`dataiku.get_custom_variables()` повертає **dict усіх розгорнутих змінних** (instance/global + project + recipe), значення — **рядки**:

```python
variables = dataiku.get_custom_variables()
country = variables["country_name"]            # каст за потреби: int(...), ...
df = df[df["country"] == country]
```

Запис project-змінних — через API client:

```python
client  = dataiku.api_client()
project = client.get_default_project()
v = project.get_variables()                     # {"standard": {...}, "local": {...}}
v["standard"]["last_run_country"] = "FR"
project.set_variables(v)
```

### SQLExecutor2 — pushdown SQL із Python (Просунуто)

Замість витягувати все у pandas — виконуйте агрегацію **у БД** і повертайте лише результат:

```python
from dataiku.core.sql import SQLExecutor2

executor = SQLExecutor2(connection="my_postgres")          # або dataset=...
df = executor.query_to_df(
    "SELECT country, COUNT(*) AS n FROM orders GROUP BY country"
)
# для INSERT/UPDATE — post_queries=['COMMIT']
```

Також: `query_to_iter()` (стрім результатів), `exec_recipe_fragment()` (динамічно згенерований SQL у SQL-рецепті), `pre_queries`/`post_queries`. Аналоги для Hadoop: `HiveExecutor`, `ImpalaExecutor`.

### Секрети / credentials у коді (Просунуто)

**Не хардкодьте паролі/токени.** Використовуйте **User Secrets** через API client:

```python
client = dataiku.api_client()
auth_info = client.get_auth_info(with_secrets=True)
secret_value = None
for secret in auth_info["secrets"]:
    if secret["key"] == "credential-for-my-api":
        secret_value = secret["value"]
        break
```

User Secrets задаються per-user у **Profile > Credentials** (адмін визначає ключі). Для connection-credentials використовуйте сам connection (DSS підставляє creds), а не власні рядки підключення.

### Збереження зовнішньої моделі (MLflow import) (Просунуто)

Імпорт MLflow-моделі як DSS **Saved Model** (детальніше — [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md)):

```python
client  = dataiku.api_client()
project = client.get_default_project()

sm = project.create_mlflow_pyfunc_model(name="my_model",
                                        prediction_type="REGRESSION")
v = sm.import_mlflow_version_from_path(version_id="v1",
                                       path="/path/to/mlflow_model_dir",
                                       code_env_name="my-mlflow-env")
v.set_core_metadata(target_column_name="target")
v.evaluate("eval_dataset")
sm.set_active_version(v.version_id)
```

> Імена параметрів злегка різняться між версіями DSS — звіряйтеся з API reference вашої версії. Стабільні entry-points: `create_mlflow_pyfunc_model(...)` → `import_mlflow_version_from_path(...)` (або `..._from_managed_folder(...)` для cloud-folders).

### Доступ до inputs/outputs рецепта (Середній)

```python
from dataiku import recipe
first_input = recipe.get_input(object_type="DATASET")     # handle
outputs     = recipe.get_outputs(object_type="DATASET")   # список handles
```

У **plugin/custom** рецептах — `from dataiku.customrecipe import get_input_names_for_role, get_output_names_for_role, get_recipe_config` (див. [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md)).

---

## Частина 3. Code environments

### Що це і навіщо (Базово)

**Code environment** — «standalone and self-contained environment to run Python or R code». Кожен env має **власний набір пакетів** і (для Python) **власну версію Python** (DSS 14 підтримує 3.9–3.14). Призначення:

- **ізоляція** — різні проєкти можуть мати різні версії пакетів без конфліктів;
- **відтворюваність** — пінінг версій, версіонування самих envs, перенесення між нодами;
- **керування пакетами** — централізоване встановлення.

У **кожному місці**, де виконується Python/R-код (recipe, notebook, scenario, webapp, ML, plugin), можна **обрати, який code env** використати.

### builtin (base) environment — і чому не для продакшну (Базово)

**builtin env** — Python/R, що поставляється разом із DSS. Містить лише **core**-пакети `dataiku` + **Jupyter**-пакети (мінімум для роботи `dataiku` і notebooks). Чому **не варто** використовувати для реальної роботи: він спільний на весь instance, base-пакети заблоковані (не можна довільно додавати/пінити версії), зміни вплинули б на весь instance. **Best practice:** створюйте окремі managed code envs під проєкт/задачу.

### Створення code env (Середній)

**Administration > Code Envs > New Python env** (або **New R env**) → задайте **глобально-унікальний identifier** (A–Z, a–z, цифри, дефіси) → для Python оберіть версію → **CREATE**. DSS авто-встановлює мінімальний/base-набір.

**Вкладка «Packages to install»** має дві секції:
- **Base Packages** (не редагуються) — **Core packages** (без них `dataiku` не працює) + **Jupyter packages** (без них недоступний Jupyter для цього env);
- **Requested Packages** (редагуються) — ваші пакети.

Робочий цикл: відредагувати **Requested packages** → **Save and update** (DSS завантажує/встановлює) → перевірити точні версії у вкладці **Actually installed packages** → логи у **Logs**.

### Керування пакетами: pip vs conda (Середній)

```text
# Requested packages (Python, синтаксис requirements.txt):
pandas==2.1.4
scikit-learn>=1.3
requests
xgboost
```

- **pip / virtualenv** — за замовчуванням і **рекомендовано** Dataiku.
- **conda** — увімкнути чекбоксом **Use conda** при створенні env. **Tier 2** support, зазвичай **не рекомендовано** (часті backward-incompatible зміни репозиторіїв; у більшості випадків conda-репозиторії **платні**). Conda потребує **двох списків**: conda-пакети + «regular» (pip/R) для решти. Рекомендовано для R, бо компіляція R-пакетів із сорсів повільна.
- **R packages** — таблиця «імʼя + опціональна мінімальна версія», з CRAN-дзеркал.

Кастомні pip/CRAN-репозиторії: **Administration > Settings > Misc** (Code Envs) глобально, або per-env через **Extra Options**.

### Призначення code env (Середній)

Три рівні з пріоритетом: **instance default → project default → recipe/notebook override**.

- **Instance default:** Administration > Settings > Misc > Global defaults.
- **Project default:** Project > **... (More options) > Settings > Code env selection**.
- **Recipe override:** у рецепті — вкладка **Advanced > Python/R environment** → dropdown **«Inherit project default»** або **«Select an environment»** (конкретний env). Має пріоритет при виконанні.
- **Notebook / webapp / ML / plugin** — кожен має свій вибір env (для notebook потрібен env із увімкненим Jupyter support).

### Containerized execution (Docker / Kubernetes) (Просунуто)

DSS називає це **Elastic AI computation** — pushdown обчислень на Kubernetes-кластери. У контейнерах можуть виконуватися: Python/R рецепти й notebooks, візуальні рецепти на DSS engine, Spark-рецепти, тренування/скоринг ML.

**Архітектура:**
1. **Image Build Configuration** — як DSS будує/публікує образи (base image + registry).
2. **Containerized Execution Configuration** — runtime: **resource restriction keys** (CPU/memory requests & limits), kubectl context, namespace для resource quota, permissions (які групи можуть використовувати).
3. **Cluster Selection** — який K8s-кластер (EKS/AKS/GKE або unmanaged).

**Звʼязок з code envs:** у вкладці **Containerized execution** code env-у обираєте, для яких container exec configurations збудувати Docker-образ (**один образ на (code env × base image)**). Оновлення env авто-перебудовує образи. CLI: `./bin/dssadmin build-container-exec-code-env-images --all`. **Після кожного апгрейду DSS** — перебудувати base images, потім code env images. GPU: `--with-cuda`.

> **Несумісність:** containerized execution **не підтримує non-managed code environments**, кастомні інтерпретатори з PATH, додаткові PYTHONPATH-записи чи вручну додані пакети.

### Відтворюваність і перенесення між нодами (Просунуто)

- **Пінінг:** `pkg==x.y.z` у Requested; вкладка **Actually installed packages** фіксує повний резолвлений набір.
- **Версіонування:** версії code env **не видаляються** — можна відкотитися до попередньої версії проєкту з тим самим env.
- **Design → Automation/Deployer:** code envs приходять у **bundles**. Перед активацією bundle роблять **Preload** (Bundles > select > Preload): DSS сканує потрібні envs, порівнює з наявними на automation-ноді й (за налаштуванням) створює/оновлює їх. Envs матчаться за **мовою та імʼям**, потім порівнюються requirements. Детальніше — [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md).

### Non-managed code environments (Просунуто)

Коли потрібна повністю кастомна інсталяція Python/R: вказуєте DSS на наявний external install/virtualenv (Python: `bin/python` + `bin/pip`; R: `bin/R`). Користувач сам підтримує інсталяцію. **Не підтримуються для containerized execution.**

---

## Частина 4. Notebooks, project libraries, Code Studios

### Jupyter notebooks у DSS (Базово)

DSS вбудовує **Jupyter notebooks** для **дослідницької/експериментальної** роботи кодом — на відміну від **Flow**, де живе продакшн. Ключовий принцип: **notebooks — для exploration, НЕ для продакшну**; щоб винести логіку в конвеєр — конвертуйте у **recipe**.

**Типи:**
- **Code notebooks** — Python, R (також Scala/Spark через kernels);
- **SQL notebooks** — спеціальне інтерактивне середовище для SQL (не Jupyter);
- **Predefined / Containerized** notebooks.

**Інтеграція з DSS:** у Python-notebook доступний весь `dataiku` API без API-ключа; `dataiku.api_client()` успадковує права користувача. Дані — `dataiku.Dataset("name").get_dataframe()` / `.write_with_schema(df)`.

**Створення:** з датасета — **Actions > Lab > Notebook > Python**; або зі списку **Notebooks**.

**SQL notebook** привʼязаний до **одного SQL-connection**; має cells+history, Table View (перші 1000 рядків), Chart, AI SQL-генерацію.

### Конвертація notebook ↔ recipe (Середній)

- **Notebook → recipe:** у відкритому notebook — кнопка **Create Recipe** (у верхній панелі) → оберіть тип (напр. Python recipe). Створює code recipe у Flow з inputs/outputs. Рекомендований шлях деплою дослідницького коду.
- **Recipe → notebook:** у редакторі code-рецепта — дія **Edit in Notebook** для інтерактивної ітерації; зміни зберігаються назад у рецепт.

**Обмеження notebooks:**
- **Scenarios НЕ запускають notebooks напряму** — автоматизація працює з **recipes**. Будь-яка логіка для розкладу/продакшну має стати рецептом.
- Завантажений notebook тримає **kernel у памʼяті** до явного unload (File > Close and Halt, або жовтий хрестик у списку). DSS авто-вивантажує неактивні kernels за idle-таймаутом.
- Вибір code env / kernel; у containerized notebook файли робочої директорії губляться при unload — персистьте через **managed folders**.

### Project libraries (lib/python, lib/R) (Середній)

**Project library** — місце для **багаторазового коду**, редагується у **Libraries editor** (меню **`</>` (Code) > Libraries**).

- Дві top-level папки: **`python`** і **`R`**. Усі Python source folders додаються до **`PYTHONPATH`** проєкту — модулі імпортуються з будь-якого code capability.
- Subfolder у Python source folder має містити `__init__.py`.
- На диску: `DATA_DIR/config/projects/PROJECT_KEY/lib`.

**Імпорт модуля у рецепті/notebook:**

```python
# lib/python/analyticfunctions.py визначає build_model()
from analyticfunctions import build_model
model = build_model()
```

Це канонічний спосіб **reuse коду між рецептами** — пишете раз у `lib/python`, імпортуєте всюди.

**Reuse між проєктами:** у Libraries editor проєкта-споживача редагуєте **`external-libraries.json`** → додаєте key проєкта-джерела у `"importLibrariesFromProjects"`. Є також **Global Shared Code** (на весь instance), але офіційно **не рекомендовано** через конфлікти імен — краще per-project.

**Імпорт з зовнішнього Git:** Libraries editor > **Git > Import from Git…** (repo URL, branch/tag/commit, subpath, target path). Синхронізація — **Git > Manage repositories…**: **Reset from remote HEAD** (це справжній `git reset` — локальні зміни губляться), **Commit and push…**, **Update all references**.

**Git-синхронізація:** DSS має вбудований version control — кожна зміна в UI (recipe, library code, dashboards) пишеться у внутрішній Git проєкта. SSH-ключі — **Profile > Credentials > SSH**; governance — **Administration > Settings > Git**. Детальніше — [13-governance-security-collaboration.md](13-governance-security-collaboration.md).

### Code Studios (коротко) (Просунуто)

**Code Studios** — персональні **контейнеризовані dev-середовища** в DSS: web-IDE (**VS Code, JupyterLab, RStudio Server**) і framework-и для webapps (**Streamlit, Dash, Gradio, Voilà**), кожне у власному контейнері (Docker/K8s). Будуються на **code environment**, мають доступ до **project library**, можуть редагувати/запускати library, recipe, notebook, webapp-код проєкта. Детально про webapps — у [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md).

---

## Best practices (зведено)

- **(Базово) Notebooks — для дослідження, recipes — для продакшну.** Конвертуйте notebook → recipe (**Create Recipe**), щоб логіка потрапила у Flow і scenarios.
- **(Базово) Власні code envs, не builtin.** Окремий managed env під проєкт/задачу; пінте версії (`pkg==x.y.z`); звіряйте **Actually installed packages**.
- **(Середній) Ідемпотентність.** Запуск рецепта має давати той самий результат при повторі. У SQL script завжди `TRUNCATE`/`DELETE` перед `INSERT`. Не накопичуйте дублі через `append`-логіку без контролю.
- **(Середній) Partition-aware код.** У partitioned-рецептах читайте/пишіть лише цільові partitions; у Python використовуйте змінні partition (`dataiku` підставляє dimension-значення). Не сканьте весь датасет, коли потрібна одна partition.
- **(Середній) Продуктивність — не збирайте все у pandas.** Для агрегацій/фільтрів робіть **pushdown у БД** через `SQLExecutor2.query_to_df()` або SQL-рецепт; для великих обсягів — `iter_dataframes(chunksize=...)` + `get_writer()` (chunked write).
- **(Середній) Reuse через project library.** Спільну логіку — у `lib/python`, імпортуйте у рецептах; не копіюйте код між рецептами.
- **(Просунуто) Секрети — через User Secrets** (`client.get_auth_info(with_secrets=True)`), ніколи не хардкодьте; для БД — використовуйте connection, а не власні рядки підключення.
- **(Просунуто) Логування.** Використовуйте `print`/`logging` — вивід йде у **Log** рецепта (Actions > Other actions > **View logs**, або вкладка Log при запуску). Структуруйте повідомлення (етап, к-ть рядків) для діагностики у scenarios.
- **(Просунуто) Containerized execution** для важких/ресурсних рецептів — ізоляція ресурсів і масштабування; не використовуйте non-managed envs у контейнерах.

---

## Типові помилки

- **`get_writer()` без попередньо встановленої схеми** → помилка/некоректний запис. Спершу `write_schema(...)`, потім writer.
- **Забутий `writer.close()`** (без `with`) → останній буфер рядків не записується, дані «зникають».
- **`write_with_schema(df)` там, де потрібна стабільна схема** → downstream-схема «пливе» при кожному запуску. Використовуйте `write_dataframe(df)` з явною схемою.
- **CTE (`WITH`) у SQL query recipe** → не переписується; перенесіть у **SQL script recipe**.
- **SQL query з входами/виходом на різних connections** без усвідомлення → DSS перейде на повільний streaming через DSS server замість in-database. Тримайте один connection або свідомо вмикайте cross-connection.
- **`Folder.get_path()` на cloud-folder (S3/GCS/Azure)** → падає. Використовуйте `get_download_stream`/`upload_stream`.
- **`get_dataframe(chunksize=...)`** → такого параметра немає; використовуйте `iter_dataframes(chunksize=...)`.
- **Збирання великого датасета у pandas** на DSS server → OOM. Pushdown у БД (`SQLExecutor2`) або chunked.
- **Очікування, що scenario запустить notebook** → ні; конвертуйте у recipe.
- **Робота у builtin env** і встановлення туди пакетів → неможливо/впливає на весь instance; створіть власний env.
- **Хардкод credentials у коді рецепта** → витік секретів у Git проєкта; використовуйте User Secrets / connection.
- **`get_custom_variables()` без касту** → значення завжди **рядки**; `int(...)`/`float(...)` за потреби.

---

## Перехресні посилання

- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — Flow, recipe, проєкти.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — datasets, connections, partitioning.
- [04-visual-recipes.md](04-visual-recipes.md) — візуальні рецепти та engines (порівняння з code).
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — Prepare recipe.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — ML, Lab, Saved Models.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — scenarios (запуск рецептів за розкладом).
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — bundles, deployment, MLflow import.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — `dataikuapi`, `api_client()`, керування обʼєктами ззовні.
- [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md) — webapps, Code Studios детально.
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — Git, секрети, permissions.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — custom recipes, plugins.
- [16-glossary.md](16-glossary.md) — глосарій термінів.
