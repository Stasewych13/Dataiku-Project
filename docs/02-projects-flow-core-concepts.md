# 02. Проєкти, Flow та базові концепції

> Версія документа орієнтована на **Dataiku DSS 13.x / 14.x** (2025–2026). Технічні терміни, назви елементів інтерфейсу (UI) та код подано в **оригіналі (англійською)**; пояснення — українською.
>
> Рівні складності позначено так: **(Базово)** — для новачка, **(Середній)** — для впевненого користувача, **(Просунуто)** — для досвідченого аналітика/інженера.

---

## Зміст

1. [Вступ](#вступ)
2. [Project: фундаментальна одиниця роботи](#project-фундаментальна-одиниця-роботи)
3. [Variables: змінні трьох рівнів](#variables-змінні-трьох-рівнів)
4. [The Flow: візуальний пайплайн](#the-flow-візуальний-пайплайн)
5. [Visual Grammar: кольори, форми, іконки](#visual-grammar-кольори-форми-іконки)
6. [Flow zones, tags та навігація](#flow-zones-tags-та-навігація)
7. [Build пайплайну: режими збірки та jobs](#build-пайплайну-режими-збірки-та-jobs)
8. [Object lifecycle та dependencies](#object-lifecycle-та-dependencies)
9. [Колаборація: Wiki, Discussions, To-do, Dashboard](#колаборація-wiki-discussions-to-do-dashboard)
10. [Git integration та version control](#git-integration-та-version-control)
11. [Project bundles (огляд)](#project-bundles-огляд)
12. [Best practices організації](#best-practices-організації)
13. [Типові помилки](#типові-помилки)
14. [Перехресні посилання](#перехресні-посилання)

---

## Вступ

**(Базово)** Уся робота в Dataiku DSS відбувається всередині **проєктів (projects)**. Проєкт — це контейнер, який об'єднує дані, логіку трансформації, моделі, дашборди, автоматизацію та документацію для однієї бізнес-задачі. Центральним об'єктом кожного проєкту є **the Flow** — візуальне зображення вашого data-пайплайну у вигляді графа.

Цей документ пояснює базові «цеглинки», на яких побудовано все інше в платформі: що таке проєкт і як ним керувати, як працюють змінні, як читати та будувати Flow, як об'єкти залежать один від одного, як організувати колаборацію та версіонування. Розуміння цих концепцій — обов'язкова основа перед переходом до datasets, recipes, ML та MLOps.

---

## Project: фундаментальна одиниця роботи

### Що таке проєкт

**(Базово)** A **project** — це «container for all your work on a particular activity» (контейнер усієї роботи над певною активністю). Кожен проєкт містить рівно один **Flow**. Проєкти забезпечують:

- **Чіткі межі (boundaries)** — визначають scope і запобігають небажаній взаємодії між різними робочими процесами.
- **Reproducibility** — пов'язані елементи зберігаються разом.
- **Collaboration** — спільний робочий простір для команди.
- **Governance** — керування доступом і дозволами.

### Project key

**(Базово)** **Project key** — це головний, постійний ідентифікатор проєкту на інстансі. Ключові властивості:

- Автоматично генерується з назви проєкту під час створення (зазвичай у верхньому регістрі, лише алфавітно-цифрові символи; пробіли та спецсимволи прибираються). Приклади: `CHURN`, `DKU_TSHIRTS`, `MYPROJECT`.
- **Immutable** — не змінюється після створення. Використовується скрізь: в URL, API-викликах та, найголовніше, у назвах таблиць (`${projectKey}_` як префікс для database connections).
- Має бути **унікальним** у межах усього інстансу.

> **Порада:** добре продумайте project key одразу — змінити його потім неможливо. Через Python API він задається явно: `client.create_project(project_key="MYPROJECT", name="My very own project", owner="alice")`.

### Project home page

**(Базово)** Це «command center» проєкту — стартова сторінка, на якій видно:

- **Overall status** проєкту та summary-інформацію.
- **Recent activity** (стрічка останніх дій) і список **contributors**.
- Інструменти колаборації: **comments/discussions**, **tags**, **project to-do list**, **project description**.
- Розділ **Project content** — швидкий огляд усього вмісту (datasets, recipes, models, notebooks, dashboards, wiki тощо).
- Меню **Actions** (праворуч угорі): **Duplicate project**, **Export**, **Delete**.
- Меню **… (More Options)** — доступ до **Variables**, **Settings**, **Version Control**, **Application Designer** та інших налаштувань.

### Project settings

**(Середній)** Основні категорії налаштувань проєкту:

| Категорія | Призначення |
|---|---|
| **Metadata** | Назва, опис, теги (`get_metadata()` / `set_metadata()` в API) |
| **Permissions** | Дозволи на групи (read/write та гранулярні права як `manageExposedElements`) |
| **Variables** | Project-level змінні (Global + Local), JSON-редактор |
| **Code envs** | Вибір/перевизначення Python/R code environment |
| **Cluster / containerized execution** | Налаштування виконання |
| **Exposed elements** | Об'єкти, поширені в інші проєкти/дашборди |
| **Security / API keys** | Project-level public API keys |
| **Flow display**, **Wiki**, **Change management** | Відображення Flow, промоут wiki, commit mode |

### Структура проєкту

**(Базово)** Що «живе» всередині проєкту:

- **Flow** (один на проєкт) — візуальний DAG.
- **Datasets** (входи/виходи).
- **Recipes** (visual + code) — трансформації даних.
- **Managed folders** — файли / неструктуровані дані.
- **Saved models / ML analyses**, **Model Evaluation Stores**.
- **Generative AI components** — prompts, agents, knowledge banks.
- **Notebooks**, **Code Studios**, **Webapps**.
- **Dashboards**, **Charts**, **Insights**, **Stories**.
- **Scenarios** (автоматизація).
- **Wiki** та **Discussions** (документація/колаборація).

### Створення проєкту

**(Базово)** На головній сторінці натисніть **+ NEW PROJECT**. Опції:

- **Blank project** — з нуля.
- **DSS Tutorials / Learning projects** — навчальні проєкти (наприклад, `+ NEW PROJECT > Learning projects > Quick Start`).
- **Sample projects** — готові приклади.
- **New Dataiku Application** — з application-шаблону.
- **Import project** — завантажити наявний проєкт із `.zip`-архіву.

Під час створення ви вводите **name**; **project key** генерується автоматично і показується (редагується лише на цьому етапі, далі — блокується).

### Дублювання проєкту

**(Середній)** На тому самому інстансі:

- Project home: **Actions > Duplicate project**, або
- Projects page: **right-click** по плитці проєкту > **Duplicate project**.

У діалозі можна задати **project name** та **project key** (унікальний), обрати **duplication mode**: дублювати **всі дані** чи лише **Flow** (структуру без даних), а також **remap connections**.

> **Best practice:** використовуйте префікс `${projectKey}_` у назвах таблиць managed datasets, щоб оригінал і копія не конфліктували в одній БД.
>
> Python API: `original_project.duplicate(target_project_key="CHURNCOPY", target_project_name="Churn (copy)")`.

### Export та Import

**(Середній)** **Export (UI):** Project home > **Actions > Export** → завантажиться `.zip`-архів. Опції експорту дозволяють обрати, **які дані включити** — можна зняти прапорці з похідних/неважливих datasets, щоб зменшити розмір архіву (потім їх можна перебудувати після імпорту). Експорт може включати/виключати: managed datasets, managed folders, uploaded datasets, models, insights data.

**Import (UI):** на цільовому інстансі — **+ NEW PROJECT > Import**, оберіть `.zip`. Потрібно:

- Розв'язати **унікальний project key** (з'явиться запит, якщо ключ уже існує).
- Мати на цільовому інстансі потрібні **connections**.
- Мати потрібні **code environments** (можна **remap** імена, якщо вони відрізняються).

**Command line (`dsscli`, на сервері DSS):**

```bash
DATA_DIR/bin/dsscli project-export PROJECT_KEY foo.zip
DATA_DIR/bin/dsscli project-import foo.zip
```

**(Просунуто)** **Python API:** експорт через `project.get_export_stream()`; імпорт через `client.prepare_project_import(f).execute()`.

---

## Variables: змінні трьох рівнів

**(Середній)** Variables дозволяють параметризувати проєкти: винести «магічні» значення (пороги, дати, шляхи, назви категорій) в одне місце й використовувати їх у формулах, коді та конфігах recipes.

### Три scope (рівні) змінних

1. **Instance-level (Global) variables** — задає адміністратор, спільні для всіх проєктів інстансу. Шлях: **Administration > Settings > Variables**. Зберігаються у `config/variables.json` (можуть перевизначатися через `local/variables.json` в DataDir — зручно для розділення dev/prod).

2. **Project-level variables** — лише для одного проєкту. Шлях: **… (More Options) > Variables**. Мають два підтипи (див. нижче).

3. **Scenario-level variables** — тимчасові, існують лише на час виконання scenario.

### Project variables: standard vs local

**(Середній)** На сторінці **… > Variables** проєкту є дві вкладки / два JSON-редактори:

- **Global variables** (вкладка) → **"standard"** змінні. **Експортуються** з project bundles, export та duplicate. Для бізнес-констант і конфігів, що мають подорожувати разом із проєктом.
- **Local variables** (вкладка) → **"local"** змінні. **НЕ** включаються в bundle/export — залишаються на вихідному інстансі. Для secrets, credentials, специфічних для середовища значень.

> **Увага щодо термінології:** вкладка «Global variables» на рівні **проєкту** (= standard, експортовані) — це НЕ те саме, що справжні instance-level global variables (Administration > Settings > Variables). Це різні scope, попри схоже слово «global».

### Як визначаються змінні (JSON editor)

Змінні задаються як JSON-об'єкт (пари ключ-значення):

```json
{
  "mean_purchase_amount": 232,
  "chosen_category": "B",
  "tx_month": "2017-01",
  "logs_cumulative_aggregate_days": 7
}
```

### Синтаксис expansion `${...}` та контексти використання

**(Середній)** Базовий синтаксис — **`${variable_name}`**. Працює в:

- **Code recipes (SQL):** `${variable_name}`; **Hive:** `${hiveconf:variable_name}` (без лапок).
- **Filesystem dataset paths**.
- **Dashboard elements** (заголовки сторінок, tile titles, text tiles).
- **Filter conditions** та **dataset names** (динамічне іменування).

**У Formulas** (Prepare recipe, computed columns, Filter/Sample) використовується **dictionary notation `variables["variable_name"]`**. Тобто `${...}` — у filter conditions та назвах datasets, а `variables["..."]` — у формульній мові. У UI є **variable picker**, який вставляє змінну з правильними дужками автоматично.

**(Просунуто)** **Override tables:** для полів, що не приймають inline `${}`, натисніть **variable icon** у конфізі recipe, щоб перевизначити конкретний JSON-path значенням `${variable}`.

**Built-in expansion filters** (pipe-стиль): `?` (пропустити undefined), `default` (fallback), `tolower`/`toupper`, `getornull`, `tojson`/`fromjson`, `formatdate`, `indent`.

### Використання у коді (точні функції)

**(Середній/Просунуто)** **Python:**

```python
import dataiku

# Усі custom variables (instance + project standard + project local), плаский dict
dataiku.get_custom_variables()["logs.preprocessing.excluded_ip"]

# Flow variables (контекст партицій/збірки)
dataiku.dku_flow_variables["DKU_DST_YEAR"]
```

Читання/запис структурованого об'єкта (standard + local) через API:

```python
project = client.get_project("MYPROJECT")
vars = project.get_variables()      # -> {"standard": {...}, "local": {...}}
vars["standard"]["my_var"] = 42
project.set_variables(vars)
```

**R:**

```r
library(dataiku)
dkuCustomVariable("logs.preprocessing_excluded_ip")
dkuFlowVariable("DKU_DST_ctry")
dkuGetProjectVariables()  # список з "standard" і "local"
```

### Built-in / автоматичні змінні

**(Середній)**

- **`${projectKey}`** — поточний project key. Часто використовується як префікс `${projectKey}_` для назв таблиць.
- **Flow / partition variables** (у `dku_flow_variables`): `DKU_DST_YEAR`, `DKU_DST_ctry` тощо.
- **Scenario time keywords:** `CURRENT_DAY`, `CURRENT_MONTH`, `PREVIOUS_DAY`, `PREVIOUS_MONTH` (для динамічного вибору партицій).
- **`stepOutput_<step_name>`** — JSON-вихід попереднього іменованого scenario-step.

### Як scenarios змінюють змінні

**(Просунуто)** Кроки scenario для запису змінних:

- **Set global variables** → записує **instance-level** змінні.
- **Set project variables** → записує **project-level** змінні.
- **Define variables** → записує **scenario-level** (тимчасові) змінні.
- **Run global variables update** → Jython-код з функцією `get_variables()`, що повертає dict.
- **Python step** → об'єкт **`Scenario`** для читання/запису (`scenario.set_scenario_variables(...)`).

> **Важливий нюанс:** поля scenario-step обчислюються як **DSS Formulas**, а НЕ як `${}` expansion. Усередині value-полів step використовуйте формульні вирази; синтаксис `${variable}` там **не** підтримується.

---

## The Flow: візуальний пайплайн

### Концепція

**(Базово)** **The Flow** — це «visual representation of how data, recipes, models, and agents are connected within a project». Це **directed acyclic graph (DAG)**, що читається **зліва направо**:

- **Nodes** = об'єкти Dataiku (datasets, recipes, models, folders…).
- **Edges** = зв'язки/залежності між ними.

Datasets і recipes **чергуються**: recipe завжди стоїть між вхідним dataset (ліворуч) і вихідним dataset (праворуч). Recipe — це крок трансформації; dataset — об'єкт даних.

**(Середній)** **Lineage та rebuild:** «DSS knows the lineage of every dataset in the flow». Завдяки цьому DSS може динамічно перебудовувати datasets, коли змінився один з їхніх батьківських datasets або recipes. Це основа для data lineage, governance та поширення збірок.

### Навігація до Flow

- Кнопка **Go to Flow** на project home.
- Меню Flow біля назви проєкту у верхній навігації.
- Клавіатурне скорочення **`g` + `f`**.

---

## Visual Grammar: кольори, форми, іконки

**(Базово)** Dataiku побудувала «visual grammar for data science» — послідовну систему форм і кольорів. Загальна конвенція:

- **Datasets = squares** (сині квадрати, злегка заокруглені).
- **Recipes = circles**, колір залежить від типу.
- **ML / model objects = green**.
- **Knowledge banks (GenAI) = pink squares**.

### Datasets (сині квадрати)

- **Blue squares** із центральною іконкою, що позначає тип/connection:
  - **стрілка вгору** = **uploaded** dataset
  - **два куби** = **Amazon S3**
  - **слон** = **HDFS**
  - (кожен тип connection — SQL, filesystem тощо — має свою іконку).
- **Shared datasets** (поширені в інший проєкт) — з **curved arrow у верхньому правому куті**.
- **Foreign datasets** (з іншого DSS-проєкту) — **чорний квадрат** замість синього.

### Recipes (кружечки, колір за типом)

- **Visual recipes = ЖОВТІ кружечки.** Внутрішня іконка вказує тип:
  - **broom (мітла)** = **Prepare**
  - **funnel (лійка)** = **Filter** / Sample
  - **pile of squares** = **Stack**
  - (Group, Join, Window, Sort, Top N, Distinct, Pivot, Split, Sync — кожен зі своєю іконкою).
- **Code recipes = ПОМАРАНЧЕВІ кружечки.** Іконка вказує мову/engine:
  - **«дві змії»** = **Python**
  - **honeycomb (стільники)** = **Hive**
  - (також R, SQL, PySpark, Scala, Shell).
- **Plugin recipes = ЧЕРВОНІ кружечки** (кастомні recipes від плагінів).

### Machine-learning об'єкти (ЗЕЛЕНІ)

**(Середній)** ML-процеси та об'єкти — **зелені**:

- **barbell (гантеля)** = подія **model training**.
- **scatter-plot** = **model scoring**.
- **trophy (кубок)** = застосування моделі до нового dataset.
- **Saved Model** = зелений об'єкт (форма diamond/ML-shape) — артефакт Flow з моделлю, задеплоєною з visual analysis. Потрібен як вхід для **Evaluate recipe**.
- **Model Evaluation Store (MES)** = зелений ML-об'єкт, що зберігає метрики продуктивності, візуалізує їх і пропонує status checks для моніторингу (model drift). Створюється **Evaluate recipe** (Saved Model + evaluation dataset).

### Інші об'єкти

- **Managed folders** — Flow-об'єкт для зберігання файлів / неструктурованих даних; має власну іконку folder, бере участь у computeі залежностей.
- **Streaming endpoints** — компонент реального часу (real-time / continuous pipelines), відображається як Flow-node.
- **Knowledge Banks** — **рожеві (pink) квадрати**. Вихід **Embed recipe** (наприклад, **Embed documents recipe**), де текстові/мультимодальні дані конвертовано у вектори для augmentation LLM. Зберігає embeddings у **vector store** (за замовчуванням **Chroma**; також **Qdrant**, **FAISS**, **Pinecone**). Augmented LLM з'являється в Prompt Studios / Prompt recipe у секції **Retrieval Augmented**.
- **Labeling tasks** — також відображаються як Flow-елементи.

### Підсумкова таблиця семантики кольорів

| Колір/форма | Значення |
|---|---|
| **Blue square** | Dataset (локальний) |
| **Black square** | Foreign dataset (з іншого проєкту) |
| **Yellow circle** | Visual recipe |
| **Orange circle** | Code recipe |
| **Red circle** | Plugin recipe |
| **Green (diamond/rounded)** | ML-об'єкт — training, scoring, Saved Model, Model Evaluation Store |
| **Pink square** | Knowledge Bank (GenAI/RAG) |

Внутрішня іконка завжди уточнює конкретний підтип; curved arrow (вгорі праворуч) позначає shared dataset.

---

## Flow zones, tags та навігація

### Flow zones

**(Середній)** **Flow zones** ділять великий проєкт на керовані секції. Вони «only used for laying out the flow» — **не створюють security boundaries**.

- **Default zone:** кожен Flow має незмінну зону **Default**. Невіднесені вузли автоматично в ній. Її не можна видалити (можна перейменувати); коли видалити всі кастомні зони, Flow знову стає без зон.
- **Створення зони:** кнопка **+ Zone** угорі, АБО виділити кілька вузлів → right-click → **Move to a flow zone**. Зони **не можна вкладати** (no nesting).
- **Ключове правило:** «Recipes and their outputs always live in the same zone». Якщо перемістити dataset, DSS перенесе і його upstream recipe. Думайте в термінах **переміщення recipes**, а не datasets.
- **Sharing:** right-click → **Share to a flow zone** робить dataset доступним у зоні **без** переміщення (показується reference node). Клік по reference node підсвічує оригінал; **Go to original** повертає до нього.
- **Взаємодія:** maximize зони (rectangular icon на caption), collapse/expand (arrows icon), **BUILD** на зоні будує всі datasets усередині.

### Tags

**(Базово)** **Tags** — «universal property of all Dataiku objects», що дозволяє категоризувати об'єкти: projects, datasets, recipes, managed folders, models, notebooks, webapps, dashboards, wiki articles.

- **Створення/керування:** користувачі з правом **Write Project content** створюють теги на льоту; перейменування, видалення, **зміна кольору** — через pop-up **Manage tags** (з кнопки **Add tags**). Адміністратори — через **Project Settings > Tags**.
- **Tag categories:** адміністратори створюють стандартні теги для всього інстансу синтаксисом `<category>:<tag>` (**Administration > Settings > Global tag categories**) — для уніфікації та запобігання помилкам.
- **Tags view:** Flow з об'єктами, пофарбованими **за тегом**, а не за типом об'єкта; маленька **крапка внизу праворуч** показує, що об'єкт має тег.
- **Фільтрація:** меню **Apply a View** (угорі ліворуч) → показати/сховати частини Flow за тегами.
- **Пошук:** у головному search bar використовуйте `tags:` keyword.

### Навігація по Flow

**(Базово)**

- **Zoom & pan:** коліщатко миші — zoom; перетягування — pan.
- **Minimap:** мінікарта для орієнтації у великому Flow.
- **Expand/collapse:** Flow Folding — згортання зон для спрощення.
- **Search / filter:** search box + filter icon шукають об'єкти; вибрані об'єкти підтримують bulk-дії.
- **Preview:** перегляд вмісту dataset прямо з Flow.

### Flow view options — меню «Apply a View»

**(Середній)** Меню **Apply a View** (угорі ліворуч) перефарбовує/фільтрує Flow за різними властивостями:

- **Flow Zones view** — підсвічування об'єктів обраних зон.
- **Tags view** — об'єкти за тегами.
- **Connections view** — групування за data connection.
- **Recipe Engines view** — recipes за execution engine (SQL, Spark, local stream тощо).
- **Heat Map view** — інтенсивність активності/навантаження.
- **File Size view** — розмір зберігання кожного dataset.
- **Count of Records view** — кількість рядків.
- **Last modification date / recent changes** — що змінилося нещодавно.

Стабільніші налаштування відображення — у **… (More Options) > Settings > Flow display**.

### Flow Actions — права панель

**(Базово)** Коли ви **виділяєте об'єкт**, праворуч з'являється **Actions panel** з контекстними діями: build options, керування об'єктом (tagging, sharing, configuring). Меню **Flow Actions** (унизу праворуч) містить дії на весь проєкт, наприклад **Build all**.

---

## Build пайплайну: режими збірки та jobs

### Що означає «build»

**(Базово)** Dataset — це **output of a recipe**. «Build» dataset означає **запустити recipe(s)**, що його продукують, щоб (пере)обчислити дані з поточних входів і логіки. Dataset, який ніколи не будувався, існує лише як metadata/schema без даних.

> Dataiku створює **job** щоразу, коли ви будуєте dataset, запускаєте recipe або тренуєте модель.

### Build modes

**(Середній)** Коли ви виділяєте dataset і натискаєте **Build** (або right-click → **Build**), з'являється діалог із вибором режиму. Три родини:

**A. Build only this (Non-recursive)**
Будує лише обраний dataset, використовуючи **тільки його батьківський recipe**. Найменше обчислень. Підхоплює зміни лише прямих входів батьківського recipe, не далі вгору.

**B. Build upstream (recursive)** — два підрежими:
- **Build only modified** (також «Smart reconstruction» / «Build required datasets») — DSS перевіряє upstream datasets/recipes, знаходить **перший outdated** dataset і перебудовує все від нього до обраного. **Рекомендований за замовчуванням.**
- **Build all upstream** (forced recursive) — перебудовує **всі залежності** до початку Flow, незалежно від того, чи вони застаріли. Найважчий за обчисленнями.
- Модифікатор: **Stop at zone boundary** — обмежує перевірки межами поточної Flow zone.

**C. Build downstream (recursive)** — два підрежими:
- **Build all downstream** — запускає recipes від обраного dataset до кінця Flow (сам обраний dataset не будується).
- **Find outputs and build recursively** — знаходить усі фінальні downstream datasets і будує потрібні upstream-залежності.

**Зіставлення термінів:**

| Термін у завданні | Режим у UI |
|---|---|
| non-recursive / «build only this dataset» | **Build only this (Non-recursive)** |
| smart reconstruction / «build required datasets» | **Build upstream → Build only modified** |
| forced recursive / «build dependent datasets» | **Build upstream → Build all upstream** |

**Додаткові опції:** **Update output schemas** (поширити структурні зміни схем на залежні datasets перед виконанням); на рівні Flow — **Build all** або build окремих зон.

> **Примітка про точність UI-рядків:** документація описує режими прозою; дослівні підписи кнопок у живому діалозі можуть відрізнятися за версіями. Поведінка вище — надійна; точні рядки варто звірити на живому інстансі DSS 14.

### Out-of-date detection

**(Середній)** DSS вважає dataset застарілим (потребує rebuild), коли:

- Змінено upstream recipe.
- Змінилися входи upstream recipe.
- Змінилися власні settings dataset.
- Для file-based datasets — змінилися метадані файлів (дати модифікації, список файлів).
- Для **Snowflake / BigQuery** — можна увімкнути перевірку last-modification timestamp.
- Для **інших SQL datasets** — DSS за замовчуванням вважає їх up-to-date (не може дешево перевірити свіжість зовнішньої таблиці), тож зміни ззовні можуть не визначатися автоматично.

Smart reconstruction використовує саме цю логіку (порівняння timestamps), щоб перебудувати мінімальний набір.

### Rebuild behavior (Advanced settings)

**(Просунуто)** Кожен dataset/folder/model має **Rebuild behavior** (**Settings → Advanced**; для моделей — **Recursive build behavior**):

- **Normal** — можна перебудовувати, зокрема рекурсивно.
- **Explicit** — перебудова лише якщо це безпосередня ціль build; рекурсивні build його пропускають (Saved models за замовчуванням Explicit).
- **Write-protected** — не можна перебудовувати взагалі, навіть явно (read-only з погляду Flow).
- **Allow build to continue when rule fails** — дозволяє рекурсивним build тривати попри провалені Data Quality rules.

### Jobs

**(Середній)** **Job** — одиниця виконання будь-якої операції build/run/train.

- **Jobs page:** меню **Jobs** у верхній навігації (іконка play) або скорочення **`g` + `j`**.
- **Activities:** job розкладається на activities — **кожен recipe / build = окрема activity**. Рекурсивний build = один job з багатьма activities. Кнопка **Show summary** на activity пояснює, чому вона була потрібна, та показує source/target datasets.
- **Job log:** меню **Actions** на job дає доступ до **full job log**, **Job diagnosis**, **Yarn logs**. Усі логи зберігаються в DSS data directory.
- **Job states:** Running, Done/Completed, Failed, Aborted, Queued/Waiting (в API — `DONE`, `FAILED`, `ABORTED`).
- **Manual vs scenario-triggered:** механіка та логи однакові; різниця лише в тригері (клік користувача vs scenario trigger).

---

## Object lifecycle та dependencies

**(Середній)**

- **Залежності задаються входами/виходами recipe.** Підключаючи recipe (обираючи його input і output datasets), ви декларуєте, що outputs залежать від inputs. Окремого кроку «оголосити залежність» немає — wiring recipe **і є** залежністю.
- **Build dependency graph:** сукупність зв'язків input→output утворює спрямований граф залежностей. Рекурсивні build обходять цей граф угору (що живить ціль) або вниз (що ціль живить).
- **Build dependencies vs dataset dependencies:** на рівні Flow це фактично одне й те саме — dataset «залежить» від datasets, що живлять його батьківський recipe (і транзитивно — від усього вгору).
- **Lifecycle об'єкта:** створено як metadata/schema → built (дані матеріалізовано через job) → може стати out of date при зміні upstream → rebuilt (smart або forced), з урахуванням Rebuild behavior.

---

## Колаборація: Wiki, Discussions, To-do, Dashboard

### Wiki

**(Базово)** Кожен проєкт має **wiki** для документування цілей, входів/виходів, внутрішньої логіки, організації команди та планів. Wiki базується на **Markdown** (форматування, зображення, таблиці, списки).

**(Середній)** **Ієрархія/taxonomy:** усі статті в ієрархічній taxonomy. Кожна стаття може мати **Parent**; статті без parent — у root. Дерево статей слугує table of contents.

**Layout modes:** **Article layout** (стандартний) та **Folder-oriented layout** (текст + список attachments); перемикання через **Actions > Switch to folder layout / Switch to article layout**.

**Attachments:** кнопка **Add attachment** → вкладка **File**; можна прикріпити і DSS-об'єкти. Посилання/embed:

```text
[label](PROJECT_KEY.attachment_id)        # download attachment
![label](PROJECT_KEY.attachment_id)       # embed image
```

**Лінкування до Flow-об'єктів із wiki:**

```text
[[PROJECT_KEY.Wiki Article 1]]            # інша wiki-стаття
(label)[dataset:PROJECT_KEY.dataset_name] # dataset
(label)[saved_model:PROJECT_KEY.model_id] # saved model
(label)[project:PROJECT_KEY]              # проєкт
```

**(Просунуто)** **Promote:** **Settings > Wiki** — promoted wikis з'являються в загальному меню **Wikis**. **Export:** у PDF (увесь wiki / стаття з нащадками / лише поточна); є scenario-step **wiki export** та **Confluence wiki plugin**.

### Discussions

**(Базово)** **Discussions** — вбудовані треди-обговорення прямо на об'єкті, щоб тримати рішення всередині Dataiku.

- **Підтримувані об'єкти:** Projects, Datasets, Recipes, Notebooks, Models, Dashboards/insights, Workspaces.
- **Доступ:** **chat icon** у навігаційному барі; **Discussions tab** (іконка comment-circle) у правій панелі; глобально — меню застосунків → **Discussions** (global inbox).
- **Threads:** кілька тредів на об'єкт; можна **resolve** і сховати (toggle **Show resolved discussions**).

**(Середній)** **@mentions:** Markdown підтримує `@username` з автодоповненням — згадка привертає увагу / повідомляє користувача. **Notifications:** усі обговорення, у яких ви берете участь або стосуються об'єктів, які ви **watching**, потрапляють у global discussions inbox. Email-повідомлення вимагають: SMTP-канал, налаштований адміністратором; валідний email у профілі; увімкнені опції в **My Settings**.

### To-do lists

**(Базово)** Проєкти мають вбудований **project to-do list** на project home page — один з інструментів колаборації. Дозволяє швидко фіксувати завдання й відмічати виконані прямо зі сторінки проєкту.

> *Уточнення:* офіційні сторінки підтверджують існування to-do list на project home, але детальний покроковий UI створення/призначення/закриття окремих пунктів у фечених сторінках не описано — варто перевірити на живому інстансі.

### Project Dashboard (коротко)

**(Базово)** **Dashboard** — окремий об'єкт-контейнер для відображення інформації. Проєкт може мати **кілька dashboards**; кожен має сторінки (**pages**), кожна — сітку **tiles**. Tiles бувають **simple** (текст, зображення, web pages, групи) та **insight tiles** (показують один **insight**). **Insights** — переюзабельні шматки інформації (таблиці datasets, charts, ML-звіти, метрики, scenario reports), що існують **незалежно** від dashboards.

> **Project home page vs Dashboard:** home page — автогенерована стартова сторінка проєкту (статус, активність, to-do). **Dashboard** — окремий, явно зібраний об'єкт для презентації звітів viewers. Детальніше — у [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md).

---

## Git integration та version control

**(Середній)** **Кожен DSS-проєкт — це звичайний Git repository.** Будь-яка зміна в UI (settings dataset, редагування recipe, зміна dashboard) **автоматично записується** в Git-репозиторій проєкту. Це дає traceability, history, revert та підтримку branching/merging.

> **Критичний нюанс:** Git-репо проєкту відстежує **лише metadata/definitions, а НЕ дані** всередині datasets. Це найчастіше джерело плутанини при revert.

### Перегляд історії

- **Project menu → Version Control** (або **… → Version Control**) відкриває history browser з усіма commits. Клік по commit показує деталі та змінені файли; кнопка **Compare** порівнює стани між ревізіями.
- На окремому об'єкті вкладка **History** показує історію саме цього об'єкта.

### Commit modes

**(Середній)** **Project Settings → Change Management → Commit Mode**:

- **Auto** (за замовчуванням) — зміни комітяться автоматично при збереженні.
- **Explicit** — зберігаються, але комітяться лише після кнопки **Commit** (контроль над межами та повідомленнями commit).

### Revert

**(Просунуто)** Три рівні гранулярності:

1. **Revert усього проєкту до ревізії** — клік по commit → **Revert to this revision** → Confirm. ⚠️ Скасовує **всю роботу всіх користувачів** з тієї ревізії; створює **новий commit** (історію не видаляє). Найбезпечніший варіант.
2. **Revert одного commit** — додає «reverse» commit. Ризикованіше: можливі Git-конфлікти (тоді DSS не виконає revert, треба вручну через CLI).
3. **Revert окремого об'єкта** — з вкладки **History** (наразі лише для recipes).

> ⚠️ **Merge commits** унеможливлюють revert проєкту в попередній стан із UI. Рекомендація: **squash merging** (через зовнішній інструмент — GitHub/Bitbucket/GitLab).
>
> ⚠️ Оскільки Git відстежує metadata, а не дані, після revert output dataset буде out of sync — зазвичай потрібен rebuild.

### Зовнішній/remote Git

**(Просунуто)** Локальне Git-репо можна підключити до **Git remote** (pull/push). Натисніть change-tracking indicator → **Add a remote** → введіть URL. **Автентифікація:** SSH keys (рекомендовано; **Profile → Credentials → SSH**) або HTTPS. Доступ до remote керується адміністратором у **Administration → Settings → Git → Group rules** (first-match, whitelist дозволених URL).

**Branches:** version control підтримує branching/merging концептуально; при export проєкту опція дозволяє **включити локальне Git-репо** (з історією та гілками). Розширені операції з гілками зазвичай виконуються проти remote через зовнішні Git-інструменти.

### Git у коді (Project Libraries)

**(Просунуто)** У **Project Libraries** можна імпортувати код із Git: **Git → Import from Git…** → URL репо, опційно branch/tag/commit ID, subpath, **Target path** → **Save and Retrieve**. DSS називає це **Git references**. **Git → Manage repositories…** дозволяє push, **reset from remote HEAD** (true git reset — локальні зміни втрачаються), редагування references. Той самий механізм використовується для plugins з Git.

---

## Project bundles (огляд)

**(Середній)** *Детально — у [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md). Тут лише орієнтир.*

- **Bundle** — «archive that contains a given version of the flow you built in the Design node»: знімок проєкту разом із даними, потрібними для відтворення задач.
- **Design node vs Automation node:** Design node — середовище розробки; Automation node — production. Bundles — механізм перенесення між ними.
- **Bundle vs export:** export переносить весь вміст для загальної міграції; **bundle** заточений під **deployment** — несе metadata + конкретні дані для відтворення production-задач, не все.
- **Створення:** **Project → Bundles** на Design node (потрібне право «Write project content»); можна переглянути diff від попереднього bundle і додати release notes. Bundle деплоїться на Automation node вручну або через **Project Deployer**.

---

## Best practices організації

**(Середній)**

### Flow zones (стадії обробки)

Діліть великий Flow на зони за **фазами обробки**: ingestion → preparation → modeling → output. Також для **ізоляції експериментальних гілок** та **розмежування зон відповідальності** учасників. Пам'ятайте: зони — лише layout, не security.

### Naming conventions

- Назви datasets — **читабельні, самопояснювальні, короткі**. Автогенеровані назви (input + operation) швидко стають нечитабельними.
- Робіть акцент на призначенні: **`foo_raw`** (сирий вхід) vs **`foo_clean`** (очищений вихід).
- Опційно — суфікси типу зберігання: **`foo_t`** (SQL), **`foo_hdfs`** (HDFS).
- Лише **алфавітно-цифрові символи та underscores**, **нижній регістр**.
- Ті самі правила — для columns, notebooks, projects, project keys.

### Tags (послідовно)

- Тегуйте гілки за задачею («insights», «preprocessing»), входи — «sources», частини в production — окремо.
- Тегуйте статусом: «work in progress», «done», «in production».
- Тегуйте іменем людини для ownership.
- Для уніфікації — **Global tag categories** (адмін), щоб уникнути різнобою та помилок.
- Використовуйте **колір** для комунікації (наприклад, червоний = важливе/термінове).

### Документація

- Ведіть **wiki**: цілі, входи/виходи, логіка, організація команди.
- Заповнюйте **object descriptions** (Summary tab, Markdown): джерела, свіжість, кроки обробки.
- Тримайте обговорення в **discussions**, прив'язаних до об'єкта, а не в email.

### Загальне

- Колаборація real-time — кілька людей одночасно в одному проєкті.
- Продумайте **project key** і структуру зон **на старті**.
- Використовуйте **Smart reconstruction** як режим збірки за замовчуванням.
- Розділяйте **standard** і **local** variables правильно (secrets — у local).

---

## Типові помилки

**(Базово/Середній)**

1. **Очікувати, що revert поверне дані.** Git відстежує лише metadata; після revert треба **rebuild** datasets.
2. **Плутати «Global variables» проєкту з instance-level global variables.** Це різні scope.
3. **Класти secrets у standard variables.** Вони експортуються з bundle/export — secrets мають бути в **local** variables.
4. **Покладатися на auto out-of-date для зовнішніх SQL-таблиць.** Для звичайних SQL datasets DSS не перевіряє свіжість автоматично — використовуйте **Build all upstream** (forced) або timestamp-checking для Snowflake/BigQuery.
5. **Намагатися перейменувати project key.** Він immutable — продумуйте одразу.
6. **Покладатися на Default-назви datasets.** Швидко стають нечитабельними; іменуйте свідомо.
7. **Не використовувати Flow zones у великих проєктах.** Flow стає нечитабельним; зони + tags рятують навігацію.
8. **Merge commits.** Унеможливлюють revert із UI — використовуйте squash merge.
9. **Очікувати `${variable}` у scenario-step полях.** Там працюють DSS Formulas, а не `${}` expansion.
10. **Force-recursive build «про всяк випадок».** Найважчий за обчисленнями; зазвичай достатньо smart reconstruction.

---

## Перехресні посилання

- [01-overview-architecture.md](01-overview-architecture.md) — архітектура DSS, Design/Automation/API nodes.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — datasets, connections, partitioning, Rebuild behavior у деталях.
- [04-visual-recipes.md](04-visual-recipes.md) — visual recipes (жовті кружечки).
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — Prepare recipe, формули, `variables["..."]`.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — code recipes (помаранчеві), `dataiku.get_custom_variables()`, Project Libraries / Git.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — Saved Models, Visual ML (зелені об'єкти).
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — Knowledge Banks (рожеві), Embed recipe, RAG, LLM Mesh.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — scenarios, set variables steps, тригери build.
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — **project bundles**, deployment, Automation node, Model Evaluation Stores у деталях.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — `dataikuapi` / Python client: create/duplicate/export проєктів, `get_variables()`.
- [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md) — **dashboards**, insights, charts, webapps.
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — permissions, governance, exposed elements.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — plugin recipes (червоні), Dataiku Applications.
- [15-best-practices.md](15-best-practices.md) — зведені best practices.
- [16-glossary.md](16-glossary.md) — глосарій термінів.

---

## Джерела (офіційна документація Dataiku)

- Projects (concepts): https://doc.dataiku.com/dss/latest/concepts/projects/index.html
- Homepage: https://doc.dataiku.com/dss/latest/concepts/homepage/index.html
- How to Copy a Project: https://doc.dataiku.com/dss/latest/concepts/projects/duplicate.html
- Custom variables expansion: https://doc.dataiku.com/dss/latest/variables/index.html
- Variables in code recipes: https://doc.dataiku.com/dss/latest/code_recipes/variables_expansion.html
- Variables in scenarios: https://doc.dataiku.com/dss/latest/scenarios/variables.html
- The Flow: https://doc.dataiku.com/dss/latest/flow/index.html
- Visual Grammar: https://doc.dataiku.com/dss/latest/flow/visual-grammar.html
- Flow Zones: https://doc.dataiku.com/dss/latest/flow/zones.html
- Rebuilding Datasets: https://doc.dataiku.com/dss/latest/flow/building-datasets.html
- Tags: https://doc.dataiku.com/dss/latest/collaboration/tags.html
- Version control of projects: https://doc.dataiku.com/dss/latest/collaboration/version-control.html
- Working with Git: https://doc.dataiku.com/dss/latest/collaboration/git.html
- Importing code from Git: https://doc.dataiku.com/dss/latest/collaboration/import-code-from-git.html
- Wiki: https://doc.dataiku.com/dss/latest/collaboration/wiki.html / Markdown: https://doc.dataiku.com/dss/latest/collaboration/markdown.html
- Discussions: https://doc.dataiku.com/dss/latest/collaboration/discussions.html
- Dashboards concepts: https://doc.dataiku.com/dss/latest/dashboards/concepts.html
- Creating a bundle: https://doc.dataiku.com/dss/latest/deployment/creating-bundles.html
- Knowledge Banks & RAG: https://doc.dataiku.com/dss/latest/generative-ai/knowledge/introduction.html
- Knowledge Base — Flow / Tags / Build modes / Naming: https://knowledge.dataiku.com/
- Developer Guide — Projects / Flow / Python API: https://developer.dataiku.com/latest/
