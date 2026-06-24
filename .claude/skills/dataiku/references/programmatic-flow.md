# Dataiku — програмна побудова Flow (`dataikuapi`)

Перевірені патерни для **програмної** побудови Flow: створення проєктів, datasets, recipes, build, scenarios — переважно через `dataikuapi` (Режим A, ззовні DSS). Концепції — `docs/11`, `docs/06`, `docs/02`.

> Сигнатури звірені з source `dataiku/dataiku-api-client-python` (master). Між версіями клієнта окремі параметри змінюються — на вашому DSS 14.5.1 валідно, але дрібниці звіряй на живому інстансі.
>
> ⚠️ **Цей код працює з реальним робочим Dataiku ПриватБанку.** Спершу — тестовий проєкт, не прод. Усе ідемпотентно. Destructive (`delete`) — лише свідомо.

---

## 1. Два режими: ззовні vs всередині

| | **Режим A** — `dataikuapi` ЗЗОВНІ | **Режим B** — `dataiku` ВСЕРЕДИНІ |
|---|---|---|
| Де | Claude Code на робочому пристрої → напряму в DSS по мережі | notebook / code recipe / scenario всередині DSS |
| Вхід | `dataikuapi.DSSClient(host, api_key)` | `import dataiku; dataiku.api_client()` |
| Ключ | потрібен API key (env var) | не потрібен (контекст користувача) |
| Призначення | оркестрація, побудова Flow, CI/CD | бізнес-логіка рецепта, читання/запис даних |

**Цей файл — переважно Режим A** (Claude будує Flow напряму). Об'єкт `DSSProject` і всі creators ідентичні в обох режимах — `dataiku.api_client()` повертає той самий `DSSClient`. Тому код звідси працює і всередині DSS, якщо замінити підключення на `client = dataiku.api_client()`.

---

## 2. Підключення та безпека

```bash
pip install dataiku-api-client          # імпортується як dataikuapi
```

```python
import dataikuapi, os

client = dataikuapi.DSSClient(
    host=os.environ["DSS_HOST"],         # internal URL інстансу, БЕЗ /public/api
    api_key=os.environ["DSS_API_KEY"],   # НІКОЛИ не в repo/коді — лише env var
)

# self-managed on-prem → ймовірно self-signed TLS:
client._session.verify = False           # ⚠️ вимикає перевірку сертифіката — MITM-ризик
# Краще: вказати CA-bundle банку замість False:
# client._session.verify = "/path/to/privatbank-ca.pem"

# Перевірка з'єднання
print(client.list_project_keys())        # якщо повернувся список — все ок
```

> `verify=False` зручний для швидкого старту, але небезпечний. У проді — CA-bundle. Personal API key має **ті самі права, що й ти** в DSS; усі дії в audit logs — від твого імені.

---

## 3. Ідемпотентність / захист "перевір-перед-створенням"

Create-операції **не** ідемпотентні: повторний create дасть `409 Conflict`. Завжди перевіряй існування.

```python
# Проєкт
PKEY = "SANDBOX_FLOW"
if PKEY not in client.list_project_keys():
    project = client.create_project(PKEY, "Sandbox Flow", owner="your_login")
else:
    project = client.get_project(PKEY)

# Dataset
existing_ds = {d.name for d in project.list_datasets()}
if "orders_raw" not in existing_ds:
    ... # створити

# Recipe
existing_rec = {r.name for r in project.list_recipes()}
if "compute_orders_clean" not in existing_rec:
    ... # створити
```

> `project.list_datasets()` / `list_recipes()` повертають list-items із атрибутом `.name`. Тримай ці helper-сети на початку скрипта — і повторний запуск не падає й не дублює.

---

## 4. Проєкти

```python
client.list_project_keys()                       # ['PROJ_A', 'PROJ_B', ...]
project = client.get_project("MYPROJECT")        # хендл існуючого
project = client.get_default_project()           # поточний (зсередини DSS)

# Створення
project = client.create_project(
    project_key="SANDBOX_FLOW",                  # UPPERCASE, без пробілів
    name="Sandbox Flow",
    owner="your_login",                          # обов'язковий
    description="Created via API",
    tags=["api-created", "sandbox"],
)
```

Повна сигнатура: `client.create_project(project_key, name, owner, description=None, settings=None, project_folder_id=None, permissions=None, tags=[])`.

