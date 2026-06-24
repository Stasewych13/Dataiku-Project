# 04. Visual recipes

> Рівень: новачок → досвідчений
> Версії DSS: 13.x / 14.x (latest резолвиться у v14)
> Джерела: [doc.dataiku.com/dss/latest](https://doc.dataiku.com/dss/latest/), [knowledge.dataiku.com](https://knowledge.dataiku.com/), Dataiku Academy

---

## Вступ

**Visual recipes** (візуальні рецепти) — це основний спосіб трансформувати дані у Dataiku **без написання коду**. Рецепт (recipe) у Flow — це обчислювальний крок, що бере один або кілька **вхідних датасетів** (inputs) і породжує один або кілька **вихідних** (outputs). Візуальні рецепти конфігуруються через UI (форми, кнопки, dropdown-меню), а Dataiku сам генерує під капотом SQL, Spark або код власного рушія.

Чому це важливо для аналітика:

- **Низький поріг входу** — більшість трансформацій (join, group, filter, pivot) робляться кліками, а не SQL/Python.
- **Прозорість Flow** — кожен рецепт видно на схемі Flow жовтим колом-іконкою; колегам зрозуміло, що відбувається з даними.
- **Pushdown** — той самий візуальний рецепт може виконуватися **в базі даних** (in-database SQL) чи на **Spark**, без зміни конфігурації; ви описуєте *що* зробити, а Dataiku вирішує *як*.
- **Партиціонування «з коробки»** — рецепти знають про partitions і будують залежності між ними.

Цей документ покриває **кожен** візуальний рецепт окремо, систему **рушіїв виконання** (engines), управління **входами/виходами та схемою**, а також коли обирати visual проти code recipes.

> Перед читанням бажано ознайомитися з [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) (Flow, recipe, dataset) та [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) (connections, partitioning).

---

## Концепції

### Recipe = крок Flow (Базово)

У Flow обʼєкти двох типів: **datasets** (сині квадрати з даними) і **recipes** (жовті/помаранчеві кола з трансформаціями). Рецепт завжди має:

- **Inputs** — один або кілька датасетів (а для деяких рецептів — managed folders чи моделі);
- **Outputs** — один або кілька датасетів (рідше — folders);
- **Конфігурацію** — налаштування трансформації;
- **Engine** — рушій, що фактично виконує обчислення.

Рецепт **не зберігає дані** — дані живуть у датасетах. Запуск (**Run**) бере дані з inputs, застосовує трансформацію і записує в outputs.

### Як створити візуальний рецепт (Базово)

Три типові способи:

1. **+ Recipe** угорі Flow → **Visual** → обрати рецепт → вказати input(и) та назву output.
2. Виділити датасет → права панель **Actions** → **Visual recipes** (іконки Sync, Group, Join, …).
3. Виділити **два** датасети → запропонується Join / Stack.

### Visual vs code recipes — коли що (Середній)

| | **Visual recipe** | **Code recipe** (SQL / Python / R / Spark) |
|---|---|---|
| Поріг входу | низький, без коду | потрібне знання мови |
| Прозорість | висока (читається з UI) | нижча (треба читати код) |
| Pushdown | авто (SQL/Spark/DSS) | SQL-рецепт = нативний SQL; Python = у памʼяті/Spark |
| Гнучкість | обмежена набором операцій | будь-яка логіка |
| Версіонування diff | гірше читається | звичайний текстовий diff |

**Правило великого пальця:** беріть **visual recipe**, поки операція покривається стандартними рецептами (sync, filter, join, group, window, pivot…). Переходьте на **code recipe** лише коли логіка не виражається візуально (складні віконні каскади, рекурсія, виклики API, нестандартні алгоритми). Детальніше — у [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md).

---

## Рушії виконання (Recipe engines)

Це **наскрізна** тема для всіх візуальних рецептів — варто зрозуміти один раз.

### Доступні рушії (Середній)

- **DSS (local stream engine)** — власний потоковий рушій Dataiku. Читає дані порядково (streaming), не вантажить усе в памʼять. Працює **завжди** і з будь-яким типом датасета (файли, SQL, cloud storage). Це універсальний fallback.
- **In-database (SQL)** — Dataiku транслює рецепт у **SQL-запит** і виконує його **всередині БД** (PostgreSQL, Snowflake, BigQuery, Redshift тощо). Дані **не виходять** із БД — це і є **pushdown**. Найшвидший варіант, коли дані вже в базі.
- **Spark** (включно зі SparkSQL) — розподілене виконання на кластері. Підходить для дуже великих обсягів та коли дані у HDFS / cloud storage. Усі візуальні рецепти трансформації підтримують Spark: Prepare, Sync, Sample/Filter, Group, Distinct, Join, Pivot, Sort, Split, Top N, Window, Stack.
- **Hive / Impala** — для датасетів на HDFS (Hadoop-екосистема). Historically для on-prem Hadoop.

### Логіка вибору engine (Середній)

DSS **автоматично** обирає рушій залежно від типу input/output датасетів:

- Усі inputs та output в **одному SQL-зʼєднанні** і всі операції перекладаються в SQL → **In-database (SQL)** (найкраще).
- Дані у HDFS / cloud storage, Spark увімкнено → **Spark**.
- Інакше → **DSS (stream)**.

> У свіжих інсталяціях (після v14.5) для малих непартиціонованих датасетів (<20MB) DSS може автоматично відкочуватися на власний рушій навіть якщо Spark увімкнено — для маленьких даних запуск Spark коштує дорожче, ніж сама обробка. Це **не** стосується Sync, Pivot, Join.

### Як подивитись / змінити engine (Базово)

У редакторі рецепта під кнопкою **Run** є посилання/іконка-шестерня з назвою поточного рушія (напр. «Running on: In-database (SQL)»). Клік відкриває діалог вибору, де доступні рушії активні, а недоступні — **сірі (greyed out)** з підказкою-причиною.

### Чому engine «not selectable» (Просунуто)

Найчастіші причини, чому потрібний рушій недоступний:

1. **Inputs у різних зʼєднаннях.** Для In-database (SQL) **усі** вхідні датасети мають бути SQL-таблицями в **одному й тому ж** connection. Join файлового датасета з PostgreSQL-таблицею в SQL не зайде — спершу зробіть **Sync** файлового датасета в ту саму БД.
2. **Output в іншому зʼєднанні.** Output теж має лежати в тій самій БД, інакше SQL-pushdown неможливий.
3. **Непідтримувана операція.** Лише підмножина процесорів/операцій транслюється в SQL. Якщо хоч один крок не перекладається — SQL-рушій недоступний (особливо актуально для Prepare; див. [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md)).
4. **SQL Query dataset як input.** Візуальні рецепти будують запит навколо *імені таблиці*; обгорнути довільний SQL-query датасет у підзапит не завжди коректно — тому SQL-рушій може бути недоступним.
5. **Fuzzy join** працює **тільки** на DSS engine — SQL/Spark недоступні взагалі.
6. **Spark не сконфігуровано** адміністратором, або вимкнено для зʼєднання у **Administration > Settings > Engines & Connections**.

Помилка під час запуску з примусово невідповідним рушієм: **`ERR_RECIPE_CANNOT_USE_ENGINE`**.

### Best practice по engine (Просунуто)

- **Тримайте дані «вдома».** Якщо джерело — БД, тримайте проміжні датасети в тій самій БД, щоб увесь Flow робив **pushdown** і не качав дані туди-сюди.
- **Не вантажте все в памʼять.** Уникайте сценаріїв, де великий датасет проходить через DSS-stream без потреби — обирайте SQL/Spark.
- **Перевіряйте фактичний engine.** Іноді Flow «тихо» падає на DSS-stream через одну несумісну операцію → деградація швидкості. Дивіться індикатор під Run і Knowledge Base: *Troubleshoot — Dataiku isn't using the optimal engine for a visual recipe*.

> Пов'язана автоматизація та метрики продуктивності — у [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md).

---

## Inputs / outputs та управління схемою (Середній)

### Inputs та outputs

- Більшість трансформаційних рецептів: **1 input → 1 output** (Sync, Prepare, Group, Window, Sort, Distinct, Pivot, Sample/Filter).
- **Join / Stack** — **кілька inputs → 1 output**.
- **Split / TopN** — **1 input → кілька outputs** (TopN додатково має «remaining rows»).
- **Download** — input може бути джерелами URL/FTP → output це **managed folder**.
- **Push to editable** — output це **editable dataset**.

### Output schema

**Output schema** (схема виходу — список колонок та їх типів) у візуальних рецептах **обчислюється автоматично** з конфігурації. Наприклад, Group recipe формує колонки = group keys + агрегати з псевдонімами.

- При зміні конфігурації Dataiku оновлює схему; зазвичай зʼявляється кнопка перевірити/застосувати схему перед Run.
- Для **Sync** є кнопка **Resync schema** — якщо схема input змінилась, синхронізувати output під неї. Sync співставляє колонки **за іменем**; можна видаляти/перевпорядковувати, але **не перейменовувати** (для rename — Prepare recipe).
- Деякі рецепти дозволяють **редагувати схему виходу** вручну (типи, drop колонок).

### Drop and recreate vs append (Середній)

Поведінка запису output контролюється у налаштуваннях output-датасета / рецепта:

- **Drop and recreate** (типово для більшості SQL/файлових outputs) — output повністю перебудовується щоразу.
- **Append** — нові рядки дописуються (для логів, інкрементального завантаження).
- Для **партиціонованих** outputs перебудовується лише цільова партиція.

> Поняття managed/external datasets, connections та partitioning — у [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md).

---

## How-to по кожному рецепту

Нижче — кожен візуальний рецтепт окремо: призначення, ключові налаштування, приклад. Назви рецептів та UI-елементів — в **оригіналі (English)**.

### Sync (Базово)

**Призначення:** копіювати датасет з одного місця/формату в інше **без зміни вмісту**. Класика: SQL → HDFS (щоб обробляти Hive), HDFS → Elasticsearch (щоб шукати), файли → SQL (щоб запитувати), або просто матеріалізувати датасет у керованому форматі.

**Ключові налаштування:**
- Output connection/формат (куди копіюємо).
- Схема за замовчуванням копіюється з input; **Resync schema** при зміні input.
- Зі stream-рушієм схеми in/out можуть відрізнятися — співставлення **за іменем**, можна прибрати/переставити колонки (rename — НЕ можна).
- Партиціонування: зазвичай реплікується з input (залежність «equals»), можна синкати в непартиціонований output.

**Engine:** Streaming, Spark, SQL, Hive, Impala. Є **оптимізовані fast-paths**: S3↔Redshift, Azure Blob↔Azure SQL DW, GCS↔BigQuery, HDFS↔Teradata, S3/Azure Blob↔Snowflake.

**Приклад:** скопіювати PostgreSQL-таблицю `orders` у HDFS, щоб далі застосувати Hive-обробку; DSS сам обере оптимальний шлях.

### Prepare (огляд) (Базово)

**Призначення:** очищення (cleansing), нормалізація та збагачення даних через скрипт із ~100 процесорів (parse date, find&replace, split column, formula тощо).

**Ключове:** Script зі steps, що виконуються послідовно; працює зі sample на дизайні, з усім датасетом на Run.

> Це найважливіший і найскладніший рецепт — повний розбір (процесори, формули, meanings, semantic types) у окремому документі [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md).

### Group (агрегації) (Базово)

**Призначення:** агрегувати дані як SQL `GROUP BY` — згорнути рядки за ключами і порахувати метрики.

**Ключові налаштування:**
- **Group keys** — одна чи кілька колонок, за якими групуємо.
- **Per-field aggregations** — для кожної колонки оберіть агрегати: `count`, `count distinct`, `sum`, `avg`, `min`, `max`, `first`, `last`, `concat`, `stddev`, тощо; кожному можна дати **alias**.
- **Global count** — додати колонку з кількістю рядків у групі (`count(*)`).
- **Pre-filter** — фільтр **до** агрегації (відсіяти зайві рядки).
- **Post-filter** — фільтр **після** агрегації (наприклад, лишити групи з `sum > X`, аналог `HAVING`).
- **Output schema** = group keys + усі обрані агрегати.

**Engine:** Hive, Impala, SparkSQL, plain SQL, DSS.

**Приклад:** агрегувати продажі за регіоном: group key = `region`, агрегат = `sum(revenue)` as `total_revenue`, global count = N; pre-filter відкидає скасовані замовлення (`status != 'cancelled'`), post-filter лишає лише `total_revenue > 100000`.

### Join (Середній)

**Призначення:** обʼєднати два (чи більше) датасети за умовами співставлення — аналог SQL `JOIN`.

**Типи join:**
- **Inner** — лише рядки, що збіглися в обох.
- **Left outer** — усі рядки лівого + збіги з правого (решта = NULL).
- **Right outer** — усі рядки правого + збіги з лівого.
- **Full outer** — усі рядки обох.
- **Cross** — декартів добуток (кожен з кожним).
- **Left/Right anti join** — лише **незбіги** з лівого/правого (рядки без пари).
- **Advanced** — нестандартні умови.

**Ключові налаштування:**
- **Join conditions** — DSS авто-підбирає колонки з однаковими іменами; **ADD A CONDITION** дозволяє задати ключі та **оператори** (не лише `=`, а й `<`, `>`, ranges → це й робить join «advanced»).
- **Selected columns** — які колонки з кожного датасета лишити у виході.
- **Column disambiguation** — при однакових іменах: alias через олівець, або **prefix** для всіх колонок таблиці (напр. `r_`).
- **Computed columns** — додаткові вихідні колонки через формулу (post-join).
- **Pre-filter** на кожен input; **post-filter** на головний вихід (НЕ на unmatched).
- **Unmatched rows** — для left/right/inner між двома датасетами можна **відправити незбіги в окремий output**.

**Engine:** Hive, Impala, SparkSQL, plain SQL, DSS. Памʼятайте: усі inputs мають бути в **одному** SQL-зʼєднанні для in-database.

**Приклад:** left join `customers` (left) з `orders` (right) за `customer_id`; selected columns: усі з customers + `order_date`, `amount` з orders з префіксом `ord_`; pre-filter на orders лишає лише останні 30 днів; незбіги (клієнти без замовлень) ідуть у окремий output `customers_without_orders`.

### Fuzzy join (Просунуто)

**Призначення:** join, коли ключі **не збігаються точно** — DSS рахує «відстань» між значеннями і порівнює з порогом. Підтримує inner/left/right/outer.

**Типи matching:**
- **Текст:** Damerau–Levenshtein (вставки/видалення/заміни/транспозиції), Hamming (для рядків рівної довжини), Jaccard (множини символів), Cosine.
- **Числа:** Euclidean distance.
- **Geopoint:** geospatial distance.
- **Інші типи:** лише сувора рівність.

**Налаштування:**
- **Normalization** тексту: lowercase, прибрати пунктуацію/звертання/стоп-слова, стемінг, сортування символів.
- **Threshold** — абсолютний або **відносний (%)**, нормований на довжину/значення ключа.

**Engine:** **тільки DSS engine** (SQL/Spark недоступні).

**Приклад:** join `propre` з `propeller` за Damerau–Levenshtein; distance = 4 операції / 9 символів = 44% ≤ поріг 50% → збіг. Практично — мерж назв компаній з різними написаннями.

### Stack (union рядків) (Базово)

**Призначення:** «поставити датасети один на одного» — конкатенація рядків, аналог SQL `UNION ALL`. (Join зʼєднує колонки; Stack — рядки.)

**Ключові налаштування:**
- Кілька inputs з потенційно різними схемами.
- Спосіб співставлення колонок: **за іменем** (union by name) або **за позицією/індексом** (by position). Колонки, відсутні в якомусь датасеті, заповнюються NULL.
- **Origin column** — додати колонку-мітку, з якого input прийшов рядок (корисно щоб не загубити походження).
- Pre-filter / post-filter.

**Engine:** усі основні (SQL/Spark/DSS).

**Приклад:** обʼєднати `sales_2023`, `sales_2024`, `sales_2025` в один `sales_all`; union by name; origin column = `source_year`.

### Window (віконні функції) (Просунуто)

**Призначення:** обчислення «по вікну» рядків зі збереженням деталізації кожного рядка — аналог SQL `OVER (...)`. На відміну від Group, **не згортає** рядки.

**Window definition:**
- **Partitioning columns** — на які групи бити (аналог `PARTITION BY`); без них вікно = весь датасет.
- **Order columns** — порядок рядків усередині партиції (потрібен для lag/lead/cumulative/rank).
- **Window frame** — які рядки беруть участь у розрахунку:
  - **Bounded** — обмежена к-сть рядків до/після (напр. «3 попередніх + 3 наступних» = рухоме вікно).
  - **Unbounded** — від початку партиції до поточного рядка (типово для cumulative).
  - Межі «rows before» і «rows after» задаються незалежно.

**Функції:**
- **Positional:** `LAG()`, `LEAD()` — значення з попереднього/наступного рядка.
- **Ranking:** `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`.
- **Cumulative:** running sum / running avg / інші агрегати по вікну.
- **Statistical:** moving average, first/last value, `VALUE_AT()` (значення на конкретній позиції).

> **Важлива поведінка:** на **DSS engine** вікно за замовчуванням = **весь frame**, якщо frame не задано явно. На SQL-рушіях семантика може відрізнятися — задавайте frame явно для передбачуваності.

**Engine:** Hive, Impala, SparkSQL, plain SQL, DSS.

**Приклад:** транзакції, partition by `customer_id`, order by `date`, frame = unbounded preceding → current row, агрегат = `sum(amount)` → колонка cumulative spend; додатково `LAG(amount)` → попередня покупка; `RANK()` за amount → ранг покупки клієнта.

### Sort (Базово)

**Призначення:** впорядкувати датасет за однією чи кількома колонками.

**Ключові налаштування:**
- Список колонок, кожна з напрямком **ascending / descending**; DSS застосовує в заданому порядку (багаторівневе сортування).
- **Null handling** (з v4.1 уніфіковано): ascending → NULL на початку, descending → NULL в кінці. Деякі БД (Vertica, Sybase IQ, DB2) можуть давати інакше.

> Output має сенс лише якщо output-формат **зберігає порядок запису** (Filesystem, HDFS). Багато SQL-таблиць не гарантують порядок без `ORDER BY` у запиті-читачі.

**Engine:** Hive, Impala, SparkSQL, SQL, DSS.

**Приклад:** sort `customers` за `region` (asc), потім `sales` (desc) — згрупувати за регіоном і показати найкращих зверху.

### TopN (Середній)

**Призначення:** дістати **перші N (і/або останні M)** рядків у кожній групі за порядком — аналог «top-K per group».

**Ключові налаштування:**
- **Grouping/partition keys** — підмножини; без них вся таблиця = одна група.
- **Order columns** — мінімум один (визначає, що «топ»).
- **N (top) та M (bottom)** — скільки взяти з початку/кінця кожної групи.
- **Два outputs:** топ/боттом рядки + **remaining rows** (решта).
- Можливе збагачення (наприклад, колонка рангу).

> **Детермінізм:** при однакових значеннях ключа і order-колонок порядок між такими рядками **недетермінований** і може мінятися між білдами.
> **TopN vs Window:** TopN одразу **обрізає** до K рядків на групу (зручніше і часто швидше для «топ-K»); Window рахує rank/функції для **всіх** рядків (потрібен наступний фільтр щоб обрізати).

**Engine:** Hive, Impala, SparkSQL, plain SQL, DSS.

**Приклад:** для `sales` grouping key = `region`, order by `revenue` desc, top 3 + bottom 2 → output найкращих/найгірших замовлень на регіон + remaining.

### Distinct (Базово)

**Призначення:** дедуплікація — лишити **унікальні** рядки.

**Ключові налаштування:**
- Колонки для порівняння (за якими визначаємо унікальність) — або всі.
- Опційно **count column** — скільки дублікатів було на кожну унікальну комбінацію.
- Pre/post-filter.

**Engine:** Hive, Impala, SparkSQL, plain SQL, DSS.

**Приклад:** distinct по `customer_id`, `email` → унікальні клієнти + колонка `dup_count` зі скільки разів зустрічався.

### Pivot (Просунуто)

**Призначення:** будувати pivot-таблицю — перетворити **значення колонки на нові колонки** з агрегацією. Більше контролю, ніж pivot-процесор у Prepare; виконується нативно в SQL/Hive.

**Ключові налаштування:**
- **Row identifiers** — колонки, що задають рядки виходу (або «всі інші колонки»).
- **Create columns from** (modalities) — колонка(и), значення яких стають новими колонками. Скільки modalities брати:
  - **Most frequent** — N найчастіших;
  - **Minimum occurrence** — ті, що зустрічаються ≥ N разів;
  - **Explicit** — задати вручну.
- **Populate content with** — агрегати на клітинку (row × modality): `count`, `sum`, `avg`, `min`, `max`, …; плюс **marginals** (підсумки по рядку).
- **Slugification** імен колонок (whitespace/пунктуація → `_`; для Hive — лише alphanumerics), нумерація/обрізання.

**Engine:** SQL/Hive (нативно), DSS.

**Приклад:** input (`Product`, `Country`, `Net`). Row id = `Product`, create columns from = `Country`, populate with = `sum(Net)` →

| Product | FR_Net_sum | GB_Net_sum | US_Net_sum |
|---|---|---|---|
| Toothpaste | 40 | 155 | 115 |
| Chocolate | 230 | 70 | — |

### Split (Середній)

**Призначення:** розкидати рядки **одного** датасета по **кількох** outputs за правилами.

**Режими розщеплення:**
- **На значеннях колонки** — окремий output на кожне значення (напр. на регіон).
- **За умовами/фільтрами** — визначити правила (rules), кожне → свій output.
- **Dispatch / random ratio** — випадковий розподіл за пропорціями (класика train/test split).
- **«Other» bucket** — рядки, що не підпали під жодне правило.

**Ключове:** кілька outputs; pre-filter; кастомні фільтри в табі **Splitting**.

**Engine:** SQL/Spark/DSS (залежно від режиму).

**Приклад:** split `events` за `country`: output `events_ua`, `events_pl`, решта → `events_other`. Або 80/20 random ratio → `train` / `test`.

### Sampling / Filtering (Середній)

**Призначення:** зменшити датасет (sample) і/або відфільтрувати рядки (filter) — рецепт **Sample/Filter**.

**Методи фільтрації (dropdown):**
- **Rules** — умови (column, operator, value) з AND/OR і групуванням.
- **Formula** — мова формул DSS (функції, імена колонок, project variables).
- **SQL expression** — для SQL-рушіїв.
- **Elasticsearch query string** — для ES v7+/OpenSearch (тоді sampling вимкнено, фільтр по всьому датасету).

**Методи семплінгу:**
- **First records** — перші N (дуже швидко, але упереджено).
- **Last records** — останні N.
- **Random (approximate ratio)** — ~X% рядків (1 прохід).
- **Random (fixed number)** — рівно N (повний прохід).
- **Random (approximate number)** — ~N (2 проходи, точніше на великих).
- **Column values subset** — підмножина значень колонки → всі рядки з ними (зберігає цілі послідовності, напр. усі дії юзера).
- **Stratified (fixed / approximate ratio)** — зберігає розподіл значень колонки.
- **Class rebalancing** — вирівнює представництво класів (тільки undersampling).
- **No sampling / Full** — без редукції.

**Engine:** SQL/Spark/DSS.

**Приклад:** rules-фільтр `status = 'active' AND signup_date > '2025-01-01'`; sampling = stratified by `plan` для збалансованого датасета під ML.

> Семплінг також керує **тим, що ви бачите** на дизайні всіх рецептів (sample у Explore). Деталі — у [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) та [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md).

### Download (Середній)

**Призначення:** завантажити файли з файлових джерел у **managed folder** (а не датасет напряму).

**Джерела:** Filesystem, HDFS, S3, GCS, Azure Blob; FTP, SFTP, SSH; **HTTP(S) URLs** (read-only).

**Ключові налаштування:**
- Список source-шляхів з **glob-патернами** (`*`, `?`).
- Синхронізація за **modification time + size**; може видаляти зниклі файли / перезавантажувати наявні.
- Output — managed folder; далі створюєте **Files in folder** dataset для парсингу.
- Партиціонування output (напр. по даті) з expansion-змінними (`$DKU_DST_YEAR` тощо).

**Приклад:** Apache-логи на FTP: output-folder партиціонований по даті (`%Y/%M/%D/.*`), source `ftp://SERVER/var/log/apache2/$DKU_DST_YEAR/$DKU_DST_MONTH/$DKU_DST_DAY/*.log`; білд партиції `2017-02-05` тягне всі `.log` з відповідної папки.

### Push to editable (Середній)

**Призначення:** скопіювати звичайний датасет в **editable dataset**, **зберігаючи ручні правки**. Кейс — виправити помилки даних, які не можна полагодити в джерелі.

**Ключові налаштування:**
- **Unique identifier** — одна чи кілька колонок як ключ рядка (для логіки злиття).
- **Ліміт 100K рядків** (обмеження editable datasets).

**Поведінка:**
- **Перший Run** — копіює весь вміст у editable dataset.
- **Наступні Run** — підтягує **нові/змінені** рядки з джерела, але **зберігає** ваші ручні правки в editable (за unique identifier).

**Приклад:** довідник `product_categories` у БД має помилки, які не правляться upstream → push to editable з ключем `product_id`, ручне виправлення в editable, решта Flow спирається на editable. Деталі про editable datasets — у [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md).

### Інші наявні рецепти (Просунуто)

Окрім трансформаційних, у Visual-меню є спеціалізовані:

- **Geo join** — join за **просторовими** відношеннями (потребує PostGIS для БД-операцій).
- **Generate features** — авто feature engineering (похідні колонки).
- **Extract document content** — текст/OCR з PDF та документів.
- **Compute statistics / Generate statistics** — зведена статистика по датасету.
- **Upsert** — оновити наявні / вставити нові рядки за ключем.
- **List folder contents** — датасет зі вмісту managed folder.
- **Extract failed rows** — ізолювати рядки, що не пройшли data quality перевірки.
- **Export** — вивантажити файли (з підтримкою партицій).

> ML-орієнтовані рецепти (Train, Score, Evaluate, Standalone evaluation) — у [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md). LLM-рецепти (Prompt, Classify, Embed) — у [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md).

### Foreach / loop через scenarios (Просунуто)

У Dataiku **немає** окремого візуального «loop recipe». Ітерування/повторення робиться:

- **партиціонуванням** — один рецепт обробляє багато партицій (по даті, по країні тощо), DSS будує залежності per-partition;
- **scenarios** — крок **Build/Train** з ітерацією по списку партицій або змінних; кроки з **custom steps** для циклів;
- **applications-as-recipes** / programmatic API для динамічних повторів.

Детально — у [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) (scenarios, кроки, тригери) та [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) (динамічна побудова Flow).

---

## Приклади (наскрізні)

**Приклад 1 — підготовка вітрини продажів (in-database pushdown).**
`raw_orders` (PostgreSQL) → **Sample/Filter** (`status='completed'`) → **Prepare** (parse dates, normalize country) → **Group** (key=`region`, `sum(amount)`, `count`) → **Sort** (desc by sum) → output `sales_by_region` (PostgreSQL). Усі датасети в одному PG-зʼєднанні → весь Flow робить SQL-pushdown, дані не покидають БД.

**Приклад 2 — збагачення замовлень клієнтами.**
`orders` + `customers` → **Join** (left, key=`customer_id`, унмечі → `orders_no_customer`) → **Window** (partition by `customer_id`, order by `date`, cumulative `sum(amount)`) → **TopN** (top 1 per `customer_id` за `amount`) → output «найбільша покупка кожного клієнта».

**Приклад 3 — консолідація з різних джерел.**
3 файли продажів → **Stack** (union by name, origin column) → **Distinct** (з count) → **Pivot** (rows=`product`, columns from=`country`, `sum(net)`) → дашборд (див. [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md)).

---

## Best practices

- **(Просунуто) Pushdown у SQL/Spark.** Тримайте проміжні датасети в тій самій БД, щоб увесь ланцюг рецептів виконувався in-database. Перевіряйте індикатор engine під Run — не дайте Flow тихо впасти на DSS-stream.
- **(Середній) Не вантажте все в памʼять.** Уникайте сценаріїв, де великий датасет проходить через DSS-stream без потреби. Для великих обсягів — SQL або Spark.
- **(Середній) Фільтруйте рано.** Pre-filter (у Group/Join/Window) і Sample/Filter **до** важких операцій зменшують обсяг, який обробляється далі.
- **(Базово) Читабельність Flow.** Давайте датасетам і рецептам **зрозумілі імена**, групуйте у **Flow zones**, тримайте лінійну логіку. Один рецепт = один зрозумілий крок; не намагайтеся «втиснути» все в один Prepare.
- **(Просунуто) Partitioning-aware рецепти.** Якщо датасети партиціоновані — налаштовуйте partition dependencies так, щоб білд оновлював лише потрібні партиції (інкрементально), а не весь датасет.
- **(Середній) Visual спершу.** Робіть візуально, поки можливо; до code recipe переходьте лише коли логіка не виражається рецептами (див. [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md)).
- **(Середній) Drop & recreate vs append.** Свідомо обирайте режим запису output: для відтворюваності — drop & recreate; для логів/інкременту — append/партиції.
- **(Базово) Aliases агрегатів.** У Group/Window/Pivot давайте агрегатам осмислені alias (`total_revenue`, а не `amount_sum`), щоб downstream-рецепти й дашборди читалися.

---

## Типові помилки

- **(Середній) Очікувати SQL-engine при різних зʼєднаннях.** Join/Stack файлу з БД-таблицею не піде in-database. Спершу **Sync** усі inputs в одну БД.
- **(Середній) Забути про детермінізм у TopN.** При рівних значеннях ключа/order порядок недетермінований — додайте tie-breaker колонку в order.
- **(Просунуто) Неявний window frame на DSS engine.** DSS бере **весь** frame за замовчуванням; на SQL семантика інша → задавайте frame **явно**, інакше cumulative/moving працюватиме «не так».
- **(Середній) Fuzzy join на великих даних.** Працює лише на DSS engine і обчислювально дорогий (попарні порівняння) — фільтруйте/блокуйте кандидатів заздалегідь, не запускайте на мільйонах рядків наосліп.
- **(Базово) Sort без збереження порядку.** Output у SQL-таблицю не гарантує порядок при подальшому читанні без `ORDER BY` — сортуйте там, де порядок реально потрібен (експорт, файли).
- **(Середній) Sync для перейменування колонки.** Sync **не вміє** rename — використовуйте Prepare recipe.
- **(Середній) Post-filter замість pre-filter.** Фільтрувати **після** важкого Group/Join — марнувати ресурси; де можна, фільтруйте **до** (pre-filter).
- **(Базово) Push to editable >100K рядків.** Editable обмежений 100K — для великих даних цей рецепт не підходить.
- **(Середній) «Тихий» DSS-stream.** Один несумісний крок у Prepare/Join робить весь рецепт небуквальним для SQL → деградація швидкості без помилки. Перевіряйте фактичний engine.

---

## Перехресні посилання

- [01-overview-architecture.md](01-overview-architecture.md) — архітектура DSS, де живе обчислення.
- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — Flow, recipe, dataset, Flow zones.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — connections, partitioning, editable datasets, sampling.
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — повний розбір Prepare recipe.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — code recipes (SQL/Python/R/Spark), коли переходити з visual.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — ML-рецепти (Train/Score/Evaluate), Lab.
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — LLM-рецепти.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — scenarios, loop/foreach, метрики, checks.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — створення рецептів через API.
- [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md) — візуалізація виходів рецептів.
- [15-best-practices.md](15-best-practices.md) — загальні best practices.
- [16-glossary.md](16-glossary.md) — глосарій термінів.
