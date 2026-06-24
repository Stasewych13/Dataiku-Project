# 05. Data Preparation: Prepare recipe

> Рівень: новачок → досвідчений
> Версії DSS: 13.x / 14.x
> Джерела: [doc.dataiku.com/dss/latest](https://doc.dataiku.com/dss/latest/), [knowledge.dataiku.com](https://knowledge.dataiku.com/), Dataiku Academy

---

## Вступ

**Prepare recipe** (рецепт підготовки) — це центральний візуальний інструмент Dataiku для очищення (cleansing), нормалізації (normalization) та збагачення (enrichment) даних **без написання коду**. Це найбільш універсальний і найчастіше використовуваний візуальний рецепт у Dataiku, який аналітик відкриває для себе одним із перших.

Prepare recipe вирізняється трьома ключовими властивостями:

1. **Візуальний і інтерактивний** — ви бачите результат кожного перетворення миттєво, на зразку (sample) даних.
2. **Складається зі скрипта (Script)** — впорядкованої послідовності кроків (steps), де кожен крок є екземпляром одного з ~100 процесорів (processors).
3. **Масштабований** — на етапі дизайну рецепт працює зі зразком, а під час запуску (Run) застосовується до всього датасета з вибором рушія виконання (engine: DSS / Spark / SQL).

Цей документ детально розкриває структуру скрипта, бібліотеку процесорів за категоріями, систему значень (meanings / semantic types), мову формул (DSS formula language, неофіційно — GREL), семплінг, рушії виконання та інструмент Analyze.

> Перед читанням бажано ознайомитися з [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) (поняття Flow та recipe) і [04-visual-recipes.md](04-visual-recipes.md) (огляд візуальних рецептів загалом).

---

## Концепції

### Script і steps

Серцем Prepare recipe є **Script** — впорядкований згори донизу список **steps** (кроків). Кожен крок:

- є екземпляром одного **процесора** з бібліотеки (наприклад, `Parse to standard date format`), коментарем або групою;
- зазвичай оперує однією колонкою чи набором колонок;
- може бути **перевпорядкований** (drag), **вимкнений/увімкнений** (toggle on/off), **згрупований**, скопійований і вставлений.

Скрипт виконується послідовно: вихід кроку N є входом кроку N+1. Це робить **порядок кроків критичним** — наприклад, спочатку треба розпарсити дату, а потім обчислювати різницю між датами.

Нові кроки додаються кнопкою **+ Add a New Step** унизу панелі **Script**, яка відкриває бібліотеку процесорів із пошуком.

### Інтерфейс дизайну (design view)

- **Панель Script** (ліворуч) — список усіх кроків; додавання, перевпорядкування, вимкнення.
- **Превʼю зразка** (центр/праворуч) — таблиця з даними, що показує ефект кроків наживо.
- **Два режими перегляду зразка:**
  - **Table view** — класична сітка рядки/колонки.
  - **Columns view** — орієнтований на колонки перегляд, що дозволяє застосовувати кроки одразу до кількох колонок.
- **Меню заголовка колонки** — клік по заголовку дає **контекстні підказки кроків** на основі **meaning** колонки (наприклад, пропозиції парсингу для дат, зміна регістру для тексту).
- **Data quality bar** — кольорова смужка під заголовком кожної колонки: **valid** (зелений), **invalid** (червоний), **empty** (сірий). Керується **storage type** і **meaning** колонки.
- **Analyze window** — вікно статистики/розподілів колонки, з якого можна одразу породжувати кроки підготовки.

Коли ви задоволені скриптом, натискаєте **Run** — і Dataiku застосовує весь Script до **всього** вхідного датасета, матеріалізуючи **вихідний датасет** у Flow.

### Prepare recipe (у Flow) vs Visual Analysis у Lab

Той самий редактор скрипта існує у двох місцях, і важливо розуміти різницю.

| Аспект | **Prepare recipe** (у Flow) | **Visual Analysis / Script** (у Lab) |
|---|---|---|
| Розташування | у **Flow** | у **Lab** (привʼязаний до датасета) |
| Вихід | створює **постійний output dataset** у Flow | **немає постійного виходу** — суто експериментальний |
| Призначення | **продакшн**, стабільність, повторне використання, автоматизація | **експерименти**, feature engineering, часто перед ML |
| Додаткові можливості | лише **Script** | має вкладки **Script** AND **Models** (Lab може будувати ML-моделі) |
| Вплив на Flow | додає обʼєкти у Flow | не захаращує Flow під час експериментів |

**Коли що використовувати:**

- **Lab visual analysis** — коли ви **експериментуєте** з перетвореннями, ще не хочете захаращувати Flow, працюєте паралельно з колегами, або наявний робочий процес уже в продакшні.
- **Prepare recipe у Flow** — коли підхід до підготовки **фіналізовано** і готовий до регулярного/продакшн-використання та автоматизації.

**Як розгорнути Lab-скрипт у Flow як Prepare recipe:**

1. У вкладці **Script** Lab-аналізу натисніть кнопку **Deploy Script**.
2. Dataiku конвертує Lab Script у **Prepare recipe** у Flow, додаючи його до конвеєра вихідного датасета і створюючи новий output dataset.
3. Натисніть **Run** на новому рецепті, щоб застосувати всі кроки до **всього** вхідного датасета (Lab працював лише зі зразком).

> Детальніше про Lab та ML-аналізи — у [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md).

---

## Бібліотека процесорів (Processors library)

DSS постачає ~100 процесорів. Нижче — організація за категоріями з **точними оригінальними назвами** (саме так вони відображаються у пошуку бібліотеки) та коротким описом найважливіших.

> Повний референс: [Processors reference — DSS 14](https://doc.dataiku.com/dss/latest/preparation/processors/index.html). Бібліотека практично **ідентична між 13.x і 14.x**.

### Cleansing / normalization (очищення / нормалізація)

- **Remove rows where cell is empty** — видаляє рядки з порожнім значенням у колонці (має опцію keep/remove).
- **Fill empty cells with fixed value** — заповнює порожні комірки константою.
- **Fill empty cells with previous/next value** — forward/backward fill.
- **Impute with computed value** — заповнення порожніх обчисленою статистикою (mean / median / most frequent).
- **Coalesce** — повертає перше непорожнє значення серед кількох колонок.
- **Simplify text** — нормалізація тексту (lowercase, прибирання діакритики/акцентів, нормалізація пробілів).
- **Transform string** — рядкові перетворення; саме цей процесор робить trim / **remove whitespace**, uppercase/lowercase тощо.
- **Normalize measure** — нормалізація числової міри до стандартної одиниці.
- **Round numbers** — round / floor / ceil.
- **Force numerical range** — обмеження (clamp) значень діапазоном min/max (також породжується при обробці викидів через Analyze).
- **Merge long-tail values** / **Group long-tail values** — згортання рідкісних категоріальних значень у єдиний кошик.
- **Negate boolean value** — інвертує boolean.

### Date parsing & formatting (парсинг і форматування дат)

- **Parse to standard date format** — парсить рядки у стандартний ISO-8601 тип дати DSS (це і є «Parse date»).
- **Format date with custom format** — рендерить дату у власний рядковий формат.
- **Compute difference between dates** — різниця між двома датами в обраній одиниці.
- **Extract date elements** — витягає компоненти (year, month, day, hour, week, day-of-week тощо). Це і є «Extract date components».
- **Convert a UNIX timestamp to a date** — epoch → дата.
- **Filter rows/cells on date** — фільтр за датою.
- **Flag rows/cells on date range** — прапорець для рядків у/поза діапазоном дат.
- **Flag holidays** — позначення свят за календарями.

> Немає окремого процесора **«Increment date»** — інкремент дати робиться через **Formula** (функція `inc(...)`) або через `Compute difference between dates`. «Split date» реалізується через **Extract date elements**.

### Split / extract / replace (розбиття / витяг / заміна)

- **Split column** — розбиває колонку за роздільником на кілька колонок або рядків.
- **Extract with regular expression** — захоплює групи (capture groups) regex у нові колонки.
- **Find and replace** — заміна (режими: literal, regular expression, whole-cell).
- **Extract numbers** — витягає числа з тексту.
- **Extract ngrams** — генерує n-грами з тексту.
- **Split into chunks** — розбиває текст на фрагменти фіксованого розміру.
- **Split invalid cells into another column** — переносить комірки, що не проходять meaning, в окрему колонку.

> Окремого процесора з назвою «Replace» немає — заміна це **Find and replace**.

### Regex-процесори

- **Extract with regular expression** — regex-захоплення в колонки.
- **Find and replace** — підтримує режим **Regular expression**.
- **Extract with grok** — витяг за grok-патернами (надмножина regex, стиль парсингу логів).
- **Extract with JSONPath** — витяг із JSON через JSONPath.
- Формульні процесори (**Filter rows/cells with formula**, **Flag rows with formula**) можуть використовувати regex через функції формул (`match`, `replace`, `split`).

### Formula step (крок Formula)

- **Formula** — найгнучкіший процесор; spreadsheet-подібна мова виразів для обчислення нової або перезапису наявної колонки. Багата бібліотека функцій (string, math, date, conditional). Редактор має **живу перевірку синтаксису, превʼю результату, повний довідник мови та приклади**.
- Споріднені: **Create if, then, else statements** (візуальний if-then-else), **Switch case**, **Python function** (рядковий Python).

### Filter / flag / keep / delete rows (фільтрація рядків)

Загальне правило: усі **Filter**-процесори мають перемикач **Keep / Remove** для рядків, що збігаються. **Flag**-процесори **додають індикаторну колонку**, а не видаляють рядки.

- **Filter rows/cells on value** — keep/remove за значенням (рівність/містить).
- **Filter rows/cells on numerical range** — keep/remove за числовим діапазоном.
- **Filter rows/cells on date** — keep/remove за датою.
- **Filter rows/cells with formula** — keep/remove за виразом-формулою.
- **Filter invalid rows/cells** — keep/remove за валідністю щодо meaning (див. розділ Meanings).
- **Flag rows with formula** — додає boolean-прапорець за формулою.
- **Flag rows on value** / **Flag rows on numerical range** / **Flag rows/cells on date range** / **Flag invalid rows** — позначення без видалення.
- **Remove rows where cell is empty** — видалити/залишити рядки з порожньою коміркою.

### Column operations (операції над колонками)

- **Rename columns** — перейменування однієї чи кількох колонок.
- **Delete/Keep columns by name** — залишити або видалити набір колонок.
- **Move columns** — перевпорядкування (move to start/end/before/after).
- **Copy column** — дублювання колонки.
- **Concatenate columns** — обʼєднання кількох колонок в одну (з роздільником).
- **Split column** — розбиття однієї колонки на кілька.
- **Column Pseudonymization** — псевдонімізація/анонімізація значень (хешування/токенізація для приватності).
- **Nest columns** — пакування кількох колонок в один обʼєкт/JSON.
- **Unnest object (flatten JSON)** — розкриття обʼєктної/JSON-колонки у кілька колонок.

### Reshaping: pivot / unpivot / fold / unfold (зміна форми)

- **Pivot** — pivot рядків у колонки (spreadsheet-pivot з агрегацією).
- **Fold multiple columns** — fold (unpivot) кількох колонок у пари key/value.
- **Fold multiple columns by pattern** — fold колонок, обраних за патерном імені.
- **Fold object keys** — fold ключів обʼєктної колонки в рядки.
- **Unfold** — unfold (spread) значень однієї колонки в кілька колонок.
- **Triggered unfold** — unfold, обумовлений значенням.
- **Transpose rows to columns** — транспонування (рядки ↔ колонки).
- **Split and fold** — розбити колонку, потім fold частин у рядки.
- **Split and unfold** — розбити колонку, потім unfold частин у колонки.

> Термінологія: «unpivot» = **Fold multiple columns**; «transpose» = **Transpose rows to columns**.

### Geo-процесори (геообробка)

- **Create GeoPoint from lat/lon** — створити geopoint із широти/довготи.
- **Extract lat/lon from GeoPoint** — навпаки, отримати lat/lon із geopoint.
- **Extract from geo column** — витяг атрибутів із geometry/geo-колонки.
- **Change coordinates system** — репроєкція між системами координат (наприклад, WGS84 ↔ інші).
- **Compute distance between geospatial objects** — відстані між гео-обʼєктами.
- **Create area around a geopoint** — буфер/зона навколо точки (кейс «zoom to area»).
- **Geo join with other dataset (memory-based)** — просторовий/гео-join з іншим датасетом.
- **Resolve GeoIP** — резолвинг IP-адреси в географічну локацію.
- **Enrich from French department** / **Enrich from French postcode** — France-специфічне збагачення.

> У core-бібліотеці **немає** нативного «Geocode / Reverse geocode» (адреса ↔ координати) — геокодування адрес постачається через **плагіни** (наприклад, Geocoder / Reverse Geocoding plugin). **Resolve GeoIP** працює з IP, **Create area around a geopoint** покриває кейс буфера/зони. Детальніше про геообробку формулами — `geoContains`, `geoDistance`, `geoBuffer` (geoformula).

### Text-процесори (обробка тексту)

- **Tokenize text** — розбиває текст на токени.
- **Simplify text** — нормалізація (lowercase, прибирання акцентів/пунктуації).
- **Extract ngrams** — генерує n-грами (це і є «split into n-grams»).
- **Count occurrences** — рахує входження патерну/підрядка в тексті.
- **Transform string** — case / whitespace / рядкові перетворення.
- **Split into chunks** — фрагментація тексту.
- **Translate values using meaning** — мапінг значень через нормалізацію meaning.

> Окремого процесора «Hash text» немає — хешування через **Column Pseudonymization** (режими хешування) або **Formula** (`md5`, `sha1`, `sha256`).

### JSON / array / nested operations

- **Unnest object (flatten JSON)** — flatten JSON/обʼєкта у колонки.
- **Nest columns** — навпаки, упакувати колонки в обʼєкт.
- **Extract from array** — витягти елементи з array-колонки.
- **Fold an array** — fold елементів масиву в рядки.
- **Unfold an array** — unfold елементів масиву в колонки.
- **Sort array** — сортування елементів масиву.
- **Concatenate JSON arrays** — конкатенація кількох JSON-масивів.
- **Zip JSON arrays** — поелементне «зіпування» кількох масивів.
- **Fold object keys** — fold ключів обʼєкта в рядки.
- **Extract with JSONPath** — витяг значень із JSON через JSONPath.

> «Split array» ≈ **Extract from array** / **Unfold an array**; «build array» ≈ **Concatenate JSON arrays** / **Zip JSON arrays**.

### Currency & units (валюти й одиниці)

- **Convert currencies** — конвертація сум між валютами.
- **Split currencies in column** — відокремити символ/код валюти від числа.
- **Numeric base conversion** — конвертація числової системи числення.
- **Discretize (bin) numerical values** — біннінг числових значень.
- **Normalize measure** — нормалізація одиниць виміру.

> Узагальненої «units conversion» поза переліченим немає — інші конверсії одиниць робляться через **Formula**.

### Інші корисні процесори

- **Split e-mail addresses** — розбити email на local-part / domain.
- **Split URL** — парсинг URL (protocol, host, port…).
- **Split HTTP Query String** — парсинг параметрів query-string.
- **Classify User-Agent** — парсинг User-Agent у OS/browser/device.
- **Join with other dataset (memory-based)** / **Fuzzy join with other dataset (memory-based)** — join-збагачення всередині рецепта.
- **Enrich with build context** / **Enrich with record context** — додає метадані build/record.

### Семплінг — це не процесор

У Prepare recipe **немає процесора семплінгу**. Семплінг — це **налаштування рівня рецепта** (вкладка/секція **Sampling**), і він керує лише інтерактивним превʼю. Щоб фізично створити семпльований датасет у Flow, використовується окремий рецепт **Sample / Filter** (див. [04-visual-recipes.md](04-visual-recipes.md)).

---

## Meanings (semantic types / значення)

### Meaning vs Storage type

Кожна колонка має **два типи**:

- **Storage type** — *технічний* тип, що каже бекенду, *як* фізично зберігаються дані. Строгий (SQL `int` не вмістить decimal). Перелік: `string, int, bigint, smallint, tinyint, float, double, boolean, date` (три варіанти datetime — «Datetime with zone», «Datetime no zone», «Date only»), `geopoint, geometry, array, map, object`.
- **Meaning (semantic type)** — *семантичний* ярлик, що каже, *що* дані означають (country, email, URL…). Meaning **не обмежує** зберігання, натомість **валідує кожну комірку**: вона **valid** або **invalid** щодо meaning. Storage type і meaning навмисно **розвʼязані**, тож DSS зберігає багате meaning навіть за наявності невалідних комірок.

### Автодетекція

Meanings **автоматично виводяться зі вмісту колонки** — DSS обирає meaning, для якого валідна більшість рядків (наприклад, виводить `URL`, якщо більшість рядків валідні для URL). Користувач може перевизначити meaning кліком по індикатору; **примусово заданий** meaning показує **іконку замка**. Вибір **Auto-detect** повертає інференс.

### Color coding — data quality bar

Під заголовком кожної колонки — **кольорова смужка**, що показує, які рядки валідні для meaning колонки. Три категорії:

- **Зелений = valid** (стан «OK» — значення відповідає meaning).
- **Червоний = invalid** (стан «NOK» — значення не відповідає meaning).
- **Сірий / порожній = empty** (missing/null — окремий кошик, не invalid).

Клік по заголовку → **Filter** дає чекбокси, щоб сфокусуватися на valid / invalid / empty рядках.

### Вбудовані meanings

**Basic:** Text, Decimal, Integer, Boolean, Datetime with zone, Datetime no zone, Date only, Object, Array, Natural language.

**Geospatial:** Latitude, Longitude, Geopoint (вкл. WKT), Geometry (WKT), Country (англ. назви + ISO-коди), US State.

**Web-specific:** IP Address (IPv4 + IPv6), URL, HTTP Query String, User Agent, E-Mail address.

**Other:** Currency code, Country code та інші (повний перелік — на сторінці [meanings-list](https://doc.dataiku.com/dss/latest/schemas/meanings-list.html)).

### User-defined meanings

Створюються у **Administration → Settings → Meanings**. Чотири типи:

1. **Declarative** — лише документація. **Без валідації, без автодетекції** — просто семантичний ярлик.
2. **Values list** — фіксований список дозволених значень; комірки валідуються за списком. Нормалізація: **exact match**, **case-insensitive** або **accent-insensitive**.
3. **Values mapping** — пари key→label, кожна з «value in storage» (key) та «label». Значення валідне, якщо збігається з key АБО label. Спеціальний процесор у Prepare може робити заміни (storage ⇄ label). Приклад: `"0", "1", "-9"` → `"no", "yes", "no answer"`.
4. **Pattern** — значення валідне, якщо відповідає наданому **Java-сумісному regex**; перемикач case-sensitive/insensitive.

> Документація **не рекомендує вмикати auto-detection** для user-defined meanings — це може придушити розпізнавання вбудованих meanings і знизити продуктивність. Типове використання — ручне призначення колонці.

### Validity та обробка invalid/non-matching значень

Валідність — це суто функція правил meaning (для `Date only` усе, що не парситься як `yyyy-MM-dd`, — invalid; для `Pattern` усе, що не відповідає regex). Empty/null — третій кошик.

Ключовий процесор — **Filter invalid rows/cells** з чотирма режимами:

- **Keep matching rows only**
- **Remove matching rows**
- **Clear content of matching cells**
- **Clear content of non-matching cells** ← це і є «очистити комірки, що не відповідають meaning».

Область: одна колонка, явний список, regex-match, або всі колонки; для кількох колонок обирається логіка **ALL** vs **ANY**.

> На виході: **за замовчуванням DSS відкидає невалідні дані** при збереженні у output dataset (значення, що не вписуються у storage type, відкидаються/обнуляються). У пікері storage type білі типи «можливі», червоні «згенерують помилки/попередження».

---

## GREL — мова формул Dataiku (DSS formula language)

> **Уточнення термінології:** офіційно Dataiku **не** бренди мову як «GREL» (General Refine Expression Language) — це **«DSS formula language»** / **«Dataiku formulas»**. «GREL» — неформальне ком'юніті-скорочення з часів OpenRefine/Google Refine. Мова застосовується **рядок за рядком**, як у spreadsheet.

**Де використовується:** процесор **Formula** у Prepare; фільтрація у **Filtering recipe**; pre/post-фільтри у grouping/window/join/stack рецептах; train/test extracts у ML; часткові extracts у Python/JS та Public API.

### Синтаксис

**Посилання на колонки:**

- **Bare name** — `mycolumn` (має починатися з літери; літери/цифри/`_`), напр. `N1 + 4`.
- **`val("column name")`** — значення з автодетекцією типу; обовʼязкове для імен із пробілами/спецсимволами. Сигнатура: `val(object o, [string defaultValue], [number offset])`.
- **`strval("column name")`** — рядкове подання (запобігає числовому coercion, зберігає `"012345"`).
- **`numval("column name")`** — числове значення.
- Аргумент **`offset`** читає попередній рядок (`1` = попередній, `0` = поточний). **Обмеження: лише в Prepare recipe і лише на DSS-рушії** (не пушиться в SQL).

**Оператори:**

- Арифметика: `+ - * /`, `//` (цілочисельне ділення), `%` (modulo).
- Порівняння (повертають boolean; **єдині, що працюють із датами**): `> >= < <= == !=`.
- Boolean: `&&` (AND), `||` (OR).
- **Конкатенація рядків: `+` працює на DSS-рушії, але НЕ на більшості SQL-рушіїв — для портативності використовуйте `concat()`.**

**Доступ до array/object:** `array[0]`, `object["key"]`, `object.key`; **method-chaining:** `foo.replace('a','b').trim().length()`.

### Основні функції за категоріями

**String:** `length(o)`, `toUppercase(s)`/`toLowercase(s)`/`toTitlecase(s)`, `substring(o, from, [to])`, `replace(s, find/regex, repl)`, `contains(s, frag)`, `startsWith(s, sub)`/`endsWith(s, tail)`, `trim(s)` (alias `strip`), `split(s, sep/regex)`, `concat((a1,a2,…))`, `indexOf`/`lastIndexOf`, `match(string, regex)` (повертає масив груп), `md5`/`sha1`/`sha256`, `uuid()`.

> Корекції: «find (regex)» → **`match()`**; «capitalize» → **`toTitlecase()`**.

**Numeric/math:** `round(n)`, `ceil(n)`/`floor(d)`/`abs(d)`, `min(...)`/`max(...)` (працюють і з датами → раніша/пізніша), `pow(a, b)`, `sum(...)`/`avg(...)`, `log(n)` (**base-10**), `ln(n)` (натуральний), `exp(n)`, `sqrt(n)`, `mod(a, b)`.

**Date (важливі корекції — поширені «звичні» назви НЕ існують):**

- `now()` — поточний datetime.
- `datePart(date, part, [tz])` — компонент. `part`: `'years'`, `'months'`, `'days'`, `'hours'`, `'minutes'`, `'seconds'`, `'weekday'`, `'dayofweek'`, `'weekOfYear'`, `'unixTime'`. **Замінює `year()`/`month()`/`day()`/`hour()`**.
- `diff(d1, d2, [unit])` — різниця дат. **Замінює `dateDiff`**.
- `inc(d, value, unit)` — інкремент/декремент дати. **Замінює `incrementDate`**.
- `trunc(d, unit)` — обрізати до одиниці.
- `asDatetimeTz(o, [fmt])`, `asDatetimeNoTz(o, [fmt])`, `asDateOnly(o, [fmt])` — парсинг/форматування. **Замінюють `parseDate`/`formatDate`**.

**Conditional / logic:** `if(boolean, then, else)`, `and(a, b)`/`or(a, b)`/`not(b)`, `isBlank(o)` (true для null або порожнього рядка), `isNull(o)`/`isNotNull(o)`, `isNonBlank(o)`, `coalesce(v1, v2, …)` (перше непорожнє), `switch(expr, m1, r1, …, [default])`.

**Array/object:** `arrayContains(a, item)`, `arrayLen(a)`, `join(a, sep)`, `get(a, from, [to])`, `slice(o, from, [to])`, `objectKeys`, `objectValues`, `getPath(o, jsonpath)`, `parseJson(s)`, `filter(array, var, expr)`, `forEach(array, var, expr)`.

---

## Приклади формул (Formula step)

```text
# 1. Середнє двох колонок, масштабоване лог-десятковим третьої
(col1 + col2) / 2 * log(col3)

# 2. Trim, перші 7 символів, lowercase
toLowercase(substring(strip(mytext), 0, 7))

# 3. Умовна категоріальна мітка
if (width > height, "wide", "tall")

# 4. Прапорець великих покупок (boolean-версія: amount > 1000)
if(amount > 1000, "high", "normal")

# 5. Fallback, коли B порожнє (рівнозначно if(isBlank(B), C, B))
coalesce(B, C)

# 6. Крос-рушієво-безпечна конкатенація (краще за +)
concat(firstName, " ", lastName)

# 7. Додати 30 днів (наприклад, due date)
inc(order_date, 30, 'days')

# 8. Місяців між двома датами
diff(start_date, end_date, 'months')

# 9. День тижня та рік замовлення
datePart(order_date, 'weekday')   # -> "Wednesday"
datePart(order_date, 'years')

# 10. Поточний мінус попередній рядок (лише Prepare, DSS-рушій)
val("amount", 0) - val("amount", 1, 1)

# 11. Багатогілкове мапінг code -> label
switch(status, "A", "Active", "I", "Inactive", "Unknown")

# 12. Очищення телефону від нецифр (regex replace)
replace(phone, "[^0-9]", "")
```

---

## Семплінг, великі дані та рушії виконання

### Дизайн на SAMPLE vs запуск на FULL

- **Етап дизайну (інтерактивний):** Prepare recipe працює зі **зразком** датасета, **завжди завантаженим у RAM**. Той самий зразок, що в Explore, застосовується до скрипта, тож ефект кожного кроку видно миттєво.
- **Етап запуску (повний):** лише коли ви натискаєте **Run**, Dataiku виконує інструкції на **всьому** датасеті, матеріалізуючи output. Тоді ж застосовується вибір рушія (зразок стає неактуальним).

### Методи семплінгу

За замовчуванням — **перші 10 000 записів**. Налаштовується у вкладці **Sampling**. Компроміс: точніші методи роблять один-два **повні проходи** даними.

| Метод (UI) | Що робить | Проходи |
|---|---|---|
| **First records** (Head) | перші N рядків; найшвидше, можлива упередженість | 1 (швидко) |
| **Random (nb. records)** | випадкові рівно N записів | 1 |
| **Random (ratio)** | випадкові ~X% записів | 1 |
| **Random (approximate number of records)** | ~N записів, точніше на великих даних | 2 |
| **Column values subset** | випадкова підмножина значень колонки + усі рядки з ними (зберігає повні послідовності, напр. усі рядки користувача) | 2 |
| **Stratified (fixed number / ratio)** | зберігає розподіл значень колонки; гарантує появу всіх значень | 2 |
| **Class rebalancing (number / ratio)** | вирівнює частоту класів; **лише undersample — ніколи не генерує зайвих** | 2 |
| **Last records** | останні N рядків | 1 |

**Налаштування:** розмір за замовчуванням 10 000; зразок завжди в RAM (**не перевищуйте 200 000 записів** для гарної інтерактивності); кнопка **Save and Refresh Sample** перераховує зразок. Зразок авто-перераховується при rebuild датасета, зміні конфігурації датасета чи семплінгу.

### Рушії виконання (engines)

Під час запуску DSS обирає **рушій**, видимий і перевизначуваний через **engine selector** (шестерня внизу рецепта). Філософія: DSS — **оркестратор**, що пушить обчислення туди, де живуть дані.

- **DSS engine (local stream):** усі дані йдуть через сервер DSS, але **потоково, рядок за рядком** (не все в памʼять). Фолбек за замовчуванням; **підтримує всі процесори**. Найменш масштабований для дуже великих даних, але завжди доступний.
- **Spark engine:** потребує Spark; рекомендований для даних на **S3, Azure Blob, GCS, Snowflake, HDFS**. **Усі процесори сумісні зі Spark** — доступність залежить від інфраструктури/конекшна, а не від підтримки процесора.
- **In-database (SQL) engine:** виконується повністю в БД лише коли **обидва** виконано: (a) input і output — SQL-таблиці на **тому самому конекшні**, і (b) **кожен** процесор у скрипті транслюється в SQL. Лише **підмножина** процесорів транслюється в SQL. Рушій можна обрати, **тільки якщо всі процесори сумісні** — один нетрансльований крок дискваліфікує весь рецепт. Кожен крок показує **зелену** (сумісно) чи **червону** (несумісно) SQL-іконку.

**Зазвичай пушиться в SQL:** Keep/Delete/Rename columns, Split column, Filter/Flag на value, Remove rows where cell empty, Fill empty cells, Concatenate/Copy column, Unfold, if-then-else / formula-подібні умови — тобто реляційні операції рівня колонок (`SELECT`/`WHERE`/`CASE`).

**Частково / може відкотитися до DSS:** Formula, Find and replace, Transform string, Extract with regular expression, частина date-процесорів, geo-процесори.

**Розширено на Snowflake через Java UDF:** деякі непушабельні процесори стають in-database на Snowflake — Discretize (binning), Convert currencies, Email/URL/IP parsing, timestamp conversion, Classify User-Agent.

> **Чому процесор непушабельний:** семантика без точної SQL-трансляції (складні текстові трансформи, regex-витяг, geo/IP/user-agent збагачення, евристичний парсинг дат, конвертація валют), edge-кейси конфлікту типів, або custom meanings у filter/flag. Оскільки SQL-рушій працює за принципом «все або нічого», один непушабельний крок штовхає весь рецепт у DSS/Spark — типовий обхід: **перевпорядкувати або розбити рецепт**, щоб SQL-пушабельні кроки лишалися в БД.

> Детальніше про рушії та продуктивність — у [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) та [15-best-practices.md](15-best-practices.md).

---

## Analyze box / quick column insights

### Доступ і призначення

Відкривається з меню заголовка колонки (caret) → **Analyze**, у вкладці **Explore** або в Prepare recipe. Модалка показує доречну статистику й візуалізації залежно від того, числові дані чи категоріальні. В Explore — для інспекції; у Prepare — дії породжують кроки-процесори.

### Статистика

**Спільне (для всіх типів):** лічильники **Valid**, **Invalid** (не відповідають meaning) та **Empty** — ті самі три кошики, що живлять кольорову смужку.

**Категоріальні / текст:** bar chart за частотою; per-value **counts і frequencies** (count + %); кількість distinct; **most frequent (top-k)**.

**Числові:** **histogram**; **box plot** (квартилі); summary stats **minimum, maximum, mean, median, standard deviation**; **квартилі** (та IQR); найчастіші значення; секція **detected outliers**.

> За замовчуванням Analyze рахує по **поточному зразку (Design Sample)**; перемикач угорі переключає на **Whole data** (увесь датасет — точніше, важче).

### Швидкі дії

- **Treat outliers (числові)** — викиди визначаються через **InterQuartile Range (IQR)**. Обробки: **Clip** (замінити на межу), **Remove** (видалити рядки), **Clear** (очистити комірки). Породжує крок **Force numerical range** (режими «Clip outliers» / «Clear outliers»).
  - ⚠️ У Prepare recipe **межі IQR заморожуються як статичні константи**, не формула — не перераховуються при зміні даних.
- **Merge selected (категоріальні)** — обрати значення чекбоксами і злити; породжує крок типу «Replace values in category». Споріднено — процесор **Merge long-tail values**.
- **Values clustering** (Fuzzy Values Clustering) — знаходить групи схожих текстових значень для стандартизації (напр. «NYC» / «New York» / «new york»).

> Розгорнута статистика (sum, variance, percentiles, скошеність) — це окрема фіча **Statistics / Interactive statistics worksheet**, не Analyze box. Див. [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) і [12-dashboards-charts-webapps-reporting.md](12-dashboards-charts-webapps-reporting.md).

---

## Best practices

1. **Ідемпотентність кроків.** Прагніть, щоб повторний запуск рецепта на тих самих вхідних даних давав той самий результат. Уникайте кроків, що залежать від «зовнішнього часу» без потреби (`now()` робить вихід недетермінованим). Якщо обробляєте викиди через Analyze — памʼятайте, що **межі IQR заморожуються як константи**; для стабільної логіки краще явний `Force numerical range` або формула.

2. **Працюйте на репрезентативному зразку.** First records — найшвидше, але упереджено; для очищення, що залежить від рідкісних значень, беріть **Random** або **Stratified**. Тримайте зразок ≤ 200 000 записів для інтерактивності.

3. **Проєктуйте під push-down.** Якщо дані в SQL-БД, **групуйте SQL-пушабельні кроки разом** і виносьте непушабельні (Formula зі складними функціями, regex-витяг, geo) в окремий рецепт нижче за течією. Перевіряйте колір SQL-іконки кожного кроку. Уникайте `+` для конкатенації рядків — використовуйте `concat()`.

4. **Повторне використання скрипта.** Скрипт можна **копіювати між рецептами** (copy/paste steps), а в Lab — деплоїти кнопкою **Deploy Script**. Для повторюваної логіки на багатьох датасетах розгляньте збереження як власного плагіна (див. [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md)).

5. **Документуйте кроки.** Додавайте **коментарі-кроки** та групуйте логічні блоки. Перейменовуйте кроки осмислено — це критично для передачі рецепта колегам.

6. **Спирайтеся на meanings.** Налаштовуйте правильний meaning замість ручних фільтрів — це активує контекстні підказки, валідацію і `Filter invalid rows/cells`. Для бізнес-словників створюйте **user-defined meanings** (Values list / Pattern).

7. **Продуктивність.** Вимикайте (не видаляйте) кроки під час дебагу, щоб ізолювати проблему. Не зловживайте memory-based join/geo на великих даних — вони тримають дані в памʼяті й не пушаться в БД.

8. **Перевіряйте на повних даних.** Зразок може приховати edge-кейси; перед продакшном запустіть рецепт і перегляньте розподіли через Analyze у режимі **Whole data**.

---

## Типові помилки

- **Неправильний порядок кроків.** Обчислення `diff(...)` до парсингу дати дасть невалідні результати. Спершу `Parse to standard date format`, потім дата-математика.
- **Очікування `+` як конкатенації в SQL.** На SQL-рушії `+` над рядками зламається або дасть NULL — використовуйте `concat()`.
- **Винайдення неіснуючих функцій.** `incrementDate`, `dateDiff`, `parseDate`, `formatDate`, `year()`, `find()`, `capitalize()` **не існують** у DSS formula language. Правильно: `inc`, `diff`, `asDateOnly`/`asDatetimeTz`, `datePart`, `match`, `toTitlecase`.
- **`val(... , offset)` поза Prepare/DSS-рушієм.** Зсув на попередній рядок працює лише в Prepare recipe на DSS-рушії — не пушиться в SQL.
- **Очікування SQL-рушія за наявності непушабельного кроку.** Один несумісний крок дискваліфікує весь рецепт із in-database виконання.
- **Плутанина invalid vs empty.** Це **різні** кошики. `Filter invalid rows/cells` не чіпає empty, а `Remove rows where cell is empty` не чіпає invalid.
- **Заморожені межі IQR.** Обробка викидів через Analyze фіксує межі як константи — на нових даних вони не перераховуються.
- **Завеликий зразок.** >200 000 записів у RAM ламає інтерактивність дизайну.
- **Увімкнена auto-detection для user-defined meaning.** Не рекомендовано — придушує вбудовані meanings і знижує продуктивність.
- **Очікування геокодування адрес у core.** Адреса↔координати потребує плагіна (Geocoder); нативний лише `Resolve GeoIP`.

---

## Перехресні посилання

- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — Flow, recipes, datasets.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — конекшни, storage types, партиціонування (впливає на семплінг і SQL push-down).
- [04-visual-recipes.md](04-visual-recipes.md) — інші візуальні рецепти (Sample/Filter, Join, Group, Pivot).
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — коли code recipe доречніший за Prepare; деталі рушіїв.
- [07-machine-learning-visual-ml.md](07-machine-learning-visual-ml.md) — Lab, visual analyses, Statistics worksheet, feature engineering.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — Data Quality Rules, автоматизація запусків рецептів.
- [14-plugins-applications-extensibility.md](14-plugins-applications-extensibility.md) — плагіни (geocoding) та власні процесори.
- [15-best-practices.md](15-best-practices.md) — продуктивність, push-down, патерни.
- [16-glossary.md](16-glossary.md) — терміни: processor, meaning, storage type, sample, engine.

---

### Джерела

- [Data preparation — DSS 14](https://doc.dataiku.com/dss/latest/preparation/index.html)
- [Processors reference — DSS 14](https://doc.dataiku.com/dss/latest/preparation/processors/index.html)
- [Execution engines — DSS 14](https://doc.dataiku.com/dss/latest/preparation/engines.html)
- [Formula language — DSS 14](https://doc.dataiku.com/dss/latest/formula/index.html)
- [Meanings list — DSS 14](https://doc.dataiku.com/dss/latest/schemas/meanings-list.html)
- [User-defined meanings — DSS 14](https://doc.dataiku.com/dss/latest/schemas/user-defined-meanings.html)
- [Sampling — DSS 14](https://doc.dataiku.com/dss/latest/explore/sampling.html)
- [Analyze — DSS 14](https://doc.dataiku.com/dss/latest/explore/analyze.html)
- [Concept | Prepare recipe — Knowledge Base](https://knowledge.dataiku.com/latest/prepare-transform-data/prepare/overview/concept-prepare-recipe.html)