Project variables (read-modify-write **усього** словника; немає per-key setter):

```python
v = project.get_variables()                      # {"standard": {...}, "local": {...}}
v["standard"]["batch_date"] = "2026-06-24"
project.set_variables(v)

# Оновити лише один scope без затирання решти:
project.update_variables({"batch_date": "2026-06-24"}, type="standard")
```

> `standard` їде з bundle (dev→prod), `local` — ні (instance-specific). `get_custom_variables()` (зсередини) повертає значення **завжди рядками** — кастуй.

---

## 5. Datasets

### 5.1 External dataset на Redshift (читання існуючої таблиці)

External = вказує на наявну таблицю в Redshift; DSS її не матеріалізує. Створюється через generic `create_dataset(name, type, params=...)`. Точну структуру `params` найнадійніше зняти з датасета, зробленого в UI (`get_dataset(...).get_settings().get_raw()`), і відтворити.

```python
# ⚠️ структуру params звірити на вашій connection (зніми з UI-датасета)
ds = project.create_dataset(
    "orders_raw",
    type="Redshift",                             # тип конектора
    params={
        "connection": "redshift_main",           # назва вашої Redshift connection
        "mode": "table",
        "table": "orders",
        "schema": "public",                      # Redshift schema
    },
)
# Підтягнути схему колонок із БД (важливо для external):
settings = ds.get_settings()
settings.set_schema_from_dataset()               # ⚠️ звірити сигнатуру; альтернатива — нижче
settings.save()
```

> Якщо `set_schema_from_dataset()` недоступний на вашій версії — використай UI-кнопку "Detect schema" один раз, або задай схему вручну через `ds.set_schema({...})` (див. 5.3). `create_dataset(dataset_name, type, params=None, formatType=None, formatParams=None)` — підтверджена сигнатура.

### 5.2 Managed output dataset (DSS сам створює таблицю/файл)

Output рецепта зазвичай зручніше створювати **прямо в creator** (`with_new_output_dataset` / `with_new_output`, див. §6). Окремо — через builder:

```python
builder = project.new_managed_dataset("orders_clean")
builder.with_store_into("redshift_main")         # connection, куди писати
ds = builder.create(overwrite=True)              # overwrite=True → ідемпотентно
```

`with_store_into(connection, type_option_id=None, format_option_id=None)`; `create(overwrite=False)`; є `already_exists()`.

> `project.new_managed_dataset(name)` повертає `DSSManagedDatasetCreationHelper`. Старий `new_managed_dataset_creation_helper(...)` — **deprecated**, не використовуй.

### 5.3 Schema: прочитати / задати

```python
schema = ds.get_schema()                         # dict з ключем 'columns'
cols = [c["name"] for c in schema["columns"]]

ds.set_schema({"columns": [
    {"name": "id",     "type": "bigint"},
    {"name": "amount", "type": "double"},
    {"name": "country","type": "string"},
]})
```

### 5.4 Читання даних ЗЗОВНІ vs ВСЕРЕДИНІ

```python
# ЗЗОВНІ (dataikuapi): немає get_dataframe() — лише iter_rows() + get_schema()
import pandas as pd
cols = [c["name"] for c in ds.get_schema()["columns"]]
df = pd.DataFrame(ds.iter_rows(), columns=cols)  # ⚠️ вантажить усе — обережно з обсягом

# ВСЕРЕДИНІ (dataiku): get_dataframe() напряму
# import dataiku
# df = dataiku.Dataset("orders_raw").get_dataframe()
```

> **Обмеження ззовні:** зручного `get_dataframe()` у `dataikuapi` немає. Для аналітики великих даних краще pushdown SQL (`client.sql_query(...)`, див. `docs/11` §10.4), а не тягнути все `iter_rows()` у pandas.

---

## 6. Recipes — побудова Flow

Загальний fluent-патерн: `project.new_recipe(type, name)` → `.with_input(...)` → output (`.with_new_output(...)` / `.with_new_output_dataset(...)` / `.with_existing_output(...)`) → `.create()`. Далі — донастройка через `recipe.get_settings()` і `recipe.compute_schema_updates().apply()`.

