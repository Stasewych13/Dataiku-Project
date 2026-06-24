# 16. Глосарій термінів Dataiku

Цей документ — **довідник термінів**, що вживаються у базі знань (документи 01–15) та в офіційній документації Dataiku (`doc.dataiku.com`). Технічні терміни лишено в оригіналі **English** як заголовки; визначення — українською (1–3 речення) з посиланням на документ, де термін розкрито детальніше.

Глосарій згруповано за темами; всередині кожної групи — за алфавітом. Якщо термін стосується кількох тем, його наведено там, де він найбільш доречний; перехресні згадки позначено словом «див.».

> **Як читати посилання.** «див. `07-machine-learning-visual-ml.md`» означає документ, де тема розкрита детально. Скорочення «див. `07-...`» вживається, коли номер документа однозначний.

---

## Зміст

1. [Архітектура, ноди та інфраструктура](#1-архітектура-ноди-та-інфраструктура)
2. [Проєкти, Flow та базові об'єкти](#2-проєкти-flow-та-базові-обєкти)
3. [Datasets, connections, schema та partitioning](#3-datasets-connections-schema-та-partitioning)
4. [Visual recipes та обробка даних](#4-visual-recipes-та-обробка-даних)
5. [Підготовка даних: Prepare recipe, meanings, GREL](#5-підготовка-даних-prepare-recipe-meanings-grel)
6. [Code recipes, code environments, notebooks](#6-code-recipes-code-environments-notebooks)
7. [Machine Learning та Visual ML](#7-machine-learning-та-visual-ml)
8. [LLM Mesh та Generative AI](#8-llm-mesh-та-generative-ai)
9. [Scenarios, автоматизація, metrics та checks](#9-scenarios-автоматизація-metrics-та-checks)
10. [MLOps, deployment та API node](#10-mlops-deployment-та-api-node)
11. [Програмний доступ: Python client та REST API](#11-програмний-доступ-python-client-та-rest-api)
12. [Dashboards, charts, webapps та reporting](#12-dashboards-charts-webapps-та-reporting)
13. [Governance, security та collaboration](#13-governance-security-та-collaboration)
14. [Plugins, applications та розширюваність](#14-plugins-applications-та-розширюваність)

---

## 1. Архітектура, ноди та інфраструктура

**API node** — нода, що обслуговує **запити реального часу** (real-time scoring, прогнози, функції) через REST. Виконує API services, задеплоєні з Design node. Див. `01-overview-architecture.md`, `10-mlops-deployment-api-node.md`.

**Automation node** — продакшен-нода, на якій **виконуються** проєкти, доставлені з Design node у вигляді bundles, та запускаються scenarios за розкладом. Розробка тут не ведеться. Див. `01-...`, `10-...`.

**Cloud Stacks** — інструмент Dataiku для розгортання й керування **самостійно керованими** інстансами DSS у власному хмарному акаунті (AWS/Azure/GCP). Адмініструється через Fleet Manager. Див. `01-overview-architecture.md`.

**Dataiku Cloud** — повністю керований **SaaS**-варіант платформи, який хостить і обслуговує сама Dataiku; користувач не керує інфраструктурою. Протиставляється self-managed. Див. `01-...`.

**Deployer** — нода (або вбудований компонент), що **керує деплоями**. Поєднує **Project Deployer** (bundles → Automation) та **API Deployer** (packages → API node). Див. `01-...`, `10-...`.

**Design node** — основна нода, де користувачі **розробляють** проєкти: будують Flow, готують дані, тренують моделі, проєктують scenarios. Точка входу для більшості роботи. Див. `01-overview-architecture.md`, `02-projects-flow-core-concepts.md`.

**DSS (Data Science Studio)** — історична й технічна назва продукту Dataiku. У цій базі знань «Dataiku» і «DSS» позначають ту саму платформу (інстанс DSS). Див. `01-...`.

**DATADIR** — кореневий каталог даних інстансу DSS (конфіги, внутрішні бази, логи, дані managed-об'єктів). Усе, що робить інстанс, живе під DATADIR. Див. `01-overview-architecture.md` (розд. 4).

**Elastic AI computation** — механізм винесення обчислень (контейнеризованих рецептів, ML-тренувань, webapps) у **Kubernetes** (EKS/AKS/GKE) для масштабованості. Див. `01-...` (розд. 7).

**FEK (Future Execution Kernel)** — воркер-процес, якому backend делегує важку за пам'яттю роботу: побудову **семплів даних**, довгі фонові задачі. Той самий Java-застосунок, що й backend, але з іншим main-класом. Див. `01-...` (розд. 4.4).

**Fleet Manager** — інструмент керування «флотом» інстансів DSS у Cloud Stacks: створення, оновлення, масштабування, бекапи віртуальних машин. Див. `01-overview-architecture.md` (розд. 6.3).

**Govern node** — окрема нода для **нагляду й узгодження**: реєстр governable items, blueprints, sign-offs, Model Risk Management. Див. `01-...`, `13-governance-security-collaboration.md`.

**Instance** — одна інсталяція DSS (одна нода певного типу), до якої заходять браузером. Див. `01-...`.

**JEK (Job Execution Kernel)** — процес, що виконує конкретний **job** (те, що запускається кнопкою **Build** у Flow): рахує залежності, виконує рецепти на DSS engine, запускає дочірні процеси (Python, R, Spark, SQL). Один JEK на один job. Див. `01-...` (розд. 4.3).

**Node** — роль інстансу DSS: **Design**, **Automation**, **API**, **Deployer**, **Govern**. Кожна нода відповідає за свій етап lifecycle. Див. `01-overview-architecture.md`.

**Project Deployer** — компонент Deployer, що публікує **bundles** з Design node й деплоїть їх на **deployment infrastructures** (Automation nodes) через стадії dev/test/prod. Див. `10-mlops-deployment-api-node.md` (розд. 4).

**supervisord** — менеджер процесів верхнього рівня, що стартує, перезапускає та моніторить головні процеси інстансу (nginx, backend, jupyter). Запускається через `./bin/dss start`. Див. `01-...` (розд. 4.1).

---

## 2. Проєкти, Flow та базові об'єкти

**Bundle** — версійований архів проєкту, створений на Design node для деплою на Automation node (через Project Deployer). Містить метадані Flow, але **не** містить даних managed-датасетів за замовчуванням. Див. `02-...`, `10-mlops-deployment-api-node.md` (розд. 3).

**Discussion** — вбудований у проєкт/об'єкт **чат-тред** для обговорень із колегами, з нотифікаціями. Див. `02-projects-flow-core-concepts.md`, `13-...`.

**Flow** — візуальний **граф залежностей** датасетів і рецептів у межах проєкту; центральний робочий простір. Датасети — сині квадрати, рецепти — кружечки. Див. `02-projects-flow-core-concepts.md`.

**Flow zone** — логічна область усередині Flow для **групування** пов'язаних об'єктів (стадій обробки) задля читабельності великих пайплайнів. Див. `02-...`.

**Git integration** — вбудоване версіонування проєкту: кожна зміна комітиться в локальний Git; підтримуються revert, історія, а також зовнішні/remote Git-репозиторії для project libraries. Див. `02-projects-flow-core-concepts.md`.

**Insight** — одиниця візуального контенту (chart, метрика, текст, модель тощо), яку публікують як **tile** на dashboard. Див. `12-dashboards-charts-webapps-reporting.md`.

**Job** — одне виконання **build** у Flow: розрахунок залежностей і запуск потрібних рецептів для побудови вихідних датасетів/партицій. Виконується JEK. Див. `02-...` (розд. Build).

**Project** — фундаментальний контейнер усієї роботи: датасети, рецепти, Flow, моделі, dashboards, scenarios, wiki. Одиниця ізоляції та прав. Див. `02-projects-flow-core-concepts.md`.

**Project key** — унікальний незмінний ідентифікатор проєкту (великими літерами), що використовується в URL, API та коді. Див. `02-...`, `11-programmatic-api-python-client.md`.

**Recipe** — крок трансформації у Flow: бере вхідні датасети й створює вихідні. Буває **visual** (без коду) або **code** (Python/R/SQL/…). Див. `04-visual-recipes.md`, `06-coding-recipes-code-envs-notebooks.md`.

**Tag** — текстова мітка для класифікації об'єктів (датасетів, рецептів, проєктів) задля фільтрації й навігації. Див. `02-projects-flow-core-concepts.md`.

**To-do** — простий список завдань на рівні проєкту для відстеження роботи. Див. `02-...`.

**Variables (project / global / instance)** — іменовані значення трьох рівнів (scope): **instance/global** (адмінські, на весь інстанс), **project standard** (експортуються з bundles) та **project local** (специфічні для інстансу, не в bundles). Підставляються синтаксисом `${...}`. Див. `02-projects-flow-core-concepts.md` (розд. Variables), `11-...` (розд. 6).

**Wiki** — вбудована в проєкт **база знань** із markdown-сторінками для документації. Див. `02-...`, `13-governance-security-collaboration.md`.

---

## 3. Datasets, connections, schema та partitioning

**Connection** — налаштоване підключення до сховища даних (SQL-БД, S3, HDFS, GCS, Azure, тощо), яке визначає, де DSS читає/пише дані. Має credentials і permissions. Див. `03-datasets-connections-partitioning.md` (розд. 5).

**Dataset (managed / external)** — логічна таблиця записів зі схемою. **External** датасет указує на наявне джерело (DSS лише читає), **managed** — створюється й керується самим DSS (вихід рецепта). Див. `03-datasets-connections-partitioning.md` (розд. 1).

**Dimension (partitioning)** — вимір партиціонування: ознака, за якою датасет розбито на партиції (наприклад, дата або значення колонки). Див. `03-...` (розд. 8.2).

**Managed folder** — керована DSS «папка» для довільних файлів (моделі, зображення, артефакти), що не є табличним датасетом. Доступна з коду через `dataiku.Folder`. Див. `03-...`, `06-coding-recipes-code-envs-notebooks.md`, `11-...` (розд. 5.4).

**Meaning (semantic type)** — **семантичний** тип колонки («rich, flexible»): що дані означають (Email, Country, URL, тощо), на відміну від технічного storage type. Можна визначати власні meanings. Див. `03-...` (розд. 2.2), `05-data-preparation-prepare-recipe.md`.

**Partition** — окремий «шматок» партиціонованого датасету, що відповідає конкретному значенню виміру (наприклад, дані за один день). Дозволяє будувати/обробляти лише потрібні частини. Див. `03-datasets-connections-partitioning.md` (розд. 8).

**Partitioning** — поділ датасету на партиції за одним чи кількома вимірами для інкрементальної обробки та масштабування. Буває file-based та SQL. Див. `03-...` (розд. 8).

**Sampling** — робота не з усіма даними, а з **вибіркою** (семплом) для швидкого дизайну й перегляду; повний прогін — окремо. Методи: head, random, stratified тощо. Див. `03-...` (розд. 3), `05-...`.

**Schema** — впорядкований перелік колонок датасету з їхніми **storage types** та **meanings**. DSS може виводити (infer) схему автоматично. Див. `03-datasets-connections-partitioning.md` (розд. 2).

**Storage type** — **технічний** тип зберігання колонки («strict»): string, int, bigint, double, boolean, date тощо. Визначає, як значення зберігається фізично. Див. `03-...` (розд. 2.1).

**Streaming endpoint** — об'єкт для **потокових** (real-time) даних (Kafka, SQS тощо), що під'єднується до streaming-рецептів. Згадується серед об'єктів Flow. Див. `02-projects-flow-core-concepts.md`.

---

## 4. Visual recipes та обробка даних

**Distinct** — visual recipe, що прибирає дублікати рядків (унікалізація за всіма чи обраними колонками). Див. `04-visual-recipes.md`.

**Engine (DSS / SQL / Spark)** — обчислювальний рушій, на якому виконується рецепт: **DSS** (in-stream, на ноді), **SQL** (pushdown у БД), **Spark**, та інші. Вибір engine впливає на швидкість і масштаб. Див. `04-...` (розд. Рушії виконання).

**Group** — visual recipe для **агрегацій** (group by): рахує суми, середні, лічильники за групами ключів. Аналог `GROUP BY`. Див. `04-visual-recipes.md`.

**Join** — visual recipe, що **з'єднує** два чи більше датасетів за ключами (inner/left/right/outer); підтримує **fuzzy join** (нечітке зіставлення). Див. `04-...`.

**Pivot** — visual recipe, що **розгортає** значення колонки в окремі колонки (cross-tab), з агрегацією. Зворотна операція — unpivot/fold. Див. `04-...`.

**Pushdown** — стратегія виконання, коли DSS **транслює** логіку рецепта в нативні запити сховища (наприклад SQL у БД), щоб обчислення відбулися «там, де дані», без перекачування. Реалізується SQL/in-database engine. Див. `04-visual-recipes.md`, `06-...` (SQLExecutor2).

**Sort** — visual recipe для впорядкування рядків за обраними колонками. Див. `04-...`.

**Split** — visual recipe, що **розділяє** один вхідний датасет на кілька вихідних за правилами (значення, діапазони, відсотки). Див. `04-...`.

**Stack** — visual recipe, що об'єднує рядки кількох датасетів (union/append) у один. Див. `04-...`.

**Sync** — visual recipe, що **копіює** датасет «як є» в іншу connection/формат (наприклад, з файлу в БД), за потреби змінюючи storage. Див. `04-visual-recipes.md`.

**TopN** — visual recipe, що повертає **перші N** рядків за впорядкуванням у межах груп (top-N per group). Див. `04-...`.

**Visual recipe** — рецепт **без коду**, що налаштовується через UI (форми, правила). Протиставляється code recipe. Найпоширеніші: Sync, Prepare, Group, Join, Stack, Window тощо. Див. `04-visual-recipes.md`.

**Window** — visual recipe для **віконних функцій** (running totals, ranks, lag/lead, ковзні агрегати) з partition/order by. Див. `04-...`.

---

## 5. Підготовка даних: Prepare recipe, meanings, GREL

**Analyze box** — швидка панель статистики й інсайтів по колонці (розподіл, топ-значення, частка пропусків) із швидкими діями очищення. Див. `05-data-preparation-prepare-recipe.md`.

**Data quality bar** — кольорова смужка над колонкою (валідні / invalid / порожні значення) на основі meaning. Дає миттєву оцінку якості даних. Див. `05-...` (Color coding).

**Formula step** — крок Prepare recipe, що обчислює нову/змінену колонку за виразом мовою **GREL**. Див. `05-data-preparation-prepare-recipe.md` (Formula step).

**GREL / DSS formula language** — мова формул Dataiku (Dataiku Reusable Expression Language) для трансформацій у Formula step, фільтрах, обчислюваних колонках. Має функції для рядків, дат, чисел, масивів. Див. `05-...` (розд. GREL).

**Prepare recipe** — ключовий visual recipe для **очищення й трансформації** даних: послідовність **steps** із бібліотеки **processors** (parse dates, split, regex, filter, тощо). Аналог у Lab — Visual analysis. Див. `05-data-preparation-prepare-recipe.md`.

**Processor** — окрема операція у Prepare recipe (наприклад «Parse date», «Split column», «Filter rows»). Бібліотека процесорів — основний інструмент підготовки. Див. `05-...` (Processors library).

**Script (Prepare)** — впорядкована послідовність **steps** у Prepare recipe (або Visual analysis), що виконуються згори вниз. Див. `05-data-preparation-prepare-recipe.md`.

**Step** — один крок у Prepare-script: застосування одного процесора. (Не плутати зі **scenario step** — див. розд. 9.) Див. `05-...`.

**Validity** — статус значення відносно meaning колонки: valid / invalid / empty; керує підсвічуванням і обробкою «поганих» значень. Див. `05-data-preparation-prepare-recipe.md`.

> **Meaning** і **Storage type** визначено у розділі 3 (schema). У контексті Prepare їх детально розкрито в `05-...`.

---

## 6. Code recipes, code environments, notebooks

**Code env (code environment)** — ізольоване **Python/R-середовище** з фіксованим набором пакетів (pip/conda), що забезпечує відтворюваність і призначається рецептам, моделям, notebooks. Builtin-env не для продакшну. Див. `06-coding-recipes-code-envs-notebooks.md` (Частина 3).

**Code recipe** — рецепт, написаний кодом: **Python, R, SQL, Shell, Spark (PySpark/SparkR/SparkSQL/Scala), Hive, Impala**. Дає повну гнучкість там, де visual recipes не вистачає. Див. `06-coding-recipes-code-envs-notebooks.md` (Частина 1).

**Code Studio** — інтегроване IDE-середовище (на базі контейнера) у DSS для розробки коду, webapps та редагування project libraries з повноцінним редактором. Див. `06-...` (Code Studios), `12-...`, `14-...`.

**Containerized execution** — виконання code recipes / ML-тренувань / code envs у **Docker/Kubernetes**-контейнерах замість локального процесу ноди, для масштабу й ізоляції. Див. `06-coding-recipes-code-envs-notebooks.md`.

**Notebook** — інтерактивний **Jupyter**-блокнот усередині DSS для дослідницького коду; можна конвертувати notebook ↔ recipe. Див. `06-...` (Частина 4).

**Project library (lib/python, lib/R)** — каталог проєкту з власним кодом (модулі, функції), імпортовним у рецепти й notebooks; версіонується Git. Див. `06-coding-recipes-code-envs-notebooks.md` (Project libraries), `02-...`.

**SQLExecutor2** — клас Dataiku Python API для виконання **SQL із Python** з pushdown у БД (запити, читання результатів у DataFrame). Див. `06-...`, `11-programmatic-api-python-client.md`.

---

## 7. Machine Learning та Visual ML

**AutoML** — режим автоматизованого створення моделей: DSS сам підбирає попередню обробку, алгоритми та гіперпараметри за шаблонами (templates). Протиставляється Expert mode. Див. `07-machine-learning-visual-ml.md` (AutoML modes).

**Champion / Challenger** — практика порівняння активної моделі (**champion**) з кандидатами (**challengers**) перед заміною у проді; підтримується для версій Saved Model та A/B на API node. Див. `07-...`, `10-mlops-deployment-api-node.md` (розд. 13.3).

**Clustering** — задача **некерованого** навчання: групування записів у кластери без міток. Один із типів ML-задач у Lab. Див. `07-machine-learning-visual-ml.md`.

**Cross-validation** — оцінювання моделі на кількох розбиттях даних для надійнішої оцінки якості; застосовується й у hyperparameter search. Див. `07-...` (Search cross-validation).

**Drift** — зміщення розподілу вхідних даних або якості моделі з часом (data drift / prediction drift), що детектується через **Model Evaluation Store**. Див. `07-...`, `10-mlops-deployment-api-node.md` (розд. 14).

**Evaluate recipe** — рецепт, що **оцінює** модель на датасеті з відомими мітками й пише метрики та (опційно) результати у Model Evaluation Store. Див. `07-...` (Evaluate recipe), `10-...`.

**Feature handling** — налаштування **попередньої обробки ознак**: типи/ролі змінних, encoding категоріальних, rescaling числових, imputation пропусків, feature generation/reduction, text handling. Див. `07-machine-learning-visual-ml.md` (Feature handling).

**Hyperparameter search** — пошук найкращих гіперпараметрів алгоритму (grid/random/інше) з крос-валідацією. Див. `07-...` (Hyperparameter search).

**Lab** — простір **експериментів** у проєкті, де створюють Visual analysis (підготовка) та ML-моделі до винесення у Flow. Див. `07-machine-learning-visual-ml.md` (The Lab).

**Model Evaluation Store (MES)** — об'єкт Flow, що зберігає **історію оцінювань** моделі (метрики, drift) у часі для моніторингу. Заповнюється Evaluate recipe. Див. `07-...`, `10-mlops-deployment-api-node.md` (розд. 14.1).

**Prediction** — задача **керованого** навчання з міткою: регресія (число), бінарна та мультикласова класифікація. Один із типів ML-задач. Див. `07-machine-learning-visual-ml.md`.

**Saved Model** — версійований об'єкт **навченої моделі** у Flow, отриманий з Lab («Deploy») або імпортом (MLflow). Має active version, історію, метрики. Використовується для scoring. Див. `07-...` (Saved Model), `10-...`.

**Score recipe** — рецепт, що застосовує **Saved Model** до датасету й додає колонки з прогнозами (батч-scoring). Див. `07-machine-learning-visual-ml.md` (Score recipe), `10-...`.

**Scoring** — застосування навченої моделі до нових даних для отримання прогнозів; буває **batch** (Score recipe) і **real-time** (API node). Див. `07-...`, `10-mlops-deployment-api-node.md` (розд. 12).

**Shapley values** — метод інтерпретації, що приписує кожній ознаці внесок у конкретний прогноз (на основі теорії ігор). Доступний серед пояснень моделі. Див. `07-machine-learning-visual-ml.md` (Shapley values).

**Subpopulation analysis** — аналіз поведінки/якості моделі на **підгрупах** даних (за значенням ознаки) задля виявлення нерівномірності та можливих bias. Див. `07-...` (Subpopulation analysis).

**Visual analysis** — об'єкт у Lab, що поєднує інтерактивну підготовку даних (script зі steps) і дизайн ML-моделей перед деплоєм у Flow. Див. `07-machine-learning-visual-ml.md`.

**Visual ML** — створення ML-моделей через UI (Lab) без коду, з повним контролем над feature handling, алгоритмами та інтерпретацією. Див. `07-machine-learning-visual-ml.md`.

---

## 8. LLM Mesh та Generative AI

**Agent** — система на базі LLM, що **планує й виконує дії** через інструменти (tools), а не лише генерує текст. У Dataiku — visual/code agents, Agent Tools, Agent Connect/Hub. Див. `08-llm-generative-ai-mesh.md` (розд. 8).

**Dataiku Answers** — готовий **корпоративний чат-застосунок** (out-of-the-box) поверх LLM Mesh з RAG, історією, посиланнями на джерела та адміністративним контролем. Див. `08-...` (розд. 9), `12-...`.

**Embedding** — числовий векторний вимір тексту/документа, що кодує його зміст; основа семантичного пошуку та RAG. Створюється Embed-рецептами. Див. `08-llm-generative-ai-mesh.md` (розд. 6.1).

**Guardrails** — захисні механізми навколо LLM: фільтрація PII, moderation, обмеження тем, перевірка відповідей; частина Advanced LLM Mesh. Див. `08-...` (розд. 12), `13-governance-security-collaboration.md` (розд. 14.5).

**Knowledge Bank** — об'єкт Dataiku, що зберігає **проіндексовані ембедінги** документів (vector store) для retrieval у RAG. Згадується серед зелених ML-об'єктів Flow. Див. `08-llm-generative-ai-mesh.md` (розд. 6.2), `02-...`.

**LLM connection** — налаштоване підключення до провайдера/моделі LLM (OpenAI, Azure, Bedrock, локальні тощо) у межах LLM Mesh, з контролем доступу та витрат. Див. `08-...` (розд. 3).

**LLM Mesh** — **абстракційний шар** Dataiku між застосунками й багатьма LLM-провайдерами: єдиний API, контроль доступу, безпеки, витрат і аудиту («virtual LLM»). Див. `08-llm-generative-ai-mesh.md` (розд. 2).

**Prompt Studio** — інтерактивне середовище для **розробки й тестування промптів** на різних LLM перед збереженням у Prompt recipe. Див. `08-...` (розд. 4.1).

**RAG (Retrieval-Augmented Generation)** — підхід, коли LLM відповідає, спираючись на **знайдені** релевантні документи з vector store, а не лише на параметри моделі. Реалізується через Knowledge Bank + retrieval. Див. `08-llm-generative-ai-mesh.md` (розд. 6.4).

**Vector store** — спеціалізоване сховище **ембедінг-векторів** із семантичним пошуком (наприклад FAISS, pgvector, ChromaDB); фізичний бекенд Knowledge Bank. Див. `08-...` (розд. 6.3).

---

## 9. Scenarios, автоматизація, metrics та checks

**Check** — перевірка, що **порівнює metric з очікуваним порогом** і дає статус (OK/Warning/Error); може зупиняти scenario. Див. `09-scenarios-automation-metrics-checks.md` (Checks).

**Data Quality rules** — сучасний механізм правил **якості даних** на рівні датасету (типи rules, hard/soft межі, change-detection, auto-run after build) з окремими dashboards моніторингу. Частково перекриває класичні metrics/checks. Див. `09-...` (Data Quality).

**Metric** — числове **вимірювання** про датасет/модель/folder (наприклад COUNT_RECORDS, % пропусків). Обчислюється probes; історія зберігається. Див. `09-scenarios-automation-metrics-checks.md` (Metrics).

**Probe** — джерело metric: вбудована probe (record count, column stats) або кастомна (**SQL probe**, **Python probe**). Див. `09-...` (Custom metrics).

**Reporter** — компонент scenario, що **надсилає нотифікації** про перебіг (email, Slack, Teams, webhook, тощо) за шаблонами повідомлень. Див. `09-scenarios-automation-metrics-checks.md` (Reporters).

**Scenario** — об'єкт **автоматизації**: послідовність steps (або custom Python), що запускається triggers й оркеструє build, перевірки, нотифікації. Див. `09-scenarios-automation-metrics-checks.md`.

**Step (scenario)** — окремий крок scenario (build датасета, run checks, set variables, execute SQL/code, тощо) із run conditions та error handling. (Не плутати з Prepare step — розд. 5.) Див. `09-...` (Scenario steps).

**Trigger** — умова **запуску** scenario: за розкладом (time-based), при зміні датасета, за SQL-умовою, через API/custom. Має evaluation interval і grace delay. Див. `09-scenarios-automation-metrics-checks.md` (Triggers).

---

## 10. MLOps, deployment та API node

**A/B testing (endpoints)** — розподіл трафіку між версіями моделі на API node для порівняння в проді. Див. `10-mlops-deployment-api-node.md` (розд. 13.4).

**API Deployer** — компонент Deployer, що деплоїть **API packages** (версії API service) на API node infrastructures (static або Kubernetes). Див. `10-...` (розд. 8.3).

**API service** — сервіс реального часу, що публікується на API node й містить один або кілька **endpoints**; версіонується через packages. Див. `10-mlops-deployment-api-node.md` (розд. 8–9).

**Batch vs real-time scoring** — два режими застосування моделі: **batch** (масово, через Score recipe у Flow) та **real-time** (поодинокі запити з низькою затримкою через API node). Див. `10-...` (розд. 12).

**Deployment infrastructure** — цільове середовище деплою (набір Automation nodes для bundles або API nodes для packages), згруповане за стадіями dev/test/prod з permissions. Див. `10-mlops-deployment-api-node.md` (розд. 4.2).

**Endpoint** — одиниця API service, що відповідає на конкретний тип запиту: prediction (Saved Model), Python/R prediction, Python/R function, SQL query, dataset lookup. Див. `10-...` (розд. 9).

**Ground truth / feedback loop** — збір реальних результатів (фактичних міток) після прогнозу для моніторингу якості й перетренування моделі. Див. `10-mlops-deployment-api-node.md` (розд. 15).

**MLOps** — практики й інструменти для **деплою, моніторингу та супроводу** ML-моделей у проді: bundles, deployer, MES, drift, ground truth. Тема всього `10-mlops-deployment-api-node.md`.

**Package (API)** — версійований архів API service для деплою на API node (аналог bundle, але для real-time). Див. `10-...` (розд. 8.2).

**Published project** — копія проєкту в Project Deployer, що накопичує версії bundles; це **не** Design-проєкт. Звідти bundles активуються на інфраструктурах. Див. `10-mlops-deployment-api-node.md` (розд. 4.1).

**Remapping** — заміна налаштувань при переході між середовищами (dev→prod): **connection remapping**, **code env remapping**, override variables, scenario triggers. Див. `10-...` (розд. 5).

**Rollback** — повернення активного bundle/версії моделі до попередньої робочої. Див. `10-mlops-deployment-api-node.md` (розд. 6).

---

## 11. Програмний доступ: Python client та REST API

**`dataiku` (package)** — **internal** Python API, що працює **зсередини** DSS (рецепти, notebooks): `Dataset`, `Folder`, доступ до змінних із контексту, читання/запис даних. Без ключа. Див. `11-programmatic-api-python-client.md` (розд. 2).

**`dataikuapi` (package)** — Python-пакет для роботи **ззовні** через REST: основний клас — `DSSClient`. Потребує URL і API key. Див. `11-...` (розд. 3).

**DSSClient** — головний клас `dataikuapi` для підключення до інстансу й керування проєктами, датасетами, scenarios, users тощо через REST API. Див. `11-programmatic-api-python-client.md` (розд. 3).

**dsscli / dssadmin** — командні утиліти інстансу DSS для адміністративних операцій (керування, бекапи, конфіги) з боку сервера. Див. `11-...`, `01-overview-architecture.md`.

**Global API key** — ключ **рівня інстансу** з адмінськими можливостями (керування користувачами, інстанс-змінними тощо); створюється адміністратором. Див. `11-programmatic-api-python-client.md` (розд. 4.1).

**Personal API key** — ключ, прив'язаний до **конкретного користувача**, що діє з його правами; для більшості програмних інтеграцій від імені користувача. Див. `11-...` (розд. 4.1).

**Public API** — публічний **REST API** інстансу DSS, поверх якого працює `dataikuapi`; покриває проєкти, об'єкти Flow, scenarios, адміністрування. Див. `11-programmatic-api-python-client.md`.

---

## 12. Dashboards, charts, webapps та reporting

**Aggregation** — спосіб згортки measure у chart (sum, avg, count, min/max, тощо) за dimensions. Див. `12-dashboards-charts-webapps-reporting.md`.

**Binning** — групування неперервних значень (дат або чисел) у діапазони/інтервали для побудови chart. Див. `12-...` (Binning).

**Chart** — візуалізація даних датасета (bar, line, scatter, map, pivot table, тощо), сконфігурована за dimensions/measures. Публікується як insight на dashboard. Див. `12-dashboards-charts-webapps-reporting.md`.

**Dashboard** — сторінка з **tiles** (charts, метрики, текст, моделі) для перегляду результатів; має власні permissions та фільтри. Див. `12-...` (Dashboards).

**Dimension (chart)** — категоріальна або часова вісь, за якою розкладаються measures у chart. (Не плутати з partitioning dimension — розд. 3.) Див. `12-dashboards-charts-webapps-reporting.md`.

**Measure** — числова величина у chart, яку агрегують за dimensions. Див. `12-...`.

**Reporting** — доставка результатів клієнтам/менеджменту: експорт dashboard у PDF/images, scheduled-звіти через scenario, експорт даних. Див. `12-dashboards-charts-webapps-reporting.md` (Reporting).

**Tile** — елемент dashboard. Буває simple (текст, image, iframe) або **insight tile** (один опублікований insight = один tile). Див. `12-...` (Tiles).

**Webapp (Standard / Shiny / Bokeh / Dash / Streamlit)** — інтерактивний застосунок усередині DSS: **Standard** (HTML/JS + Python/R backend), а також **Shiny**, **Bokeh**, **Dash**, **Streamlit**. Має run-as-user безпеку й доступ до даних DSS. Розробляється, зокрема, у Code Studios. Див. `12-dashboards-charts-webapps-reporting.md` (Webapps).

---

## 13. Governance, security та collaboration

**Audit** — журналювання дій і подій в інстансі (хто що зробив, login, доступ до даних) для безпеки й комплаєнсу; централізується EventServer. Див. `13-governance-security-collaboration.md` (розд. 10).

**Blueprint** — у Govern: **шаблон** governable item (поля, статуси, workflow), за яким реєструють і ведуть об'єкти (проєкти, моделі, bundles). Див. `13-...` (розд. 11.4).

**Compute Resource Usage (CRU)** — аудит **споживання обчислювальних ресурсів** (jobs, контейнери) для моніторингу й costing. Див. `13-governance-security-collaboration.md` (розд. 10.3).

**EventServer** — центральний компонент **збору audit-логів** з кількох нод в одне місце для аналізу й retention. Див. `13-...` (розд. 10.2).

**Exposed / Shared object** — об'єкт (датасет, модель, folder, тощо), **наданий** іншому проєкту чи workspace для повторного використання без копіювання. Розрізняють кілька видів «sharing». Див. `13-governance-security-collaboration.md` (розд. 8), `12-...`.

**Governable item** — об'єкт, зареєстрований у Govern node (проєкт, Saved Model, bundle, тощо) для нагляду, sign-offs і ризик-менеджменту. Див. `13-...` (розд. 11.2).

**Group** — основна одиниця **прав доступу**: користувачі належать групам, групам надають global/project/connection permissions (права кумулятивні). Див. `13-governance-security-collaboration.md` (розд. 3, 7).

**Model Risk Management (MRM)** — процес у Govern для **оцінки й контролю ризиків** ML-моделей (документація, sign-offs, статуси). Див. `13-...` (розд. 13).

**Permission** — конкретне право (read/write/admin, запуск scenarios, використання connection, тощо), що надається групі на global/project/connection рівні. Див. `13-governance-security-collaboration.md` (розд. 7).

**Sign-off** — формальне **узгодження/затвердження** governable item (наприклад, перед деплоєм) визначеними ролями; може бути gate для deployment. Див. `13-...` (розд. 12).

**SSO (Single Sign-On)** — єдиний вхід через зовнішнього провайдера ідентичності (**SAML**, **OIDC/OAuth2**); з SCIM — автоматичне provisioning користувачів. Див. `13-governance-security-collaboration.md` (розд. 5).

**UIF (User Isolation Framework)** — механізм **ізоляції користувачів** на рівні ОС/compute: код і доступ до даних виконуються від імені реального користувача (impersonation), а не сервісного акаунта. Див. `13-...` (розд. 9).

**User** — обліковий запис у DSS (local або з LDAP/AD/SSO), що належить групам, від яких успадковує права. Див. `13-governance-security-collaboration.md` (розд. 3).

**Workspace** — простір для **колаборації та шерингу** обраних об'єктів між командами поза межами одного проєкту. Див. `13-...` (розд. 8.2).

---

## 14. Plugins, applications та розширюваність

**Application-as-recipe** — спосіб «упакувати» цілий **Dataiku Application** як один **рецепт** у Flow іншого проєкту (вхід/вихід + параметри), для перевикористання пайплайну. Див. `14-plugins-applications-extensibility.md` (Частина 8).

**Dataiku Application** — «упакований» проєкт із параметрами та спрощеним UI (Application Designer), який можна **інстанціювати** як шаблон для нових кейсів. Має app instances і versioning. Див. `14-...` (Частина 7).

**Macro** — plugin-компонент (runnable), що виконує **адміністративну/допоміжну дію** (наприклад, cleanup) поза стандартним рецептом, через UI чи scenario. Див. `14-plugins-applications-extensibility.md` (Частина 6).

**Parameter set** — іменований набір параметрів plugin-компонента (наприклад, набір налаштувань підключення), з якого створюють **presets**. Див. `14-...` (Частина 5).

**Plugin** — пакет розширення DSS, що додає нові **компоненти** (рецепти, датасети, процесори, macros, тощо); встановлюється зі Store, zip або Git. Див. `14-plugins-applications-extensibility.md` (Частина 1).

**Plugin component** — окремий елемент plugin: custom recipe, custom dataset, processor, macro, webapp, parameter set тощо. Один plugin містить кілька компонентів. Див. `14-...` (Частина 3).

**Preset** — конкретний, заздалегідь заповнений набір значень параметрів (екземпляр parameter set), що його обирають при використанні компонента; presets можуть бути глобальними чи per-user. Див. `14-plugins-applications-extensibility.md` (Частина 5).

---

## Підсумок

Цей глосарій **доповнює** змістові документи бази знань і служить точкою швидкого пошуку визначень. За детальним розкриттям кожного терміна звертайтеся до відповідного документа:

- `01-overview-architecture.md` — ноди, інфраструктура, lifecycle, deployment-варіанти.
- `02-projects-flow-core-concepts.md` — проєкти, Flow, variables, колаборація, Git.
- `03-datasets-connections-partitioning.md` — datasets, connections, schema, partitioning.
- `04-visual-recipes.md` — visual recipes та engines.
- `05-data-preparation-prepare-recipe.md` — Prepare recipe, processors, meanings, GREL.
- `06-coding-recipes-code-envs-notebooks.md` — code recipes, code envs, notebooks, libraries.
- `07-machine-learning-visual-ml.md` — Visual ML, AutoML, інтерпретація, scoring.
- `08-llm-generative-ai-mesh.md` — LLM Mesh, RAG, агенти, Dataiku Answers, guardrails.
- `09-scenarios-automation-metrics-checks.md` — scenarios, triggers, metrics, checks, data quality.
- `10-mlops-deployment-api-node.md` — bundles, deployers, API node, drift, scoring.
- `11-programmatic-api-python-client.md` — `dataiku`/`dataikuapi`, DSSClient, API keys.
- `12-dashboards-charts-webapps-reporting.md` — charts, dashboards, webapps, reporting.
- `13-governance-security-collaboration.md` — users/groups, permissions, UIF, audit, Govern.
- `14-plugins-applications-extensibility.md` — plugins, presets, applications, application-as-recipe.

> Терміни, що не ввійшли сюди як окремі статті, зазвичай розкрито у профільному документі; шукайте за оригінальною English-назвою у відповідному файлі.
