# 09. Scenarios, Automation, Metrics та Checks

> Версія документа орієнтована на **Dataiku DSS 13.x / 14.x** (2025–2026). Технічні терміни, назви елементів інтерфейсу (UI) та код подано в **оригіналі (англійською)**; пояснення — українською.
>
> Рівні складності позначено так: **(Базово)** — для новачка, **(Середній)** — для впевненого користувача, **(Просунуто)** — для досвідченого аналітика/інженера.

---

## Зміст

1. [Вступ](#вступ)
2. [Scenarios: концепція](#scenarios-концепція)
3. [Step-based vs Custom Python scenario](#step-based-vs-custom-python-scenario)
4. [Scenario steps: повний довідник](#scenario-steps-повний-довідник)
5. [Run conditions та error handling](#run-conditions-та-error-handling)
6. [Triggers](#triggers)
7. [Reporters / notifications](#reporters--notifications)
8. [Metrics](#metrics)
9. [Checks](#checks)
10. [Data Quality (rules)](#data-quality-rules)
11. [Співвідношення: Data Quality vs Metrics/Checks](#співвідношення-data-quality-vs-metricschecks)
12. [Scenario runs, логи, monitoring, diagnostics](#scenario-runs-логи-monitoring-diagnostics)
13. [Приклади (end-to-end)](#приклади-end-to-end)
14. [Best practices](#best-practices)
15. [Типові помилки](#типові-помилки)
16. [Перехресні посилання](#перехресні-посилання)
17. [Джерела](#джерела-офіційна-документація-dataiku)

---

## Вступ

**(Базово)** Усе, що в DSS можна зробити вручну (build dataset, train model, обчислити metric), можна **автоматизувати**. Центральний інструмент автоматизації — **scenarios**. Scenario визначає **що робити** (послідовність steps), **коли** (triggers) і **кого повідомити** (reporters).

Поряд із scenarios працюють два механізми контролю якості та стану даних:

- **Metrics** — числові вимірювання об'єктів Flow (record count, середнє по колонці, розмір файлу тощо), які DSS обчислює та зберігає в історії.
- **Checks** (legacy) та **Data Quality rules** (новіше) — правила, що перевіряють metrics/дані й видають статус **OK / WARNING / ERROR**. ERROR може «завалити» build або scenario.

Разом це утворює основу **automation & data observability**: пайплайн будується за розкладом, дані перевіряються, команда отримує сповіщення лише коли щось дійсно зламалося.

---

## Scenarios: концепція

**(Базово)** **Scenario** — об'єкт проєкту (меню **Jobs > Scenarios** або іконка автоматизації). Він дозволяє «build datasets or train models in an automated fashion» — будувати датасети, тренувати моделі, рахувати metrics, запускати checks, надсилати повідомлення тощо без ручного втручання.

Анатомія scenario:

| Вкладка | Призначення |
|---|---|
| **Steps** (або **Script**) | Що робити: послідовність steps або Python-скрипт |
| **Settings** | Triggers, reporters, concurrency, активність auto-triggers |
| **Last runs** | Історія запусків, статуси, логи, timeline |

**(Середній)** Ключовий перемикач у верхній частині scenario — **Auto-triggers (Active / Off)**. Якщо scenario неактивний, triggers **не оцінюються** — scenario можна запустити лише вручну (**Run** в Actions). Це головний запобіжник, щоб тестовий scenario не запустився на проді.

> **Важливо щодо variables у полях steps.** У текстових полях scenario steps використовуються **DSS Formulas** (наприклад, `outcome == 'SUCCESS'`), а **не** `${variable}`-expansion як у code recipes. Для підстановки змінних у формулах звертайтеся до них напряму за іменем або через `variables["myvar"]`.

---

## Step-based vs Custom Python scenario

**(Середній)** При створенні scenario обираєте тип:

### Step-based (Sequence of steps)

Фіксована **послідовність parametrized steps**, що виконуються по порядку. Кожен step має свою конфігурацію в UI, run condition та налаштування error handling. Можливий обмежений flow control (умовний запуск step). Це **рекомендований дефолт** — простіше читати, версіонувати, передавати колегам.

Використовуйте, коли потрібні стандартні build / train / compute metrics / report у зрозумілій лінійній логіці.

### Custom Python scenario

Повноцінний **Python-скрипт**, який може робити все, що й step-based, плюс довільну логіку: цикли, складні умови, виклики зовнішніх API, динамічний вибір датасетів. Скрипт використовує клас `Scenario` з пакета `dataiku.scenario`.

Використовуйте, коли:
- потрібна **складна умовна логіка** (запускати step чи ні залежно від metric / зовнішнього стану);
- треба слати **повідомлення посеред виконання** (не лише на start/end);
- треба читати **детальні metadata** про виконані steps (статуси, кількість warnings, перелік побудованих датасетів);
- треба програмно **активувати версію моделі**.

**(Просунуто)** Базовий приклад custom scenario:

```python
from dataiku.scenario import Scenario

scenario = Scenario()

# Build датасета (smart-режим за замовчуванням)
scenario.build_dataset("customers_enriched")

# Build конкретних партицій
scenario.build_dataset("sales_daily", partitions="2026-06-23,2026-06-24")

# Train моделі за її ID (видно в URL налаштувань моделі)
scenario.train_model("epae130z")

# Прочитати результат і відреагувати
last_step = scenario.get_previous_steps_outputs()[-1]
if last_step["result"]["outcome"] != "SUCCESS":
    sender = scenario.get_message_sender("ops-slack")
    sender.send(message="Build failed, see scenario run")
```

> **Порада:** навіть для custom scenario тригери та reporters зручніше тримати у вкладці **Settings** (UI), а в Python-скрипті описувати лише логіку steps.

---

## Scenario steps: повний довідник

**(Середній)** Нижче — типові step-types step-based scenario (вкладка **Steps > Add step**).

### Робота з даними та Flow

| Step | Що робить |
|---|---|
| **Build / Train** | Будує datasets (включно з партиціями), managed folders, saved models з Flow. Підтримує build modes (smart / forced recursive / тощо) |
| **Clear** | Очищає дані датасетів або вміст managed folders та їхніх партицій |
| **Synchronize Hive table** | Оновлює схему зовнішньої Hive-таблиці під схему пов'язаного HDFS-датасета |
| **Create / clear (folder)** | Створення/очищення managed folder як частина пайплайну |

### Якість даних та вимірювання

| Step | Що робить |
|---|---|
| **Compute metrics** | Запускає probes (зі Status-tab датасета/folder/моделі). Значення metrics стають доступними як variables для наступних steps |
| **Run checks** | Виконує legacy checks на folders, saved models, evaluation stores. ERROR → step fails |
| **Verify rules / Data Quality** | Обчислює **Data Quality rules** на датасетах. Статус ERROR валить step (і scenario) |

### Variables

| Step | Що робить |
|---|---|
| **Define scenario variables** | Задає **scenario-local** variables (живуть лише в межах цього запуску), JSON або key-value з формулами |
| **Set project variables** | Перезаписує **project-level** variables (зберігаються між запусками) |
| **Set global variables** | Перезаписує instance-level (global) variables |
| **Run global variables update** | Виконує заздалегідь налаштоване в Administration оновлення глобальних variables |

### Виконання коду / SQL

| Step | Що робить |
|---|---|
| **Execute SQL** | Виконує SQL/HiveQL на DSS connection; результат query доступний як variables |
| **Execute Python code** | Виконує Python-чанк з доступом до Dataiku API та параметрів trigger |
| **Execute Python unit test** | Запускає pytest-тести з Libraries-теки |

### Інтеграція, експорт, оркестрація

| Step | Що робить |
|---|---|
| **Send message** | Надсилає повідомлення через reporter-канал посеред виконання (аналог reporter, але як крок) |
| **Run another scenario** | Запускає інший scenario й **чекає** завершення; успадковує його outcome |
| **Package API service** | Створює пакет API service (підтримує variable-expressions для авто-нумерації версій) |
| **Create dashboard export** | Генерує експорт дашборду в managed folder (PDF/зображення, кастомні розміри) |
| **Create notebook export** | Перепубліковує Python notebook у пов'язаний insight (опц. виконує його перед цим) |
| **Run integration test** / **WebApp test** | Non-regression тести, перевірка доступності webapp |
| **Kill scenario** | Зупиняє виконання scenario (наприклад, у гілці error-handling) |

> **Порядок steps** має значення: спершу **Build**, потім **Compute metrics**, потім **Run checks / Verify rules**, наприкінці **Send message**. Якщо чек має валити пайплайн — ставте його **до** кроків, що залежать від валідних даних.

---

## Run conditions та error handling

**(Середній)** Кожен step має дві важливі групи налаштувань.

### Run condition (умовний запуск step)

Визначає, **чи виконувати** цей step. Типові опції:

- **Always** — завжди (дефолт);
- **Never** — вимкнено (зручно для тимчасового відключення);
- **If no prior step failed** — лише якщо попередні steps успішні;
- **If some prior step failed** — лише якщо щось зламалося (гілка обробки помилок: cleanup, alert);
- **Run this step based on a condition** — довільна **DSS Formula**, наприклад:

```text
outcome == 'SUCCESS' && variables["records_count"] > 1000
```

Це дає базовий flow control: «надіслати success-нотифікацію лише якщо все ОК» або «зробити rollback лише якщо впав build».

### Error handling (поведінка при падінні step)

Для кожного step налаштовується, що робити, якщо він **fails**:

- **Fail the scenario** (зупинити подальші steps) — за замовчуванням для критичних кроків;
- **Mark scenario as failed but continue** — позначити scenario як failed, але виконати решту steps (наприклад, щоб дійти до error-notification step);
- **Continue (ignore failure)** — продовжити, наче нічого не сталося (для необов'язкових кроків).

**(Просунуто)** Типовий патерн надійного scenario:

1. Build (Fail scenario on error).
2. Compute metrics + Verify rules (Fail scenario on error).
3. Send message — **Send on SUCCESS** (run condition `outcome == 'SUCCESS'`).
4. Send message — **Send on FAILURE** (run condition `If some prior step failed`), error handling = Continue.

---

## Triggers

**(Базово)** **Triggers** автоматично запускають scenario «based on several conditions». Кожен trigger можна **окремо ввімкнути/вимкнути**, але вони оцінюються **лише коли весь scenario Active** (перемикач Auto-triggers). Якщо активних triggers кілька — scenario запускається, коли **будь-яка** умова стає true.

### Типи triggers

#### 1. Time-based (cron-like)

**(Базово)** Запуск за розкладом. Периодичність: **monthly / weekly / daily / hourly**, плюс точний час у межах періоду (напр., daily о **22:00**). Time-based trigger має вбудований grace delay ~2 сек.

#### 2. Trigger on dataset change

**(Середній)** Спрацьовує при зміні вмісту/налаштувань датасета:

- **Filesystem datasets** (HDFS, uploaded): DSS виявляє зміни в іменах файлів, розмірах, timestamps модифікації.
- **SQL datasets**: **НЕ** виявляють зміни даних — для них використовуйте SQL-trigger.
- Опція **marker file** — вказати конкретний файл, зміна якого = «датасет оновився» (захист від false-triggers через інші файли).

#### 3. Trigger on SQL query result change

**(Середній)** «Run a query and activate when the output of the query changes» — DSS періодично виконує query й порівнює результат з попереднім. Типові приклади: `COUNT(*)` (з'явилися нові рядки) або `MAX(updated_at)`.

#### 4. Trigger on change of file in a folder

**(Середній)** Спрацьовує при зміні файлів у managed folder (поява/зміна/видалення) — корисно для inbox-патернів (зовнішня система кладе файл → пайплайн стартує).

#### 5. After another scenario

**(Середній)** Запуск «after the end of another scenario, optionally with a condition on the followed scenario's outcome» — оркестрація ланцюжків (напр., запускати downstream-scenario лише якщо upstream завершився SUCCESS).

#### 6. Custom Python trigger

**(Просунуто)** Максимальна гнучкість: Python-скрипт сам вирішує, чи активувати trigger (можна звертатися до зовнішніх API, бази, файлів).

```python
from dataiku.scenario import Trigger

t = Trigger()

# Будь-яка логіка: тут — приклад звернення до зовнішньої умови
import requests
ready = requests.get("https://my-api.example/ready").json().get("ready", False)

if ready:
    # fire() запускає scenario; можна передати параметри
    t.fire()
    # t.fire(params={"source": "external_api"})  # параметри доступні steps
```

### Evaluation interval та grace delay

**(Середній)** Усі triggers, крім time-based, мають **evaluation interval** — DSS перевіряє умову (timestamps файлів, SQL query, Python) на кожному інтервалі.

- **Grace delay** — захист від передчасного запуску, коли ресурс ще змінюється. З **Re-check enabled** DSS чекає стабілізації стану; без re-check — чекає фіксований проміжок (напр., 90 сек), потім запускає.
- **Suppress triggers while running** (увімкнено за замовчуванням) — triggers не оцінюються, поки scenario вже виконується (захист від перекриття запусків).
- **Delay runs** + **Squash delayed runs** — якщо suppress вимкнено, можна чергувати запуски; squash залишає лише найсвіжіший у черзі.

### Активація / деактивація

**(Базово)** Два рівні:

1. **Scenario Auto-triggers: Active / Off** — глобальний перемикач для всіх triggers scenario.
2. **Окремий trigger: enabled / disabled** — у налаштуваннях конкретного trigger.

> На **Design node** scenarios зазвичай тримають **Off** (запуск вручну для тестів). На **Automation node** — **Active**. Див. розділ best practices.

---

## Reporters / notifications

**(Базово)** **Reporters** інформують команду про активність scenario (успіх, падіння, зміни якості даних). Повідомлення можна слати у **трьох точках**:

| Момент | Як налаштувати |
|---|---|
| **Start of scenario** | Reporter у Settings, **Send on scenario = Start** |
| **During execution** | Step **Send message** (step-based) або `get_message_sender()` (Python) |
| **End of scenario** | **Send on scenario = End** + run condition: `outcome == 'SUCCESS'` / `'FAILED'` / `'ABORTED'` |

### Канали (reporter types)

**(Середній)** Перед використанням адміністратор має налаштувати канал у **Administration > Settings > Notifications & Integrations**.

| Канал | Налаштування / нюанси |
|---|---|
| **Email (SMTP)** | Mail channel у Administration. Templating: Freemarker (дефолтний шаблон) або inline через **DSS Formulas**. Recipients — звичайний рядок або JSON-array. Підтримка Microsoft 365 (OAuth2, `Mail.Send`) |
| **Slack** | Incoming webhook (простіше) або bot user (керований вигляд). Templating через DSS Formulas; HTML рендериться як є |
| **Microsoft Teams** | Workflow-шаблон «Post to a channel when a webhook request is received». Простий текст або складні **Adaptive Cards** (JSON: заголовки, кольори, fact sets) |
| **Google Chat** | Webhook на каналі; cards не підтримуються |
| **Webhook** | Довільний HTTP-endpoint; додаткові параметри (колір, alias) через DSS Formulas |
| **Twilio (SMS)** | Через DSS Formulas |
| **Shell** | Передає результати scenario у shell-скрипт через environment variables |
| **Dataset (audit log)** | Записує результати запусків scenario у dataset для подальшого аналізу/моніторингу |

### Шаблони повідомлень та змінні

**(Середній)** Доступні variables: дані про DSS instance, project, scenario run. Корисні:

- `${scenarioRunURL}` — пряме посилання на run (потребує налаштованого **DSS URL** в Administration > Notifications & Integrations > DSS Location);
- `outcome` — SUCCESS / FAILED / ABORTED / WARNING;
- назва scenario, project key, час запуску.

**(Просунуто)** Надсилання повідомлень із Python (custom scenario або Execute Python step):

```python
from dataiku.scenario import Scenario
scenario = Scenario()

# Канал, заздалегідь налаштований адміністратором
sender = scenario.get_message_sender("ops-teams")
sender.send(message="Scenario is working as expected")

# Email з HTML
mail = scenario.get_message_sender("ops-mail")
html = "<p style='color:orange'>Daily build finished with WARNING</p>"
mail.send(subject="Daily build", message=html, sendAsHTML=True)
```

> **На яких подіях слати.** Найкорисніша практика: **FAILURE** завжди; **SUCCESS** — лише для критичних щоденних пайплайнів (інакше канал зашумлюється). Див. best practices.

---

## Metrics

**(Базово)** **Metrics** — автоматичні числові вимірювання об'єктів Flow: datasets, managed folders, saved models. Налаштовуються у вкладці **Status** відповідного об'єкта. Уся система побудована навколо концепції **probe**.

> **Probe** — компонент, що обчислює **кілька metrics за один прохід** даними. DSS автоматично об'єднує probes зі спільними обчисленнями в один прохід для ефективності.

### Вбудовані probes датасетів

**(Середній)**

| Probe | Metrics |
|---|---|
| **Basic info** | Розмір датасета, кількість файлів (де доречно) |
| **Records** | Кількість записів (`COUNT_RECORDS`). На non-Hadoop/non-SQL — повний enumerate, може бути дорого |
| **Partitioning** | Кількість партицій, перелік партицій |
| **Basic column statistics** | MIN, MAX, AVG по обраних колонках |
| **Advanced column statistics** | Most frequent / top-N значення (дорожче — окремий probe) |
| **Data validity statistics** | Кількість/частка невалідних значень за meaning/storage type (missing, min, max, mean, distinct тощо) |

**(Середній)** Metrics доступні також на:

- **Managed folders** — розмір, кількість файлів, кастомні probes;
- **Saved models** — performance-метрики (AUC, accuracy, тощо) активної версії;
- **Evaluation stores** — drift та performance metrics.

### Partition-level metrics

**(Просунуто)** На партиційованих датасетах metrics обчислюються **окремо на кожній партиції та на цілому датасеті**, бо для багатьох metrics «metric на цілому датасеті — це не сума metrics партицій» (наприклад, median). Перегляд — у вікнах **Partitions Table** / **Partitions Chart** (особливо корисно для time-based партицій).

### Custom metrics: SQL probe

**(Просунуто)** Для SQL-датасетів можна писати кастомні probes на SQL. Кожна колонка результату стає окремою metric. З опцією **«is a single aggregate»** ваш вираз автоматично обгортається `SELECT ... FROM ... WHERE ...`:

```sql
-- single aggregate:
SUM(cost) / COUNT(customers)
-- автоматично перетвориться на:
-- SELECT SUM(cost) / COUNT(customers) AS "col_0" FROM orders WHERE order_date='2026-06-23'
```

### Custom metrics: Python probe

**(Просунуто)** Python-probe — функція `process()`, що отримує об'єкт датасета (і опц. `partition_id`) і повертає **dict {ім'я метрики: значення}** або одне значення (тоді metric отримує ім'я `value`):

```python
import numpy as np

def process(dataset, partition_id):
    df = dataset.get_dataframe()
    df_per_cat = (df.groupby('item_category')
                    .agg({'authorized_flag': np.mean})
                    .sort_values('authorized_flag', ascending=False))
    return {
        'num_rows': df.shape[0],
        'num_cols': df.shape[1],
        'most_authorized_category': df_per_cat.index[0],
        'least_authorized_category': df_per_cat.index[-1],
    }
```

> Підпис без партицій — `def process(dataset):` — також валідний для непартиційованих об'єктів.

### Перегляд та історія metrics

**(Середній)** На вкладці **Status** доступні режими:

- **Tile view** — останні значення metrics; клік відкриває history-modal;
- **Ribbon view** — історичні тренди обраних metrics;
- **Partitions Table / Chart** — значення по партиціях.

Вибір показаних metrics — кнопка **«X/Y metrics»**. DSS **автоматично історизує** metrics: кожне обчислення зберігається, що дозволяє аналізувати тренди («як змінювався середній чек по місяцях»). Probes можна налаштувати на **auto-run after build**.

### Обчислення та читання metrics з коду

**(Просунуто)**

```python
import dataiku

dataset = dataiku.Dataset("orders")

# Обчислити всі metrics (probes)
dataset.compute_metrics()

# Прочитати останні значення (ComputedMetrics)
metrics = dataset.get_last_metric_values()
count = metrics.get_global_value('records:COUNT_RECORDS')       # ціле по датасету
# для партиції:
# metrics.get_partition_value('records:COUNT_RECORDS', '2026-06-23')

# Перелік усіх metric id
all_ids = metrics.get_all_ids()

# Історія однієї metric
history = dataset.get_metric_history('records:COUNT_RECORDS')
```

У scenario це покривається step **Compute metrics**, який ще й кладе значення metrics у variables для наступних steps.

---

## Checks

**(Базово)** **Checks** (legacy-механізм) автоматично **валідують** значення metrics об'єктів Flow. Тісно інтегровані з metrics: check читає **найсвіжіше значення metric** і порівнює з порогами. Сьогодні checks залишаються основним механізмом для **managed folders, saved models, evaluation stores**, тоді як для датасетів рекомендують **Data Quality rules** (див. нижче).

### Типи checks

**(Середній)**

1. **Numeric range** — metric має бути в діапазоні:
   - нижче **minimum** → ERROR;
   - нижче **soft minimum** → WARNING;
   - у межах → OK;
   - вище **soft maximum** → WARNING;
   - вище **maximum** → ERROR.
2. **Value in a set** — значення metric має бути в списку допустимих.
3. **Python (custom)** — довільна логіка.

### Статуси

**(Базово)** Check видає один із чотирьох статусів:

| Статус | Значення |
|---|---|
| **OK** | Умова виконана |
| **WARNING** | Перевищено soft-поріг (не валить build) |
| **ERROR** | Перевищено hard-поріг → **валить build/scenario** |
| **EMPTY** | Значення metric недоступне для оцінки |

### Вплив на scenario

**(Середній)** Checks виконуються під час build (якщо налаштовано auto-run) і в scenario через step **Run checks**. Якщо хоч один check повертає **ERROR** — step fails, і (за дефолтного error handling) **scenario fails**. Це і є механізм «зупинити пайплайн, якщо дані погані».

### Custom Python check

**(Просунуто)** Функція `process(last_values, dataset)`:

- `last_values` — **dict {metric id → MetricDataPoint}** з останніми значеннями metrics;
- повертає статус-рядок (`'OK'`/`'WARNING'`/`'ERROR'`) і **опційне повідомлення** другим елементом.

```python
def process(last_values, dataset):
    # Дістаємо metric value (рядок -> приводимо до потрібного типу)
    n = int(last_values['records:COUNT_RECORDS'].get_value())

    if n == 0:
        return 'ERROR', 'Dataset is empty'
    elif n < 1000:
        return 'WARNING', f'Only {n} records (expected >= 1000)'
    else:
        return 'OK', f'{n} records'
```

Для партиційованих об'єктів / folders підпис може містити `partition_id`:

```python
def process(last_values, managed_folder, partition_id):
    paths = managed_folder.list_paths_in_partition()
    if len(paths) == 0:
        return 'ERROR', 'Data is missing'
    return 'OK', 'Clear build'
```

> `MetricDataPoint` має `get_value()` (рядок), `get_type()` (BIGINT/DOUBLE/STRING/BOOLEAN), `get_compute_time()`.

---

## Data Quality (rules)

**(Базово)** **Data Quality rules** (DSS 12.6+) — **сучасна заміна** legacy checks **для датасетів**: спрощена конфігурація, окремі dashboards моніторингу на трьох рівнях, історичне відстеження, статистичне виявлення аномалій. Налаштовуються у вкладці **Data Quality** датасета.

Кожне виконання rule дає **OK / Warning / Error / Empty** (Empty — коли умову неможливо оцінити, напр., замало історії).

### Типи rules

**(Середній)**

**Статистичні діапазони (по колонці):**
- `Column min/max/avg/sum/median/std dev in range` — статистика в hard/soft межах;
- `... is within its typical range` — динамічне виявлення аномалій за історією (IQR).

**Порожнеча та унікальність:**
- `Column values are not empty` / `are empty` (усі або поріг %);
- `Column empty value is within its typical range`;
- `Column values are unique` / `unique value within typical range`.

**Обмеження значень:**
- `Column values in set` / `in range`;
- `Column values are valid according to meaning` (email, phone, URL…);
- `Column top N values in set` / `most frequent value in set`.

**На рівні датасета:**
- `Record count in range` / `within its typical range`;
- `Column count in range` / `within its typical range`;
- `File size in range` (file-based datasets).

**Схема:**
- `Dataset schema equals` (точна відповідність: колонки, типи, meanings, порядок);
- `Dataset schema contains` (наявність очікуваних колонок, порядок неважливий).

**Кастомні:**
- `Python code` — повертає `(status, observed_value)`;
- `Compare values of two metrics`;
- `Plugin rules`.

### Hard vs soft межі та change-detection

**(Просунуто)** Range-rules підтримують 4 пороги (minimum / soft minimum / soft maximum / maximum) з тією ж логікою, що й numeric-range check.

Rules «within its typical range» встановлюють діапазон автоматично з історії:
- **Learning period** — мінімум спостережень перед порівняннями (до того — EMPTY);
- **Lookback window** — горизонт історії (Days або Runs);
- **IQR factor** / **Soft IQR factor** — множники для hard/soft меж: `Q1/Q3 ± (factor × IQR)`.

### Auto-run after build

**(Середній)** З увімкненим **Auto-run after build** rule обчислюється після кожного build датасета — статус завжди синхронний із вмістом. Rule зі статусом **Error валить build job** (захист від поширення поганих даних downstream). Input datasets (джерела, не побудовані recipe) auto-run пропускають.

У scenario rules обчислюються через step **Verify rules** (Error → step fails → scenario fails).

### Custom Python rule

**(Просунуто)**

```python
def process(last_values, dataset):
    df = dataset.get_dataframe()
    avg_revenue = df['revenue'].mean()
    if avg_revenue >= 10000:
        return 'OK', f'Average revenue: {avg_revenue:,.2f}'
    elif avg_revenue >= 5000:
        return 'WARNING', f'Low average revenue: {avg_revenue:,.2f}'
    else:
        return 'ERROR', f'Critical: avg revenue {avg_revenue:,.2f}'
```

### Dashboards моніторингу

**(Середній)** Три рівні:

| Рівень | Що показує |
|---|---|
| **Dataset** | Current Status (поточні статуси rules), Timeline (еволюція по днях), Rule History (повний аудит обчислень) |
| **Project** | Агрегований найгірший статус по кожному monitored-датасету; Timeline по проєкту |
| **Instance** | Високорівневий dashboard (Navigation menu) — статуси всіх monitored-проєктів, доступних користувачу |

Датасет із ≥1 rule стає **monitored** за замовчуванням (керується **Monitoring flag**). Є також **Data Quality Flow view** (статуси прямо на Flow) і **Right Panel** (швидка перевірка з explorer/catalog).

### Data Quality Templates

**(Середній)** **Templates** дозволяють зберігати набори rules як шаблони й перевикористовувати на різних датасетах. Templates **глобальні** для інстансу — стандартні конфігурації якості шаряться по всій організації. Окремі rules можна copy/paste між датасетами і без templates.

---

## Співвідношення: Data Quality vs Metrics/Checks

**(Середній)** Коротко, як це поєднується:

- **Metrics** — фундамент: вони **вимірюють**. Існують для datasets, folders, models, evaluation stores.
- **Checks** (legacy) — **валідують metrics** і дають статус. Залишаються для **folders / saved models / evaluation stores**.
- **Data Quality rules** — **сучасний механізм валідації датасетів**, що замінює checks саме для датасетів. Обчислення rule **автоматично генерує відповідні metrics**; можна також будувати rules поверх наявних metrics (`Metric value in range / in set`).

**Коли що:**

| Ситуація | Інструмент |
|---|---|
| Перевірка якості **датасета** (новий проєкт) | **Data Quality rules** |
| Перевірка **managed folder / saved model / evaluation store** | **Checks** |
| Лише виміряти (без валідації) | **Metrics / probes** |
| Старі проєкти з вже налаштованими dataset checks | Працюють далі; нове — на Data Quality |

> Історичні нюанси: timeline Data Quality має «прогалини» до моменту впровадження фічі; legacy checks не з'являються у статусі, доки не перераховані.

---

## Scenario runs, логи, monitoring, diagnostics

**(Середній)** Вкладка **Last runs** scenario — головне місце спостереження:

- **Список запусків** з outcome (SUCCESS / WARNING / FAILED / ABORTED), часом старту, тривалістю, тригером, що ініціював запуск.
- Для кожного run — **timeline steps**: який step коли запустився, його статус, warnings.
- **Логи** кожного step та jobs, які він породив (build job має власний детальний лог із job diagnostics).
- Прив'язка до **trigger params** (видно, який trigger і з якими параметрами запустив).

**(Просунуто)** Моніторинг автоматизації на рівні інстансу:

- **Automation monitoring** (Automation node) — зведення по всіх scenarios: останні запуски, fail-rate, тривалості.
- **Dataset reporter** — пише результати кожного run у dataset, який можна аналізувати чартами/scenario (мета-моніторинг).
- **Data Quality instance dashboard** — окремо для статусів якості даних.
- Python-доступ до історії: через `dataikuapi` можна читати `scenario.get_last_runs()`, статуси, логи (див. doc 11).

**(Просунуто)** **Diagnostics** при падінні: відкрийте failed run → знайдіть перший step зі статусом FAILED → відкрийте його лог / пов'язаний job. Для build-кроків корисний **job > activities** з повним стектрейсом recipe. Перевіряйте також, чи не **EMPTY** check (часто означає, що metric не обчислилася — треба Compute metrics **перед** Run checks).

---

## Приклади (end-to-end)

### Приклад 1 (Середній): щоденний ETL з перевіркою якості та alert

Step-based scenario `daily_refresh`:

1. **Build** — `customers_enriched` (smart mode). Error handling: *Fail scenario*.
2. **Compute metrics** — на `customers_enriched`. *Fail scenario*.
3. **Verify rules** — Data Quality rules на `customers_enriched`:
   - `Record count in range` (min 10000),
   - `Column values are not empty` для `customer_id`,
   - `Column values are unique` для `customer_id`.
   Error handling: *Fail scenario*.
4. **Send message** (Teams) — run condition `If some prior step failed`, error handling *Continue*:
   `Daily refresh FAILED: ${scenarioRunURL}`.
5. **Send message** (Teams) — run condition `outcome == 'SUCCESS'`:
   `Daily refresh OK. Records: загляньте у dashboard`.

**Trigger:** time-based, daily о 04:00. **Auto-triggers: Active** (на Automation node).

### Приклад 2 (Просунуто): event-driven через SQL-trigger + custom scenario

**Trigger:** SQL query change на `SELECT MAX(updated_at) FROM staging.orders`, evaluation interval 5 хв, grace delay з re-check.

**Custom Python scenario:**

```python
from dataiku.scenario import Scenario
import dataiku

scenario = Scenario()

# 1. Будуємо downstream
build = scenario.build_dataset("orders_clean")

# 2. Рахуємо metrics і читаємо record count
ds = dataiku.Dataset("orders_clean")
ds.compute_metrics()
n = int(ds.get_last_metric_values().get_global_value('records:COUNT_RECORDS'))

# 3. Умовна логіка
sender = scenario.get_message_sender("ops-slack")
if n == 0:
    sender.send(message=":red_circle: orders_clean is empty after build!")
    scenario.set_scenario_variables(empty_build=True)
else:
    # тренуємо модель лише якщо даних достатньо
    if n > 50000:
        scenario.train_model("epae130z")
    sender.send(message=f":white_check_mark: orders_clean built: {n} rows")
```

### Приклад 3 (Просунуто): кастомний Python trigger за зовнішнім розкладом

```python
from dataiku.scenario import Trigger
from datetime import datetime

t = Trigger()
# Запускати лише в робочі дні після 9:00
now = datetime.now()
if now.weekday() < 5 and now.hour >= 9:
    t.fire()
```

---

## Best practices

**(Середній / Просунуто)**

1. **Ідемпотентні білди.** Scenario має давати однаковий результат при повторному запуску. Уникайте append-логіки, що дублює дані при ретраї; будуйте overwrite-датасети або використовуйте партиції з чітким partition spec.

2. **Надійні тригери.** Для SQL-джерел — SQL-trigger (а не dataset-change, що його не бачить). Вмикайте **grace delay з re-check**, коли джерело пишеться поступово, щоб не стартувати на півдорозі. Тримайте **Suppress triggers while running**, щоб запуски не накладалися.

3. **Sensible checks / rules.** Перевіряйте те, що дійсно ламає downstream: непорожність, унікальність ключів, record count у розумному діапазоні, валідність критичних колонок. Використовуйте **typical range** (IQR) для волатильних метрик замість жорстких меж, які доведеться постійно крутити.

4. **Нотифікації лише на справжні проблеми.** **FAILURE** — завжди; **SUCCESS** — лише для критичних щоденних пайплайнів. Зашумлений канал перестають читати. Використовуйте WARNING-статуси для «варто глянути, але не критично».

5. **Compute metrics ПЕРЕД Run checks / Verify rules.** Інакше check читатиме застаріле або відсутнє значення (EMPTY). У scenario це окремі послідовні steps.

6. **Dev vs Automation node.** На **Design node** scenarios зазвичай тримають **Auto-triggers Off** (запуск вручну для тестів). Бойові triggers активуються на **Automation node** після деплою bundle. Не вмикайте проднотифікації на dev (через bundle налаштування часто переозначують канали/змінні).

7. **Data Quality Templates** для стандартизації: однакові набори rules (унікальність ключів, непорожність) — у шаблон, що шериться по інстансу.

8. **Step-based за замовчуванням, custom Python — лише за потреби.** Step-based легше читати, версіонувати й передавати. Custom Python — коли реально потрібна нелінійна логіка.

9. **Run condition `If some prior step failed`** для error-handling-гілки (cleanup + alert), з error handling **Continue**, щоб гілка сама не валила scenario до того, як надішле сповіщення.

10. **Версіонуйте scenarios через Git** (project version control) — scenario є частиною bundle; зміни steps/triggers відстежуються.

---

## Типові помилки

**(Базово / Середній)**

1. **Очікувати `${variable}`-expansion у полях scenario steps.** Там працюють **DSS Formulas**, а не `${}` як у code recipes.
2. **Dataset-change trigger на SQL-датасеті.** Він не бачить змін даних — потрібен **SQL query change trigger**.
3. **Run checks без попереднього Compute metrics.** Check повертає **EMPTY** (немає актуального значення metric).
4. **Залишити Auto-triggers Active на Design node.** Тестовий scenario стартує сам і може зіпсувати дані/завалити канал нотифікацій.
5. **Тригер без grace delay на джерелі, що пишеться поступово.** Scenario стартує на півзаписаних даних → нестабільні результати.
6. **Спам SUCCESS-нотифікаціями** на кожен дрібний scenario → канал ігнорують, реальні FAILURE губляться.
7. **Жорсткі numeric-range межі на волатильних метриках** → постійні false-WARNING/ERROR. Використовуйте typical range (IQR).
8. **Плутати scenario-local variables з project variables.** `Define scenario variables` живе лише в межах run; `Set project variables` зберігається між запусками.
9. **Покладатися на legacy dataset checks у нових проєктах.** Для датасетів — **Data Quality rules** (краще моніторинг та аномалії).
10. **Error handling «Continue» на критичному build-кроці.** Scenario «успішно» завершиться з порожніми/битими даними downstream.
11. **Не вмикати `Run another scenario` для оркестрації, а робити час-залежні triggers «з запасом».** Краще ланцюжок (after-scenario trigger), ніж сподіватися, що о 05:00 upstream уже точно завершився.

---

## Перехресні посилання

- [01-overview-architecture.md](01-overview-architecture.md) — Design / Automation / API nodes; де живуть бойові scenarios.
- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — variables (3 рівні), build modes, jobs, Flow.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — datasets, partitioning, rebuild behavior (база для metrics та Build steps).
- [04-visual-recipes.md](04-visual-recipes.md) / [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — рецепти, що їх будують scenario steps.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — Python у DSS, `dataiku` API, Project Libraries (де живуть pytest-тести).
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — Saved Models, performance metrics, model checks; Train step.
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — побудова LLM-компонентів у scenarios.
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — **project bundles**, деплой на Automation node, активація triggers, Model Evaluation Stores та drift-checks.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — `dataikuapi`: керування scenarios, читання runs/metrics/checks ззовні.
- [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md) — dashboards (зокрема dashboard export step), Data Quality insights.
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — права на запуск scenarios, налаштування каналів нотифікацій.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — plugin-rules для Data Quality, кастомні reporter-плагіни.
- [15-best-practices.md](15-best-practices.md) — зведені best practices.
- [16-glossary.md](16-glossary.md) — глосарій термінів.

---

## Джерела (офіційна документація Dataiku)

- Automation scenarios (overview): https://doc.dataiku.com/dss/latest/scenarios/index.html
- Scenario definitions: https://doc.dataiku.com/dss/latest/scenarios/definitions.html
- Scenario steps: https://doc.dataiku.com/dss/latest/scenarios/steps.html
- Triggers: https://doc.dataiku.com/dss/latest/scenarios/triggers.html
- Reporters / notifications: https://doc.dataiku.com/dss/latest/scenarios/reporters.html
- Custom (Python) scenarios: https://doc.dataiku.com/dss/latest/scenarios/custom_scenarios.html
- Variables in scenarios: https://doc.dataiku.com/dss/latest/scenarios/variables.html
- Metrics, checks & Data Quality (index): https://doc.dataiku.com/dss/latest/metrics-check-data-quality/index.html
- Metrics: https://doc.dataiku.com/dss/latest/metrics-check-data-quality/metrics.html
- Checks: https://doc.dataiku.com/dss/latest/metrics-check-data-quality/checks.html
- Custom probes and checks: https://doc.dataiku.com/dss/latest/metrics-check-data-quality/custom_metrics_and_checks.html
- Data Quality rules: https://doc.dataiku.com/dss/latest/metrics-check-data-quality/data-quality-rules.html
- Data Quality templates: https://doc.dataiku.com/dss/latest/metrics-check-data-quality/data-quality-templates.html
- Data Quality insight (dashboard): https://doc.dataiku.com/dss/latest/dashboards/insights/data-quality.html
- Developer Guide — Metrics & Checks API: https://developer.dataiku.com/latest/api-reference/python/metrics.html
- Developer Guide — Data Quality: https://developer.dataiku.com/latest/concepts-and-examples/data-quality.html
- Knowledge Base — Automation / Data Quality: https://knowledge.dataiku.com/latest/automation/data-quality/concept-data-quality.html
- Knowledge Base — Tutorial: Custom metrics, checks, DQ rules: https://knowledge.dataiku.com/latest/automation/data-quality/tutorial-custom-metrics-checks-dqr.html
