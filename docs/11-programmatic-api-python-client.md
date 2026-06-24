# 11. Програмний доступ: Python client та REST API

> Документ про те, як керувати Dataiku DSS **програмно**: зсередини (пакет `dataiku`) і ззовні (пакет `dataikuapi` / `DSSClient`, а також прямий REST API по HTTP). Покриває аутентифікацію (personal / global / project API keys, OAuth2/SSO), базові CRUD-операції над projects / datasets / recipes / scenarios / managed folders, запуск scenarios та jobs, читання metrics, керування variables, адмін-операції (users / groups / connections / code envs), роботу з даними (schema, chunked read, write, SQL), ML через API, scoring через **API node**, прямі HTTP-виклики (`curl`), CLI-інструменти (`dssadmin`, `dsscli`, `apinode-admin`, Fleet Manager) та best practices (секрети, retry, dev/prod, CI/CD). Версії DSS **13.x** і **14.x**. Технічні/UI/код терміни залишені в **оригіналі (англійською)**.

---

## Зміст

1. [Вступ: два світи Python API](#1-вступ-два-світи-python-api)
2. [`dataiku` (internal API): стисло](#2-dataiku-internal-api-стисло)
3. [`dataikuapi` / `DSSClient`: підключення ззовні](#3-dataikuapi--dssclient-підключення-ззовні)
4. [Аутентифікація: personal / global / project keys, OAuth2/SSO](#4-аутентифікація)
5. [Базові операції: projects, datasets, recipes, folders](#5-базові-операції)
6. [Variables: project-level та instance-level](#6-variables)
7. [Запуск scenarios та jobs](#7-запуск-scenarios-та-jobs)
8. [Читання metrics та checks](#8-читання-metrics-та-checks)
9. [Адмін-операції: users, groups, connections, code envs](#9-адмін-операції)
10. [Робота з даними через API: schema, read, write, SQL](#10-робота-з-даними-через-api)
11. [ML через API: тренування, Saved Models, deploy](#11-ml-через-api)
12. [API node: real-time scoring (APINodeClient)](#12-api-node-real-time-scoring)
13. [REST API напряму (HTTP / curl)](#13-rest-api-напряму-http--curl)
14. [CLI-інструменти: dssadmin, dsscli, apinode-admin, Fleet Manager](#14-cli-інструменти)
15. [Помилки, пагінація, rate limiting, retry, ідемпотентність](#15-помилки-пагінація-rate-limiting-retry-ідемпотентність)
16. [Best practices](#16-best-practices)
17. [Типові помилки](#17-типові-помилки)
18. [Перехресні посилання](#18-перехресні-посилання)
19. [Джерела](#19-джерела)

> **Орієнтир.** Деталі роботи зсередини DSS (code recipes, code envs, notebooks) — у `06-coding-recipes-code-envs-notebooks.md`. Деталі deployment (bundles, Project/API Deployer, lifecycle) — у `10-mlops-deployment-api-node.md`. Тут — повний огляд **програмного інтерфейсу** як такого.

---

## 1. Вступ: два світи Python API

Dataiku має **два окремі Python-пакети**, і їх не можна плутати. (Базово)

| | `dataiku` (**internal API**) | `dataikuapi` (**public / REST API**) |
|---|---|---|
| Де виконується | **ВСЕРЕДИНІ** DSS: code recipes, notebooks, webapps, scenarios, Code Studios | **ЗЗОВНІ** DSS: будь-який Python-процес (ваш ноутбук, CI/CD, інша система) |
| Точка входу | `dataiku.Dataset(...)`, `dataiku.Folder(...)`, `dataiku.api_client()` | `dataikuapi.DSSClient(host, api_key)` |
| Аутентифікація | автоматична (контекст користувача/рецепта) | потрібен **API key** (або OAuth2 token) |
| Повертає DataFrame напряму | **так** (`get_dataframe()`, потрібен pandas) | **ні** — стрімить сирі рядки, DataFrame будуєте самі |
| Типове призначення | читати/писати дані рецепта, бізнес-логіка | оркестрація, адміністрування, інтеграції, CI/CD |

Ключовий нюанс: **зсередини DSS доступні обидва пакети**. Тобто з notebook/recipe можна імпортувати `dataiku` для роботи з даними і одночасно отримати **повноцінний REST-клієнт** без хардкоду ключа:

```python
import dataiku
client = dataiku.api_client()        # авто-автентифікований DSSClient (той самий, що dataikuapi.DSSClient)
project = client.get_default_project()  # поточний проєкт контексту
```

Тобто `dataiku.api_client()` — це міст: він повертає об'єкт того ж класу `DSSClient`, що й `dataikuapi.DSSClient(host, key)`, але без ключа. Усе, що нижче в розділах про `DSSClient`, працює і через `dataiku.api_client()`.

---

## 2. `dataiku` (internal API): стисло

Детально — у `06-coding-recipes-code-envs-notebooks.md`. Тут лише головне, щоб підкреслити **відмінність** від `dataikuapi`. (Базово)

```python
import dataiku
import pandas as pd

# Читання даних рецепта (повертає pandas.DataFrame напряму)
ds = dataiku.Dataset("input_dataset")
df = ds.get_dataframe()                  # вантажить усе в пам'ять — для великих даних див. §10

# Запис
out = dataiku.Dataset("output_dataset")
out.write_with_schema(df)

# Managed folder
folder = dataiku.Folder("FOLDER_ID")
with folder.get_download_stream("model.pkl") as f:
    data = f.read()

# Змінні проєкту/інстансу (read-only, з контексту)
variables = dataiku.get_custom_variables()  # завжди рядки!

# REST-клієнт без ключа
client = dataiku.api_client()
```

**Чим `dataiku` принципово відрізняється від `dataikuapi`:**

- `dataiku.Dataset(...).get_dataframe()` повертає DataFrame **напряму**; у `dataikuapi` такого методу немає — там лише `iter_rows()` (сирі списки значень).
- `dataiku` працює в контексті **поточного користувача/рецепта** — ключ не потрібен; `dataikuapi` вимагає API key.
- `dataiku` має зручні I/O-обгортки (`SQLExecutor2`, `Folder.get_path()` тощо), оптимізовані під виконання на самому DSS server.
- `dataiku.get_custom_variables()` повертає значення **завжди як рядки** — кастуйте (`int(...)`, `float(...)`).

> Якщо код може виконуватись і зсередини, і ззовні — пишіть його через `dataiku.api_client()` (зсередини) / `dataikuapi.DSSClient(...)` (ззовні), щоб логіка оркестрації була однаковою.

---

## 3. `dataikuapi` / `DSSClient`: підключення ззовні

### 3.1 Встановлення (Базово)

```bash
pip install dataiku-api-client
```

> Пакет називається `dataiku-api-client` (з дефісами) при встановленні, а **імпортується** як `dataikuapi` (без дефісів). Версію клієнта бажано тримати **сумісною з версією DSS** (13.x / 14.x).

### 3.2 Базове підключення (Базово)

```python
import dataikuapi

host = "https://dss.mycompany.com"     # БЕЗ завершального /public/api — лише корінь інстансу
api_key = "YOUR_PERSONAL_API_KEY"

client = dataikuapi.DSSClient(host, api_key)

# Перевірка зв'язку
print(client.list_project_keys())
```

Повна сигнатура конструктора:

```python
dataikuapi.DSSClient(host, api_key=None, internal_ticket=None,
                     extra_headers=None, no_check_certificate=False,
                     client_certificate=None, jwt_bearer_token=None)
```

### 3.3 Self-signed TLS / корпоративні сертифікати (Середній)

```python
# Варіант 1 — параметр конструктора
client = dataikuapi.DSSClient(host, api_key, no_check_certificate=True)

# Варіант 2 — на нижньому рівні requests.Session
client = dataikuapi.DSSClient(host, api_key)
client._session.verify = False         # вимкнути перевірку сертифіката
# або вказати свій CA-bundle:
client._session.verify = "/path/to/corporate-ca.pem"
```

> `no_check_certificate=True` зручний для dev, але **небезпечний у проді** — краще додати корпоративний CA-bundle.

### 3.4 Зсередини DSS — без ключа (Базово)

```python
import dataiku
client = dataiku.api_client()          # той самий DSSClient, але автентифікований контекстом
```

---

## 4. Аутентифікація

Dataiku розрізняє **три типи API keys** на design/automation node плюс окремий концепт ключів на **API node**. (Середній)

| Тип ключа | Де створюється (UI) | Scope / права |
|---|---|---|
| **Personal API key** | `Profile & Settings > API keys` | **Ті самі права, що й у користувача**, який його створив. У audit logs дії показуються від його імені. Потрібен для операцій з restrictive-access connections (створення dataset, SQL queries). |
| **Project-level key** | `Project > Settings/Security > API Keys` | Обмежений **одним проєктом**. Права: `READ_CONF`, `WRITE_CONF`, `EXPORT_DATASETS_DATA`, `RUN_SCENARIOS`, та per-dataset `READ_DATA`/`WRITE_DATA`/`READ_SCHEMA`/… Не дає доступу поза проєктом. |
| **Global API key** | `Administration > Security > Global API keys` | Може охоплювати **кілька проєктів**; з прапорцем `global admin` — повний доступ + адмін-задачі (керування користувачами). |

### 4.1 Personal vs Global — коли що (Середній)

- **Personal** — для скриптів, що мають діяти «від імені людини» (трасування в audit logs, права = права людини). Найбезпечніший для повсякденних інтеграцій.
- **Global (admin)** — лише для адмін-автоматизації (provisioning users, connections, code envs). Тримайте окремо, з мінімальним поширенням.
- **Project-level** — для вузьких інтеграцій з одним проєктом (наприклад, зовнішня система лише запускає сценарій).

### 4.2 OAuth2 / SSO / JWT (Просунуто)

Якщо DSS налаштований на SSO/OAuth2, замість API key можна передати **JWT bearer token**:

```python
client = dataikuapi.DSSClient(host, jwt_bearer_token="eyJhbGciOi...")
```

Це актуально, коли організація не дозволяє статичні ключі, а вимагає короткоживучі токени від identity provider (OAuth2/OIDC). Токен отримуєте поза Dataiku (через свій IdP) і передаєте в конструктор.

### 4.3 Створення ключів програмно (Просунуто)

```python
# Personal key для поточного користувача
pk = client.create_personal_api_key(label="ci-pipeline", description="used by CI")
print(pk["key"], pk["secret"])         # secret показується лише раз!

# Global key (потрібні admin-права)
gk = client.create_global_api_key(label="admin-automation", description="...", as_admin=True)

# Список / видалення
client.list_global_api_keys()
```

---

## 5. Базові операції

### 5.1 Projects: list / get / create (Базово)

```python
# Список усіх ключів проєктів
keys = client.list_project_keys()                  # ['PROJECT_A', 'PROJECT_B', ...]

# Список з метаданими
projects = client.list_projects()                  # list of dict
projects = client.list_projects(include_location=True)

# Хендл існуючого проєкту
project = client.get_project("MYPROJECT")

# Поточний проєкт (зсередини DSS)
project = client.get_default_project()

# Створення нового проєкту
project = client.create_project(
    project_key="NEWPROJECT",
    name="New Project",
    owner="admin",
    description="Created via API",
    tags=["api-created"]
)
```

Повна сигнатура:

```python
client.create_project(project_key, name, owner, description=None,
                      settings=None, project_folder_id=None,
                      permissions=None, tags=[])
```

### 5.2 Дублювання, експорт, видалення проєкту (Середній)

```python
# Дублювати
project.duplicate(target_project_key="MYPROJECT_COPY",
                  target_project_name="My Project (copy)")

# Експорт у потік (бекап/перенесення)
with open("/tmp/export.zip", "wb") as f:
    with project.get_export_stream() as s:
        for chunk in s:
            f.write(chunk)

# Імпорт архіву
client.prepare_project_import(open("/tmp/export.zip", "rb"))

# Видалення (обережно!)
project.delete(clear_managed_datasets=True)

# Permissions / metadata / tags
perms = project.get_permissions()
project.set_permissions(perms)
meta = project.get_metadata()
tags = project.get_tags()
```

### 5.3 Datasets, recipes у проєкті (Базово)

```python
# Datasets
datasets = project.list_datasets()                 # list of DSSDatasetListItem (.name, .type, .connection)
ds = project.get_dataset("customers")

# Recipes
recipes = project.list_recipes()
recipe = project.get_recipe("compute_customers")

# Scenarios
scenarios = project.list_scenarios()
scenario = project.get_scenario("DAILY_REFRESH")
```

### 5.4 Managed folders (Базово)

```python
# Створити managed folder
folder = project.create_managed_folder(name="myfolder")

# Отримати існуючу (ззовні — за ID)
folder = project.get_managed_folder("FOLDER_ID")

# Робота з файлами (ззовні: list_contents / get_file / put_file)
for item in folder.list_contents()["items"]:
    print(item["path"])

with open("/tmp/x.csv", "rb") as f:
    folder.put_file("/uploaded/x.csv", f)

data = folder.get_file("/some/file.bin").raw   # потік для читання
```

> **Увага на асиметрію імен.** Зсередини (`dataiku.Folder`): `list_paths_in_partition()`, `get_download_stream()`, `upload_stream()`. Ззовні (`dataikuapi` `DSSManagedFolder`): `list_contents()`, `get_file()`, `put_file()`. Це різні API над тим самим об'єктом.

---

## 6. Variables

### 6.1 Project-level variables (Базово)

`project.get_variables()` повертає словник із **двох** словників:

```python
variables = project.get_variables()
# {
#   "standard": { ... },   # звичайні змінні, ЕКСПОРТУЮТЬСЯ з bundles
#   "local":    { ... }    # НЕ частина bundles (instance-specific overrides)
# }
```

Оновлення (немає per-key setter — read-modify-write **усього** словника або `update_variables`):

```python
# Спосіб 1: read-modify-write
variables = project.get_variables()
variables["standard"]["batch_date"] = "2026-06-24"
project.set_variables(variables)

# Спосіб 2: оновити лише один scope
project.update_variables({"batch_date": "2026-06-24"}, type="standard")
```

> `standard` мандрує з bundle (dev→prod); `local` — НІ, тому `local` зручний для секретів/середовище-залежних значень, які не повинні їхати в прод-bundle. Деталі remapping — `10-mlops-deployment-api-node.md`.

### 6.2 Instance-level (global) variables (Просунуто, admin)

```python
# Сучасний (не deprecated) спосіб
gv = client.get_global_variables()     # DSSInstanceVariables
# ... мутуєте gv ...
gv.save()

# Legacy (працює, але deprecated): потребує admin-ключа, треба задати ВСІ змінні одразу
variables = client.get_variables()
variables["my_instance_var"] = "value"
client.set_variables(variables)
```

### 6.3 Зсередини scenario (Просунуто)

```python
import dataiku.scenario
s = dataiku.scenario.Scenario()
all_vars = s.get_all_variables()       # merged instance + project + scenario
s.set_scenario_variables(my_var="value")
```

---

## 7. Запуск scenarios та jobs

### 7.1 Scenarios: trigger vs wait (Середній)

Ключове розрізнення: `run()` **не** запускає сценарій миттєво — він **fires a manual trigger** і повертає `DSSTriggerFire`. Сам run з'являється лише після `wait_for_scenario_run()`.

```python
scenario = project.get_scenario("DAILY_REFRESH")

# Синхронно: запустити І дочекатися завершення
run = scenario.run_and_wait()
print(run.outcome)                     # SUCCESS / WARNING / FAILED / ABORTED

# Асинхронно: fire trigger → дочекатися появи run
trigger_fire = scenario.run()
scenario_run = trigger_fire.wait_for_scenario_run()

# Ручний поллінг
import time
while True:
    scenario_run.refresh()
    if scenario_run.running:
        time.sleep(5)
    else:
        break
print(scenario_run.outcome)
```

Сигнатури:

```python
scenario.run_and_wait(params=None, no_fail=False)   # -> DSSScenarioRun
scenario.run(params=None)                            # -> DSSTriggerFire
scenario.get_last_runs(limit=10, only_finished_runs=False)
scenario.get_last_successful_run()
scenario.get_status()                                # .next_run, .running
scenario.abort()
```

Передача параметрів у сценарій:

```python
run = scenario.run_and_wait(params={"refreshDate": "2026-06-24"})
```

> Немає методу з назвою `run_as_async()` — асинхронний шлях це `run()` → `DSSTriggerFire.wait_for_scenario_run()`.

### 7.2 Jobs: будування datasets програмно (Середній)

Два способи запустити job.

**(a) Сирий definition + `start_job`:**

```python
definition = {
    "type": "NON_RECURSIVE_FORCED_BUILD",
    "outputs": [{"id": "customers_enriched", "type": "DATASET", "partition": "NP"}]
}
job = project.start_job(definition)
```

**(b) Fluent builder через `new_job` (повертає `JobDefinitionBuilder`):**

```python
job = (project.new_job("RECURSIVE_FORCED_BUILD")
       .with_output("customers_enriched")
       .start())

# або одразу start + чекати
res = (project.new_job("RECURSIVE_BUILD")
       .with_output("customers_enriched")
       .start_and_wait())
```

Поллінг статусу:

```python
import time
state = ""
while state not in ("DONE", "FAILED", "ABORTED"):
    time.sleep(1)
    state = job.get_status()["baseStatus"]["state"]
```

Значення `object_type`: `DATASET`, `MANAGED_FOLDER`, `SAVED_MODEL`, `STREAMING_ENDPOINT`, `KNOWLEDGE_BANK`.
Значення `job_type` (build mode): `RECURSIVE_BUILD`, `NON_RECURSIVE_FORCED_BUILD`, `RECURSIVE_FORCED_BUILD`, `RECURSIVE_MISSING_ONLY_BUILD`.

```python
# Інші операції
project.get_job(job_id).abort()
project.list_jobs()
project.get_job(job_id).get_log()
```

> Клас будівника називається `JobDefinitionBuilder` (не `DSSJobDefinitionBuilder`); `project.new_job(...)` повертає саме його.

---

## 8. Читання metrics та checks

Детальна концепція metrics/checks/Data Quality — у `09-scenarios-automation-metrics-checks.md`. Тут — API-зріз. (Середній)

```python
ds = project.get_dataset("customers")

# Перерахувати metrics
ds.compute_metrics()

# Прочитати останні значення -> ComputedMetrics
last = ds.get_last_metric_values()
value = last.get_global_value("records:COUNT_RECORDS")

# Усі id
all_ids = last.get_all_ids()

# Історія однієї метрики
history = ds.get_metric_history("records:COUNT_RECORDS")

# Checks
ds.run_checks()
```

Сигнатури `DSSDataset`:

```python
ds.compute_metrics(partition='', metric_ids=None, probes=None)
ds.get_last_metric_values(partition='')      # -> ComputedMetrics
ds.get_metric_history(metric, partition='')
ds.run_checks(partition='', checks=None)
```

Корисні аксесори `ComputedMetrics`:

```python
last.get_metric_by_id(metric_id)
last.get_global_value(metric_id)
last.get_partition_value(metric_id, partition)
last.get_all_ids()
```

> Немає методу `get_last_check_values` — outcomes отримуєте через `run_checks()` та об'єкти `ComputedChecks` (аналогічні аксесори: `get_check_by_name`, `get_global_value`, `get_all_names`).

---

## 9. Адмін-операції

> Усі операції нижче потребують **admin-ключа** (global API key з `global admin`). Будьте обережні — вони змінюють інстанс глобально. (Просунуто)

### 9.1 Users

```python
# Список
users = client.list_users()                        # list of dict
user = client.get_user("jdoe")
me = client.get_own_user()

# Створення
client.create_user(
    login="jdoe",
    password="initial-pass",
    display_name="John Doe",
    source_type="LOCAL",                            # або LDAP / SAML_SSO ...
    groups=["data-scientists"],
    profile="DATA_SCIENTIST",
    email="jdoe@example.com"
)

# Bulk
client.create_users(users_list)
client.edit_users(changes)
client.delete_users(["jdoe"], allow_self_deletion=False)
```

### 9.2 Groups

```python
client.list_groups()
client.get_group("data-scientists")
client.create_group(name="analysts", description="BI analysts", source_type="LOCAL")
```

### 9.3 Connections

```python
# Список (as_type: 'dictitems' | 'listitems' | 'objects')
conns = client.list_connections()
conn = client.get_connection("prod-postgres")

# Створення
client.create_connection(
    name="new-s3",
    type="EC2",
    params={"bucket": "my-bucket", "...": "..."},
    usable_by="ALL",
    allowed_groups=None,
    description="S3 connection"
)
```

### 9.4 Code environments

```python
client.list_code_envs()
env = client.get_code_env("PYTHON", "py39_ml")

client.create_code_env(
    env_lang="PYTHON",
    env_name="py39_ml",
    deployment_mode="DESIGN_MANAGED",
    params={"pythonInterpreter": "PYTHON39"},
    wait=True
)

# Оновити пакети / спеки
settings = env.get_settings()
# settings.set_required_packages(...) тощо
settings.save()
env.update_packages()
```

> Керування code envs через API критичне для **CI/CD**: відтворюваність середовища між dev і prod (див. `10-...` та `06-...`).

---

## 10. Робота з даними через API

Тут найбільша різниця між `dataiku` (internal) і `dataikuapi` (external). (Середній)

### 10.1 Schema

```python
# Зсередини (dataiku)
cols = dataiku.Dataset("customers").read_schema()          # list of {name, type, ...}

# Ззовні (dataikuapi)
schema = project.get_dataset("customers").get_schema()     # dict з ключем 'columns'
column_names = [c["name"] for c in schema["columns"]]

# Редагування схеми ззовні
ds = project.get_dataset("customers")
settings = ds.get_settings()
settings.add_raw_schema_column({"name": "new_col", "type": "string"})
settings.save()
```

### 10.2 Читання даних: internal vs external (Середній)

**Зсередини (`dataiku`)** — DataFrame напряму:

```python
import dataiku
df = dataiku.Dataset("customers").get_dataframe()   # ВСЕ в пам'ять
```

**Chunked read зсередини** (для великих даних — НЕ збирайте все):

```python
ds = dataiku.Dataset("big_dataset")
for chunk in ds.iter_dataframes(chunksize=100000):
    process(chunk)                                   # кожен chunk ≤ 100k рядків

# Ще легше — без pandas:
for row in ds.iter_rows():        # dict-like
    ...
for tup in ds.iter_tuples():      # tuple значень
    ...
```

> Параметра `get_dataframe(chunksize=...)` **не існує** — для chunked використовуйте `iter_dataframes(chunksize=...)`.

**Ззовні (`dataikuapi`)** — DataFrame напряму **немає**; стрімите сирі рядки і будуєте DataFrame самі:

```python
import pandas as pd

ds = project.get_dataset("customers")
columns = [c["name"] for c in ds.get_schema()["columns"]]

# iter_rows() повертає список значень у порядку схеми
rows = ds.iter_rows()
df = pd.DataFrame(rows, columns=columns)

# або порядкова обробка без матеріалізації
for row in ds.iter_rows():
    record = dict(zip(columns, row))
    ...
```

> Найважливіше: зручний `get_dataframe()` існує **лише** в пакеті `dataiku`. У `dataikuapi` маєте `iter_rows()` + `get_schema()` і збираєте DataFrame вручну.

### 10.3 Запис даних (зсередини, `dataiku`)

```python
out = dataiku.Dataset("output_dataset")

# Найпростіше
out.write_with_schema(df)

# Стабільна схема + chunked write (для великих даних)
out.write_schema([{"name": "id", "type": "bigint"}, {"name": "v", "type": "double"}])
with out.get_writer() as writer:
    for chunk in dataiku.Dataset("input").iter_dataframes(chunksize=100000):
        # ... трансформація ...
        writer.write_dataframe(chunk)
```

### 10.4 SQL через API (Середній)

**Зсередини (`dataiku.SQLExecutor2`)** — pushdown у БД:

```python
from dataiku import SQLExecutor2

executor = SQLExecutor2(connection="prod-postgres")     # або dataset="customers"
df = executor.query_to_df("SELECT region, COUNT(*) c FROM sales GROUP BY region")

# Ітератор (не вантажить усе):
reader = executor.query_to_iter("SELECT * FROM big_table")
for tup in reader.iter_tuples():
    ...
```

**Ззовні (`client.sql_query` → `DSSSQLQuery`)** — стрімінг:

```python
q = client.sql_query("SELECT * FROM train_set",
                     connection="prod-postgres",
                     type="sql")
for row in q.iter_rows():
    process(row)
q.verify()        # ОБОВ'ЯЗКОВО після ітерації — інакше помилки стрімінгу мовчазні

# Hive/Impala — через database замість connection:
q = client.sql_query("SELECT * FROM train_set", database="test_area", type="hive")
```

Сигнатура:

```python
client.sql_query(query, connection, database=None, type=None,
                 pre_queries=None, post_queries=None, project_key=None)
```

> `verify()` треба викликати **після** `iter_rows()` — помилки під час стрімінгу не кидаються інлайн.

### 10.5 Managed folders і cloud (Середній)

```python
# Зсередини — для cloud (S3/GCS/Azure) get_path() ПАДАЄ, лише stream-API:
folder = dataiku.Folder("FOLDER_ID")
for path in folder.list_paths_in_partition():
    with folder.get_download_stream(path) as f:
        data = f.read()

with folder.get_writer("out/result.bin") as w:
    w.write(b"...")

# get_path() — ТІЛЬКИ для local-filesystem folders:
local_path = folder.get_path()    # падає на S3/HDFS/Azure/GCS
```

---

## 11. ML через API

Детально про Visual ML — у `07-machine-learning-visual-ml.md`. Тут — програмний потік. (Просунуто)

### 11.1 Тренування моделі

```python
p = client.get_project("MYPROJECT")

# Створити ML Task (сучасний API — напряму з проєкту)
mltask = p.create_prediction_ml_task(
    input_dataset="trainset",
    target_variable="target",
    ml_backend_type="PY_MEMORY",
    guess_policy="DEFAULT"
)
mltask.wait_guess_complete()          # дочекатися auto-guess фіч/налаштувань

# Налаштувати алгоритм
settings = mltask.get_settings()
settings.set_algorithm_enabled("GBT_CLASSIFICATION", True)
settings.use_feature("amount")
settings.reject_feature("internal_id")
settings.save()

# Тренувати
mltask.start_train()
mltask.wait_train_complete()

# Інспекція результатів
ids = mltask.get_trained_models_ids()
details = mltask.get_trained_model_details(ids[0])
```

Інші конструктори ML Task: `create_clustering_ml_task(...)`, `create_timeseries_forecasting_ml_task(...)`.

### 11.2 Deploy у Flow + Saved Models

```python
# Розгорнути натреновану модель у Flow (створює Saved Model + train recipe)
ret = mltask.deploy_to_flow(model_id=ids[0], model_name="fraud_model",
                            train_dataset="trainset")
sm_id = ret["savedModelId"]

# Керування Saved Model
model = p.get_saved_model(sm_id)
for v in model.list_versions():
    print(v)
model.get_active_version()
model.set_active_version("<version_id>")
```

### 11.3 API service: створення та публікація (Просунуто)

```python
# Створити API service у API Designer
service = p.create_api_service("fraud_service")

# Додати visual-prediction endpoint на основі Saved Model
settings = service.get_settings()
settings.add_prediction_endpoint("predict_fraud", sm_id)
settings.save()

# Створити package (версію) і опублікувати на API Deployer
service.create_package("v1.0", release_notes="Initial release")
service.publish_package("v1.0", published_service_id="fraud_service")
```

Деплой через API Deployer (інфра + deployment):

```python
deployer = client.get_apideployer()
infra = deployer.create_infra("prod-api", stage="Production", type="STATIC")
dep = deployer.create_deployment("fraud-deploy", service_id="fraud_service",
                                 infra_id="prod-api", version="v1.0")
dep.start_update()
print(dep.get_status())
```

> Повний lifecycle (static vs Kubernetes infra, HA, test queries, query enrichment) — у `10-mlops-deployment-api-node.md`.

---

## 12. API node: real-time scoring

Коли модель уже задеплоєна на **API node**, її скорять через окремий клієнт `dataikuapi.APINodeClient` (не `DSSClient`!). (Просунуто)

### 12.1 APINodeClient

```python
import dataikuapi

client = dataikuapi.APINodeClient(
    "http://apinode-host:12000",       # uri API node
    "fraud_service",                   # service_id
    "myapikey"                         # api_key (якщо service захищений ключами)
)

record = {"customer_id": 2136, "transaction_amount": 233.33, "payment_type": "CB"}

# Один запис
prediction = client.predict_record("predict_fraud", record)
print(prediction["result"]["prediction"])
print(prediction["result"]["probas"])

# Пакет записів
records = [
    {"customer_id": 2136, "transaction_amount": 233.33, "payment_type": "CB"},
    {"customer_id": 9921, "transaction_amount": 12.10,  "payment_type": "CASH"},
]
res = client.predict_records("predict_fraud", records)
for r in res["results"]:
    print(r["result"]["prediction"])
```

Конструктор та основні методи:

```python
dataikuapi.APINodeClient(uri, service_id, api_key=None, bearer_token=None,
                         no_check_certificate=False, client_certificate=None)

predict_record(endpoint_id, features, with_explanations=None, explanation_method=None, ...)
predict_records(endpoint_id, records, ...)
forecast(endpoint_id, records, ...)            # time-series
predict_effect(endpoint_id, features, ...)     # causal
sql_query(endpoint_id, parameters)
lookup_record(endpoint_id, record, ...)        # dataset lookup endpoint
run_function(endpoint_id, **kwargs)            # custom Python/R function endpoint
```

> **Немає методу `predict(...)`.** Один запис — `predict_record`; пакет — `predict_records`. Для OAuth2/JWT-захищеного service передавайте `bearer_token=` замість `api_key=`.

### 12.2 Explanations при scoring

```python
pred = client.predict_record(
    "predict_fraud", record,
    with_explanations=True,
    explanation_method="SHAPLEY",     # або 'ICE'
    n_explanations=3
)
print(pred["result"]["explanations"])
```

---

## 13. REST API напряму (HTTP / curl)

Якщо клієнтом не Python — викликайте REST напряму. (Середній)

### 13.1 Структура URL

```
http://DSS_HOST:DSS_PORT/public/api/<resource>/
```

Приклад: `http://dss.mycompany.com:12000/public/api/projects/`. Повний машинний reference усіх ендпоінтів — `https://doc.dataiku.com/dss/api/14/rest`.

POST/PUT-тіла — JSON з `Content-Type: application/json`. Відповіді — JSON. Успіх — 2xx, помилки — 4xx/5xx з JSON-деталями.

### 13.2 Аутентифікація по HTTP

> "When using Bearer Token Authentication, use the API key as the token. When using Basic Authentication, leave the username blank and use the API key as the password."

**Bearer token (рекомендовано):**

```bash
curl --request GET \
  --url http://dss.mycompany.com:12000/public/api/projects/ \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json'
```

**HTTP Basic (ключ як username, пароль порожній):**

```bash
curl --user "YOUR_API_KEY:" -H "Content-Type: application/json" \
  -X GET http://dss.mycompany.com:12000/public/api/projects/
```

> Зверніть увагу на двокрапку в `--user "KEY:"` — це задає username=ключ і порожній password.

Запуск scenario по HTTP:

```bash
curl --user "YOUR_API_KEY:" -H "Content-Type: application/json" \
  -X POST \
  http://dss.mycompany.com:12000/public/api/projects/MYPROJECT/scenarios/DAILY_REFRESH/run
```

### 13.3 API node по HTTP (scoring)

API node має **окремий простір** і **окремі public keys** (Bearer-only):

```
POST http://APINODE_HOST:PORT/public/api/v1/<service_id>/<endpoint_id>/predict
```

```bash
curl -X POST \
  'http://apinode-host:12000/public/api/v1/fraud_service/predict_fraud/predict' \
  -H 'Authorization: Bearer 1234APIKEY56789' \
  -H 'Content-Type: application/json' \
  -d '{
        "features": {
          "customer_id": 2136,
          "transaction_amount": 233.33,
          "payment_type": "CB"
        }
      }'
```

Приклад відповіді (classification):

```json
{
  "result": {
    "prediction": "FRAUDULENT",
    "probaPercentile": 94,
    "probas": { "FRAUDULENT": 0.878, "NOT_FRAUDULENT": 0.122 }
  }
}
```

Інші форми ендпоінта: `/predict-multi` (batch, тіло з масивом `items`), `/predict-simple?feat1=...` (GET для дебагу), `/run` (custom function), `/swagger` (OpenAPI-спека сервісу).

> **Public API keys на API node ≠ keys на design/automation node.** На API node ключі специфічні для service, дають **повний доступ до всього service** (немає per-endpoint security), керуються через `apinode-admin` або API Deployer.

---

## 14. CLI-інструменти

Інструменти живуть у `DSS_DATADIR/bin/`. (Просунуто)

### 14.1 Старт/стоп інстансу

```bash
./bin/dss start      # піднімає supervisord -> nginx, backend, jupyter
./bin/dss stop
./bin/dss status
./bin/dss restart
```

### 14.2 `dssadmin` — installation/config

«Інсталяційні» команди (R, Spark, Hadoop, graphics, regenerate-config):

```bash
./bin/dssadmin install-spark-integration -sparkHome /opt/spark3
./bin/dssadmin install-R-integration
./bin/dssadmin install-hadoop-integration -standalone generic-hadoop3 -standaloneArchive /path/...tar.gz
./bin/dssadmin regenerate-config        # застосувати зміни install.ini (напр., Java memory)
```

### 14.3 `dsscli` — day-to-day операції

```bash
# Users / groups / keys
./bin/dsscli users-list
./bin/dsscli user-create -login jdoe -password ... -displayName "John Doe"
./bin/dsscli api-key-create ...

# Jobs / scenarios
./bin/dsscli build PROJECT_KEY dataset_name
./bin/dsscli scenario-run PROJECT_KEY SCENARIO_ID
./bin/dsscli jobs-list PROJECT_KEY

# Projects / bundles
./bin/dsscli projects-list
./bin/dsscli bundle-export PROJECT_KEY BUNDLE_ID
./bin/dsscli bundle-activate PROJECT_KEY BUNDLE_ID

# Datasets / connections / code envs
./bin/dsscli datasets-list PROJECT_KEY
./bin/dsscli connections-list
./bin/dsscli code-envs-list
```

> Розподіл праці: `dssadmin` — «інсталяційні» речі (R/Spark/Hadoop), `dsscli` — рутина (users, jobs, scenarios, bundles).

### 14.4 `apinode-admin` — керування API node

```bash
./bin/apinode-admin services-list
./bin/apinode-admin service-create <id>
./bin/apinode-admin service-import-generation <id> <path>
./bin/apinode-admin admin-key-create
./bin/apinode-admin predict <service> <endpoint> ...
```

### 14.5 Fleet Manager (cloud)

> Уточнення: окремого `fmcli` **немає**. Fleet Manager керується через **web UI**, **Python `FMClient`** (`FMClientAWS` / `FMClientAzure` / `FMClientGCP`), а локальний бінар `fmadmin` потрібен лише для **створення API key** для FM Python API.

```python
import dataikuapi
fm = dataikuapi.fmclient.FMClientAWS("https://fleet-manager-host", "API_KEY_ID", "API_KEY_SECRET")
# fm.list_instances(), fm.create_instance(...), снапшоти/restore тощо
```

> `dku` теж **не** публічна команда — це внутрішній префікс (`DKU_*` env vars, внутрішні kernels: backend, jek, fek, hproxy, apimain, governserver). Користувацькі CLI — `dss`, `dssadmin`, `dsscli`, `apinode-admin`.

---

## 15. Помилки, пагінація, rate limiting, retry, ідемпотентність

(Середній / Просунуто)

### 15.1 HTTP-помилки

Клієнт кидає виняток на 4xx/5xx. Типово ловлять загально:

```python
try:
    project = client.get_project("MISSING")
    project.get_metadata()
except Exception as e:
    # повідомлення містить HTTP-код і тіло помилки DSS
    print("DSS API error:", e)
```

Поширені коди: **401/403** — поганий/недостатній ключ; **404** — об'єкт не існує; **409** — конфлікт (наприклад, проєкт із таким key вже є); **500** — помилка на стороні DSS (дивіться backend logs).

### 15.2 Пагінація

Більшість `list_*` методів повертають **повний** список одразу (DSS не пагінує більшість public-API list-ендпоінтів). Для важких вибірок (jobs, scenario runs) використовуйте параметри-обмежувачі:

```python
scenario.get_last_runs(limit=20, only_finished_runs=True)
project.list_jobs()                     # фільтруйте на клієнті
```

### 15.3 Rate limiting та retry

Public API на design/automation node зазвичай не має жорсткого rate-limit, але **API node** під навантаженням може повертати помилки. Робастний патерн із retry/backoff:

```python
import time

def with_retry(fn, attempts=5, base_delay=1.0):
    for i in range(attempts):
        try:
            return fn()
        except Exception as e:
            if i == attempts - 1:
                raise
            time.sleep(base_delay * (2 ** i))   # exponential backoff

result = with_retry(lambda: api_client.predict_record("predict_fraud", record))
```

Для продакшен-scoring краще `requests.Session` з `urllib3.util.retry.Retry` (status_forcelist=[429,500,502,503,504]).

### 15.4 Ідемпотентність

- **Read-операції** (`list_*`, `get_*`) — ідемпотентні, безпечно ретраїти.
- **Create-операції** — НЕ ідемпотентні: повторний `create_project("X", ...)` дасть 409. Перед створенням перевіряйте існування:

```python
if "NEWPROJECT" not in client.list_project_keys():
    client.create_project("NEWPROJECT", "New", "admin")
```

- **`set_variables` / `update_variables`** — фактично «PUT»: повторний виклик з тим самим значенням безпечний (idempotent за результатом), але read-modify-write на `set_variables` має **race condition** при паралельних апдейтах — там краще `update_variables` або серіалізуйте записи.

---

## 16. Best practices

- **(Базово) НЕ хардкодьте api_key.** Беріть з env vars / secrets manager:

  ```python
  import os, dataikuapi
  client = dataikuapi.DSSClient(os.environ["DSS_HOST"], os.environ["DSS_API_KEY"])
  ```

- **(Базово) Зсередини DSS — `dataiku.api_client()`, не зовнішній ключ.** Ніколи не вставляйте personal key у recipe/notebook — це витік у Git проєкта. Зсередини використовуйте `dataiku.api_client()` (контекстна автентифікація) або **User Secrets** (`client.get_auth_info(with_secrets=True)`).
- **(Середній) Dev vs prod хости через конфіг, не код.** Тримайте `{host, key}` у середовище-залежному конфізі (env vars, `.env` не в Git, vault). Той самий скрипт має працювати на dev і prod лише зі зміною змінних оточення.
- **(Середній) Мінімальні права ключа.** Для вузької інтеграції — **project-level key**, а не global admin. Global admin — лише для provisioning-скриптів, окремо й під контролем.
- **(Середній) Перевіряйте існування перед create (ідемпотентність).** `if key not in client.list_project_keys(): create_project(...)`. Робить CI-пайплайни повторно-запускними.
- **(Середній) Не збирайте великі дані ззовні через `iter_rows()` у DataFrame** без потреби — для аналітики краще pushdown SQL (`client.sql_query`) або chunked обробка.
- **(Просунуто) Версіюйте скрипти оркестрації** в Git разом із проєктом (або окремим репо). API-скрипти — це інфраструктурний код; до них застосовуйте code review та CI-тести.
- **(Просунуто) Обережність з admin-операціями.** `delete_users`, `delete()` проєкту, `create_connection` — незворотні/глобальні. Робіть dry-run (логуйте, що буде зроблено) перед реальним запуском; ставте підтвердження для destructive-кроків.
- **(Просунуто) CI/CD.** Типовий пайплайн: збірка bundle/API package на dev (`service.create_package`), `publish_package`, `deployer.create_deployment(...).start_update()`, потім smoke-test через `APINodeClient.predict_record`. Деталі — `10-mlops-deployment-api-node.md` §17.
- **(Просунуто) Retry/timeout для real-time scoring.** `APINodeClient`/HTTP-виклики обгортайте в retry з backoff та явним timeout; не блокуйте пайплайн нескінченно.
- **(Просунуто) Сумісність версій клієнта.** Тримайте `dataiku-api-client` сумісним з версією DSS (13.x/14.x). Нові методи API можуть бути відсутні у старому клієнті.

---

## 17. Типові помилки

- **Плутати `dataiku` і `dataikuapi`.** `get_dataframe()` є тільки в `dataiku`; ззовні (`dataikuapi`) — лише `iter_rows()` + `get_schema()`.
- **Передати `host` із `/public/api`** у `DSSClient(host, key)` → 404. Host — це **корінь інстансу** (`https://dss.host`), без шляху API.
- **Хардкод personal key у recipe/notebook** → витік у Git. Зсередини — `dataiku.api_client()`.
- **Викликати `client.predict(...)`** на `APINodeClient` → AttributeError. Метод — `predict_record` (один) / `predict_records` (пакет).
- **Забути `q.verify()` після `client.sql_query(...).iter_rows()`** → мовчазно пропущені помилки стрімінгу.
- **`get_dataframe(chunksize=...)`** → такого параметра немає; chunked — це `iter_dataframes(chunksize=...)`.
- **`Folder.get_path()` на cloud (S3/GCS/Azure/HDFS)** → падає; використовуйте `get_download_stream`/`upload_stream`.
- **Повторний `create_project(key, ...)` з наявним key** → 409 Conflict. Перевіряйте `list_project_keys()` спершу.
- **`set_variables` без read-modify-write** → затирає інші змінні (треба задавати ВЕСЬ словник, або `update_variables`).
- **`get_custom_variables()` без касту** → значення завжди **рядки**; `int(...)`/`float(...)`.
- **Очікувати, що `scenario.run()` одразу запустить run** → ні; це fire trigger. Потрібен `wait_for_scenario_run()` (або `run_and_wait()`).
- **`no_check_certificate=True` у проді** → MITM-ризик; додайте корпоративний CA-bundle.
- **Шукати `fmcli` / `dku` як CLI** → їх немає; для Fleet Manager — Python `FMClient*`; внутрішні процеси `dku`-prefixed не викликаються напряму.
- **Збирати весь великий dataset ззовні через `iter_rows()` у `pd.DataFrame`** → OOM/повільно; робіть pushdown SQL або chunked.

---

## 18. Перехресні посилання

- [01-overview-architecture.md](01-overview-architecture.md) — ноди (Design/Automation/API/Deployer), архітектура інстансу.
- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — projects, Flow, recipes (об'єкти, якими керує API).
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — datasets, connections, partitioning.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — **`dataiku` internal API детально**, code envs, notebooks, User Secrets.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — Visual ML, Lab, Saved Models (контекст для §11).
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — LLM Mesh (також доступний програмно).
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — scenarios, metrics, checks (концепція; тут API-зріз).
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — **bundles, Project/API Deployer, API node lifecycle, CI/CD** (контекст для §11–§12, §16).
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — security, permissions, секрети, Git.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — plugins, custom recipes (теж керовані через API).
- [15-best-practices.md](15-best-practices.md) — зведені best practices.
- [16-glossary.md](16-glossary.md) — глосарій термінів.

---

## 19. Джерела (офіційна документація Dataiku)

- Developer Guide — DSSClient reference: https://developer.dataiku.com/latest/api-reference/python/client.html
- Developer Guide — Getting started: https://developer.dataiku.com/latest/getting-started/index.html
- Developer Guide — Projects: https://developer.dataiku.com/latest/concepts-and-examples/projects.html
- Developer Guide — Datasets (data): https://developer.dataiku.com/latest/concepts-and-examples/datasets/datasets-data.html
- Developer Guide — Datasets API reference: https://developer.dataiku.com/latest/api-reference/python/datasets.html
- Developer Guide — SQL: https://developer.dataiku.com/latest/concepts-and-examples/sql.html
- Developer Guide — Managed folders: https://developer.dataiku.com/latest/concepts-and-examples/managed-folders.html
- Developer Guide — Scenarios: https://developer.dataiku.com/latest/concepts-and-examples/scenarios.html
- Developer Guide — Scenarios API reference: https://developer.dataiku.com/latest/api-reference/python/scenarios.html
- Developer Guide — Jobs: https://developer.dataiku.com/latest/concepts-and-examples/jobs.html
- Developer Guide — Metrics & Checks API: https://developer.dataiku.com/latest/api-reference/python/metrics.html
- Developer Guide — ML (concepts/examples): https://developer.dataiku.com/latest/concepts-and-examples/ml.html
- Developer Guide — API services (Designer/Deployer): https://developer.dataiku.com/latest/api-reference/python/api-services.html
- Developer Guide — Fleet Manager (FMClient): https://developer.dataiku.com/latest/api-reference/python/fleetmanager.html
- Developer Guide — REST API tutorial: https://developer.dataiku.com/latest/tutorials/devtools/rest-api/index.html
- DSS docs — Public REST API: https://doc.dataiku.com/dss/latest/publicapi/rest.html
- DSS docs — API keys: https://doc.dataiku.com/dss/latest/publicapi/keys.html
- DSS docs — Python API client: https://doc.dataiku.com/dss/latest/publicapi/client-python.html
- DSS docs — API node index: https://doc.dataiku.com/dss/latest/apinode/index.html
- DSS docs — API node security: https://doc.dataiku.com/dss/latest/apinode/security.html
- DSS docs — apinode-admin CLI: https://doc.dataiku.com/dss/latest/apinode/operations/cli-tool.html
- DSS docs — dsscli / dssadmin: https://doc.dataiku.com/dss/latest/operations/dsscli.html
- DSS docs — Machine REST API reference: https://doc.dataiku.com/dss/api/14/rest
- GitHub — dataiku-api-client-python (джерело сигнатур): https://github.com/dataiku/dataiku-api-client-python
