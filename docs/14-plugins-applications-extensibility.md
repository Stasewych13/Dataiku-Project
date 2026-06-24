# 14. Plugins, Applications та розширюваність

> Рівень: новачок → досвідчений
> Версії DSS: 13.x / 14.x
> Джерела: [doc.dataiku.com/dss/latest](https://doc.dataiku.com/dss/latest/), [developer.dataiku.com](https://developer.dataiku.com/), [knowledge.dataiku.com](https://knowledge.dataiku.com/)

---

## Вступ

Dataiku — це **white-box** платформа, але навіть вона не покриває «з коробки» геть усі джерела даних, алгоритми чи бізнес-операції. Для цього існують механізми розширюваності, що дозволяють обгорнути власний код у **багаторазові, GUI-керовані компоненти** та поширити їх на весь instance або навіть між організаціями.

Цей документ покриває два головних механізми та екосистему навколо них:

1. **Plugins** — пакети багаторазових **компонентів** (custom recipe, connector, processor, macro, prediction algorithm, webapp, LLM tool тощо). Кожен компонент — це GUI-обгортка над користувацьким кодом, що додає до DSS новий елемент.
2. **Dataiku Applications** — перетворення цілого проєкту на **багаторазовий, параметризований застосунок** (із tiles-інтерфейсом) або на **Application-as-recipe** (рецепт у Flow іншого проєкту).
3. **Загальні reuse-механізми** — project duplication, project templates, shared code libraries — і коли який з них доречний.

> Передумови: знання Flow і recipe ([02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md)), code recipes та code environments ([06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md)), scenarios ([09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md)). Для ML-компонентів — [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md), для агентних інструментів — [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md).

---

## Частина 1. Plugins: концепції

### Що таке plugin і навіщо він (Базово)

**Plugin** — це структурована папка (упакована у zip або git-репозиторій), що містить **один або декілька related components**. Component — це **GUI-обгортка над користувацьким кодом**, яка додає у DSS один елемент певного типу: recipe, dataset, processor, webapp тощо.

Чому варто писати plugin замість «просто code recipe»:

- **Reusability** — компонент стає доступним у **всіх проєктах** instance (а не лише там, де лежить код).
- **GUI для не-кодерів** — параметри коду перетворюються на форму. Аналітик без Python користується вашою логікою через звичний UI.
- **Стандартизація** — одна перевірена реалізація замість десятків копій коду в різних проєктах.
- **Дистрибуція** — plugin можна опублікувати у Plugin Store, передати через git/zip, поширити між instance.
- **Governance** — компонент має версію, опис, автора; адміністратор контролює, що встановлено.

> Орієнтир: пишіть **code recipe**, поки логіка потрібна в одному-двох проєктах і часто змінюється. Виносьте у **plugin**, коли та сама параметризована логіка потрібна багатьом командам, або коли її мають використовувати не-кодери.

### Permissions: хто що може (Базово)

| Дія | Потрібне право |
|---|---|
| Переглядати Plugin Store та **використовувати** встановлені компоненти | будь-який користувач |
| **Встановлювати** / оновлювати / видаляти plugin | **administrator** (global admin) |
| **Розробляти** dev plugins (редагувати код у plugin editor) | глобальне право **Develop plugins** |
| Подавати **запит на встановлення** plugin | користувач без admin (запит іде адміну) |

> Безпекова теза: **тільки admin** ставить plugins, бо компонент виконує довільний код на сервері (часто від імені `dataiku` користувача ОС). Це навмисне обмеження. Деталі моделі прав — у [13-governance-security-collaboration.md](13-governance-security-collaboration.md).

### Plugin code environments (Середній)

Багато plugins мають власні Python/R-залежності (наприклад, SDK зовнішнього API). Plugin може декларувати **власний code environment**:

- специфікація лежить у папці `code-env/python/spec/` (`requirements.txt`, `desc.json`);
- при встановленні plugin DSS пропонує **збудувати** його code env — він ізольований і прив'язаний до plugin;
- оновлення plugin може вимагати **rebuild** code env.

> Важливий нюанс (Просунуто): **prediction algorithm** та **deck деякі ML-компоненти НЕ можуть використовувати plugin code-env** напряму. Якщо алгоритм потребує зовнішніх бібліотек — на instance створюють **окремий звичайний code env**, який обирається у Visual ML. Завжди звіряйтесь із документацією конкретного типу компонента.

---

## Частина 2. Встановлення та керування plugins

### З Plugin Store (Базово)

Найпростіший шлях (потрібен інтернет на instance):

1. **Application menu (☰) → Plugins**.
2. Вкладка **Store**.
3. Знайдіть plugin (пошук/категорії) → **Install**.
4. Слідуйте інструкціям, зокрема **build code environment**, якщо потрібен.

### З Zip-архіву (Базово)

Для offline-instance або власних/приватних plugins:

1. Завантажте zip (офіційні — у [plugins gallery](https://www.dataiku.com/product/plugins/)).
2. **Plugins → Add plugin → Upload**.
3. **Choose file** → виберіть zip.
4. Позначте, чи це **оновлення** наявного plugin.
5. **Upload**, за потреби — рестарт за інструкцією.

### З Git-репозиторію (Середній)

1. **Plugins → Add plugin → Fetch from Git repository**.
2. Вкажіть **repository URL**.
3. Оберіть **deployment mode**:
   - **standard** — звичайне встановлення (read-only);
   - **development** — створює **dev plugin**: код можна редагувати локально та контрибутити upstream.
4. Опціонально — **branch** і **path/subdirectory** (для монорепо).

> Більшість офіційних plugins живуть на GitHub за конвенцією `dss-plugin-*` або у репозиторії `dataiku/dataiku-contrib`.

### Керування версіями та оновлення (Середній)

- Plugin має **`version`** у `plugin.json` (рекомендовано **semantic versioning** `MAJOR.MINOR.PATCH`).
- **Update**: для store-плагінів — кнопка Update; для zip — повторний Upload із позначкою «update»; для git — pull/re-fetch.
- Після оновлення часто потрібен **rebuild plugin code env**.
- **Before upgrading**: перевірте **usages** — де plugin застосований у Flow, бо зміна параметрів може зламати наявні рецепти. Через API: `plugin.list_usages()`.

### Через Python API (Просунуто)

```python
import dataikuapi
client = dataikuapi.DSSClient("https://dss.example.com", "API_KEY")

# Встановлення
client.install_plugin_from_store("googlesheets")
client.install_plugin_from_git("https://github.com/org/dss-plugin-foo.git")
with open("my-plugin.zip", "rb") as f:
    client.install_plugin_from_archive(f)

# Керування
plugin = client.get_plugin("googlesheets")
usages = plugin.list_usages()          # де використовується
fut = plugin.start_update_code_env()   # rebuild code env
plugin.delete(force=True)              # видалити (force ігнорує usages)
```

Деталі Python client — у [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md).

---

## Частина 3. Типи компонентів plugin

Plugin може містити багато компонентів різних типів. Нижче — карта типів, що вони роблять, і **descriptor-файл** для кожного. Найактуальніший список завжди — у **plugin editor → New component** всередині продукту.

| Компонент | Що додає | Папка / descriptor |
|---|---|---|
| **Custom recipe** | Visual recipe з власною логікою та формою параметрів | `custom-recipes/<id>/recipe.json` + `recipe.py` |
| **Custom dataset / connector** | Нове джерело даних (read/write) | `python-connectors/<id>/connector.json` + `connector.py` |
| **Custom format** | Парсер/серіалізатор файлового формату | `python-formats/<id>/format.json` + `format.py` |
| **Custom FS provider** | Нова файлова система для connections | `fs-providers/<id>/...` |
| **Custom processor (prepare step)** | Новий крок у Prepare recipe (Jython) | `custom-jython-processors/<id>/processor.json` |
| **Custom trigger** | Тригер запуску scenario | `python-triggers/<id>/trigger.json` + `trigger.py` |
| **Custom scenario step** | Крок усередині scenario | `python-steps/<id>/step.json` + `step.py` |
| **Custom check** | Перевірка значення metric | `python-checks/<id>/check.json` + `check.py` |
| **Custom probe (metric)** | Обчислення власної metric датасета | `python-probes/<id>/probe.json` + `probe.py` |
| **Custom prediction algorithm** | Алгоритм у Visual ML (Python/R) | `python-prediction-algos/<id>/algo.json` + `algo.py` |
| **Custom webapp** | Інтерактивний застосунок (Standard/Bokeh/Dash/Shiny) | `webapps/<id>/webapp.json` + код |
| **Macro (runnable)** | Дія/автоматизація з кнопки чи scenario | `python-runnables/<id>/runnable.json` + `runnable.py` |
| **Custom fields editor** | Власний UI-віджет для параметра | `custom-fields/<id>/...` |
| **Custom chart** | Палітра кольорів / map background для charts | `custom-charts/...` |
| **Custom policy hooks** | Перехоплення/валідація дій (governance) | `policy-hooks/...` |
| **Custom exporter** | Експорт датасета у власний формат/призначення | `python-exporters/<id>/exporter.json` |
| **LLM connection** | Власне підключення до LLM (LLM Mesh) | `python-llms/<id>/...` |
| **Custom agent tool** | Tool для visual/code agents | `python-agent-tools/<id>/tool.json` + `tool.py` |
| **Custom agent** | Самостійний агент, не прив'язаний до проєкту | `python-agents/<id>/...` |
| **Code env / parameter sets** | Спільні залежності та набори параметрів | `code-env/`, `parameter-sets/` |

> Усі `python-*` компоненти мають аналоги для R, де це підтримується (наприклад, `r-recipes/`).

### Найвживаніші типи — детальніше

**Custom recipe (Базово→Середній).** Найпопулярніший компонент. Бере inputs/outputs за **ролями**, показує форму параметрів, виконує `recipe.py`. Підходить для повторюваних трансформацій (виклик API, нормалізація, кастомний join).

**Custom dataset / connector (Просунуто).** Підключає **нове джерело** (REST API, нестандартна БД). Клас успадковує `Connector`, реалізує `get_read_schema()`, `generate_rows()` (yield-генератор рядків), за потреби `get_writer()`. Дає DSS можливість читати/писати джерело так, наче це звичайний dataset.

**Custom processor (Середній).** Додає **крок у Prepare recipe** (бачить кожен рядок як dict). Пишеться на **Jython** (JVM-Python), тож без `pandas`/нативних бібліотек. Для важкої логіки краще custom recipe. Зв'язок із Prepare — у [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md).

**Macro / runnable (Середній).** Дія, що не вписується у Flow: масова реорганізація проєкту, виклик зовнішньої системи, генерація звіту. Запускається з меню проєкту/датасета або як **scenario step**.

**Custom prediction algorithm (Просунуто).** Додає алгоритм у **Visual ML**. `get_clf()` повертає scikit-learn-сумісний об'єкт (`fit`, `predict`, `get_params`, `set_params`). Інтегрується у весь pipeline Visual ML: preprocessing, grid search, interpretability ([07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md)).

**Custom agent tool / agent (Просунуто).** Розширює LLM Mesh: tool, яким користуються visual agents, або самостійний агент. Деталі агентних патернів — у [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md).

---

## Частина 4. Розробка plugin: структура та workflow

### Структура папок (Базово)

Plugin із id `myplugin`:

```
myplugin/
├── plugin.json                      # метадані + plugin-level params
├── code-env/
│   └── python/
│       └── spec/
│           ├── requirements.txt     # залежності plugin code env
│           └── desc.json
├── python-lib/
│   └── mymodule.py                  # спільний код (авто в PYTHONPATH компонентів)
├── resource/                        # read-only ресурси (дані, скрипти choices)
├── custom-recipes/
│   └── my-recipe/
│       ├── recipe.json              # descriptor: meta, kind, roles, params
│       └── recipe.py                # логіка
├── python-runnables/
│   └── my-macro/
│       ├── runnable.json
│       └── runnable.py
└── parameter-sets/
    └── my-preset/
        └── parameter-set.json
```

> `python-lib/` автоматично додається до `PYTHONPATH` усіх custom recipes/datasets plugin — тут тримають спільні функції. `resource/` призначений як **read-only** (дані, lookup-таблиці, скрипти динамічних choices).

### plugin.json (Базово)

```json
{
  "id": "myplugin",
  "version": "1.2.0",
  "meta": {
    "label": "My Plugin",
    "description": "Reusable components for X",
    "author": "Stanislav Marynovych",
    "icon": "icon-puzzle-piece",
    "licenseInfo": "Apache-2.0",
    "url": "https://github.com/org/dss-plugin-myplugin",
    "tags": ["Data Processing", "API"]
  },
  "params": [
    { "name": "api_key", "label": "API Key", "type": "PASSWORD", "mandatory": true }
  ]
}
```

- **`id`** — глобально унікальний (літери, цифри, `_`, `-`).
- **`version`** — semantic versioning.
- **`meta.icon`** — **FontAwesome 3.2.1** (саме ця версія).
- **`params`** на рівні plugin — глобальні налаштування, видимі **тільки адміністраторам** (зручно для instance-level credentials).

### Dev plugins / Plugin Developer mode (Середній)

Розробка ведеться у **dev plugin** — редагованому plugin із вбудованим **plugin editor**:

1. **Plugins → Add plugin → Write your own** (створити новий dev plugin) — DSS згенерує `plugin.json` і структуру.
2. **New component** → оберіть тип (Recipe, Macro, …) → DSS згенерує descriptor + skeleton-код.
3. Редагуйте `recipe.json` / `recipe.py` у вбудованому редакторі.
4. **Git integration**: dev plugin можна прив'язати до git-репозиторію, комітити/пушити прямо з plugin editor.

**Convert code → plugin recipe** (зручний фастпас): з готового **Python code recipe** можна натиснути **Actions → Convert to plugin** — DSS перенесе код у новий чи наявний plugin і допоможе винести параметри у форму.

> **Dev plugin vs installed plugin**: dev plugin **редагований** на цьому instance (для розробки/тестування); installed plugin (зі store/zip/git-standard) — **read-only**. На production-instance тримають лише installed-версії.

### Dev workflow та тестування (Середній)

1. Створіть dev plugin і потрібний компонент.
2. Винесіть параметри у `params` (типи — у Частині 5).
3. Реалізуйте логіку в `recipe.py`/`runnable.py`; спільне — у `python-lib/`.
4. **Тестуйте прямо в UI**: створіть тестовий проєкт, додайте компонент у Flow, проганяйте на реальних даних. Зміни в dev plugin підхоплюються одразу (інколи потрібен reload форми).
5. Логіку з `python-lib/` можна юніт-тестувати окремо (звичайний `pytest`), бо вона не залежить від UI.
6. Версіонуйте через git; перед релізом зафіксуйте `version` у `plugin.json`.

---

## Частина 5. Параметри, presets та parameter sets

### Типи параметрів (Середній)

Параметри декларуються масивом `params` у descriptor-і компонента (або в `plugin.json`). Кожен параметр має: `type` (обов'язково), `name` (snake_case, не з `__`), `label`, опційно `description`, `defaultValue`, `mandatory`, `visibilityCondition`.

**Базові типи:**

| Тип | Значення |
|---|---|
| `STRING`, `TEXTAREA`, `PASSWORD` | текст (однорядковий / багаторядковий / прихований) |
| `STRINGS` | список рядків (`allowDuplicates`) |
| `INT`, `DOUBLE` | число (межі `minI/maxI`, `minD/maxD`) |
| `INTS`, `DOUBLES` | списки чисел |
| `BOOLEAN` | чекбокс |
| `DATE`, `DATE_ONLY`, `DATETIME_WITH_TZ`, `DATETIME_WITHOUT_TZ` | дати/час |
| `SELECT`, `MULTISELECT` | вибір зі `selectChoices` або динамічно (`getChoicesFromPython`) |
| `MAP`, `KEY_VALUE_LIST` | пари ключ-значення |
| `OBJECT_LIST` | список складних об'єктів (`subParams`) |
| `SEPARATOR` | візуальний розділювач секцій |

**DSS-object типи** (вибір сутностей DSS): `DATASET`, `DATASETS`, `MANAGED_FOLDER`, `COLUMN`/`COLUMNS` (із input-ролі), `DATASET_COLUMN(S)` (з конкретного датасета), `SAVED_MODEL`, `SCENARIO`, `PROJECT`, `CODE_ENV`, `LLM`, `KNOWLEDGE_BANK`, `API_SERVICE`, `BUNDLE`, `CLUSTER` тощо.

```json
{
  "name": "input_column",
  "label": "Input column",
  "type": "COLUMN",
  "columnRole": "input_role_1",
  "allowedColumnTypes": ["int", "bigint", "float", "double"]
}
```

**Conditional visibility** — показ поля залежно від інших (`model` = поточні значення форми):

```json
{ "name": "auth_type", "type": "SELECT",
  "selectChoices": [{"value":"token","label":"Token"},{"value":"basic","label":"Basic"}] },
{ "name": "username", "type": "STRING",
  "visibilityCondition": "model.auth_type == 'basic'" },
{ "name": "api_token", "type": "PASSWORD",
  "visibilityCondition": "model.auth_type == 'token'" }
```

**Динамічні SELECT** (choices із Python): додайте `"getChoicesFromPython": true` до параметра і `"paramsPythonSetup": "compute_choices.py"` у descriptor; функція в `resource/`:

```python
# resource/compute_choices.py
def do(payload, config, plugin_config, inputs):
    return {"choices": [
        {"value": "v1", "label": "Value 1"},
        {"value": "v2", "label": "Value 2"},
    ]}
```

### Parameter sets та presets (Середній→Просунуто)

**Parameter set** — повторно вживаний **набір параметрів**, що визначається один раз і використовується багатьма компонентами через тип `PRESET`. Класичний кейс — облікові дані до зовнішнього сервісу: визначаєте один раз, а кожен recipe/dataset вибирає **preset** замість повторного введення секретів.

```json
// у recipe.json — посилання на parameter set
{ "type": "PRESET", "name": "api_account", "label": "API Account",
  "parameterSetId": "api_accounts" }
```

Адміністратор створює **presets** цього parameter set у **Plugins → <plugin> → Settings**. Presets бувають:
- **instance-level** (керує admin, спільні для всіх) — для production credentials;
- **user-level / per-user** — кожен користувач вводить свої облікові дані.

**CREDENTIAL_REQUEST** (тільки в parameter set) — per-user секрети, зокрема OAuth2:

```json
{ "type": "CREDENTIAL_REQUEST", "name": "oauth_creds", "label": "OAuth2 Credentials",
  "credentialRequestSettings": {
    "type": "OAUTH2",
    "authorizationEndpoint": "https://auth.example.com/authorize",
    "tokenEndpoint": "https://auth.example.com/token",
    "scope": "user.email profile" } }
```

Типи credential request: `SINGLE_FIELD`, `BASIC`, `OAUTH2`.

### Доступ до параметрів у коді (Середній)

```python
import dataiku
from dataiku.customrecipe import (
    get_recipe_config, get_plugin_config,
    get_input_names_for_role, get_output_names_for_role,
    get_recipe_resource,
)

config = get_recipe_config()              # параметри цього recipe
multiplier = config.get("multiplier", 1)

plugin_cfg = get_plugin_config()          # plugin-level params (admin)
preset = config["api_account"]            # значення обраного PRESET

# OAuth-токен із credential request (авто-refresh)
from dataiku.core import plugin
oauth = plugin.OAuthCredentials(preset["__credentials"]["oauth_creds"])
token = oauth.access_token
```

R-аналог:

```r
config <- dataiku::dkuRecipeConfig()
value  <- config[["param_name"]]
```

---

## Частина 6. Приклад: повний custom recipe

### recipe.json

```json
{
  "meta": {
    "label": "Multiply column",
    "description": "Multiplies a numeric column by a configurable factor",
    "icon": "icon-cogs",
    "iconColor": "blue"
  },
  "kind": "PYTHON",
  "selectableFromDataset": "input",
  "inputRoles": [
    {
      "name": "input",
      "label": "Input dataset",
      "arity": "UNARY",
      "required": true,
      "acceptsDataset": true
    }
  ],
  "outputRoles": [
    {
      "name": "main_output",
      "label": "Output dataset",
      "arity": "UNARY",
      "required": true,
      "acceptsDataset": true
    }
  ],
  "params": [
    { "name": "column", "label": "Column", "type": "COLUMN", "columnRole": "input",
      "allowedColumnTypes": ["int", "bigint", "float", "double"], "mandatory": true },
    { "name": "factor", "label": "Factor", "type": "DOUBLE", "defaultValue": 2.0,
      "mandatory": true }
  ]
}
```

- **`inputRoles`/`outputRoles`** описують слоти Flow за **ролями**. `arity`: `UNARY` (один) чи `NARY` (багато). Прапори `acceptsDataset`, `acceptsManagedFolder`, `acceptsSavedModel`.
- **`selectableFromDataset`** — recipe з'явиться в меню Actions датасета, прив'язаного до ролі `input`.

### recipe.py

```python
import dataiku
from dataiku.customrecipe import (
    get_input_names_for_role,
    get_output_names_for_role,
    get_recipe_config,
)

config = get_recipe_config()
column = config["column"]
factor = config.get("factor", 1.0)

input_name  = get_input_names_for_role("input")[0]
output_name = get_output_names_for_role("main_output")[0]

df = dataiku.Dataset(input_name).get_dataframe()
df[column] = df[column] * factor

dataiku.Dataset(output_name).write_with_schema(df)
```

### Приклад: macro (runnable.json + runnable.py)

```json
// python-runnables/cleanup/runnable.json
{
  "meta": { "label": "Cleanup temp datasets", "icon": "icon-trash" },
  "impersonate": true,
  "permissions": ["WRITE_CONF"],
  "resultType": "HTML",
  "resultLabel": "Cleanup report",
  "macroRoles": [{ "type": "PROJECT_MACROS" }],
  "params": [
    { "name": "prefix", "label": "Name prefix", "type": "STRING", "defaultValue": "tmp_" }
  ]
}
```

```python
# python-runnables/cleanup/runnable.py
from dataiku.runnables import Runnable
import dataiku

class CleanupRunnable(Runnable):
    def __init__(self, project_key, config, plugin_config):
        self.project_key = project_key
        self.config = config

    def get_progress_target(self):
        return (100, "NONE")

    def run(self, progress_callback):
        client = dataiku.api_client()
        project = client.get_project(self.project_key)
        prefix = self.config.get("prefix", "tmp_")
        deleted = []
        for ds in project.list_datasets():
            if ds["name"].startswith(prefix):
                project.get_dataset(ds["name"]).delete()
                deleted.append(ds["name"])
            progress_callback(50)
        return "<h4>Deleted %d datasets</h4><ul>%s</ul>" % (
            len(deleted), "".join("<li>%s</li>" % n for n in deleted))
```

- **`impersonate: true`** — виконання від імені користувача (рекомендовано, sandboxing).
- **`macroRoles`** — де макрос з'являється: `PROJECT_MACROS`, `DATASET`, `DATASETS` тощо.
- **`resultType`** — `HTML`, `URL`, `RESULT_TABLE` або немає.

---

## Частина 7. Dataiku Applications

### Що це і навіщо (Базово)

**Dataiku Application** — це **цілий проєкт, перетворений на багаторазовий, параметризований застосунок**. Користувач створює **app instance** (копію проєкту з підставленими своїми даними) через простий **tiles-інтерфейс**, не торкаючись Flow.

Типові кейси:
- **Самообслуговування**: бізнес-користувач завантажує свій файл, тисне Run, забирає звіт/дашборд — без розуміння Flow.
- **Стандартизований процес** для багатьох команд/клієнтів: однакова логіка, різні вхідні дані.
- **Демо/портал** поверх готового pipeline.

Створюються два типи (через **Application designer**):
1. **Dataiku Application** (Visual application) — окремий tiles-інтерфейс, кожен instance = окремий проєкт.
2. **Application-as-recipe** — рецепт у Flow **іншого** проєкту (Частина 8).

### Application Designer (Середній)

**Project menu → Application designer**. Тільки **Admin** проєкту може конвертувати проєкт в application або змінювати тип; **Write project content** — редагувати наявний.

Ключові секції:

- **Application Header**: image, title, description, **version** (рядок), хто може інстанціювати, discoverability, tags.
- **Application Features**: які елементи показувати на instance — Flow menu, Lab/Code menus, version control, кнопка «Switch back to project view».
- **Included Content**: які додаткові дані з проєкту-шаблону копіювати в кожен instance (datasets, folders, variables).
- **Application Instance Home**: дизайн UI з **tiles**, згрупованих у **sections**. Доступні tiles:
  - upload/download file, edit/select dataset, view dashboard, markdown, variable controls, **run scenario**.
- **Advanced**: конвертація типу, mapping connections та code envs, перегляд raw JSON manifest.

> **Важливо**: зміни в designer застосовуються **тільки до нових instances**. Наявні instances для оновлення треба видалити й створити заново.

### App instances та versioning (Середній)

- З одного application користувач створює **кілька named instances** (кожен — окремий проєкт).
- Кожен instance отримує авто-змінну **`projectRandomKey`** — короткий (8 символів) випадковий рядок. Зручно для іменування SQL-таблиць, щоб уникнути колізій між instances.
- **Версіонування**: розробник змінює рядок `version` у header, щоб **повідомити** користувачів про оновлення (нова tile, зміна у Flow). DSS не оновлює наявні instances автоматично.

### Дозволи та instantiation requests (Середній)

- Для створення instance потрібне право **Execute app**.
- Користувач без права бачить кнопку **request access**; запит іде адміністратору (через security settings/inbox).
- Статус запиту авто-оновлюється, коли користувачу дають право, видаляють користувача, або видаляють проєкт.

---

## Частина 8. Application-as-recipe

### Концепція (Середній)

**Application-as-recipe** упаковує Flow проєкту-шаблону в **рецепт**, який інші проєкти вставляють у свій Flow. Тобто складний багатокроковий pipeline стає **одним блоком** із чіткими inputs/outputs.

### Створення (Середній)

У **Application designer** (тип Application-as-recipe) налаштовують:

- **Icon** (FontAwesome 3.2.1) та **Category** (групування в меню «New recipe»).
- **Inputs/Outputs** — мапінг елементів шаблону (Dataset / Managed Folder / Saved Model) на слоти рецепта; кожен має label і type.
- **Scenario** (обов'язково) — який scenario будує outputs, коли рецепт запускається.
- **Settings** — runtime-форма, яку бачить користувач при налаштуванні рецепта.
- **Included Content** — додаткові дані з шаблону.

### Використання та поведінка (Просунуто)

- В іншому проєкті: **Flow → New recipe → (ваша Category) → <recipe>**. Потрібні право інстанціювати шаблон і **Write project content**.
- При запуску DSS **копіює проєкт-шаблон**, підставляє inputs і запускає scenario. **Код рецептів не модифікується** — тому в Python code recipes посилайтесь на входи **за індексом** (`recipe.get_input(0)`), а не за іменем.

### Обмеження (Просунуто)

- **Partitioned** inputs/outputs **не підтримуються**.
- Outputs мають бути **writable DSS** (не read-only зовнішні БД на кшталт BigQuery як вихід).

### Поширення як plugin-компонент (Середній)

Будь-який application (visual чи as-recipe) можна винести у plugin: **Application designer → Actions → Create or update plugin application**. У descriptor задають `meta.label`, `meta.description`, `appImageFilename` (картинка з `resource/` plugin). Так application стає **встановлюваним компонентом** і поширюється через Plugin Store/git/zip.

---

## Частина 9. Reuse-механізми: коли що

Dataiku пропонує кілька способів повторного використання. Орієнтир вибору:

| Механізм | Що це | Коли обирати |
|---|---|---|
| **Project duplication** | Копія проєкту (одноразова) | Швидкий старт від подібного проєкту; **немає** зв'язку з оригіналом |
| **Project template** (instance-level) | Шаблон, з якого створюють нові проєкти | Стандартний «скелет» проєкту для команди |
| **Dataiku Application** | Параметризований tiles-застосунок | Самообслуговування для не-кодерів; багато instances із різними даними |
| **Application-as-recipe** | Pipeline-шаблон як блок у чужому Flow | Складний sub-pipeline, що повторюється в багатьох проєктах |
| **Shared code libraries** (project / instance) | Спільний Python/R-код у `lib/` | Спільні функції для code recipes/notebooks; часті зміни; для кодерів |
| **Plugin** | Багаторазові GUI-компоненти | Параметризована логіка для багатьох проєктів/команд; UI для не-кодерів; дистрибуція |

**Як вирішити:**
- Логіка для **кодерів**, часто змінюється, в кількох проєктах → **shared code library** ([06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md)).
- Логіка для **не-кодерів** із формою, стабільна, для багатьох → **plugin**.
- Треба запакувати **весь процес** із власним UI → **Dataiku Application**.
- Треба вставити **весь процес блоком** у чужий Flow → **Application-as-recipe**.
- Просто почати від схожого → **duplication** або **template**.

---

## Частина 10. Дистрибуція plugins

### Публікація (Середній)

- **Public Plugin Store** — для офіційних/community-плагінів (через Dataiku, зазвичай GitHub `dss-plugin-*`).
- **Git** — приватний/корпоративний репозиторій; інші instance ставлять через **Fetch from Git**. Підтримує branch/path → природний CI/CD.
- **Zip** — для air-gapped/offline instances; передається вручну.

### Приватний / корпоративний store (Просунуто)

Організація може тримати **внутрішній каталог plugins** (git-репозиторій або внутрішня сторінка з zip). На практиці це git-монорепо чи набір `dss-plugin-*` репозиторіїв, звідки instances фетчать через **Fetch from Git** із фіксованим tag/branch. Це дає версіонування, code review та контроль того, що потрапляє на production.

### Підтримка та оновлення (Середній)

- Версіонуйте через **semantic versioning**; ведіть changelog.
- Перед публікацією оновлення — перевірте **backward compatibility** параметрів (перейменування `name` параметра зламає наявні рецепти).
- Документуйте plugin: README, описи `description` у кожному компоненті/параметрі, приклади.
- Після оновлення на instance — повідомте про потенційний **rebuild code env** і перевірте **usages**.

---

## Best practices

**(Базово)**
- **Plugin vs code recipe**: одиночна/змінна логіка → code recipe; багаторазова параметризована для багатьох команд → plugin.
- Не дублюйте логіку між компонентами — виносьте у **`python-lib/`**.
- Заповнюйте `description` для plugin, кожного компонента і **кожного параметра** — це і є вбудована документація.

**(Середній)**
- **Credentials** — лише через `PASSWORD`/`CREDENTIAL_REQUEST`/parameter set presets, **ніколи** не хардкодьте у код і не зберігайте у plaintext params. Instance-level credentials тримайте у admin-only **plugin params** або instance presets.
- **Semantic versioning** + changelog; стабільні `name` параметрів (перейменування ламає usages).
- Юніт-тестуйте логіку з `python-lib/` окремо від UI; інтеграційно тестуйте у dev plugin на реальних даних.
- Для plugin із зовнішніми залежностями — декларуйте **plugin code env** (`code-env/python/spec/`).

**(Просунуто)**
- **Безпека встановлення**: тільки **admin** ставить plugins; на production — лише **installed** (read-only) версії, dev plugins тримайте на dev-instance.
- **Prediction algorithm** не використовує plugin code-env — створіть окремий code env на instance.
- В **Application-as-recipe** з code recipes посилайтесь на входи **за індексом** (`recipe.get_input(0)`), бо код не переписується при інстанціюванні.
- Версіонуйте plugins у git і фетчте на instances із фіксованих tags → відтворюваність і відкат.

---

## Типові помилки

- **Пишуть plugin, коли достатньо code recipe** — передчасне ускладнення. Спочатку code recipe; виносьте у plugin, коли reuse реально доведено.
- **Хардкод секретів** у `recipe.py` чи у `STRING`-параметрі замість `PASSWORD`/preset → витік у логи/git.
- **Перейменування `name` параметра** в новій версії → усі наявні рецепти втрачають значення й ламаються. Зберігайте старі `name`, додавайте нові поряд.
- **Очікують, що зміни в Application designer оновлять наявні instances** — ні, лише нові. Наявні треба перестворити.
- **Забувають rebuild plugin code env** після оновлення → ImportError/несумісні версії.
- **Звертання до входів за іменем** в Application-as-recipe code recipe → ламається при інстанціюванні (код не переписується); треба за індексом.
- **Намагаються підключити plugin code-env до prediction algorithm** — не підтримується; потрібен окремий instance-level code env.
- **Custom processor (Jython) для важкої логіки з pandas** — Jython не має нативних бібліотек; беріть custom recipe.
- **Ставлять dev plugin на production** замість installed read-only версії → ризик випадкових правок коду на проді.
- **Іконки не з FontAwesome 3.2.1** → не відображаються (DSS використовує саме цю стару версію).

---

## Перехресні посилання

- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — Flow, recipe, проєкти (фундамент для компонентів і applications).
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — куди вбудовується custom processor.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — code recipes, code environments, shared code libraries (альтернатива plugin).
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — custom prediction algorithm у Visual ML.
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — LLM connections, custom agent tools/agents.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — custom trigger/step/check/probe як scenario- та quality-компоненти.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — керування plugins через Python client.
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — права (Develop plugins, Execute app), безпека встановлення.
- [15-best-practices.md](15-best-practices.md) — зведені best practices.
- [16-glossary.md](16-glossary.md) — терміни (plugin, component, application, preset, parameter set).
