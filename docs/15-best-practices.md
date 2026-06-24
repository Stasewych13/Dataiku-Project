# 15. Зведені Best Practices

> Наскрізний документ бази знань. Синтезує найважливіші рекомендації з документів **01–14** в одну впорядковану за темами шпаргалку та додає нове за офіційними джерелами (`knowledge.dataiku.com`, `doc.dataiku.com`). Орієнтовано на **DSS 13.x / 14.x (2025–2026)**. Технічні, UI- та code-терміни залишено в **оригіналі (English)**; пояснення — українською.
>
> Кожна практика подана за формулою **чому → як**, з рівнем складності **(Базово)** / **(Середній)** / **(Просунуто)** і посиланням на детальний документ. Це not-replacement, а навігатор: за глибиною йдіть у відповідний розділ.

---

## Зміст

1. [Як читати цей документ](#1-як-читати-цей-документ)
2. [Організація проєктів і Flow](#2-організація-проєктів-і-flow)
3. [Управління даними](#3-управління-даними)
4. [Продуктивність та обчислення](#4-продуктивність-та-обчислення)
5. [Підготовка даних](#5-підготовка-даних)
6. [Код і відтворюваність](#6-код-і-відтворюваність)
7. [Machine Learning](#7-machine-learning)
8. [Автоматизація](#8-автоматизація)
9. [MLOps та середовища](#9-mlops-та-середовища)
10. [Governance та безпека](#10-governance-та-безпека)
11. [Колаборація](#11-колаборація)
12. [LLM / GenAI](#12-llm--genai)
13. [Антипатерни: чеклист «чого не робити»](#13-антипатерни-чеклист-чого-не-робити)
14. [Чеклист готовності проєкту до продакшну](#14-чеклист-готовності-проєкту-до-продакшну)
15. [Перехресні посилання](#15-перехресні-посилання)
16. [Джерела](#16-джерела)

---

## 1. Як читати цей документ

Документ організовано **за темами**, а не за рівнями. У межах теми практики йдуть від базових до просунутих. Рівень у дужках — це орієнтир, кому практика першочергово адресована:

- **(Базово)** — застосовуйте з першого дня, навіть на навчальному інстансі.
- **(Середній)** — для командної роботи та проєктів, які підуть далі прототипу.
- **(Просунуто)** — для продакшну, масштабу, регульованих індустрій.

> **Принцип над усіма принципами:** Dataiku — це **layout + governance над обчисленнями, які виконуються деінде** (SQL, Spark, K8s, LLM-провайдери). Майже всі best practices зводяться до трьох ідей: (1) **тримайте дані й обчислення близько одне до одного** (push-down), (2) **робіть кроки відтворюваними та ідемпотентними**, (3) **розділяйте dev і prod**.

Чотири наскрізні теми, що повторюються в усіх документах і яких варто триматися завжди:

| Тема | Суть | Де поглибити |
|---|---|---|
| **Push-down** | Не тягніть дані на DSS server без потреби — рахуйте в SQL/Spark/БД | §4, [03](03-datasets-connections-partitioning.md), [04](04-visual-recipes.md), [06](06-coding-recipes-code-envs-notebooks.md) |
| **Ідемпотентність** | Той самий вхід → той самий вихід; overwrite, а не сліпий append | §5, §8, [09](09-scenarios-automation-metrics-checks.md) |
| **Dev/prod separation** | Окремі ноди, connections, credentials; деплой лише через bundles | §9, §10, [10](10-mlops-deployment-api-node.md), [13](13-governance-security-collaboration.md) |
| **Least privilege & no secrets in code** | Права через групи, секрети через presets/User Secrets | §6, §10, [13](13-governance-security-collaboration.md) |

---

## 2. Організація проєктів і Flow

**Flow zones — діліть великий Flow за фазами обробки.**
*Чому:* складний Flow з десятками datasets/recipes стає нечитабельним; zones відображають фази й ізолюють експериментальні гілки. *Як:* діліть на **ingestion → preparation → modeling → output**; виносьте experiments в окрему зону. **(Середній)** ⟶ деталі: [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md).
> **Пастка:** zones — це **layout, не security**. Розмежування доступу робиться правами на проєкт, а не зонами.

**Naming conventions — `foo_raw` / `foo_clean`.**
*Чому:* добре названі datasets — найважливіший елемент колаборації: легше повернутися до своєї роботи й зрозуміти чужу. *Як:* імена **читабельні, самопояснювальні, короткі**; акцент на призначенні (`foo_raw` → `foo_clean` → `foo_agg`); опційно суфікс сховища (`foo_t` для SQL, `foo_hdfs`). Лише **lowercase, alphanumerics та underscore**. Project name — повний і явний («Data Ingestion», а не «p001_di»). **(Базово)** ⟶ [02](02-projects-flow-core-concepts.md).

**Tags — послідовно й з кольором.**
*Чому:* теги дають «з першого погляду» зрозуміти роль кожної частини Flow. *Як:* тегуйте за задачею, статусом (`wip`/`done`/`in production`), власником (ім'я людини), джерелами; уніфікуйте через **Global tag categories** (адмін); використовуйте **колір** для акценту. **(Середній)** ⟶ [02](02-projects-flow-core-concepts.md).

**Документація — wiki + descriptions + discussions.**
*Чому:* добре задокументований workflow забезпечує відтворюваність, спрощує підтримку й передачу. *Як:* ведіть **project wiki** (Markdown); заповнюйте **object descriptions** (Summary tab); тримайте обговорення в **discussions, прив'язаних до об'єкта**, а не в email. **(Базово → Середній)** ⟶ [13-governance-security-collaboration.md](13-governance-security-collaboration.md).

**Структура — продумайте на старті.**
*Чому:* `project key` **immutable**; перебудова zones постфактум — біль. *Як:* визначте project key, межі зон і naming-схему **до** першого dataset. **(Середній)** ⟶ [02](02-projects-flow-core-concepts.md).

**Build mode — Smart reconstruction за замовчуванням.**
*Чому:* force-recursive build — найважчий режим; зазвичай надлишковий. *Як:* лишайте **Smart reconstruction** як дефолт (перебудовує лише застарілі датасети); force-recursive — лише коли свідомо потрібно повністю перебудувати гілку. Пам'ятайте: revert у Git повертає **лише metadata**, не дані — після revert потрібен rebuild. **(Середній)** ⟶ [02](02-projects-flow-core-concepts.md).

**Variables — standard vs local; secrets завжди в local.**
*Чому:* standard variables **експортуються з bundle**, тож секрет у них витече в прод-артефакт. *Як:* конфіг — у standard variables; секрети/токени — у **local** variables; не плутайте project-level з instance-level global variables. **(Середній)** ⟶ [02](02-projects-flow-core-concepts.md).

---

## 3. Управління даними

**Обирайте connection під етап життя проєкту.**
*Чому:* uploaded files зручні для прототипу, але не масштабуються; SQL/cloud дають push-down. *Як:* прототип — uploaded/filesystem; продакшн — **SQL або cloud storage**. Тримайте дані якомога ближче до обчислень. **(Базово → Середній)** ⟶ [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md).

**Parquet (або ORC) для великих даних, не CSV.**
*Чому:* columnar + стиснення + predicate pushdown дають прискорення до **×100**; CSV не пушиться й роздуває обсяг. *Як:* для managed-датасетів на Hadoop/cloud — **Parquet**; CSV лишайте для import/export/обміну. Дотримуйтесь **lowercase identifiers** (Parquet/Hive). **(Середній)** ⟶ [03](03-datasets-connections-partitioning.md).

**Partitioning — за потреби, без over-partitioning.**
*Чому:* партиціонування за **time** дає інкрементальні білди; але дрібні partitions → тисячі файлів і повільний listing. *Як:* time-партиціонування для логів/подій; discrete — для розбивки по категоріях (country/tenant); використовуйте **Build required datasets** + Time range / Equals залежності. Не дробіть надміру. **(Просунуто)** ⟶ [03](03-datasets-connections-partitioning.md).
> **Пастка:** SQL time-partition column має бути **string** у DSS-синтаксисі, не Date.

**Не дублюйте дані.**
*Чому:* зайві managed-копії = більше сховища, складніший lineage, ризик розсинхрону. *Як:* якщо рецепт лише читає — лишайте дані в джерелі; для незмінних reference-таблиць — **external dataset** замість копіювання; налаштуйте **naming rules** на connections, щоб managed datasets різних проєктів не перекрилися. **(Середній)** ⟶ [03](03-datasets-connections-partitioning.md).

**Sampling під час дизайну — репрезентативний, не «перші рядки».**
*Чому:* усе в Explore/prep — це **sample**; `First records` на відсортованих даних дає biased вибірку. *Як:* для рідкісних значень — **Random / Stratified**; тримайте зразок ≤ **200 000** записів; перед продакшном перевіряйте на **Whole data**. **(Середній)** ⟶ [03](03-datasets-connections-partitioning.md), [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md).

**Окремі connections для dev / test / prod.**
*Чому:* спільний connection між середовищами = ризик прочитати/записати не ті дані під час bundle remapping. *Як:* заведіть `sql-dev` / `sql-test` / `sql-prod` (та аналоги для cloud storage) і керуйте доступом через **groups**; не покладайтеся на «той самий ім'ям» — налаштовуйте explicit remapping при деплої (див. §9). **(Середній)** ⟶ [03](03-datasets-connections-partitioning.md), [10](10-mlops-deployment-api-node.md).

**Свідомий режим запису output: drop & recreate vs append.**
*Чому:* неконтрольований append ламає ідемпотентність і дублює дані при ретраї. *Як:* для відтворюваних трансформацій — **drop & recreate**; для логів/інкременту — append із чітким partition spec. **(Середній)** ⟶ [04](04-visual-recipes.md), [09](09-scenarios-automation-metrics-checks.md).

---

## 4. Продуктивність та обчислення

**Push-down у SQL/Spark — головна оптимізація DSS.**
*Чому:* DSS-stream engine тягне дані на сервер; SQL/Spark виконують in-database/in-cluster без переміщення. *Як:* тримайте проміжні датасети **в тій самій БД**, щоб ланцюг рецептів виконувався in-database; **перевіряйте індикатор engine** під Run — не дайте Flow тихо впасти на DSS-stream. **(Просунуто)** ⟶ [04-visual-recipes.md](04-visual-recipes.md).
> **Тиха деградація:** один несумісний крок (різні connections, непушабельна функція) дискваліфікує весь рецепт із SQL-engine — без помилки, лише повільніше. Завжди звіряйте фактичний engine.

**Вибір recipe engine — visual спершу, code за потреби.**
*Чому:* visual recipes автоматично обирають найкращий доступний engine; код важче оптимізувати й тестувати. *Як:* виражайте логіку visual recipes; до **code recipe** переходьте, лише коли логіка не виражається інакше. **(Середній)** ⟶ [04](04-visual-recipes.md), [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md).

**Фільтруйте рано.**
*Чому:* обробляти зайві рядки дорого. *Як:* pre-filter у Group/Join/Window та Sample/Filter **до** важких операцій, а не після. **(Середній)** ⟶ [04](04-visual-recipes.md).

**Не вантажте все у pandas.**
*Чому:* збирання великого датасета в DataFrame на DSS server → **OOM**. *Як:* для агрегацій/фільтрів — pushdown у БД через `SQLExecutor2.query_to_df()` або SQL-рецепт; для великих обсягів — `iter_dataframes(chunksize=...)` + `get_writer()` (chunked read/write). **(Середній)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md).
> **Пастка API:** параметра `get_dataframe(chunksize=...)` **не існує**; chunked — це `iter_dataframes(chunksize=...)`.

**Containerized execution для важких рецептів.**
*Чому:* ізоляція ресурсів і еластичне масштабування на Kubernetes; гасіть кластери за розкладом заради економії. *Як:* виносьте важкі Python/ML-рецепти в containerized execution; **не використовуйте non-managed code envs у контейнерах**. **(Просунуто)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md), [01-overview-architecture.md](01-overview-architecture.md).

**Детермінізм у visual recipes.**
*Чому:* SQL-таблиця **не гарантує порядок** без `ORDER BY`; TopN/Window без явного впорядкування дають недетермінований результат між запусками. *Як:* у TopN додавайте tie-breaker в order; для Window на DSS engine задавайте **frame явно** (DSS бере весь frame за замовчуванням — семантика інша за SQL); fuzzy join на великих даних блокуйте кандидатів заздалегідь (тільки DSS engine, дорогий). **(Середній → Просунуто)** ⟶ [04](04-visual-recipes.md).

---

## 5. Підготовка даних

**Ідемпотентні Prepare-кроки.**
*Чому:* той самий вхід має давати той самий вихід — інакше білди недетерміновані. *Як:* уникайте `now()` без потреби; межі IQR від Analyze **заморожуються як константи** (для стабільності — явний `Force numerical range` або формула); продумайте порядок кроків (`diff()` лише **після** парсингу дати). **(Середній)** ⟶ [05](05-data-preparation-prepare-recipe.md).

**Спирайтеся на meanings, не на ручні фільтри.**
*Чому:* правильний meaning активує підказки, валідацію та `Filter invalid rows/cells`. *Як:* призначайте коректний meaning; для бізнес-словників — **user-defined meanings** (Values list / Pattern). **Не вмикайте auto-detection** для user-defined meanings — це вимикає built-in meanings й гальмує. **(Середній)** ⟶ [05](05-data-preparation-prepare-recipe.md), [03](03-datasets-connections-partitioning.md).

**Проєктуйте під push-down.**
*Чому:* один непушабельний крок дискваліфікує весь рецепт із SQL-engine. *Як:* групуйте SQL-пушабельні кроки разом; непушабельні (складні Formula, regex-витяг, geo) виносьте в **окремий рецепт нижче**; перевіряйте колір SQL-іконки; для конкатенації — `concat()`, не `+`. **(Просунуто)** ⟶ [05](05-data-preparation-prepare-recipe.md).

**Валідація схеми.**
*Чому:* mistyped storage type ламає jobs при переході SQL↔Spark↔Hadoop. *Як:* свідомо задавайте storage types; перевіряйте edge-кейси на повних даних через Analyze (Whole data); документуйте кроки (коментарі, осмислені назви, логічні блоки). **(Середній)** ⟶ [05](05-data-preparation-prepare-recipe.md), [03](03-datasets-connections-partitioning.md).
> **Не винаходьте функцій:** у DSS-формулах НЕ існує `incrementDate`, `dateDiff`, `parseDate`, `year()`, `capitalize()`. Правильно: `inc`, `diff`, `asDateOnly`, `datePart`, `toTitlecase`. Повний список — у [05](05-data-preparation-prepare-recipe.md).

**Повторне використання та дебаг.**
*Чому:* копіювати логіку між рецептами — множити помилки; видаляти кроки під час дебагу — втрачати їх. *Як:* copy/paste steps між Prepare-рецептами; у Lab — **Deploy Script**; для повторюваної логіки — власний плагін; під час дебагу кроки **вимикайте**, а не видаляйте. **(Середній)** ⟶ [05](05-data-preparation-prepare-recipe.md), [14](14-plugins-applications-extensibility.md).

---

## 6. Код і відтворюваність

**Власні code environments, не builtin.**
*Чому:* встановлення пакетів у builtin (base) env впливає на **весь instance**; пінінг версій дає відтворюваність. *Як:* окремий managed env під проєкт/задачу; пінте версії (`pkg==x.y.z`); звіряйте **Actually installed packages**; **перебудовуйте base images після кожного апгрейду DSS**. **(Базово)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md).

**Project libraries замість copy-paste.**
*Чому:* копіювання коду між рецептами множить помилки. *Як:* спільну логіку — у `lib/python` (`lib/R`), імпортуйте у рецептах; тестуйте її окремо. **(Середній)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md).

**Notebooks — для дослідження, recipes — для продакшну.**
*Чому:* scenario **не** запустить notebook; код у Flow має бути рецептом. *Як:* конвертуйте notebook → recipe (**Create Recipe**), щоб логіка потрапила у Flow і scenarios. **(Базово)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md).

**Секрети — ніколи в коді.**
*Чому:* хардкод credentials у recipe/notebook = **витік у Git проєкта**. *Як:* зсередини DSS — `dataiku.api_client()` (контекстна автентифікація) або **User Secrets** через `client.get_auth_info(with_secrets=True)`; для БД — використовуйте **connection**, а не власний рядок підключення; у plugins — `PASSWORD`/`CREDENTIAL_REQUEST`/parameter-set presets. **(Просунуто)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md), [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md), [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md).

**Не хардкодьте api_key і шляхи.**
*Чому:* той самий скрипт має працювати на dev і prod лише зі зміною оточення. *Як:* `{host, key}` — з env vars / vault (`.env` не в Git); для вузької інтеграції — **project-level key**, а не global admin; `Folder.get_path()` не працює на cloud-folder (S3/GCS/Azure) — use `get_download_stream`/`upload_stream`. **(Середній → Просунуто)** ⟶ [11](11-programmatic-api-python-client.md), [06](06-coding-recipes-code-envs-notebooks.md).

**Логування та версіювання.**
*Чому:* `print`/`logging` йдуть у Log рецепта; orchestration-скрипти — це інфраструктурний код. *Як:* структуруйте лог-повідомлення (етап, к-ть рядків); версіюйте Python-скрипти у Git разом із проєктом і застосовуйте до них code review. **(Просунуто)** ⟶ [06](06-coding-recipes-code-envs-notebooks.md), [11](11-programmatic-api-python-client.md).
> **Пастка:** `get_custom_variables()` повертає **рядки** — кастуйте перед арифметикою. Внутрішній `dataiku` (in-DSS, `get_dataframe()`) ≠ зовнішній `dataikuapi` (`iter_rows()` + `get_schema()`).

---

## 7. Machine Learning

**Уникайте data leakage.**
*Чому:* фічі, недоступні на момент прогнозу, або похідні від target дають нереалістично високий AUC, що падає в проді. *Як:* не включайте майбутні/target-залежні фічі; для часових даних — **Time ordering** split (не random!); стережіться target/impact encoding поза CV; реагуйте на діагностику «Too good to be true?». **(Середній)** ⟶ [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md).

**Чесний train/test і cross-validation.**
*Чому:* search-CV обирає hyperparameters; фінальну якість оцінює **окремий** test-набір. *Як:* K-fold для надійної оцінки; не плутайте search-CV із test; для незбалансованих класів — ROC AUC / F1 / Average Precision (не Accuracy). **(Середній)** ⟶ [07](07-machine-learning-visual-ml.md).

**Design vs deployment.**
*Чому:* «бойову» модель не редагують наживо. *Як:* експерименти — в **Lab**; продакшн — **Saved Model** + Train/Score/Evaluate recipes у scenarios; нові дані мають пройти **ті самі** Prepare-кроки, що й train. **(Середній)** ⟶ [07](07-machine-learning-visual-ml.md).

**Контролюйте активацію версій.**
*Чому:* при `Activate new versions = Automatically` retrain мовчки замінює модель — можна промотувати **гіршу**. *Як:* у продакшні — **Manually**; активуйте лише після перевірки метрик/drift (умовний Model Comparison / Run checks перед `Make Active`). **(Просунуто)** ⟶ [07](07-machine-learning-visual-ml.md), [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md).

**Моніторте drift і ретренте за тригером.**
*Чому:* модель деградує; без моніторингу ви про це дізнаєтесь останніми. *Як:* будуйте **MES** з самого початку (регулярний Evaluate recipe через scenario на свіжих прод-даних); без ground truth — **Input Data Drift + Prediction Drift**, Performance Drift додавайте щойно з'являться мітки; ретренте на **drift-тригер**, а не лише за розкладом. **(Просунуто)** ⟶ [07](07-machine-learning-visual-ml.md), [10](10-mlops-deployment-api-node.md).

**Інтерпретуйте перед деплоєм + відтворюваність.**
*Чому:* модель без пояснення — ризик. *Як:* Feature importance + PDP + Subpopulation analysis (+ Fairness report за потреби); фіксуйте seed, версіонуйте code env, документуйте через Model Document Generator. **(Середній → Просунуто)** ⟶ [07](07-machine-learning-visual-ml.md).

**Починайте з Quick Prototypes; rescaling — для scale-sensitive алгоритмів.**
*Чому:* швидкі прототипи показують складність задачі до інвестицій у High Performance; rescaling критичний для SVM/KNN/лінійних/NN, а для дерев — ні. *Як:* перший прохід — Quick Prototypes; обирайте бізнес-метрику свідомо (Cost Matrix Gain для вартісних рішень); для спец-задач (time series / causal / computer vision) — окремий code env; текстові змінні за замовчуванням відхиляються — увімкніть і оберіть векторизацію явно. **(Базово → Середній)** ⟶ [07](07-machine-learning-visual-ml.md).

---

## 8. Автоматизація

**Ідемпотентні scenarios.**
*Чому:* при ретраї append-логіка дублює дані. *Як:* будуйте **overwrite**-датасети або партиції з чітким partition spec; у SQL script завжди `TRUNCATE`/`DELETE` перед `INSERT`. **(Середній)** ⟶ [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md), [06](06-coding-recipes-code-envs-notebooks.md).

**Надійні тригери.**
*Чому:* dataset-change trigger **не бачить** змін у SQL-таблиці. *Як:* для SQL-джерел — **SQL query change trigger**; для джерел, що пишуться поступово — **grace delay з re-check**; тримайте **Suppress triggers while running**; для оркестрації — ланцюжок `Run another scenario` / after-scenario trigger, а не час-залежні «з запасом». **(Середній)** ⟶ [09](09-scenarios-automation-metrics-checks.md).

**Metrics / Checks / Data Quality rules.**
*Чому:* перевіряти треба те, що ламає downstream. *Як:* контролюйте непорожність, унікальність ключів, record count у розумному діапазоні, валідність критичних колонок; для волатильних метрик — **typical range (IQR)** замість жорстких меж; у нових проєктах — **Data Quality rules**, не legacy checks. **(Середній)** ⟶ [09](09-scenarios-automation-metrics-checks.md).
> **Порядок критичний:** **Compute metrics ПЕРЕД Run checks / Verify rules** — інакше check читає застаріле/відсутнє значення (EMPTY).

**Нотифікації лише на справжні проблеми.**
*Чому:* зашумлений канал перестають читати, і реальні FAILURE губляться. *Як:* FAILURE — завжди; SUCCESS — лише для критичних щоденних пайплайнів; гілка error-handling через `If some prior step failed`. **(Середній)** ⟶ [09](09-scenarios-automation-metrics-checks.md).
> **Dev vs prod:** на **Design node** scenarios тримають **Auto-triggers Off** (ручний запуск для тестів); бойові triggers активуються на **Automation node** після деплою bundle. Не вмикайте прод-нотифікації на dev.

---

## 9. MLOps та середовища

**Розділяйте Design / Automation / API ноди.**
*Чому:* продакшен-навантаження на Design node = нестабільність і відсутність контрольованих релізів. *Як:* фізично розділяйте dev (Design) і prod (Automation/API); різні групи-адміни й credentials. **(Середній)** ⟶ [01](01-overview-architecture.md), [10](10-mlops-deployment-api-node.md), [13](13-governance-security-collaboration.md).

**Деплойте лише через bundles, не «сирий» проєкт.**
*Чому:* версійований bundle через Project Deployer = відтворюваність і rollback. *Як:* збирайте **bundle**; не забудьте додати Saved Model / reference-datasets / plugins в **Additions** (інакше у проді scoring падає); просувайте **той самий bundle** через stages **Development → Test → Production**, а не збирайте новий під кожне. **(Середній)** ⟶ [10](10-mlops-deployment-api-node.md), [01](01-overview-architecture.md).

**Connection / code env remapping — завжди явний.**
*Чому:* без remapping bundle вказує на `sql-dev`, і прод читає/пише dev-дані. *Як:* тримайте окремі connections (`sql-dev`/`sql-prod`); **завжди** налаштовуйте explicit connection remapping; переконайтесь, що prod-connections, code envs **і plugins вже існують** на цільовій ноді перед першим деплоєм; override local variables у Settings deployment, не хардкодьте. **(Середній)** ⟶ [10](10-mlops-deployment-api-node.md).

**Безпечний rollback + CI/CD.**
*Чому:* реліз ламається — потрібен миттєвий відкат. *Як:* давайте bundle ID / API version змістовні імена (`v2.3-prod`) і заповнюйте **Release notes**; зберігайте відповідність **bundle ↔ git-комміт**; у CI/CD зберігайте поточний `bundle_id` перед кожним update; прогоняйте **smoke-тест** після деплою; для API — активацію без простою + rolling upgrade по нодах. **(Просунуто)** ⟶ [10](10-mlops-deployment-api-node.md), [11](11-programmatic-api-python-client.md).
> **HA:** один API node без load balancer — це не прод. Мінімум 2 ноди за LB із `/isAlive/`-probe.

**Project Standards — pre-deployment quality gates.**
*Чому:* «брак спільного розуміння, що таке якість» гальмує команди; стандарти автоматично перевіряють готовність. *Як:* **адмін** вмикає built-in / custom checks (приклади: «проєкт не має забагато datasets», вимоги до документації, патерни Flow, якість коду); звіт показується **під час дизайну, при створенні bundle і як deployment gate**. Переглядайте звіт і виправляйте порушення **до** спроби деплою. **(Просунуто)** ⟶ [13](13-governance-security-collaboration.md), [10](10-mlops-deployment-api-node.md).

---

## 10. Governance та безпека

**Least privilege.**
*Чому:* надлишкові права — головний вектор ризику; права в DSS **кумулятивні** (union усіх груп, «deny» не існує). *Як:* видавайте мінімум (не «Administrator» там, де досить «Create projects»); права — **тільки через групи**, не окремим користувачам; з code envs знімайте дефолт «Usable by all»; connection «Details readable by» — лише групам, яким реально треба читати креденшали з коду. **(Базово)** ⟶ [13](13-governance-security-collaboration.md).
> «Write unisolated code» / «Write unsafe code» — фактично **обхід безпеки** (код виконується як `dssuser`). Видавайте лише вузькому колу довірених.

**Separation of duties.**
*Чому:* той самий, хто розробляє, не має сам пушити в прод без контролю. *Як:* фізично розділяйте Design і Automation/API ноди; деплой — **тільки bundles через Deployer**; для prod-infrastructure Governance Policy = **Prevent** (не дефолтний NO_CHECK); право Deploy на Production — окремій групі; для прод — **Deploy-as-User** з технічним акаунтом. **(Середній)** ⟶ [13](13-governance-security-collaboration.md), [10](10-mlops-deployment-api-node.md).

**Audit — не покладайтесь на UI-в'ю.**
*Чому:* UI audit показує лише **останні 1000 подій** і скидається при рестарті — це не compliance-запис. *Як:* розгорніть **Event Server** і відвантажуйте логи в durable S3/Azure/GCS із власною lifecycle-політикою; моніторте topic `authfail` на brute-force / misconfig. **(Середній → Просунуто)** ⟶ [13](13-governance-security-collaboration.md).

**PII та per-user credentials через UIF.**
*Чому:* без **User Isolation Framework** усе виконується як єдиний `dssuser`, а connection-доступ і «Details readable by» — лише косметика. *Як:* у secure mode DSS impersonує end-user (per-user credentials стають реальними); PII — через Column Pseudonymization processor (SHA + Pepper + Salt) і LLM Mesh PII Detector. Native column/row-level masking **немає** — робіть у базі. **(Просунуто)** ⟶ [13](13-governance-security-collaboration.md).
> **Критична пастка:** **не вмикайте UIF на заповнений instance** — highly not recommended, downgrade не підтримується. Плануйте UIF до масового наповнення. Тримайте хоча б один **Local admin** (break-glass через `/login/`).

---

## 11. Колаборація

**Wiki + discussions замість email.**
*Чому:* контекст, прив'язаний до об'єкта, не губиться. *Як:* документуйте workflow у project wiki; обговорення — у threaded **discussions** з @mentions, прив'язаних до dataset/recipe/моделі. **(Базово)** ⟶ [13](13-governance-security-collaboration.md), [02](02-projects-flow-core-concepts.md).

**Code review через Git.**
*Чому:* orchestration-скрипти, webapp-код і plugins — це код; до них застосовні CI-тести й рев'ю. *Як:* версіюйте у Git; розробляйте webapp через **Code Studio** + version control (webapp є частиною bundle); plugins фетчте з **фіксованих git tags** (відтворюваність і відкат), зберігайте `name` параметрів стабільними. **(Середній → Просунуто)** ⟶ [11](11-programmatic-api-python-client.md), [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md), [14](14-plugins-applications-extensibility.md).
> **Очікування ≠ реальність:** у DSS немає real-time co-edit чи object locks — на Git діє **last-write-wins** (warn). Code Studios — single-user. Узгоджуйте, хто що редагує.

**Безпека спільного доступу до дашбордів.**
*Чому:* read-only глядач не має бачити сирі PII-датасети. *Як:* свідомо керуйте **dashboard authorizations** (глядач бачить лише явно authorized об'єкти); public webapp/dashboard працює під **run-as-user** — не давайте йому зайвих прав, публікуйте лише **aggregated** дані. **(Середній)** ⟶ [12](12-dashboards-charts-webapps-reporting.md), [13](13-governance-security-collaboration.md).

---

## 12. LLM / GenAI

**Усе через LLM Mesh, ніколи в обхід.**
*Чому:* прямі виклики провайдерів = втрата governance, аудиту, квот, кешу й guardrails. *Як:* звертайтесь за **LLM ID** (модель легко замінити); порівнюйте кандидатів у **Prompt Studio** / **Evaluate LLM**. **(Базово → Середній)** ⟶ [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md).

**Вибір моделі під задачу й бюджет.**
*Чому:* потужна модель на простій класифікації — марна витрата. *Як:* дешевші/менші для класифікації/підсумовування, потужніші — для складної генерації й агентів; для агентів — модель із надійним **tool/function calling**. **(Середній)** ⟶ [08](08-llm-generative-ai-mesh.md).

**Безпека даних.**
*Чому:* PII не має витікати у зовнішні API. *Як:* чутливі дані — на **локальні HuggingFace** моделі; перед відправкою у зовнішні — **PII Detection** guardrail (mask/pseudonymize) на connection-рівні; для user-facing — **Prompt Injection** + **Toxicity** guardrails. **(Середній → Просунуто)** ⟶ [08](08-llm-generative-ai-mesh.md).

**Контроль витрат.**
*Чому:* LLM-витрати ростуть непомітно. *Як:* налаштуйте **cost quotas** з blocking + email alerts (per project/user) + **Fallback Quota**; увімкніть **response caching**; покладайтеся на automatic throttling. Пам'ятайте: **LLM Cost Guard — оцінка**, не білінг. **(Середній)** ⟶ [08](08-llm-generative-ai-mesh.md).
> **Пастка квот:** rate-limit і квоти **не діють** на SageMaker, Databricks Mosaic AI, Snowflake Cortex та локальні HuggingFace — для них плануйте контроль витрат окремо.

**Оцінювання RAG.**
*Чому:* погані чанки = погані відповіді; галюцинації. *Як:* перевіряйте retrieval у режимі **Search** Knowledge Bank **до** підключення LLM; тюньте chunking / top-k / hybrid search / reranking; вмикайте **RAG guardrails** (faithfulness/relevancy); оцінюйте через **Evaluate LLM/Agent** (Context Precision/Recall, Faithfulness). **(Просунуто)** ⟶ [08](08-llm-generative-ai-mesh.md).

**Агенти — від простого до детермінованого.**
*Чому:* складний агент важче дебажити й контролювати. *Як:* починайте з **Simple Visual Agent**, переходьте на Structured Visual Agent / Code Agent для детермінованого контролю; давайте інструментам **чіткі описи** (LLM обирає tool за описом + JSON-schema); дебажте через **Trace Explorer**, оцінюйте через Agent Evaluation. Агент потребує моделі з надійним tool calling. **(Просунуто)** ⟶ [08](08-llm-generative-ai-mesh.md).

---

## 13. Антипатерни: чеклист «чого не робити»

Зведення типових помилок з усіх документів. Якщо хоч один пункт «про вас» — поверніться у відповідний документ.

| # | Антипатерн | Документ |
|---|---|---|
| 1 | Сплутати Design і Automation як «одну машину для всього»; деплоїти вручну в обхід bundles/Deployer | [01](01-overview-architecture.md), [10](10-mlops-deployment-api-node.md) |
| 2 | Покладатися на Default-назви datasets; не використовувати Flow zones у великих проєктах | [02](02-projects-flow-core-concepts.md) |
| 3 | Класти secrets у **standard** variables (експортуються з bundle) замість local | [02](02-projects-flow-core-concepts.md) |
| 4 | Force-recursive build «про всяк випадок» замість Smart reconstruction | [02](02-projects-flow-core-concepts.md) |
| 5 | CSV для big data у Flow; uppercase identifiers з Parquet/Hive; over-partitioning | [03](03-datasets-connections-partitioning.md) |
| 6 | Biased sample («First records» на відсортованих даних); зразок > 200 000 у RAM | [03](03-datasets-connections-partitioning.md), [05](05-data-preparation-prepare-recipe.md) |
| 7 | «Тихий» DSS-stream — не звіряти фактичний engine після Run | [04](04-visual-recipes.md) |
| 8 | Винаходити неіснуючі DSS-формули (`year()`, `parseDate`, `capitalize()`...); `+` як конкатенація | [05](05-data-preparation-prepare-recipe.md) |
| 9 | Хардкод credentials / api_key у recipe/notebook (витік у Git) | [06](06-coding-recipes-code-envs-notebooks.md), [11](11-programmatic-api-python-client.md) |
| 10 | Збирати великий датасет у pandas на DSS server (OOM); ставити пакети у builtin env | [06](06-coding-recipes-code-envs-notebooks.md) |
| 11 | Random split для часових даних; data leakage; Accuracy на незбалансованих класах | [07](07-machine-learning-visual-ml.md) |
| 12 | `Activate new versions = Automatically` у проді (мовчки промотує гіршу модель) | [07](07-machine-learning-visual-ml.md), [10](10-mlops-deployment-api-node.md) |
| 13 | Run checks без Compute metrics (EMPTY); dataset-change trigger на SQL; спам SUCCESS-нотифікаціями | [09](09-scenarios-automation-metrics-checks.md) |
| 14 | Не налаштувати connection remapping; забути Additions; один API node без LB як «прод» | [10](10-mlops-deployment-api-node.md) |
| 15 | Плутати `dataiku` і `dataikuapi`; `no_check_certificate=True` у проді | [11](11-programmatic-api-python-client.md) |
| 16 | Head sampling на дашборді; public webapp під привілейованим run-as-user; витік через dashboard authorizations | [12](12-dashboards-charts-webapps-reporting.md) |
| 17 | Видавати права окремим користувачам; покладатись на connection permissions без UIF; UIF на заповнений instance | [13](13-governance-security-collaboration.md) |
| 18 | Хардкод секретів у plugin-params; перейменування `name` параметра (ламає usages); dev plugin на prod | [14](14-plugins-applications-extensibility.md) |
| 19 | Прямі виклики LLM-провайдерів в обхід LLM Mesh; PII у зовнішні API без guardrails | [08](08-llm-generative-ai-mesh.md) |

---

## 14. Чеклист готовності проєкту до продакшну

Actionable-список «перед натисканням Deploy». Більшість пунктів покривають **Project Standards** автоматично — але звіряйте вручну.

**Структура та документація**
- [ ] Flow поділено на **zones**; datasets названо за схемою (`foo_raw`/`foo_clean`); немає Default-назв.
- [ ] Project wiki описує workflow; ключові об'єкти мають descriptions.
- [ ] Немає зайвих/тимчасових datasets («проєкт не має забагато datasets» — Project Standards).

**Дані та обчислення**
- [ ] Прод-дані на **SQL/cloud** connection (не uploaded files); великі managed datasets — **Parquet**.
- [ ] Перевірено фактичні **engines** ключових рецептів (push-down, не тихий DSS-stream).
- [ ] Білди **ідемпотентні** (overwrite/partition spec, не неконтрольований append).

**Якість та автоматизація**
- [ ] Scenario будує пайплайн; **Metrics → Checks/Data Quality rules** у правильному порядку.
- [ ] Тригери надійні (SQL trigger для SQL-джерел; grace delay де треба).
- [ ] Reporters: **FAILURE завжди**, SUCCESS лише де критично.
- [ ] На Design node scenarios мають **Auto-triggers Off**.

**ML (якщо є модель)**
- [ ] Немає data leakage; для часових даних — Time ordering split.
- [ ] Saved Model з `Activate new versions = Manually`; train/score проходять **ті самі** Prepare-кроки.
- [ ] Налаштовано **MES** (Input/Prediction drift) + checks + alerts.

**Деплой та середовища**
- [ ] Зібрано **bundle** (не «сирий» проєкт); усе потрібне в **Additions** (Saved Models, reference-datasets, plugins).
- [ ] Explicit **connection / code env remapping** dev→prod; цільові connections/envs/plugins **існують** на Automation/API node.
- [ ] Bundle ID/API version названо змістовно; заповнено **Release notes**; збережено `bundle_id` для rollback.
- [ ] Прогнано через **Test stage**; підготовлено smoke-тест; для API — ≥2 ноди за LB.

**Безпека та governance**
- [ ] Права за **least privilege**, через **групи**; немає «Write unisolated code» широко.
- [ ] Секрети — через connection / User Secrets / presets, **не в коді**.
- [ ] PII не виставлено публічно (dashboard authorizations, run-as-user, aggregated-only).
- [ ] Для регульованих: Governance Policy = **Prevent**, Final Approval (sign-off) отримано.
- [ ] **Event Server** збирає audit у durable storage.

**LLM (якщо є)**
- [ ] Усі виклики через **LLM Mesh**; cost quotas + guardrails (PII, injection) увімкнено.
- [ ] RAG retrieval перевірено; RAG guardrails активні.

> Якщо всі чекбокси заповнено — проєкт **production-ready**. Якщо ні — поверніться у відповідний тематичний розділ вище.

---

## 15. Перехресні посилання

Цей документ синтезує best practices з усіх розділів бази знань. За глибиною йдіть у першоджерело:

- [01-overview-architecture.md](01-overview-architecture.md) — архітектура, типи нод, lifecycle dev→prod, Kubernetes.
- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — Flow zones, tags, naming, variables, build modes.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — connections, Parquet, partitioning, sampling, storage types.
- [04-visual-recipes.md](04-visual-recipes.md) — visual recipes, recipe engines, push-down.
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — Prepare recipe, meanings, DSS formulas, ідемпотентність.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — code recipes, code envs, project libraries, секрети, containerized execution.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — Visual ML, leakage, train/test, Saved Models, drift.
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — LLM Mesh, guardrails, cost control, RAG, agents.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — scenarios, triggers, metrics, checks, Data Quality rules.
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — bundles, Project Deployer, remapping, MES, rollback, API node.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — Python client, REST API, ключі, CI/CD.
- [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md) — dashboards, charts, webapps, run-as-user, reporting.
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — permissions, UIF, audit, PII, wiki, sign-offs, Project Standards.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — plugins, applications, credentials, versioning.
- [16-glossary.md](16-glossary.md) — повний словник термінів.

---

## 16. Джерела

Офіційна документація та knowledge base Dataiku (станом на DSS 13.x/14.x):

- **Good dataset naming schemes** — `knowledge.dataiku.com/latest/data-sourcing/datasets/tip-dataset-naming-scheme.html`
- **Flow zones** — `knowledge.dataiku.com/latest/getting-started/dataiku-ui/tutorial-flow-zones.html`, `doc.dataiku.com/dss/latest/flow/zones.html`
- **Best Practices for Collaborating in Dataiku** — `knowledge.dataiku.com/latest/kb/governance/collaboration/best-practices-collaboration.html`
- **Automation Scenarios (production readiness)** — `knowledge.dataiku.com/latest/courses/mlops/readiness/automation-best-practices.html`
- **Workflow documentation in a wiki** — `knowledge.dataiku.com/latest/courses/mlops/readiness/workflow-documentation.html`
- **Project Standards** — `doc.dataiku.com/dss/latest/mlops/project-standards/index.html`, `knowledge.dataiku.com/latest/mlops-o16n/project-standards/tutorial-project-standards.html`
- **MLOps overview & architecture** — `knowledge.dataiku.com/latest/mlops-o16n/index.html`, `doc.dataiku.com/dss/latest/mlops/index.html`
- **Project deployment / batch deployment** — `knowledge.dataiku.com/latest/mlops-o16n/batch-deployment/concept-batch-deployment.html`

> Решта посилань на конкретні фічі — у секціях **Джерела** відповідних тематичних документів 01–14.