> `project.new_recipe(type, name=None)` — підтверджена сигнатура. `build()` на creator — **deprecated alias до `create()`**; нижче скрізь `create()`.
>
> **Важливо щодо назв класів:** `new_recipe("python")` повертає **`CodeRecipeCreator`** (не `PythonRecipeCreator`!). `PythonRecipeCreator` існує як окремий клас, але `new_recipe` його не повертає. `CodeRecipeCreator` має `with_script()` і `with_new_output_dataset()` — саме ними й користуйся.

### 6.1 Python recipe

```python
code = """
import dataiku
df = dataiku.Dataset("orders_raw").get_dataframe()
df = df[df["amount"] > 0]                         # проста чистка
dataiku.Dataset("orders_clean").write_with_schema(df)
"""

builder = project.new_recipe("python", "compute_orders_clean")   # -> CodeRecipeCreator
builder.with_input("orders_raw")
builder.with_new_output_dataset("orders_clean", "redshift_main", overwrite=True)
builder.with_script(code)
recipe = builder.create()
```

`with_new_output_dataset(name, connection, type=None, format=None, copy_partitioning_from="FIRST_INPUT", append=False, overwrite=False, **kwargs)` — створює managed dataset **одразу**. `with_script(script)` — задає код.

### 6.2 Sync recipe (копія dataset → іншу connection)

```python
builder = project.new_recipe("sync", "compute_orders_synced")    # -> SyncRecipeCreator
builder.with_input("orders_clean")
# with_new_output: output не створюється одразу, а при .create()
builder.with_new_output("orders_synced", "redshift_main", overwrite=True)
recipe = builder.create()
```

`SyncRecipeCreator` успадковує `SingleOutputRecipeCreator`: `with_new_output(name, connection, type=None, format=None, ..., append=False, object_type='DATASET', overwrite=False)`, `with_existing_output(output_id, append=False)`.

### 6.3 Grouping (Group) recipe

```python
builder = project.new_recipe("grouping", "compute_orders_by_country")  # -> GroupingRecipeCreator
builder.with_input("orders_clean")
builder.with_new_output("orders_by_country", "redshift_main", overwrite=True)
builder.with_group_key("country")                # ключ групування (один)
recipe = builder.create()

# Агрегації задаються ПІСЛЯ створення, через settings:
st = recipe.get_settings()
st.set_column_aggregations("amount", sum=True)   # ⚠️ звірити set_column_aggregations на вашій версії
st.save()
recipe.compute_schema_updates().apply()          # застосувати нову схему до output
```

`with_group_key(group_key)` — підтверджено. Додаткові group keys і агрегації — через `recipe.get_settings()` (`GroupingRecipeSettings`).

### 6.4 Join recipe

```python
builder = project.new_recipe("join", "compute_orders_joined")    # -> JoinRecipeCreator
builder.with_input("orders_clean")
builder.with_input("customers")                  # JoinRecipeCreator приймає кілька inputs
builder.with_new_output("orders_joined", "redshift_main", overwrite=True)
recipe = builder.create()
# Умови join / тип join — донастроюються через recipe.get_settings() (JoinRecipeSettings)
```

`JoinRecipeCreator` успадковує `VirtualInputsSingleOutputRecipeCreator` → `with_input(input_id, project_key=None)` (multi-input), `with_new_output(...)`.

### 6.5 Prepare (shaker) recipe

```python
builder = project.new_recipe("prepare", "compute_orders_prepared")  # -> PrepareRecipeCreator
# ("shaker" — синонім "prepare", обидва приймаються new_recipe)
builder.with_input("orders_raw")
builder.with_new_output("orders_prepared", "redshift_main", overwrite=True)
recipe = builder.create()
```

