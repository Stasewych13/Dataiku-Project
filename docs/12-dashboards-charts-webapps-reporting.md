# 12. Dashboards, Charts, Webapps та Reporting

> Версія документа орієнтована на **Dataiku DSS 13.x / 14.x** (2025–2026). Технічні терміни, назви елементів інтерфейсу (UI) та код подано в **оригіналі (англійською)**; пояснення — українською.
>
> Рівні складності позначено так: **(Базово)** — для новачка, **(Середній)** — для впевненого користувача, **(Просунуто)** — для досвідченого аналітика/інженера.

---

## Зміст

1. [Вступ](#вступ)
2. [Charts: концепція](#charts-концепція)
3. [Dimensions, measures, aggregations](#dimensions-measures-aggregations)
4. [Binning дат і чисел](#binning-дат-і-чисел)
5. [Типи charts (повний довідник)](#типи-charts-повний-довідник)
6. [Filters, color, tooltip, reference lines](#filters-color-tooltip-reference-lines)
7. [Sampling та execution engine](#sampling-та-execution-engine)
8. [Публікація chart на dashboard](#публікація-chart-на-dashboard)
9. [Dashboards: концепція та структура](#dashboards-концепція-та-структура)
10. [Tiles: повний довідник](#tiles-повний-довідник)
11. [Filters та interactivity на dashboard](#filters-та-interactivity-на-dashboard)
12. [Дозволи на перегляд (dashboard authorizations)](#дозволи-на-перегляд-dashboard-authorizations)
13. [Sharing & exposing об'єктів](#sharing--exposing-обєктів)
14. [Export dashboard у PDF/images](#export-dashboard-у-pdfimages)
15. [Webapps: типи та архітектура](#webapps-типи-та-архітектура)
16. [Як webapp читає дані з DSS](#як-webapp-читає-дані-з-dss)
17. [Webapp security та run-as-user](#webapp-security-та-run-as-user)
18. [Code Studios для розробки webapp](#code-studios-для-розробки-webapp)
19. [Reporting: експорт даних клієнтам](#reporting-експорт-даних-клієнтам)
20. [RAG / LLM-powered apps та Dataiku Answers](#rag--llm-powered-apps-та-dataiku-answers)
21. [Приклади (end-to-end)](#приклади-end-to-end)
22. [Best practices](#best-practices)
23. [Типові помилки](#типові-помилки)
24. [Перехресні посилання](#перехресні-посилання)
25. [Джерела](#джерела-офіційна-документація-dataiku)

---

## Вступ

**(Базово)** Зробити аналіз — це лише пів справи. Друга половина — **донести результат** до колег, замовників, керівництва. У DSS за це відповідає **«шар комунікації»**, який складається з трьох рівнів складності:

| Інструмент | Що це | Коли застосовувати |
|---|---|---|
| **Charts** | Вбудований no-code chart builder поверх датасета | Швидка візуалізація для аналізу та для дашборда |
| **Dashboards** | Сторінки з tiles (charts, таблиці, metrics, текст, webapp…) | Готовий звіт/панель для перегляду стейкхолдерами |
| **Webapps** | Повноцінні веб-застосунки (HTML/JS, Dash, Streamlit, Shiny, Bokeh) | Інтерактивні застосунки, кастомні візуалізації, форми |

Поверх них працює **reporting** — автоматична доставка результатів (email з вкладеннями, експорт у файли, scheduled PDF-експорт дашбордів через scenarios — див. [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md)).

Ключова ідея — **прогресія складності**: почати з chart на датасеті (хвилина роботи), зібрати кілька charts у dashboard (звіт без коду), і лише коли потрібна справжня інтерактивність або кастомний UI — будувати webapp.

---

## Charts: концепція

**(Базово)** **Chart** — це візуалізація, що будується поверх **датасета** (вкладка **Charts** будь-якого dataset) або всередині **visual analysis** (Lab). Інтерфейс повністю **drag&drop**: ви перетягуєте колонки в зони `X`, `Y`, `Color`, `Size`, `Tooltip`, а DSS сам обирає тип агрегації та малює графік.

DSS пропонує **25+ форматів** charts, згрупованих у **families**:

- **Basic charts** — bar, lines, area, stacked/grouped, pie/donut.
- **Tables** — pivot table.
- **Scatter charts** — basic, multi-pair, grouped, binned.
- **Map charts** — scatter map, density map, grid (binned) map, administrative (filled) map, geometry map.
- **Other charts** — boxplot, 2D distribution (heatmap / density), histogram, lift chart, treemap, KPI (number), radar, sankey, gauge, waterfall.

**(Середній)** Важлива відмінність двох місць, де живуть charts:

- **Charts на датасеті** — можна **опублікувати на dashboard** як insight.
- **Charts у visual analysis (Lab)** — **НЕ можна** напряму опублікувати на dashboard. Спочатку розгорніть рецепт (deploy), щоб отримати output dataset, і будуйте chart уже на ньому.

> Це одна з найчастіших точок плутанини у новачків: «Чому кнопка Publish неактивна?» — бо chart створено в Lab, а не на датасеті.

---

## Dimensions, measures, aggregations

**(Базово)** DSS розрізняє два типи ролей колонок у chart:

- **Dimensions** — категоріальні поля, за якими дані **групуються** (наприклад `country`, `product_category`, `order_date`). Це зазвичай вісь `X` або поле `Color`.
- **Measures** — числові поля, які **агрегуються** (наприклад `revenue`, `quantity`). Це зазвичай вісь `Y`.

**(Середній)** Коли measure потрапляє у зону `Y`, DSS застосовує **aggregation**. Стандартні режими:

| Aggregation | Опис |
|---|---|
| `SUM` | Сума значень |
| `AVG` (Average) | Середнє |
| `MIN` / `MAX` | Мінімум / максимум |
| `COUNT` | Кількість записів у групі |
| `COUNT DISTINCT` | Кількість унікальних значень |
| `STDDEV` / `VARIANCE` | Стандартне відхилення / дисперсія |
| `MEDIAN`, percentiles | Медіана та перцентилі (in-memory engine) |

Спеціальна measure — **`Count of records`** (псевдоколонка): рахує кількість рядків у кожній групі без вибору конкретного поля.

**(Просунуто)** Aggregation можна змінити кліком на pill (badge) measure. Для co-occurrence-аналізу одну й ту саму колонку можна покласти і в dimension, і в measure з різними агрегаціями.

---

## Binning дат і чисел

**(Середній)** Коли dimension — це **дата** або **число** з великою кількістю унікальних значень, потрібно **binning** (групування у відрізки), інакше графік розсиплеться на сотні стовпчиків.

### Date binning

Date-колонку у зоні dimension можна групувати за рівнями:

- `Year`, `Quarter`, `Month`, `Week`, `Day`, `Hour`, `Minute`;
- комбіновані (`Year + Month`, `Year + Week`…);
- **«Automatic»** — DSS сам обирає гранулярність за діапазоном дат.

### Number binning

Числову dimension можна бінувати трьома способами:

| Режим | Опис |
|---|---|
| **Fixed number of bins** | Розбити діапазон на N рівних інтервалів |
| **Fixed-size intervals** | Інтервали фіксованої ширини (наприклад по 100) |
| **Natural binning** | DSS обирає «природні» межі (для гарних round-чисел) |
| **Treat as alphanumerical / no binning** | Кожне значення = окрема категорія |

**(Просунуто)** Binning виконується **в engine** (in-memory або in-database), а не на стороні браузера — тож працює навіть на великих датасетах.

---

## Типи charts (повний довідник)

**(Базово / Середній)** Нижче — практичний довідник: який тип для якої задачі.

### Basic charts

| Тип | Призначення |
|---|---|
| **Bar / Column** | Порівняння категорій (одна dimension + одна measure) |
| **Grouped bars** | Порівняння за двома dimensions (друга — у Color) |
| **Stacked bars / 100% stacked** | Структура цілого по категоріях |
| **Lines** | Динаміка в часі (date dimension по X) |
| **Area / Stacked area** | Накопичена динаміка |
| **Pie / Donut** | Частки одного цілого (бажано ≤ 6 категорій) |

### Tables

| Тип | Призначення |
|---|---|
| **Pivot table** | Крос-таблиця: dimensions у рядках/колонках, measures у клітинках; підтримує **conditional formatting** (кольорове кодування клітинок) |

### Scatter charts

| Тип | Призначення |
|---|---|
| **Scatter (basic)** | Кореляція двох measures (X, Y); опційні `Color`, `Size` |
| **Binned scatter** | Scatter із bin'ами для дуже великих датасетів |
| **Grouped / multi-pair scatter** | Кілька серій точок |

### Map charts (geo)

> Потребують **Geo-колонки** (geopoint або geometry). Для деяких — плагін **Reverse Geocoding**.

| Тип | Призначення |
|---|---|
| **Scatter map** | Точка на кожен geopoint; опційні `Color`, `Size` |
| **Density map** | Теплова карта щільності точок з урахуванням просторової близькості |
| **Grid (binned) map** | Прямокутна сітка-агрегація по карті; опційний `Color` |
| **Administrative (filled) map** | Заливка реальних полігонів адмінодиниць (BETA; потребує Reverse Geocoding) |
| **Geometry map** | Відображення складних geometries (полігони, лінії) |

### Other charts

| Тип | Призначення |
|---|---|
| **Histogram** | Розподіл однієї числової змінної (з binning) |
| **Boxplot** | Розподіл measure по категоріях (медіана, квартилі, outliers) |
| **2D distribution / Heatmap** | Дві dimensions + інтенсивність (кольорова матриця) |
| **Density 2D** | Згладжений 2D-розподіл |
| **Treemap** | Ієрархічні частки прямокутниками |
| **KPI / Number** | Одне велике число (з conditional formatting та порівнянням) |
| **Lift chart** | Оцінка якості бінарної моделі/скорингу |
| **Radar**, **Sankey**, **Gauge**, **Waterfall** | Спеціалізовані візуалізації |

> **Heatmap** реалізується через **2D distribution** (для агрегованих measures по двох dimensions) або через **pivot table з conditional formatting**.

---

## Filters, color, tooltip, reference lines

**(Середній)**

- **Filters** — у chart є власна панель `Filters`. Два режими aggregation-фільтрів:
  - **Constraint mode** — залишити рядки за умовою (range/значення/дата);
  - **Top N mode** — показати лише топ-N категорій за measure.
- **Color** — зона `Color` приймає dimension (дискретна палітра) або measure (continuous gradient). Палітри: **discrete**, **continuous**, **diverging**; налаштовуються в **Color palettes**.
- **Tooltip** — у зону `Tooltip` можна перетягнути додаткові measures, що з'являться при наведенні.
- **Reference lines** — горизонтальні/вертикальні лінії (constant, average, median) для позначення цілей/порогів. Особливо корисні на bar/line charts.

**(Просунуто)** Conditional formatting доступний для **KPI** і **pivot table** — клітинки/число фарбуються за правилами (наприклад червоне, якщо нижче порогу).

---

## Sampling та execution engine

**(Середній)** Це **критично для продуктивності** дашбордів.

### Sampling

За замовчуванням chart рахується **не на всьому датасеті, а на sample** (вкладка **Sampling & Engine** у chart). Режими sampling:
- **Head** (перші N рядків) — швидко, але **зміщено** (перші рядки нерепрезентативні!);
- **Random** (N випадкових рядків) — репрезентативно;
- **Stratified**, **Class rebalancing** — для збалансованих вибірок;
- **Full / No sampling** — повний датасет (повільніше; має сенс із in-database engine).

> **Пастка новачка:** дефолтний **Head sampling** на відсортованому датасеті показує спотворену картину. Для аналітичних дашбордів переходьте на **Random** або **No sampling**.

### Execution engine

DSS може рахувати агрегації chart двома engine:

| Engine | Де працює | Коли |
|---|---|---|
| **DSS (in-memory)** | DSS рахує у пам'яті по sample | Файлові датасети, малі/середні обсяги |
| **In-database (SQL / Impala/Hadoop)** | Агрегація **push down** у БД через SQL `GROUP BY` | SQL-датасети; великі обсяги; точні результати на повних даних |

**(Просунуто)** In-database engine дозволяє будувати chart на **повному датасеті** без вивантаження в DSS — БД сама агрегує. Це найшвидший і найточніший шлях для великих SQL-таблиць. Вибір engine — у вкладці **Sampling & Engine**.

---

## Публікація chart на dashboard

**(Базово)** Два шляхи:

1. **З датасета:** вкладка **Charts** → побудувати chart → кнопка **Publish** → обрати dashboard (новий або існуючий) і page. Потрібен дозвіл **Read project content**.
2. **З dashboard:** в edit-режимі натиснути **`+`** → **Chart** → обрати dataset → побудувати chart прямо в tile.

**(Середній)** При публікації chart стає **insight** (об'єкт у списку **Insights** проєкту). Tile на дашборді **посилається** на цей insight. Залежно від дозволу:
- **Write** на dashboard — можна змінювати всі налаштування chart (axis, legends, filters);
- **Read** — користувач лише бачить chart, без зміни осей/даних. Зміни фільтрів зберігаються лише якщо їх збереже хтось із Write-доступом.

У tile можна керувати відображенням: показувати чи приховувати **axis**, **legends**, **tooltips**.

---

## Dashboards: концепція та структура

**(Базово)** **Dashboard** — об'єкт проєкту (меню **Dashboards**). Один проєкт може мати **багато dashboards**, а кожен dashboard — **багато pages** (сторінок/слайдів).

**Анатомія:**

```
Dashboard
 ├── Page 1
 │    ├── Tile (chart)
 │    ├── Tile (dataset table)
 │    └── Tile (text)
 ├── Page 2
 │    └── Tile (metrics)
 └── Filters panel (опційно)
```

**(Середній)** Dashboard має **два режими**:
- **Edit** (редагування) — додавання/розташування/налаштування tiles; доступний з Write-дозволом;
- **View** (перегляд) — «чистий» звіт для стейкхолдерів.

**Навігація між pages:** ліва preview-панель, стрілки навігації, клавіатурні скорочення (**⌥/Ctrl + arrows**), циклічний перехід (з останньої на першу).

**(Просунуто)** Display-налаштування:
- **Dashboard format** — розмір/пропорції полотна;
- **Tile positioning** — точне позиціонування та розмір tiles (grid);
- **Group tiles** (DSS 14+) — згрупувати tiles, щоб рухати як єдине ціле, задати фон групи, керувати spacing.

---

## Tiles: повний довідник

**(Середній)** Tile — «комірка» дашборда. Типи tiles поділяються на **simple** (статичний контент) та **insight** (посилання на об'єкт DSS).

### Simple tiles

| Tile | Призначення |
|---|---|
| **Text** | Заголовки, описи, markdown-текст |
| **Image** | Зображення (логотип, схема) |
| **Embedded page** (iframe) | Вбудована веб-сторінка / зовнішній контент |
| **Group** | Контейнер для групування інших tiles (DSS 14+) |
| **Filters** | Панель фільтрів дашборда |

### Insight tiles (один insight = один tile)

| Tile (insight) | Що показує |
|---|---|
| **Chart** | Графік з chart builder |
| **Dataset table** | Табличне представлення датасета |
| **Managed folder** | Вміст folder (файли, зображення) |
| **Metrics** | DSS metrics об'єкта (record count тощо) |
| **Data Quality** | Статус Data Quality rules (зв'язок з [09](09-scenarios-automation-metrics-checks.md)) |
| **Model report** | Звіт saved model (ROC, feature importance — див. [07](07-machine-learning-visual-ml.md)) |
| **Scenario** | Звіт run + **кнопка запуску** scenario з дашборда |
| **Jupyter notebook** | Статичний експорт notebook |
| **Webapp** | Вбудований webapp |
| **Wiki article** | Стаття проєктного wiki |
| **Activity / Comments** | Стрічка активності та коментарі |
| **Macro** | Кнопка запуску macro |

> **Scenario tile із кнопкою запуску** — потужний прийом для self-service: користувач без доступу до Flow може натиснути «Run» прямо на дашборді (наприклад «Оновити звіт»).

---

## Filters та interactivity на dashboard

**(Середній)** Окрім фільтрів усередині кожного chart, dashboard має **глобальні фільтри**:

- **Filters panel** — панель на сторінці, де читач обирає значення (наприклад `region = EU`), і всі сумісні tiles оновлюються.
- **Cross-filtering** — клік по елементу одного chart фільтрує інші tiles на сторінці.
- **URL query parameters** — фільтр можна задати **через параметр URL** (наприклад `?region=EU`). Це дозволяє надіслати колезі посилання на дашборд з **попередньо застосованим** фільтром.

**(Просунуто)** URL-параметри + scenario-генеровані посилання = персоналізовані звіти: один dashboard, різні URL для різних регіонів/клієнтів.

---

## Дозволи на перегляд (dashboard authorizations)

**(Середній)** Це **ключова безпекова концепція**, яку часто недооцінюють.

**Ownership:**
- Creator **володіє** дашбордом; редагувати можуть власник, користувачі з **Write Dashboard** та admin.

**Visibility:**
- Усі з **Read dashboards** можуть переглядати дашборди проєкту;
- **unpromoted** дашборд видно лише за прямим URL; **promoted** — з'являється у списках.

**Dashboard authorizations (критично):**
- Користувач із **тільки** правом «Read dashboards» (тобто **без** доступу до Flow/датасетів) бачить **лише ті об'єкти, які явно authorized** для дашбордів.
- Якщо аналітик додає на dashboard tile з **неавторизованим** об'єктом — DSS попереджає / просить підтвердити авторизацію.

> Тобто read-only-глядач не отримує доступ до сирих даних «через задні двері» — він бачить лише те, що автор свідомо виставив на дашборд. Це і є механізм **безпечного шерингу** з людьми, у яких немає повного доступу до проєкту.

**(Просунуто)** Адміністратор може видавати **Read dashboards** окремим групам і **надсилати посилання на дашборд email-ом** (опційно зі скриншотом).

---

## Sharing & exposing об'єктів

**(Середній)** Щоб chart/dataset з **проєкту A** показати на дашборді **проєкту B**, об'єкт треба **expose**:

1. У проєкті-джерелі: **Share to another project** (expose dataset / saved model / managed folder / тощо).
2. У цільовому проєкті об'єкт стає доступним (read-only посилання) і його можна покласти в tile.

Це той самий механізм **exposing**, що описаний у [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — dashboards його використовують для крос-проєктних звітів (наприклад центральний «executive dashboard», що тягне insights з кількох проєктів).

**(Просунуто)** Для зовнішнього шерингу:
- **Email dashboard link** (адмін) — посилання + опційний скриншот;
- **Scheduled PDF export** через scenario (див. нижче) — для тих, у кого взагалі немає доступу до DSS;
- **Public webapps / public dashboards** — для анонімного доступу (потребує явного увімкнення; обережно з безпекою).

---

## Export dashboard у PDF/images

**(Середній)** Dashboard можна експортувати у **PDF, PNG або JPEG**.

### Налаштування інстансу (адмін)

Експорт використовує **embedded Chrome browser** для рендеру (snapshotting). Перед першим використанням адміністратор має увімкнути фічу: **Setting up DSS item exports to PDF or images** (graphics export) — під час встановлення DSS завантажує актуальний Chrome.

### Ручний експорт

З view-режиму дашборда: **Actions → Export** → обрати формат і сторінки.

### Автоматичний (scheduled) експорт через scenario

**(Просунуто)** Два способи в scenario (зв'язок з [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md)):

1. **Step «Export dashboard»** — генерує файл (PDF/PNG/JPEG) і кладе у **managed folder**. Параметри: який dashboard (обов'язково, інакше step падає), тип файлу, сторінки.
2. **Mail reporter** — у налаштуваннях reporter обрати **dashboard export attachment**: scenario згенерує PDF і **надішле вкладенням** email-ом через mail channel.

Приклад: scenario щопонеділка о 7:00 будує датасети → рахує metrics → експортує «Weekly KPI» dashboard у PDF → шле менеджменту email-ом.

```
Trigger: time-based (Mon 07:00)
 ├── Step: Build datasets
 ├── Step: Compute metrics
 └── Reporter (mail): attach dashboard export "Weekly KPI" (PDF)
```

---

## Webapps: типи та архітектура

**(Середній)** **Webapp** — повноцінний веб-застосунок, **hosted by DSS** (меню **Webapps**). Використовуйте, коли chart/dashboard недостатньо: потрібна кастомна інтерактивність, форми, складна логіка, унікальна візуалізація.

DSS підтримує **п'ять типів** webapp:

| Тип | Мова | Опис |
|---|---|---|
| **Standard** | HTML/CSS/JS (+ опц. Python backend) | Frontend на JS звертається до DSS API; Python backend (Flask) для SQL/адмін-задач |
| **Shiny** | R | Інтерактивні застосунки **лише R-кодом**, без CSS/JS; повний R API DSS |
| **Bokeh** | Python | «Python-аналог Shiny»; візуалізації Python-кодом, без CSS/JS; повний Python API |
| **Dash** | Python | Фреймворк Dash (на Flask); UI + сервер Python-кодом, реактивні callbacks |
| **Streamlit** | Python | Streamlit-застосунки лише Python-кодом, реактивні до вводу |

**(Просунуто)** **Backend** усіх типів — це **Flask/Python** (або R для Shiny), що працює всередині DSS і має доступ до **Python/R API** DSS. Standard webapp — це фактично HTML/JS frontend + опційний Flask backend; решта (Dash/Streamlit/Bokeh/Shiny) — фреймворки, що генерують UI з коду.

Типові use cases:
- **Custom visualizations** — Sankey, d3.js, кастомні графіки;
- **Applicative frontends** — форма «знайти сегмент клієнта за ID»;
- **GraphQL interfaces** — програмний доступ до датасетів.

---

## Як webapp читає дані з DSS

**(Середній)** Webapp читає дані через **DSS API**, що працює в backend під ідентичністю **run-as-user**:

- **Python backend** (Dash/Streamlit/Bokeh/Standard):
  ```python
  import dataiku
  import pandas as pd

  ds = dataiku.Dataset("customers_enriched")
  df = ds.get_dataframe()          # повний/частковий датасет у pandas

  # Прямий SQL-запит до БД
  from dataiku.core.sql import SQLExecutor2
  executor = SQLExecutor2(connection="my_postgres")
  rows = executor.query_to_df("SELECT region, SUM(revenue) FROM sales GROUP BY region")
  ```
- **Standard webapp (JS frontend)** — звертається до **Dataiku JS API** (`getWebAppBackend`, AJAX до Flask endpoint), а Flask backend уже читає датасети Python-кодом.
- **Shiny (R)** — `dku` R API (`dkuReadDataset(...)`).

**(Просунуто)** Backend може робити SQL-запити, читати/писати датасети, виконувати адмін-задачі — **за умови, що run-as-user має достатні дозволи**. Великі датасети краще не вантажити цілком у pandas — фільтруйте/агрегуйте SQL-ом на боці БД.

---

## Webapp security та run-as-user

**(Просунуто)** **Хто виконує backend-код webapp?**

- Backend працює як **єдиний користувач** — **run-as-user**. За замовчуванням це **останній користувач, який змінював webapp**. Admin може змінити run-as-user у налаштуваннях webapp.
- Тобто всі глядачі бачать дані під **однією** ідентичністю backend — навіть якщо самі мають менше прав. Це треба усвідомлювати: webapp може «показати» дані глядачу, до яких він напряму не має доступу.

**Ідентифікація кінцевого користувача** (хто саме відкрив webapp) — через headers:
- Standard — request headers;
- Bokeh — `get_session_headers()` + `get_auth_info_from_browser_headers()`;
- Dash — headers з Flask request у callback;
- Streamlit — `st.context.headers` (v1.37+);
- Shiny — R API auth.

**Impersonation** — `WebappImpersonationContext()` дозволяє виконувати API-виклики **від імені кінцевого користувача** (а не run-as-user). Потребує:
- група run-as-user має дозвіл **«Impersonation in webapps»**;
- група кінцевого користувача явно дозволена.

**Authentication:** за замовчуванням webapp **вимагає автентифікації**. Опція **public webapp** дає ширший/анонімний доступ — використовуйте **обережно** (бо backend все одно працює під run-as-user з його правами).

**Exposing на dashboard:** webapp публікується як **insight** і вставляється у **Webapp tile**, або шериться напряму (**Share to another project**).

---

## Code Studios для розробки webapp

**(Просунуто)** **Code Studio** — повноцінне IDE-середовище всередині DSS (VS Code, Jupyter тощо) у контейнері. Воно дозволяє розробляти webapp, **які DSS не керує нативно** (Flask, Bokeh, Shiny, Dash керуються нативно; решту — Gradio, кастомні Streamlit-стеки тощо — через Code Studio).

Процес публікації webapp із Code Studio:
1. **Code Studio template** — образ контейнера з кастомізаціями та **entrypoints** (мінімум один для webapp, зазвичай ще один для IDE).
2. Інстанціювати Code Studio з template, написати код застосунку.
3. **Publish** (у action-панелі Code Studio) — створює webapp.

Ключові деталі:
- Webapp і Code Studio **поділяють той самий набір файлів**. Після редагування Code Studio **не треба** публікувати повторно — webapp завжди вказує на актуальний стан.
- Webapp має **read-only** доступ до файлів Code Studio (не може писати на сервер).
- На Kubernetes рекомендовано **TCP probing**; експозиція — через **port forwarding** за Nginx-prefix.

Code Studio — рекомендований шлях для команд, що хочуть **версіювати webapp-код у Git** і використовувати звичні IDE-інструменти.

---

## Reporting: експорт даних клієнтам

**(Середній)** «Reporting» у широкому сенсі — **доставка результатів** одержувачам. Способи в DSS:

| Спосіб | Як |
|---|---|
| **Email з вкладенням** | Mail reporter у scenario: attach dataset export (CSV/Excel) або dashboard PDF |
| **Email з посиланням** | Посилання на dashboard / public link (без важких вкладень) |
| **Export dataset → file** | Експорт датасета у файл (CSV, Excel) вручну або **Export to folder** recipe |
| **Managed folder** | Scenario пише файли у folder; клієнт завантажує (або folder на S3/SFTP) |
| **Dashboard PDF (scheduled)** | Step «Export dashboard» + mail reporter (див. вище) |
| **API / data delivery** | Програмний доступ через API node або public webapp/GraphQL |

**(Просунуто)** Типовий «report-орієнтований scenario» (зв'язок з [09](09-scenarios-automation-metrics-checks.md)):

```
Trigger (schedule / after upstream scenario)
 ├── Build report dataset
 ├── Run checks / Data Quality (qgate)
 ├── Export dataset → managed folder (Excel)
 ├── Export dashboard → PDF
 └── Mail reporter:
       to: client@example.com
       attach: Excel + dashboard PDF
       body: формула з KPI-значеннями (metrics)
```

> Найкраща практика: великі дані — **посиланням/у folder**, а не важким вкладенням; в email-тіло вставляйте кілька **ключових metrics** через DSS Formula, щоб одержувач бачив суть без відкривання файлу.

---

## RAG / LLM-powered apps та Dataiku Answers

**(Середній)** Окрім класичних webapp, DSS дозволяє будувати **LLM-powered застосунки**:

- **RAG-чатботи** поверх ваших документів (Knowledge Banks, retrieval) — будуються як webapp (Streamlit/Dash) поверх **LLM Mesh**.
- **Dataiku Answers** — готовий, конфігурований **chat-UI / no-code RAG-застосунок** для безпечного Q&A над корпоративними даними з цитуванням джерел.

Детально — у [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md). Тут лише фіксуємо: такі застосунки **публікуються як webapp** і шеряться/вставляються на dashboard так само, як звичайні webapps.

---

## Приклади (end-to-end)

### Приклад 1 (Базово): KPI-дашборд за 10 хвилин

1. На датасеті `sales` → вкладка **Charts**:
   - Chart 1: **Lines**, X = `order_date` (binning Month), Y = `SUM(revenue)`.
   - Chart 2: **Bar**, X = `region`, Y = `SUM(revenue)`, Color = `region`.
   - Chart 3: **KPI**, measure = `SUM(revenue)`, conditional formatting (зелене, якщо > target).
2. **Publish** усіх трьох → новий dashboard «Sales Overview».
3. У dashboard додати **Text tile** із заголовком і **Filters panel** (`region`, `order_date`).
4. Перейти у **View** — готовий інтерактивний звіт.

### Приклад 2 (Середній): scheduled PDF-звіт менеджменту

```
Scenario "Weekly Sales Report"
 Trigger: time-based, Mon 07:00
 Steps:
   1. Build dataset "sales_weekly"  (smart rebuild)
   2. Compute metrics on "sales_weekly"
   3. Export dashboard "Sales Overview" → PDF (managed folder "reports")
 Reporter (mail channel "corp-smtp"):
   to: leadership@corp.com
   subject: "Weekly Sales — ${dku_flow_state}"
   attach: dashboard export "Sales Overview" (PDF)
   body: "Revenue this week: ${sales_weekly.revenue_sum}"
```

### Приклад 3 (Просунуто): Streamlit webapp із прямим SQL

```python
# Streamlit webapp (backend run-as-user з доступом до connection "warehouse")
import streamlit as st
import dataiku
from dataiku.core.sql import SQLExecutor2

st.title("Customer Lookup")
cust_id = st.text_input("Customer ID")

if cust_id:
    safe_id = cust_id.replace("'", "")          # проста санітизація
    executor = SQLExecutor2(connection="warehouse")
    df = executor.query_to_df(
        f"SELECT * FROM customers WHERE id = '{safe_id}'"
    )
    st.dataframe(df)
```

Публікація: **Webapps → New → Streamlit** (або через **Code Studio** для версіювання в Git) → налаштувати **run-as-user** з мінімально необхідними правами → **Publish** як insight → вставити **Webapp tile** на dashboard.

---

## Best practices

**(Середній / Просунуто)**

1. **Sampling під задачу.** Для аналітичних дашбордів уникайте дефолтного **Head sampling** на відсортованих даних — беріть **Random** або переходьте на **in-database engine** (повні точні дані без вивантаження).
2. **In-database engine для великих SQL-датасетів.** Push-down `GROUP BY` у БД швидший і точніший, ніж агрегація in-memory по sample.
3. **Pre-aggregation для важких дашбордів.** Якщо chart агрегує мільйони рядків — зробіть **окремий агрегований dataset** (Group recipe) у Flow і будуйте chart на ньому. Дашборд відкривається миттєво.
4. **Chart vs webapp — правильний вибір.** Стандартна візуалізація/KPI → **chart на dashboard** (no-code, дешево підтримувати). Кастомний UI, форми, складна інтерактивність, LLM-чат → **webapp**. Не будуйте webapp там, де вистачає chart.
5. **Безпека публічних посилань.** Public webapp/dashboard працює під **run-as-user**; не давайте run-as-user зайвих прав. Для зовнішніх — публікуйте лише **agreggated** дані, не сирі PII.
6. **Dashboard authorizations свідомо.** Перевіряйте, які об'єкти authorized для read-only глядачів — щоб не «протекли» сирі датасети тим, хто має лише Read dashboards.
7. **Версіюйте webapp-код у Git.** Розробляйте через **Code Studio** + project version control; webapp є частиною bundle і деплоїться на Automation node.
8. **Reporting: посилання замість важких вкладень.** Великі CSV/Excel — у managed folder / на SFTP, а в email — посилання + кілька ключових metrics у тілі.
9. **Один dashboard — багато аудиторій через URL-параметри.** Замість дублювання дашбордів використовуйте **URL query parameters** для попередньо застосованих фільтрів.
10. **Group tiles (DSS 14+) для чистого layout.** Групуйте споріднені tiles — легше рухати, фон/spacing під контролем.

---

## Типові помилки

**(Базово / Середній)**

1. **«Publish неактивний».** Chart створено у **visual analysis (Lab)** — його не можна публікувати напряму; будуйте chart на **датасеті** (deploy рецепт спершу).
2. **Спотворені цифри через Head sampling.** Дефолтний sample = перші N рядків; на відсортованому датасеті це нерепрезентативно. Перемкніть на **Random** / **No sampling**.
3. **Повільний дашборд на сирому великому датасеті.** Не агрегуйте мільйони рядків у chart — зробіть **pre-aggregated dataset** або in-database engine.
4. **Pie chart на 20 категоріях.** Нечитабельно — використовуйте **bar** або **Top N filter**.
5. **Забути увімкнути graphics export.** Step «Export dashboard» / PDF-вкладення впадуть, якщо адмін не налаштував **DSS item exports** (embedded Chrome).
6. **Export dashboard step без вказаного dashboard.** Step падає, якщо dashboard не заданий явно.
7. **Витік даних через dashboard authorizations.** Додали tile з неавторизованим об'єктом → read-only глядачі або нічого не бачать, або (за неуважного підтвердження) отримують доступ до зайвого. Перевіряйте authorizations.
8. **Public webapp під привілейованим run-as-user.** Анонімні користувачі фактично отримують права run-as-user. Давайте мінімальні дозволи; не виставляйте PII.
9. **Будувати webapp там, де вистачає chart.** Зайва складність підтримки. Webapp — лише коли потрібна справжня кастомність/інтерактивність.
10. **Не версіювати webapp-код.** Webapp без Git/Code Studio складно ревʼювити та відкочувати. Тримайте код під version control.
11. **Важкі email-вкладення.** Великі CSV «забивають» пошту й ліміти. Шеріть посиланням / через folder.
12. **Очікувати, що chart у Lab оновиться на дашборді.** На дашборді живуть лише **dataset charts** (insights); Lab-charts туди не потрапляють.

---

## Перехресні посилання

- [01-overview-architecture.md](01-overview-architecture.md) — Design / Automation / API nodes; де хостяться webapps та виконуються scheduled-експорти.
- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — **exposing об'єктів** між проєктами (основа крос-проєктних дашбордів), insights, project permissions.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — datasets та connections (джерело charts і даних для webapp; SQL connections для in-database engine).
- [04-visual-recipes.md](04-visual-recipes.md) — Group recipe для **pre-aggregation** під швидкі дашборди; Export recipe для file-delivery.
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — підготовка/збагачення даних перед візуалізацією; geo-обробка для map charts.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — code envs для webapp backend; notebooks (export на dashboard); `dataiku` API.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — **Model report insight** на дашборді; saved models у webapp.
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — **RAG/LLM-powered webapps** та **Dataiku Answers**; LLM Mesh.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — **scheduled dashboard export step**, mail reporters, **Metrics/Data Quality insights**, scenario tile з кнопкою запуску.
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — деплой webapps у bundle на Automation node; public access та інфраструктура.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — `dataikuapi`: програмне керування dashboards, insights, webapps.
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — **dashboard authorizations**, webapp **run-as-user** та impersonation, public access policy, права на перегляд.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — **chart-плагіни** та **webapp-плагіни**, Dataiku Applications (de-факто «упакований» dashboard+webapp).
- [15-best-practices.md](15-best-practices.md) — зведені best practices.
- [16-glossary.md](16-glossary.md) — глосарій термінів (insight, tile, run-as-user, sampling, engine…).

---

## Джерела (офіційна документація Dataiku)

- Dashboards (overview): https://doc.dataiku.com/dss/latest/dashboards/index.html
- Dashboard concepts: https://doc.dataiku.com/dss/latest/dashboards/concepts.html
- Chart insight: https://doc.dataiku.com/dss/latest/dashboards/insights/chart.html
- Insights reference: https://doc.dataiku.com/dss/latest/dashboards/insights/index.html
- Data Quality insight: https://doc.dataiku.com/dss/latest/dashboards/insights/data-quality.html
- Exporting dashboards to PDF or images: https://doc.dataiku.com/dss/latest/dashboards/exports.html
- Setting up DSS item exports (graphics export): https://doc.dataiku.com/dss/latest/installation/custom/graphics-export.html
- Charts (visualization overview): https://doc.dataiku.com/dss/latest/visualization/index.html
- Scatter charts: https://doc.dataiku.com/dss/latest/visualization/charts-scatters.html
- Map charts: https://doc.dataiku.com/dss/latest/visualization/charts-maps.html
- Other charts (boxplot, treemap, KPI, lift…): https://doc.dataiku.com/dss/latest/visualization/charts-other.html
- Webapps (overview): https://doc.dataiku.com/dss/latest/webapps/index.html
- Introduction to DSS webapps: https://doc.dataiku.com/dss/latest/webapps/introduction.html
- Webapp security (run-as-user, impersonation): https://doc.dataiku.com/dss/latest/webapps/security.html
- Bokeh web apps: https://doc.dataiku.com/dss/latest/webapps/bokeh.html
- Streamlit web apps: https://doc.dataiku.com/dss/latest/webapps/streamlit.html
- Code Studios as webapps: https://doc.dataiku.com/dss/latest/code-studios/code-studios-as-webapps.html
- Application tiles: https://doc.dataiku.com/dss/latest/applications/tiles.html
- Scenario steps (Export dashboard, reporters): https://doc.dataiku.com/dss/latest/scenarios/steps.html
- Knowledge Base — Tutorial: Build reports with dashboards: https://knowledge.dataiku.com/latest/visualize-data/dashboards/tutorial-build-reports-with-dashboards.html
- Knowledge Base — Charts: https://knowledge.dataiku.com/latest/visualize-data/charts/index.html
- Knowledge Base — Concept: Dashboards: https://knowledge.dataiku.com/latest/courses/lab-to-flow/reporting/concept-dashboards.html
- Developer Guide: https://developer.dataiku.com/latest/
