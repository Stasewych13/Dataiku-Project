# Dataiku — патерни коду

Перевірені патерни для `dataiku` (всередині DSS) та `dataikuapi` (ззовні). Деталі — `docs/06` та `docs/11`.

> Звіряй точні сигнатури з версією свого DSS — окремі параметри змінюються між 13.x/14.x.

---

## 1. `dataiku` — всередині recipe / notebook / scenario

### Читання та запис (малі/середні дані)
```python
import dataiku

# read
ds_in = dataiku.Dataset("input_dataset")
df = ds_in.get_dataframe()           # вся таблиця у pandas

# transform
df["new_col"] = df["a"] + df["b"]

# write (output dataset рецепта)
ds_out = dataiku.Dataset("output_dataset")
ds_out.write_with_schema(df)         # пише дані + оновлює схему
```

### Великі дані — chunked (не вантажити все в пам'ять)
```python
import dataiku

ds_in = dataiku.Dataset("big_input")
ds_out = dataiku.Dataset("big_output")

# поки читаємо порціями — одразу пишемо
with ds_out.get_writer() as writer:
    for chunk in ds_in.iter_dataframes(chunksize=100_000):
        chunk["flag"] = chunk["amount"] > 0
        writer.write_dataframe(chunk)
# Важливо: схему output задай заздалегідь (UI) або скористайся write_schema_from_dataframe
```

### Порядкова ітерація рядків (дуже великі, без pandas)
```python
ds = dataiku.Dataset("input")
for row in ds.iter_rows():           # row — dict-подібний
    process(row["col"])
```

### SQL pushdown із Python (обчислення в БД)
```python
import dataiku
from dataiku import SQLExecutor2

executor = SQLExecutor2(connection="my_postgres")   # або dataset=...
df = executor.query_to_df("""
    SELECT country, COUNT(*) AS n
    FROM "${projectKey}_orders"
    GROUP BY country
""")
```

### Managed folders (файли)
```python
folder = dataiku.Folder("folder_id_or_name")
paths = folder.list_paths_in_partition()
with folder.get_download_stream("/data/file.csv") as f:
    content = f.read()
with folder.get_writer("/out/result.json") as w:
    w.write(b'{"ok": true}')
```

### Variables та секрети
```python
import dataiku
client = dataiku.api_client()
proj = client.get_default_project()

vars = dataiku.get_custom_variables()   # project + global variables
threshold = float(vars.get("threshold", "0.5"))

# секрети користувача (User Secrets), не хардкод:
auth = client.get_auth_info(with_secrets=True)
secret = next((s["value"] for s in auth["secrets"] if s["key"] == "MY_KEY"), None)
```

### Scenario step — Python (всередині scenario)
```python
from dataiku.scenario import Scenario
s = Scenario()

# побудувати dataset у межах scenario
s.build_dataset("clean_orders")

# виставити scenario-змінну для наступних кроків
s.set_scenario_variables(run_date="2026-06-24")

# надіслати кастомне повідомлення через налаштований reporter
# (зазвичай простіше використати visual step "Send message")
```

### Custom metric (probe) і custom check
```python
# Custom metric: повертає dict {metric_id: value}
def process(dataset, partition_id):
    df = dataset.get_dataframe()
    return {"avg_amount": float(df["amount"].mean())}

# Custom check: оцінює останні значення metric -> (status, message)
def process(last_values, dataset):
    v = last_values["avg_amount"].get_value()
    if v is None:
        return "WARNING", "no value"
    return ("OK", "ok") if v > 0 else ("ERROR", f"avg<=0: {v}")
```

---

## 2. `dataikuapi` — ззовні DSS (автоматизація, CI/CD)

### Підключення
```python
import dataikuapi, os

client = dataikuapi.DSSClient(
    host=os.environ["DSS_HOST"],          # https://dss.example.com
    api_key=os.environ["DSS_API_KEY"],    # ніколи не хардкодь
)
client._session.verify = True             # TLS

projects = client.list_project_keys()
```

### Робота з проєктом / датасетами
```python
project = client.get_project("MYPROJECT")

for d in project.list_datasets():
    print(d["name"], d["type"])

ds = project.get_dataset("orders")
schema = ds.get_schema()

# читання даних ззовні (porційно)
data = ds.get_data(format="tsv-excel-header")   # streaming
```

### Запуск scenario та очікування
```python
scenario = project.get_scenario("DAILY_BUILD")
run = scenario.run_and_wait()                  # блокує до завершення
print(run.get_info().get("result", {}).get("outcome"))   # SUCCESS / FAILED
```

### Запуск job (build) програмно
```python
definition = {
    "type": "RECURSIVE_BUILD",
    "outputs": [{"id": "final_dataset", "type": "DATASET"}],
}
job = project.start_job(definition)
job.wait_for_completion()
```

### Metrics ззовні
```python
ds = project.get_dataset("orders")
metrics = ds.get_last_metric_values()
val = metrics.get_global_value("records:COUNT_RECORDS")
```

### Variables (override між середовищами)
```python
v = project.get_variables()
v["standard"]["target_connection"] = "prod_snowflake"
project.set_variables(v)
```

### MLOps: bundle (Design → Deployer)
```python
# на Design node: створити та опублікувати bundle
project.export_bundle("v_2026_06_24")
project.publish_bundle("v_2026_06_24")

# через Project Deployer: deploy на Automation infrastructure
deployer = client.get_projectdeployer()
# ... get_deployment(...).start_update() — див. docs/10
```

### API node — real-time scoring
```python
from dataikuapi import APINodeClient

apinode = APINodeClient("http://apinode:12000", "my_service")
pred = apinode.predict_record("my_endpoint", {
    "feature_a": 1.0, "feature_b": "x",
})
print(pred["result"]["prediction"], pred["result"].get("probaPercentile"))
```

---

## 3. Прямий REST / curl
```bash
# Public API під /public/api/, Basic auth з API key
curl -u "$DSS_API_KEY:" \
  "https://dss.example.com/public/api/projects/MYPROJECT/datasets/"

# API node prediction
curl -X POST "http://apinode:12000/public/api/v1/my_service/my_endpoint/predict" \
  -H "Content-Type: application/json" \
  -d '{"features": {"feature_a": 1.0, "feature_b": "x"}}'
```

---

## Чеклист безпеки коду
- [ ] api_key / паролі — з env vars або `get_secret`, не в коді.
- [ ] Хости dev/prod — параметризовані, не зашиті.
- [ ] Великі дані — chunked або SQL pushdown, не суцільний `get_dataframe()`.
- [ ] Запис ідемпотентний (overwrite зі схемою), не сліпий append.
- [ ] Admin-операції (`dataikuapi` users/connections) — окремо, з підтвердженням.