> **Чесно про обмеження:** `PrepareRecipeCreator` створює **порожній** Prepare (без steps). Зручного API "додай processor X" немає — steps додаються через `recipe.get_settings().raw_steps` (список dict; структура кожного step специфічна для processor'а) + `.save()`. Найнадійніше: зроби 1 step у UI, зніми його через `get_settings()` і відтвори структуру. Складні Prepare краще будувати в UI, а через API — лише оркеструвати build.

### 6.6 SQL query recipe

```python
sql = """
SELECT country, COUNT(*) AS n, SUM(amount) AS total
FROM "${projectKey}_orders_clean"
GROUP BY country
"""

builder = project.new_recipe("sql_query", "compute_orders_sql")  # -> SQLQueryRecipeCreator
builder.with_input("orders_clean")
builder.with_new_output("orders_sql_agg", "redshift_main", overwrite=True)
recipe = builder.create()

# Текст запиту задається через settings (немає with_query на creator):
st = recipe.get_settings()
st.set_payload(sql)                              # ⚠️ звірити set_payload на вашій версії
st.save()
```

> `SQLQueryRecipeCreator` успадковує `SingleOutputRecipeCreator` (тільки input/output, **без** методу для тексту запиту). SQL задавай через `recipe.get_settings()` (`set_payload(...)` / `get_payload()` — звір на версії). Для in-database pushdown тримай і input, і output на одній SQL connection (Redshift).

---

## 7. Build / jobs

```python
# Найпростіше — build одного dataset (синхронно, чекає):
ds = project.get_dataset("orders_clean")
job = ds.build()                                 # NON_RECURSIVE_FORCED_BUILD, wait=True
print("done:", job.id)

# Recursive build (перебудувати весь upstream):
ds.build(job_type="RECURSIVE_BUILD")
```

`dataset.build(job_type="NON_RECURSIVE_FORCED_BUILD", partitions=None, wait=True, no_fail=False)`. Типи: `RECURSIVE_BUILD`, `NON_RECURSIVE_FORCED_BUILD`, `RECURSIVE_FORCED_BUILD`, `RECURSIVE_MISSING_ONLY_BUILD`.

Fluent job-builder (кілька outputs / явний контроль):

```python
res = (project.new_job("RECURSIVE_BUILD")
       .with_output("orders_by_country")
       .start_and_wait())                        # блокує до завершення

# Або раздільно + поллінг:
job = project.new_job("RECURSIVE_BUILD").with_output("orders_by_country").start()
import time
state = ""
while state not in ("DONE", "FAILED", "ABORTED"):
    time.sleep(2)
    state = job.get_status()["baseStatus"]["state"]
print(state)
```

> `project.new_job(type)` повертає `JobDefinitionBuilder` (`.with_output()`, `.start()`, `.start_and_wait()`).

---

## 8. Scenarios

```python
# Створити step-based scenario (ідемпотентно)
existing = {s.id for s in project.list_scenarios(as_type="objects")}
if "DAILY_BUILD" not in existing:
    scenario = project.create_scenario("Daily build", type="step_based")
else:
    scenario = project.get_scenario("DAILY_BUILD")

# Додати build-step. Зручного add_build_step НЕМАЄ — редагуємо raw_steps:
st = scenario.get_settings()                     # StepBasedScenarioSettings
st.raw_steps.append({
    "id": "build_orders",
    "name": "Build orders_by_country",
    "type": "build_flowitem",                    # тип build-кроку
    "params": {
        "builds": [{"type": "DATASET", "itemId": "orders_by_country",
                    "partitionsSpec": ""}],
        "buildMode": "RECURSIVE_BUILD",
    },
})
st.add_periodic_trigger(every_minutes=1440)      # опц. тригер: раз на добу
st.save()

# Запустити синхронно і дочекатися
run = scenario.run_and_wait()
print(run.outcome)                               # SUCCESS / WARNING / FAILED / ABORTED
```

> `create_scenario(scenario_name, type, definition=None)`, `type` ∈ `'step_based'` / `'custom_python'`. `raw_steps` — **список dict** (структуру step звір з UI-сценарієм через `get_settings().raw_steps`). Тригери: `add_periodic_trigger(every_minutes=5)`, `add_daily_trigger(...)`, `add_hourly_trigger(...)`. `run_and_wait(params=None, no_fail=False)` → `DSSScenarioRun` (`.outcome`). `run()` НЕ запускає миттєво — fires trigger; для async: `run().wait_for_scenario_run()`.

---

## 9. End-to-end приклад (ідемпотентний)

Підключитись → get/create тестовий проєкт → external Redshift dataset → Python recipe (чистка) → managed output → build → scenario.

```python
import dataikuapi, os

client = dataikuapi.DSSClient(os.environ["DSS_HOST"], os.environ["DSS_API_KEY"])
client._session.verify = False                   # ⚠️ on-prem self-signed; краще CA-bundle

PKEY, CONN = "SANDBOX_FLOW", "redshift_main"      # ⚠️ підстав свою Redshift connection

# 1) Проєкт
if PKEY not in client.list_project_keys():
    project = client.create_project(PKEY, "Sandbox Flow", owner="your_login",
                                    tags=["api-created", "sandbox"])
else:
    project = client.get_project(PKEY)

ds_names = {d.name for d in project.list_datasets()}
rec_names = {r.name for r in project.list_recipes()}

# 2) External dataset на Redshift (читання наявної таблиці)
if "orders_raw" not in ds_names:
    raw = project.create_dataset("orders_raw", type="Redshift", params={
        "connection": CONN, "mode": "table", "table": "orders", "schema": "public",
    })  # ⚠️ params звірити з UI-датасетом; потім детектнути схему (UI або set_schema_from_dataset)

# 3) Python recipe (чистка) + managed output orders_clean — створюються разом
if "compute_orders_clean" not in rec_names:
    code = (
        "import dataiku\n"
        "df = dataiku.Dataset('orders_raw').get_dataframe()\n"
        "df = df[df['amount'] > 0]\n"
        "dataiku.Dataset('orders_clean').write_with_schema(df)\n"
    )
    b = project.new_recipe("python", "compute_orders_clean")     # CodeRecipeCreator
    b.with_input("orders_raw")
    b.with_new_output_dataset("orders_clean", CONN, overwrite=True)
    b.with_script(code)
    b.create()

# 4) Build
project.get_dataset("orders_clean").build(job_type="RECURSIVE_BUILD")

# 5) (опц.) Scenario з build-кроком
if "DAILYBUILD" not in {s.id for s in project.list_scenarios(as_type="objects")}:
    sc = project.create_scenario("Daily build", type="step_based")
    sst = sc.get_settings()
    sst.raw_steps.append({
        "id": "build_clean", "name": "Build orders_clean", "type": "build_flowitem",
        "params": {"builds": [{"type": "DATASET", "itemId": "orders_clean"}],
                   "buildMode": "RECURSIVE_BUILD"},
    })
    sst.save()
    # sc.run_and_wait()  # запустити вручну за потреби

print("Flow готовий:", [d.name for d in project.list_datasets()])
```

---

## Чеклист безпеки
- [ ] API key — з env var (`DSS_API_KEY`), **ніколи** не в repo/коді/MEMORY.md.
- [ ] Працюємо в **тестовому** проєкті (`SANDBOX_*`), не в проді.
- [ ] `verify` TLS: усвідомлено `False` лише для dev; у проді — CA-bundle банку.
- [ ] Права ключа = **твої** права в DSS; усі дії в audit logs від твого імені.
- [ ] Усі create — за патерном "перевір-перед-створенням" (повторний запуск безпечний).
- [ ] `overwrite=True` на output робить build ідемпотентним (не append наосліп).
- [ ] `project.delete()` / будь-який delete — лише свідомо, з підтвердженням; незворотно.
- [ ] Великі дані ззовні — pushdown SQL (`client.sql_query`), не `iter_rows()` у pandas.

---

## Перехресні посилання
- [`docs/11-programmatic-api-python-client.md`](../../../../docs/11-programmatic-api-python-client.md) — повний reference `dataiku`/`dataikuapi`, auth, jobs, metrics, sql_query, API node.
- [`docs/06-coding-recipes-code-envs-notebooks.md`](../../../../docs/06-coding-recipes-code-envs-notebooks.md) — `dataiku` internal API, code recipes, code envs (Режим B).
- [`docs/02-projects-flow-core-concepts.md`](../../../../docs/02-projects-flow-core-concepts.md) — projects, Flow, recipes, build, zones (об'єкти, якими керує цей API).
- [`MEMORY.md`](../../../../MEMORY.md) — фактичний інстанс: версія 14.5.1, Design node, connections (Redshift/SQL/Sheets/Jira), рішення про Режим A.
- `references/code-snippets.md` — патерни `dataiku`/`dataikuapi` (читання/запис, SQL pushdown, scenarios).

### Офіційні джерела (звіряти сигнатури)
- Recipes API reference: https://developer.dataiku.com/latest/api-reference/python/recipes.html
- Projects (concepts): https://developer.dataiku.com/latest/concepts-and-examples/projects.html
- Datasets (concepts): https://developer.dataiku.com/latest/concepts-and-examples/datasets/datasets-other.html
- Scenarios API reference: https://developer.dataiku.com/latest/api-reference/python/scenarios.html
- GitHub (джерело сигнатур): https://github.com/dataiku/dataiku-api-client-python — `dataikuapi/dss/recipe.py`, `project.py`, `dataset.py`, `scenario.py`
