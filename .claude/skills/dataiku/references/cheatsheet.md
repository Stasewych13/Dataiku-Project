# Dataiku — швидка шпаргалка

Стисла довідка для роботи в Dataiku. Для деталей — профільний документ у `docs/` (вказано в дужках).

## Flow: семантика елементів (02, 04)
- **Dataset** — синій квадрат (іконка = тип connection). Чорний контур = foreign (з іншого проєкту).
- **Recipe** — кружок: жовтий = visual, помаранчевий = code, червоний = plugin.
- **Saved Model / Model Evaluation Store** — зелений.
- **Knowledge Bank** — рожевий (RAG / GenAI).
- Стрілки = lineage (зліва направо). Recipe завжди між input і output dataset.

## Build modes (02)
- **Non-recursive** — будує лише обраний dataset (вхідні мають бути актуальні).
- **Recursive / smart reconstruction** — добудовує застарілі предки автоматично.
- **Forced recursive** — перебудовує всіх предків незалежно від стану.

## Visual recipes — коли що (04)
| Потрібно | Recipe |
|----------|--------|
| Копія/конвертація формату/connection | **Sync** |
| Агрегації (sum/avg/count за ключем) | **Group** |
| З'єднати два датасети за ключем | **Join** |
| Неточне з'єднання | **Fuzzy join** |
| Об'єднати рядки (UNION) | **Stack** |
| Віконні функції (lag/lead/rank/cumulative) | **Window** |
| Перші N за групою | **TopN** |
| Унікальні рядки / дублікати | **Distinct** |
| Long↔wide | **Pivot** |
| Розділити на кілька виходів | **Split** |
| Фільтр/семпл | **Sampling/Filtering** |
| Очистка/трансформація колонок | **Prepare** (05) |

## Recipe engines (04)
- **DSS (local stream)** — у пам'яті DSS. Дефолт, але повільно для великих даних.
- **In-database (SQL)** — обчислення в БД (pushdown). Швидко, коли input+output на одній SQL-connection.
- **Spark** — розподілено. Для дуже великих даних.
- Engine видно/змінюється під кнопкою **Run** (шестерня). «Not selectable» → різні connections, непідтримувана операція, SQL-query input тощо.

## Prepare — типове (05)
- **Step** = крок скрипта. Бібліотека processors: cleansing, dates parse/format, split/extract/replace, regex, **Formula**, filter/flag rows, columns ops, pivot/fold, geo, text, JSON.
- **Storage type** (int/string/date…) ≠ **Meaning** (semantic: Email, URL, Country…). Meaning дає валідацію.
- **Formula language** (часто звуть «GREL»): `if(cond, a, b)`, `length(x)`, `strSplit()`, `toUppercase()`, дати `datePart()`, `inc()`, `diff()`. Офіційна назва — *DSS formula language*.

## Datasets & connections (03)
- **Managed** = DSS володіє storage (output recipes). **External/uploaded** = читаєш чуже.
- Великі дані → **Parquet** (columnar, стиснення) замість CSV.
- **Partitioning**: file-based (`%Y/%M/%D`) або SQL column-based; будуй конкретні партиції, не весь датасет.
- Credentials: **global** (один сервісний акаунт) vs **per-user** (потребує UIF).

## ML (07)
- **Lab** → Visual analysis → дизайн моделі. Tasks: binary/multiclass/regression, clustering.
- AutoML templates: Quick Prototypes / Interpretable / High Performance / Expert.
- Deploy у Flow → **Saved Model** (версії, activate). **Score recipe** (batch) vs API endpoint (real-time, 10).
- **Evaluate recipe** + **Model Evaluation Store** → drift. Уникай **data leakage** (no target-derived features).

## Scenarios (09)
- **Trigger** (time/cron, dataset change, SQL change, custom Python) → **Steps** (Build, Run checks, Compute metrics, Set variables, Run scenario, Custom Python…) → **Reporter** (email/Slack/Teams/webhook).
- **Metric** = виміряне число (record count, col stats…). **Check** = умова на metric (OK/WARNING/ERROR), може зафейлити scenario.
- **Data Quality rules** — новіший шар поверх metrics/checks (статуси, dashboard).

## MLOps lifecycle (10)
- Design node → створюєш **bundle** → publish у **Project Deployer** → deploy/activate на **Automation node**. Remap connections/variables між середовищами. Rollback = activate попередній bundle.
- **API node** + **API Deployer** — real-time. Endpoints: prediction (Saved Model), Python function, SQL query, dataset lookup.

## Програмний доступ (11)
- **`dataiku`** — ВСЕРЕДИНІ DSS (recipes, notebooks, scenarios). `dataiku.Dataset("name")`.
- **`dataikuapi`** — ЗЗОВНІ. `dataikuapi.DSSClient(host, api_key)`.
- Усередині notebook: `dataiku.api_client()` дає DSSClient без ключа.
- Ключі: personal / global API key. Не хардкодь — env var.

## Антипатерни (стоп-сигнали) (15)
- ❌ `get_dataframe()` на величезному датасеті (OOM) → chunked / SQL.
- ❌ Recipe engine = DSS, коли все в SQL → втрата pushdown.
- ❌ Append без overwrite → недетермінований білд.
- ❌ Хардкод credentials у коді.
- ❌ Розробка прямо на Automation/prod.
- ❌ Один гігантський Prepare на 100 кроків замість логічно поділеного Flow.
- ❌ Нотифікації на кожен запуск (alert fatigue) — лише на реальні фейли.
