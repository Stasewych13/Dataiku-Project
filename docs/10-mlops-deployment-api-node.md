# 10. MLOps, Deployment та API node

> Документ про **продакшен-частину Dataiku**: як проєкт і модель проходять шлях **dev → prod**, як версіонувати й деплоїти проєкти через **bundles** та **Project Deployer**, як виставляти моделі як **real-time API** через **API node** та **API Deployer**, як працює **batch vs real-time scoring**, і як організувати повний **lifecycle моделі** з моніторингом, drift detection та авто-ретренінгом. Версії DSS **13.x** і **14.x** (актуально на 2025–2026). Технічні/UI/код терміни залишені в **оригіналі (англійською)**.

---

## Зміст

1. [Вступ: що таке MLOps в Dataiku](#1-вступ-що-таке-mlops-в-dataiku)
2. [Lifecycle dev → prod та ноди (короткий повтор)](#2-lifecycle-dev--prod-та-ноди)
3. [Project bundles: створення, зміст, версії](#3-project-bundles-створення-зміст-версії)
4. [Project Deployer: infrastructures, stages, deploy/activate](#4-project-deployer)
5. [Remapping connections / code envs / variables між середовищами](#5-remapping-між-середовищами)
6. [Rollback та deployment lifecycle](#6-rollback-та-deployment-lifecycle)
7. [Governance: sign-off як gate для деплою](#7-governance-sign-off-як-gate)
8. [API node та API Deployer: real-time scoring](#8-api-node-та-api-deployer)
9. [API services та типи endpoints](#9-api-services-та-типи-endpoints)
10. [Deploy API service: static vs Kubernetes, test queries, HA](#10-deploy-api-service)
11. [Query enrichment та explanations при scoring](#11-query-enrichment-та-explanations)
12. [Batch vs real-time scoring](#12-batch-vs-real-time-scoring)
13. [Model lifecycle: Saved Model, versions, champion/challenger](#13-model-lifecycle)
14. [Model Evaluation Stores та drift detection](#14-model-evaluation-stores-та-drift-detection)
15. [Ground truth, feedback loop та моніторинг у проді](#15-ground-truth-feedback-loop-та-моніторинг)
16. [Auto-retraining через scenarios](#16-auto-retraining-через-scenarios)
17. [CI/CD інтеграція (Python API, git, dssadmin)](#17-cicd-інтеграція)
18. [Best practices](#18-best-practices)
19. [Типові помилки](#19-типові-помилки)
20. [Перехресні посилання](#20-перехресні-посилання)

> **Примітка про назви нод.** Стислий огляд нод (Design / Automation / API / Deployer / Govern) та архітектури інстансу — в `01-overview-architecture.md`. Тут ми зосереджуємось на **операційному потоці**, а ноди згадуємо лише настільки, наскільки треба для розуміння deployment.

---

## 1. Вступ: що таке MLOps в Dataiku

**MLOps** — це практики й інструменти, що переносять модель з ноутбука data scientist-а в **надійний, відтворюваний, моніторений продакшен** і тримають її там здоровою. Dataiku покриває весь цей цикл нативно: версіонування проєктів (**bundles**), розподіл по середовищах (**Project Deployer**), real-time serving (**API node**), пакетне scoring (**Score recipe**), моніторинг та виявлення деградації (**Model Evaluation Stores**, **drift detection**), а також авто-ретренінг через **scenarios**. (Базово)

Ключова ідея: **розробка і продакшен фізично розділені**. Ви ніколи не пушите «сирий» код напряму у прод. Натомість збираєте **версійований артефакт** (bundle для batch, API package для real-time), реєструєте його в **Deployer**, і деплоїте на цільову **infrastructure** з контрольованим remapping connections і змінних. Це робить деплої передбачуваними, відтворюваними й такими, що відкочуються (rollback).

Цей документ читається разом із:
- `07-machine-learning-visual-ml.md` — як модель тренується (Lab, Visual ML), що таке Saved Model;
- `09-scenarios-automation-metrics-checks.md` — деталі scenarios, metrics, checks, reporters (тут лише застосування до MLOps);
- `01-overview-architecture.md` — типи нод і варіанти розгортання;
- `11-programmatic-api-python-client.md` — повний Python client (`dataikuapi`), тут лише MLOps-зрізи.

---

## 2. Lifecycle dev → prod та ноди

Короткий повтор ролей (деталі — `01-...`):

| Нода | Роль у lifecycle |
|---|---|
| **Design node** | Розробка: Flow, рецепти, тренування моделей, **створення bundles** і **API services packages**. |
| **Automation node** | Продакшен **batch**: виконує scenarios за розкладом, перебудови Flow, batch scoring проти продакшен-даних. Сюди деплоять **bundles**. |
| **API node** | Продакшен **real-time**: відповідає на HTTP/REST scoring-запити за мілісекунди. Сюди деплоять **API service packages**. |
| **Deployer** | Централізоване керування деплоями. Дві половини: **Project Deployer** (bundles → Automation nodes) та **API Deployer** (API services → API nodes). |
| **Govern node** | Sign-off / approval workflows, що **гейтять** перехід dev→prod. |

Два паралельні потоки деплою (Базово):

```
                         ┌─────────────────────────┐
   Design node  ──bundle──►  Project Deployer  ──► Automation node (batch)
       │                 └─────────────────────────┘
       │                 ┌─────────────────────────┐
       └──API package────►   API Deployer     ──►  API node (real-time)
                         └─────────────────────────┘
```

**Deployer** можна розгорнути двома способами:
- **Local Deployer** — вбудований у Design (або Automation) node; за замовчуванням на Dataiku Cloud, простіший варіант для невеликих установок;
- **Standalone / Remote Deployer** — окрема нода, що централізує деплої для багатьох Design/Automation/API nodes (рекомендовано для зрілих установок).

---

## 3. Project bundles: створення, зміст, версії

### 3.1 Що таке bundle (Базово)

**Bundle** — це **версійований снапшот проєкту** разом із даними, потрібними для відтворення задач у продакшені. Це **не** просто project export: мета bundle — перенести **метадані + дані, потрібні для replay** задач на Automation node, а не скопіювати весь вміст проєкту.

### 3.2 Що завжди всередині bundle

Захоплюється автоматично (метадані):
- Project settings;
- Recipes, Scenarios, Notebooks, Analyses;
- **Dataset metadata** (схема/конфіг — **не** самі дані);
- **Saved Models metadata** (за замовчуванням без файлів натренованої моделі);
- **Managed Folders metadata** (без вмісту);
- **Model Evaluation Store metadata**;
- **Project-level shared code** (project libraries).

### 3.3 Чого НЕМАЄ в bundle за замовчуванням

- Самі **дані датасетів** у Flow (вони **перераховуються** в проді);
- **Global / instance-level shared code** (тільки *project*-level код мандрує в bundle);
- **Plugins** (мають бути встановлені на Automation node окремо).

### 3.4 «Additions» — що додати вручну (Середній)

Під час створення bundle ви явно додаєте **реальні дані** для об'єктів, які **не перераховуються** в проді:
- **Datasets** — статичні reference/enrichment-таблиці, що не білдяться в проді;
- **Saved Models** — натреновані моделі, потрібні для продакшен-scoring;
- **Managed Folders** — допоміжні файли (зображення, серіалізовані об'єкти, PDF тощо).

> **Правило:** усе, що в проді **обчислюється з нуля** (наприклад, проміжні managed-датасети), у bundle класти не треба — лише метадані. Усе, що в проді **не перебудовується** (натренована модель, reference-таблиця), треба явно додати в Additions, інакше в проді його не буде.

### 3.5 Створення bundle на Design node (how-to)

1. Відкрийте проєкт → вкладка **Bundles** (потрібне право *Write project content*).
2. Якщо bundle ще немає — натисніть **Create your first bundle**.
3. Задайте **Bundle ID** — це **ідентифікатор версії**. Auto-semver немає: версіонуєте самі (`v1.0`, `v2.0`, `2026-06-24-prod` тощо).
4. Оберіть **Additions** (які датасети / saved models / managed folders включити з даними).
5. Додайте **Release notes** — DSS порівняє з попереднім bundle, щоб допомогти задокументувати зміни.

Дії на вкладці Bundles:
- **Publish on Deployer** — відправити bundle у Project Deployer;
- **Download** — експортувати bundle як `.zip` для ручного перенесення;
- **Revert** — відкотити проєкт до попередньої версії bundle (заміняє поточні метадані; з'являється попередження про зміни User-defined Meanings).

> **Залежності між проєктами.** Якщо bundle містить об'єкти, **share**-нуті з інших проєктів, він залежить від upstream-проєктів: їхні bundles треба **активувати раніше** за downstream.

### 3.6 Через Python (Середній)

```python
import dataiku
client = dataiku.api_client()
project = client.get_project("MY_PROJECT")

# Створити bundle на Design node
bundle = project.export_bundle("v2.0", "Додано feature engineering, оновлено модель")

# Опублікувати у Project Deployer (published project створиться, якщо його ще немає)
published = project.publish_bundle("v2.0", "MY_PROJECT_PUBLISHED")
```

---

## 4. Project Deployer

### 4.1 Published project ≠ Design project (Середній)

Натиснувши **Publish on Deployer**, ви створюєте **project на Project Deployer** і імпортуєте туди bundle. Важливо: **published project** на Deployer — це **колекція bundles (версій)**, а не копія Design-проєкту.

### 4.2 Deployment infrastructures

**Infrastructure** — це зареєстрована ціль деплою, тобто **Automation node** (або кілька) під керуванням Project Deployer. Щоб деплоїти в прод, треба створити хоча б одну. Типи цільових нод:
- **Custom Dataiku** — самостійно встановлені Automation nodes; потрібен **global admin API key** (Administration > Security > Global API keys);
- **Dataiku Cloud Stacks** — Automation-інстанси через fleet management;
- **Dataiku Cloud** — через Launchpad > Extensions > Add an extension > Automation node.

Створення infrastructure (Project Deployer > Infrastructures):
- унікальний **Infrastructure ID**;
- **base URL** Automation node (з протоколом/портом);
- **admin API key secret**;
- **permissions** на вкладці Settings.

### 4.3 Lifecycle stages: dev / test / prod

Infrastructures організовано за **lifecycle stage**. Передвстановлені stages: **Development**, **Test**, **Production**. Налаштовуються в **Administration > Settings > Deployer > Project deployment stages**. Stage — це мітка/групування, що керує організацією та правами (наприклад, обмежити, хто може деплоїти в Production).

Типовий staged-потік: bundle спершу деплоять на **dev/test** Automation node, проганяють перевірки, і лише потім просувають на **prod** Automation node.

### 4.4 Permissions на infrastructure (Середній)

| Рівень | Що дає |
|---|---|
| **Not granted** | Infrastructure невидима/недоступна. |
| **View** | Бачити deployments і деталі (read-only). |
| **Deploy** | Створювати, оновлювати, видаляти deployments. |
| **Admin** | Редагувати всі налаштування infrastructure. |

**Deployment policies** на infrastructure (Просунуто):
- **Permissions propagation**: *None* (тільки той, хто деплоїть, стає Project Owner) / *Limited* (design-користувачі стають readers) / *Full* (репліка security-схеми Design);
- **Deploy as User** — застосовувати зміни деплою під технічним акаунтом, а не під користувачем;
- **Run-as User** — технічний користувач для виконання Scenarios і WebApps (щоб не залежати від конкретних людей).

**Deployment hooks** — кастомний Python-код **до/після** деплою (створити тікет, надіслати у Slack/Teams, запустити post-deployment scenario). Бачити/редагувати hooks можуть лише адміни infrastructure.

### 4.5 Deploy / activate bundle (how-to)

1. У Project Deployer оберіть завантажений bundle.
2. Натисніть **Deploy**.
3. Оберіть **target infrastructure** (Automation node).
4. Задайте **Deployment identifier** (посилання лише в межах Deployer).
5. (Опц.) задайте **Target Project Key** і **Target Project Folder**.
6. **Create**.

**Deploy vs Update:**
- **Deploy** — первинне встановлення bundle на Automation node;
- **Update** — оновити наявний deployment новішим bundle **або** запушити змінені налаштування.

**Що означає Activate:** «коли ви тиснете Deploy (або Update), DSS відправляє bundle на Automation node і **активує** його». Через Project Deployer активація **автоматична** (окремого кроку немає). При ручному імпорті (без Deployer) **Activate** — окрема кнопка. Активація розпаковує метадані + bundled-дані й робить їх **поточним станом** проєкту.

**Pre-activation перевірки:**
- відсутні connections → **fatal errors**, треба remapping;
- конфлікти user-defined meanings → warnings;
- недоступні plugins → alerts (проблема лише якщо проєкт їх реально використовує).

> **Критичне правило:** будь-яка зміна налаштувань deployment вимагає натиснути **Update**, щоб запушити її на Automation node — **навіть якщо сам bundle не змінився**. Стан розсинхрону позначається як **Out of sync**.

---

## 5. Remapping між середовищами

Це **серце** коректного dev→prod в Dataiku. Без remapping connections з іменами `sql-dev` тощо «приїхали» б у прод і вказували б на dev-дані. (Середній/Просунуто)

Remapping задається у **двох місцях**:
- **Infrastructure settings** — дефолти для всіх deployments на цій infrastructure;
- **Settings** конкретного deployment — per-deployment override (local variables, connection remapping, code environments, scenarios, активний bundle).

### 5.1 Connection remapping

- Deployer **зіставляє connections за іменем** між Design і Automation.
- Ви задаєте явні remapping-и, наприклад `sql-dev → sql-prod`.
- Поведінка: «якщо налаштовано remapping `sql-dev → sql-prod`, при деплої Project Deployer вимагатиме connection `sql-prod` і спрямує все використання на нього».
- Якщо remapping **не задано**, DSS автоматично шукає connection із **тим самим іменем** на Automation node.
- **Важливо:** цільовий продакшен-connection має **вже існувати** на Automation node (узгодьте з адмінами).
- Зміни remapping діють **лише для майбутніх** deployments/updates.

### 5.2 Code environment remapping

- Будь-який **code env**, що використовується проєктом, має існувати на Automation node.
- Дефолтне налаштування імпорту code env зазвичай: **створити новий code env**, якщо проєкт посилається на той, якого ще немає на Automation node.
- Деталі code envs — `06-coding-recipes-code-envs-notebooks.md`.

### 5.3 Variables override

- З вкладки Settings deployment можна **перевизначити local variables** проєкту для продакшену (production-специфічні значення параметрів — шляхи, ліміти, prod-bucket тощо).

### 5.4 Scenario triggers

- У **Settings > Scenarios** контролюєте, чи **enabled / disabled / unchanged** тригери scenarios на Automation node; можна **вимкнути всі автоматичні тригери** одразу.
- **Automation local** scenarios (створені прямо на Automation node) звідси не керуються.
- Стани активації scenarios **зберігаються** між update-ами; локальні scenarios/notebooks теж зберігаються — **окрім** випадку, коли об'єкт bundle має той самий **ID**, що й локальний (тоді версія з bundle перезаписує).

---

## 6. Rollback та deployment lifecycle

### 6.1 Rollback (Середній)

- **У Deployer UI:** оберіть **попередню версію bundle** і **задеплойте її як Update**. Список версій (deployment history) дозволяє обрати ранній реліз і передеплоїти.
- **На рівні проєкту:** **Revert** заміняє поточний стан метаданими зі збереженого bundle.
- **У CI/CD:** стандартний патерн — **передеплоїти попередній bundle**. Пайплайн **зберігає ID поточного працюючого bundle перед update**, тож якщо деплой або smoke-тест провалюються — відновлює збережений `bundle_id`.

### 6.2 Стани та статус deployment

- **Health status** — на дашборді deployments і сторінці статусу;
- **Timeline** недавніх scenario runs з результатами;
- **Last Updates** — збережена інформація про оновлення;
- **Out of sync** — розбіжність налаштувань між Deployer і Automation node; лікується кнопкою **Update**.

### 6.3 Безпечний rollback через Python

```python
deployment = client.get_projectdeployer().get_deployment("MY_DEPLOYMENT")
settings = deployment.get_settings()
settings.bundle_id = "v1.0"          # повернути попередню версію
settings.save()
deployment.start_update().wait_for_result()
```

---

## 7. Governance: sign-off як gate

(Просунуто; деталі governance — `13-governance-security-collaboration.md`.)

- **Sign-off гейтить деплой.** Без затвердження у відповідному sign-off-процесі **Govern node** bundle **не може** перейти з Deployer на Automation (чи API) node — тобто з dev у prod.
- Sign-off підтримує два типи рев'ю: **feedback reviews** і **final approvals**; reviewers задаються на рівні governed-проєкту (**Sign-off reviewers and approvers**).
- **Blueprint Designer** (просунуті ліцензії) дозволяє чіпляти sign-offs там, де вони потрібні (наприклад, на Qualification-кроці).
- **Технічний механізм:** governance enforced через **pre-deployment hook** на infrastructure; параметр `govern_check_policy` (дефолт `'NO_CHECK'`). Hook звертається до Govern node, перевіряє статус sign-off і повертає `HookResult.error(...)` (блокує) або `HookResult.success(...)` (дозволяє).

---

## 8. API node та API Deployer

### 8.1 Навіщо окрема нода (Базово)

**API node** — сервер застосунків реального часу, що відповідає на **HTTP/REST scoring-запити** (наприклад, прогноз для одного запису за мілісекунди). Кожен інстанс API node **повністю незалежний**: будь-який клієнт може звертатися до будь-якої ноди. Це дає горизонтальне масштабування й high availability.

URL endpoint-а має вигляд:
```
/public/api/v1/<service_id>/<endpoint_id>/<action>
```

### 8.2 API service vs endpoint vs version/package (Середній)

| Поняття | Що це |
|---|---|
| **API service** | Одиниця керування й деплою на API node. Створюється в **API Designer** (у проєкті на Design/Automation node). Містить **один або кілька endpoints**, які оновлюються/керуються **разом**. |
| **API endpoint** | Один шлях (URL), що виконує одну функцію: повернути прогноз, виконати функцію, SQL-запит чи lookup. Service містить 1..N endpoints. |
| **Version (generation)** | Версія service. Кілька версій можуть працювати одночасно (для A/B та оновлень без простою). У ручному деплої версія зветься **generation**. |
| **Package** | Фізичне втілення версії — `.zip` з усіма endpoints одного service, що переноситься на API nodes для активації. |

### 8.3 API Deployer

**API Deployer** — централізований компонент, що деплоїть **published service** на infrastructures. Це друга половина **Dataiku Deployer** (перша — Project Deployer для bundles). Можливості: створення infrastructures, deploy/update на одну чи кілька infrastructures, моніторинг здоров'я/статусу нод, version control із rollback, permissions, логування/аудит запитів, deployment hooks. Режими: **Local** (вбудований у Design/Automation node) або **Remote/Standalone** (окрема нода).

---

## 9. API services та типи endpoints

Тип endpoint обирається при його створенні в **API Designer**. (Середній)

### 9.1 Prediction endpoint зі Saved Model (Visual Model)

- **Що:** виставляє модель, натреновану у Visual ML і задеплоєну у Flow як **Saved Model**, як real-time prediction API.
- **Як налаштувати:** з Flow оберіть модель → **Create an API** → задайте **Service identifier** і **Endpoint identifier** → потрапляєте в API Designer. Або в API Designer створіть service → новий endpoint типу **Prediction** → **Select a saved model**.
- **Action URLs:** `/predict` (звичайний), `/predict-effect` (causal), `/forecast` (time series).
- **Опції:** **Java scoring** для сумісних моделей (значно швидше; **несумісне** з explanations); explanations (**SHAPLEY** або **ICE**); вкладка **Test queries**.
- **Коли:** низьколатентний продакшен-scoring supervised Visual ML моделі. Це **основний** тип для більшості ML-кейсів.

### 9.2 Python prediction (custom)

- **Що:** виставити модель, написану напряму на Python (поза Visual ML).
- **Як:** клас, що розширює `dataiku.apinode.predict.predictor.ClassificationPredictor` або `RegressionPredictor`; метод `def predict(self, features_df)` приймає **pandas DataFrame** рядків; опційний `__init__(self, folder_path)` завантажує серіалізовану модель із **managed folder**.
- Задається **Code environment** у Settings. URL: `/predict`.
- **Увага:** більшість API, що працюють у Python-рецептах, тут не працюють (код виконується на **API node**, не Design) — доступ до даних робіть через **query enrichment**.

### 9.3 R prediction (custom)

- Аналог Python prediction в R. Тип **Custom Prediction (R)**; функція приймає named feature-аргументи, повертає число/вектор або list з `prediction` (+ `probas`, `customKeys`). Managed folders через `dkuAPINodeGetResourceFolders()`. URL: `/predict`.

### 9.4 Python function endpoint

- **Що:** виставити будь-яку Python-функцію (будь-яка дія, будь-який JSON-серіалізовний результат — операції з БД, збереження файлів, кастомні обчислення).
- **Як:** код містить ≥1 функцію, її ім'я задається в **Settings**; параметри функції відповідають атрибутам JSON-запиту, напр. `def my_api_function(age, days, price):` або гнучка `def api_py_function(z=0, **data):`. Managed folders через глобальну змінну `folders` (`folders[0]`, …). URL: `/run`.

### 9.5 R function endpoint

- Те саме, що Python function, але в R. Ім'я функції в Settings; результат JSON-серіалізовний; managed folders через `dkuAPINodeGetResourceFolders()`. URL: `/run`. Dataset-based enrichment недоступний.

### 9.6 SQL query endpoint

- **Що:** виставити параметризований SQL-запит. URL: `/query`.
- **Як:** у **Settings** оберіть **database connection**; у **Query** напишіть SQL із плейсхолдерами `?` (без лапок навколо — БД сама конвертує типи), і додайте **ім'я параметра** на кожен `?`. Приклад: `select * from mytable where email = ?;` з параметром `target_email`. SELECT повертає columns+rows; INSERT/DELETE — `updatedRows`. **Add a query** дозволяє кілька стейтментів. Тільки «звичайні» SQL-connections (Hive/Impala — ні); JDBC-драйвери мають бути на API node.

### 9.7 Dataset lookup endpoint

- **Що:** пошук записів у DSS-датасеті за **lookup keys**. URL: `/lookup`. Підтримує кілька датасетів, кілька вхідних записів, кілька ключів, довільні retrieved columns.
- **Обмеження:** максимум **один рядок** на вхідний запис (кілька збігів → помилка/відкидаються). Конфігурація така сама, як для query enrichment (див. розділ 11).

> Існує також **MLflow model** endpoint (`endpoint-mlflow.html`) для імпортованих MLflow-моделей — той самий `/predict`-контракт, що й у Visual Model.

### 9.8 Формат запиту/відповіді prediction endpoint (Середній)

**Запит** (POST на `/public/api/v1/<service_id>/<endpoint_id>/predict`):
```json
{
  "features": { "feature1": "value1", "feature2": 42 }
}
```

**Запит з explanations:**
```json
{
  "features": { "feature1": "value1", "feature2": 42 },
  "explanations": { "enabled": true, "method": "SHAPLEY", "nMonteCarloSteps": 100, "nExplanations": 5 }
}
```

**Відповідь (classification):**
```json
{
  "result": {
    "prediction": "FRAUDULENT",
    "probaPercentile": 94,
    "probas": { "FRAUDULENT": 0.878, "NOT_FRAUDULENT": 0.122 },
    "explanations": { "transaction_amount": 0.485, "payment_type": -0.304 }
  }
}
```

Інші контракти: **function** (`/run`) — плоский JSON named-аргументів, відповідь = результат функції; **SQL** (`/query`) — `{ "columns": [...], "rows": [[...]] }` або `{ "updatedRows": N }`; **lookup** (`/lookup`) — об'єкт `data` з retrieved-колонками. Є й GET-форма: `/predict-simple?feat1=val1&feat2=val2`.

---

## 10. Deploy API service

### 10.1 Створення package / version (how-to)

У **API Designer** натисніть:
- **Push to API Deployer** — створює **Version** (generation) на основі поточної active-версії saved model і публікує її в API Deployer (створюючи **Published Service**); або
- **Prepare package** — робить завантажуваний `.zip` у вкладці **Packages** (для ручного деплою).

Давайте версіям змістовні ідентифікатори, напр. `v4-new-customer-features`.

### 10.2 Infrastructures: static vs Kubernetes (Середній)

Треба створити хоча б одну API infrastructure. Типи:
1. **Static infrastructure** — набір **заздалегідь встановлених API nodes**. Встановлюєте API nodes окремо, генеруєте admin keys (`./bin/apinode-admin admin-key-create`), реєструєте URL + API key кожної ноди в налаштуваннях **API Nodes** Deployer-а.
2. **Kubernetes infrastructure** — API Deployer **динамічно** створює containers/pods у вашому K8s-кластері (GKE/AKS/EKS/Minikube), кожен deployment = **Kubernetes Deployment** з кількох **replicas** + **Kubernetes Service** для публічного URL. Підтримка GPU-образів (CUDA), Ingress/Gateway API / Load Balancer / Node Port.
3. **External platform** — AWS SageMaker, Azure ML, Databricks, Google Vertex AI, Snowflake Snowpark.

**Stages** (lifecycle stage) для API теж є: **Development / Test / Production** (Administration > Settings > Deployer > API deployment stages).

### 10.3 Deploy (how-to)

**З Deployer:** Push to API Deployer → у лівій панелі оберіть версію → **Deploy** → оберіть target **infrastructure** → задайте **deployment identifier** → validate. Сторінка результату дає публічні URL, графіки моніторингу, sample-код.

**Без Deployer (manual, на standalone API node):**
```bash
./bin/apinode-admin service-create <SERVICE_ID>
./bin/apinode-admin service-import-generation <SERVICE_ID> <PATH_TO_ZIP>
./bin/apinode-admin service-switch-to-newest <SERVICE_ID>
```
SQL/enrichment-connections у ручному режимі задаються в `DATA_DIR/config/server.json` через `remappedConnections` (referenced data) і `bundledConnection` (bundled data).

### 10.4 Test queries

У endpoint є секція **Test queries / Test**: натисніть **add test queries** (можна підтягнути рядки **з test-датасету**), потім **Run test queries** (**Play test queries**), щоб перевірити прогнози/результати ще **до** деплою. Для Python/R-endpoints є **Development server**: **Deploy to Dev Server** завантажує код/модель для прогонки test queries.

### 10.5 High availability та load balancing (Просунуто)

- **Архітектура HA:** кілька незалежних API node інстансів на окремих машинах за **load balancer** — апаратним (**F5**), програмним (**HAProxy**) чи хмарним (**AWS ELB**). LB роздає запити по здорових нодах і дає прозорий failover.
- **Health probe (`isAlive`):** API node виставляє глобальний probe на mount point **`/isAlive/`** — HTTP **2xx**, коли нода жива, **5xx** — коли ні. Показує статус ноди, не окремих services.
- **Ручне виведення з пулу:** створіть файл **`apinode-not-alive.txt`** у data directory ноди → `isAlive` поверне not-alive (LB дренує трафік); видаліть — відновиться.
- **Активація версії без простою:** «існуючі запити дообслуговуються старою generation, нові йдуть на нову generation, щойно вона готова» — без втрати запитів.
- **Rolling upgrade:** (1) `isAlive` → false (вивести з пулу), (2) стоп/апгрейд, (3) рестарт + завантажити новий package, (4) повернути в пул; повторити по нодах.

---

## 11. Query enrichment та explanations

(Середній)

**Query enrichment** автоматично доповнює scoring-запит **додатковими features**, які підтягуються з lookup-таблиць **під час scoring** — щоб клієнт надсилав лише мінімальні **lookup keys**, а не всі features моделі.

Як працює:
1. запит приходить зі значеннями lookup-ключів (напр. `customer_id`, `product_id`);
2. API node запитує enrichment-датасети за цими ключами;
3. отримані колонки **додаються** до запису перед scoring.

Приклад: fraud-модель тренована на «поточне замовлення + історичні агрегати клієнта/товару» — бекенд надсилає лише деталі замовлення + ID, а таблиці `customer_data` / `product_data` підтягуються автоматично.

Налаштування — у секції **Features mapping** endpoint:
- **Lookup mapping** — колонки lookup-ключа в enrichment-датасеті + їх відповідність features запиту;
- **Retrieved columns** — які колонки забрати + відповідність очікуваним features моделі;
- error handling для **Unspecified key / No match / Several matches**.

Варіанти розгортання даних enrichment:
- **Bundled data** (малі/середні таблиці): копіюються в package і вантажаться в приватну SQL-БД кожної API node (`bundledConnection` у `server.json`);
- **Referenced data** (великі/часто оновлювані): у package лише посилання (connection + таблиця); кожна нода використовує `remappedConnections` на продакшен-БД.

**Prediction explanations** обчислюються при scoring (якщо enabled): **SHAPLEY** або **ICE**, повертаються в об'єкті `explanations` як feature→influence. **Несумісні з Java scoring**.

---

## 12. Batch vs real-time scoring

Ключове розрізнення MLOps в Dataiku. (Базово)

| | **Batch scoring** | **Real-time scoring** |
|---|---|---|
| **Інструмент** | **Score recipe** у Flow | **API endpoint** на API node |
| **Нода** | Design / Automation | API node |
| **Вхід** | цілий dataset (тисячі–мільярди рядків) | один запис (або малий batch) по HTTP |
| **Латентність** | хвилини–години; за розкладом | мілісекунди; на запит |
| **Запуск** | scenario / ручний build | HTTP/REST виклик клієнта |
| **Кейс** | нічний скоринг бази клієнтів, сегментація | онлайн-фрод, рекомендації в UI, рішення в реальному часі |

**Score recipe** (batch): бере **Saved Model** + вхідний dataset, дописує колонку(и) прогнозу у вихідний dataset. Виконується у Flow, версіонується через bundle, оркеструється scenario на Automation node.

**API endpoint** (real-time): та сама модель, але виставлена як HTTP-сервіс на API node для синхронного scoring окремих записів.

> Одну й ту саму **Saved Model** можна використати **і** в Score recipe (batch), **і** в prediction endpoint (real-time) — це дає консистентність між двома режимами.

---

## 13. Model lifecycle

### 13.1 Lab → Flow → Saved Model (Базово)

Модель спершу будується в **Lab** (всередині **Visual Analysis**, сторінка **Models**): designed, trained, explored, selected. Коли влаштовує — **Deploy** з Lab у **Flow**, де вона стає зеленим ромбом — **Saved Model**. Разом із нею у Flow з'являється **Training recipe** (Retrain recipe), що дозволяє **ретренити** модель із **ідентичною** конфігурацією (алгоритм, гіперпараметри, feature handling), але на нових даних. Деталі тренування — `07-machine-learning-visual-ml.md`.

### 13.2 Versions та active version (Середній)

Saved Model містить кілька **versions**. Перша задеплоєна — active. Правило: «модель, задеплоєна у Flow, — перша й active версія, тобто та, що використовується у Retrain, Score та Evaluate recipes».
- Змінити продакшен-версію: відкрийте Saved Model (або Training recipe) → **Make Active** на потрібній версії.
- При **Retrain** дизайн-налаштування зберігаються, модель перенавчається на нових даних і за замовчуванням стає **новою active версією**.
- Python: `DSSSavedModel.set_active_version(...)`; redeploy-операції мають параметр `activate` (дефолт `True`).

### 13.3 Champion / challenger (Середній)

- **Champion** — модель, активна в проді зараз (або найкраща наразі).
- **Challengers** — кандидати на порівняння з champion.
- Порівняння — інструмент **Model Comparison** (top-nav: **ML menu > Model Comparisons**): дозволяє порівнювати моделі з трьох джерел — **Saved Model versions**, **Lab models**, **Evaluations** (з Model Evaluation Stores). Показує метрики, decision charts, ROC, confusion matrices, feature handling; у **Summary** призначаєте champion.
- У проді champion/challenger реалізується через **shadow scoring** (challengers скорять ті ж запити, але **не** повертають прогноз клієнту) або **A/B testing** (див. 13.4). Challenger «промотують», зробивши його версію active / передеплоївши.

### 13.4 A/B testing endpoints (Просунуто)

API node підтримує A/B напряму: кілька активних **generations** одного service з ймовірнісним розподілом трафіку.
```json
{"entries": [
  {"generation": "v1", "proba": 0.5},
  {"generation": "v2", "proba": 0.5}
]}
```
`proba` мають у сумі давати **1**. Admin-команди:
```bash
echo '<JSON>' | ./bin/apinode-admin service-set-mapping <SERVICE_ID>
./bin/apinode-admin service-list-generations <SERVICE_ID>
```
Query log фіксує, **яка версія** обслужила запит — це дозволяє порівнювати A/B на реальному трафіку. Відмінність від shadow scoring: при A/B кожна generation реально обслуговує частку трафіку й **повертає** прогнози; при shadow — challenger скорить ті ж запити, але прогнози клієнту не повертаються.

---

## 14. Model Evaluation Stores та drift detection

### 14.1 Model Evaluation Store (MES) (Середній)

**MES** — зелений Flow-об'єкт, що зберігає **часовий ряд evaluations**: відстежує поведінку й перформанс моделі в часі та є фундаментом drift detection.

**Model Evaluation (ME)** — один запис: результат обчислення перформансу/поведінки моделі на **Evaluation set** у конкретний момент. Кожен прогон **Evaluate recipe** додає **новий ME** (рядок) у store.

**Evaluate recipe** (Evaluation recipe):
- **Inputs:** (1) **Saved Model** з Flow (можна обрати версію; дефолт — **active**) + (2) **evaluation dataset**.
- **Standalone Evaluate recipe** — для зовнішніх моделей: порівнює evaluation dataset із **reference dataset**, без saved model.
- **Три виходи** (будь-яка комбінація): **Output dataset** (features + прогноз + правильність), **Metrics dataset** (один рядок метрик на прогон), **Model Evaluation Store** (сам store з усіма екранами).

Метрики/екрани: **ROC AUC**, precision, recall, F1, регресійні метрики, confusion matrix, decision/density/lift charts, error deciles, partial dependence, subpopulation analysis. Можна додати **Custom metrics** (Python `score`-функція, що повертає float).

> **Обмеження MES:** не працює з **clustering**, **ensembling** та **partitioned** моделями. Evaluate recipe підтримує Classification, Regression, Time Series Forecasting (non-partitioned).

### 14.2 Типи drift (Просунуто)

- **Data Drift** — зміна статистичного розподілу features.
- **Concept Drift** — зміна зв'язку між features і target.
- Принцип: «drift analysis — це завжди порівняння даних поточного ME з reference».

Усередині ME є секція **Drift** з трьома аналізами:

**Input Data Drift** (ground truth **не** потрібен):
- **Domain classifier (drift model):** бінарний класифікатор тренується на конкатенації семплів train-даних і даних, що оцінюються. «Чим вища accuracy drift-моделі, тим краще вона розрізняє, звідки рядок» — висока accuracy = сильний drift.
- **Global Drift Score:** drift model accuracy + lower/upper bounds (довірчий інтервал) + **binomial test**.
- **Univariate data drift:** на кожен feature — відповідна метрика залежно від типу (**PSI** / Population Stability Index, **Kolmogorov–Smirnov** для числових, **Chi-square** для категорійних).
- **Feature Drift Importance:** scatter-plot важливості features оригінальної моделі vs drift-моделі.

**Prediction Drift** (ground truth **не** потрібен): аналізує розподіл **прогнозів** на оцінюваних даних. Суттєва зміна розподілу прогнозів сигналізує про можливий concept drift.

**Performance Drift** (ground truth **потрібен**): порівнює виміряний перформанс (AUC, precision, recall…) поточного ME vs reference — чи деградує модель.

### 14.3 Metrics & checks для моделей

- **Metrics** на MES: і performance-, і data-drift-метрики зберігаються як **DSS metrics** на кожному обчисленні ME — можна графіти будь-яку в часі.
- **Checks** валідують метрику проти **threshold range** і повертають **OK / WARNING / ERROR**. Налаштовуються в **Evaluation Store > Settings > Status checks**. Можна підняти error, якщо performance впав надто низько **або** drift-метрика зросла надто високо.

---

## 15. Ground truth, feedback loop та моніторинг

### 15.1 Ground truth і чому це складно (Середній)

**Ground truth** — реальні правильні мітки, що зазвичай стають доступні **після** прогнозу (наприклад, чи клієнт реально churn-нув — відомо лише через місяці). Часто повільно й дорого збирати.

Два підходи до моніторингу:
- **Ground Truth Monitoring** — коли мітки приходять, з'єднуємо їх із продакшен-прогнозами й прямо міряємо accuracy → **Performance Drift**;
- **Input Drift Monitoring** — аналізуємо зсув розподілів features ще **до** появи міток (без ground truth) → **Input Data Drift** + **Prediction Drift**.

### 15.2 Як actuals повертаються в цикл (Просунуто)

1. Модель скорить продакшен-записи (Score recipe batch або API node real-time). API-прогнози фіксуються як **prediction logs / audit logs**.
2. Логи централізуються через **Event Server** у file-based сховище.
3. Коли ground truth доступний — будуєте Flow, що **reconcile/join**-ить реальні результати зі збереженими прогнозами (зазвичай **Join recipe** за request/record id).
4. Запускаєте **Evaluate recipe** на цьому labeled-датасеті → **Performance Drift** у MES.

Компоненти feedback loop: **MES** (центр порівняння кандидатів і прод-версії в часі), **online comparison** (shadow scoring / A/B), **logging system** (дані з прод-сервера повертаються у прототипне середовище для ітерацій).

### 15.3 Моніторинг у проді (Просунуто)

**Unified Monitoring** — єдиний DSS-екран операційного здоров'я задеплоєних ML-активів через Design / Automation / API nodes. Для кожного API Endpoint — **Global Status**, **Deployment Status** і (якщо лінкується до DSS Saved Model) **Model Status**.
- **Model Status** обчислюється так: Unified Monitoring дивиться на всі MES, де ця Saved Model — вхід Evaluation recipe, бере **останній** результат кожного check і агрегує як **найгірший** результат.
- Scope: для **Project Models** запитує Automation node, де задеплоєно проєкт; для **API Endpoints** — Design node, де зібрано package (потрібен URL + API key).
- Налаштування self-managed: **Administration > Settings > Deployer**.

**Monitoring loop для API endpoint:**
- налаштовуєте **Event Server** на Design/Automation node, конфігуруєте API nodes слати логи туди — всі прогнози всіх deployments повертаються в Event Server;
- у prediction endpoint відкриваєте панель **Monitoring** → **Configure** (на Cloud / self-managed ≥ 13.2 є one-click-метод; раніше — ручне підключення Event Server + Evaluate recipe);
- збережені prediction logs споживає **Evaluate recipe** для обчислення drift.

**Моніторинг без ground truth (standalone monitoring loop):** коли міток немає — **Standalone Evaluate recipe** порівнює продакшен-дані з reference dataset → **Input Data Drift** + **Prediction Drift**, рання попередня сигналізація деградації.

---

## 16. Auto-retraining через scenarios

Повний автоматичний champion/challenger + auto-redeploy будується зі **Scenarios + Metrics + Checks** (деталі scenarios — `09-scenarios-automation-metrics-checks.md`). (Просунуто)

Кроки scenario:
- **Build/Train** — ретренити Saved Models (**Add Model to Build**); цей самий крок із **Evaluation Store як виходом** обчислює новий ME;
- **Compute metrics** — перерахувати метрики на datasets/models/MES;
- **Run checks** — перевірити checks проти thresholds (OK/WARNING/ERROR);
- кастомні Python/SQL-кроки + умовна логіка на результатах metrics/checks.

Типовий drift-triggered потік авто-ретренінгу:
1. **Check** у **Evaluation Store > Settings > Status checks** із порогом (по AUC або по input-data-drift score), за яким хочемо ретренінг.
2. У **Scenario** крок **Build/Train**, що перебудовує MES (Evaluate recipe на свіжих прод-даних) → крок **Run checks**.
3. Якщо check = **WARNING/ERROR** (drift / падіння перформансу) — умовно запускаємо **Build/Train**, що **ретренить** Saved Model; за замовчуванням нова версія стає active (challenger → champion).
4. (Опц.) умовне порівняння challenger vs champion перед промоушеном + крок **redeploy** нової active-версії на Automation/API node.

**Тригери scenario:** time-based (щогодини/щодня), на **dataset change**, на **custom/metric-threshold** умови, або вручну. Тобто весь retrain-цикл можна тригерити **на drift/metric-умову**.

**Alerting:** scenarios інтегрують **reporters** — **Email**, **Slack**, **Webhook**, custom. Фаєрите їх (або ретренінг) за результатами checks: коли drift/metric-поріг перетнуто — команду повідомлено та/або запускається ремедіація.

---

## 17. CI/CD інтеграція

(Просунуто; повний Python client — `11-programmatic-api-python-client.md`.)

### 17.1 Python client для деплою

Project Deployer отримують через `client.get_projectdeployer()`. Ключові класи: `DSSProjectDeployer`, `DSSProjectDeployerDeployment`, `DSSProjectDeployerDeploymentSettings`, `DSSProjectDeployerInfra`, `DSSProjectDeployerProject`.

End-to-end (Dataiku ≥ 13.4, Python ≥ 3.10):
```python
import dataiku
client = dataiku.api_client()

# 1. Export bundle на Design node
project = client.get_project("MY_PROJECT")
project.export_bundle("v2.0", "release notes")

# 2. Publish у Project Deployer
project.publish_bundle("v2.0", "MY_PROJECT_PUBLISHED")

# 3. Створити й запустити deployment
pd = client.get_projectdeployer()
deployment = pd.create_deployment(
    deployment_id="my-deployment",
    project_key="MY_PROJECT_PUBLISHED",
    infra_id="prod-infra",
    bundle_id="v2.0",
)
update = deployment.start_update()
update.wait_for_result()
print(f"Deployment state: {update.state}")
```

`create_deployment(...)` повертає handle, але **не запускає** деплой — треба `start_update()` + `wait_for_result()`. Зміна `settings.bundle_id` + `save()` + `start_update()` — це механізм перемикання/rollback активного bundle у коді.

### 17.2 CI/CD-пайплайни

Референсний **Jenkins**-пайплайн (із Project Deployer): Design node + **дві Automation nodes** (Pre-Prod, Prod) + **JFrog Artifactory** для зберігання артефактів. Стадії: **Bundle Creation** → **Artifact Publishing** → **Pre-Production Deployment** → **Smoke Testing** (`TEST_SMOKE`) → **Production Deployment** → **Rollback** (передеплой попереднього bundle, якщо стадія впала — ID поточного bundle зберігається перед update). Є й варіант **Azure Pipelines**. Принцип: потрібен оркестратор, що вміє запускати **Python**, і ви використовуєте `dataikuapi`, а не сирий HTTP REST.

### 17.3 Git та dssadmin

- **Кожен Dataiku-проєкт — це git-проєкт:** історія комітів, revert, гілки; можна налаштувати **remote repository** (GitLab/Bitbucket/GitHub). Можливий **GitOps**.
- **dssadmin / технічний акаунт:** продакшен-деплой можна обмежити технічним service-акаунтом (часто `dssadmin`), яким керує CI/CD-оркестратор через Python API. Разом із політиками **Deploy-as-User / Run-as-User** на infrastructure це дає повністю автоматизований, identity-контрольований пайплайн.

---

## 18. Best practices

### Версіонування (Базово/Середній)
- Давайте **bundle ID** та **API version** змістовні, послідовні імена (`v2.3-prod`, `v4-new-features`), а не випадкові — це ваша історія релізів і основа rollback.
- Завжди заповнюйте **Release notes**: DSS порівнює з попереднім bundle і дає матеріал для аудиту.
- Зберігайте **відповідність bundle ↔ git-комміт** проєкту, щоб однозначно відтворити будь-який реліз.

### Ізоляція середовищ (Середній)
- Тримайте **окремі connections** для dev і prod (`sql-dev` / `sql-prod`) і **завжди** налаштовуйте explicit **connection remapping** — не покладайтесь на «той самий ім'ям».
- Розділяйте stages **Development / Test / Production** і прогоняйте bundle через test-Automation node перед prod.
- Обмежуйте право **Deploy** на Production окремій групі; для prod-деплою використовуйте **Deploy-as-User** з технічним акаунтом.

### Connection / code env remapping (Середній)
- Перед першим деплоєм переконайтесь, що **prod-connections і code envs вже існують** на Automation/API node.
- Override **local variables** під прод (шляхи, ліміти, prod-bucket) у Settings deployment, а не хардкодьте у Flow.

### Моніторинг drift (Просунуто)
- Будуйте **MES** з самого початку: налаштуйте регулярний **Evaluate recipe** (через scenario) на свіжих прод-даних.
- Якщо ground truth недоступний одразу — моніторте **Input Data Drift** + **Prediction Drift** (standalone loop), а **Performance Drift** додавайте, щойно з'являться мітки.
- Налаштуйте **checks** на ключові метрики (AUC, drift score) + **reporters** (Slack/Email), щоб дізнаватися про деградацію **до** того, як її помітить бізнес.

### Безпечний rollback (Середній)
- **Зберігайте поточний `bundle_id` перед кожним update** у CI/CD — це ваш миттєвий шлях відкоту.
- Прогоняйте **smoke-тест** (test queries для API, scenario-run для batch) одразу після деплою; при провалі — автоматично передеплойте попередню версію.
- Для API використовуйте **активацію без простою** і **rolling upgrade** по нодах, а не одночасний рестарт усього пулу.

### Champion/challenger та ретренінг (Просунуто)
- Не передеплоюйте challenger автоматично без порівняння: додайте умовний крок **Model Comparison / Run checks** перед `Make Active`.
- Ретренте на **drift-тригер**, а не лише за розкладом — це економить ресурси й реагує на реальну деградацію.

---

## 19. Типові помилки

- **Пушити «сирий» проєкт замість bundle.** Прямий project export ламає dev/prod-ізоляцію; для продакшену — лише версійовані bundles через Deployer.
- **Забути додати Saved Model / reference-dataset в Additions.** У проді модель/таблиця не перебудовуються — без них деплой «активується», але scoring падає.
- **Не налаштувати connection remapping.** Bundle вказує на `sql-dev` → прод читає/пише dev-дані (або падає fatal-error на відсутній connection).
- **Цільового connection немає на Automation node.** Remapping вимагає, щоб `sql-prod` **вже існував** — інакше активація провалюється.
- **Змінити налаштування deployment і не натиснути Update.** Стан **Out of sync**: Deployer показує одне, Automation node працює зі старим.
- **Чекати, що Python-рецептні API працюватимуть в API endpoint.** Код виконується на API node, не Design — доступ до даних лише через **query enrichment**, не через `dataiku.Dataset`.
- **Увімкнути explanations разом із Java scoring.** Вони **несумісні** — або швидкість (Java), або explanations.
- **Будувати batch і real-time на різних версіях моделі.** Використовуйте **одну Saved Model** і для Score recipe, і для prediction endpoint — інакше прогнози розходяться.
- **Моніторити лише Performance Drift.** Без ground truth він недоступний; без **Input/Prediction Drift** ви «сліпі» між поставками міток.
- **Намагатися покласти clustering / ensembling / partitioned модель у MES.** Не підтримується — drift на них так не порахуєш.
- **Ретренити сліпо за розкладом і робити нову версію active автоматично.** Без **Model Comparison / checks** можна промотувати гіршу модель у прод.
- **Plugin не встановлено на Automation/API node.** Bundle не везе plugins — установіть їх на цільовій ноді окремо.
- **Один API node без load balancer як «прод».** Немає HA: падіння ноди = простій. Мінімум 2 ноди за LB із `/isAlive/`-probe.

---

## 20. Перехресні посилання

- `01-overview-architecture.md` — типи нод (Design/Automation/API/Deployer/Govern), варіанти розгортання, Kubernetes/elastic AI.
- `02-projects-flow-core-concepts.md` — Flow, проєкти, project export vs bundle.
- `03-datasets-connections-partitioning.md` — connections (основа remapping), partitioned scoring, storage у проді.
- `06-coding-recipes-code-envs-notebooks.md` — code envs (remapping code env при деплої), Python/R для custom endpoints.
- `07-machine-learning-visual-ml.md` — Lab, Visual ML, тренування, що таке Saved Model і Training recipe.
- `09-scenarios-automation-metrics-checks.md` — scenarios, metrics, checks, reporters, тригери (база для auto-retrain і alerting).
- `11-programmatic-api-python-client.md` — повний `dataikuapi`: `DSSProjectDeployer`, `DSSSavedModel`, MES-класи, API node client.
- `12-dashboards-charts-webapps-reporting.md` — webapps у проді (deploy через bundle, Run-as User).
- `13-governance-security-collaboration.md` — Govern node, sign-off, blueprints, permissions propagation.
- `16-glossary.md` — терміни: bundle, deployment, infrastructure, stage, API service/endpoint, Saved Model, MES, drift, champion/challenger.

---

> Джерела: doc.dataiku.com/dss/latest (розділи `deployment/`, `apinode/`, `mlops/`, `machine-learning/`), knowledge.dataiku.com (MLOps & O16N), developer.dataiku.com (Python API, xOps tutorials). DSS 13.x/14.x, перевірено червень 2026.
