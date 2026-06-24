# 07. Machine Learning: Visual ML / AutoML

> Рівень: новачок → досвідчений
> Версії DSS: 13.x / 14.x
> Джерела: [doc.dataiku.com/dss/latest](https://doc.dataiku.com/dss/latest/), [knowledge.dataiku.com](https://knowledge.dataiku.com/), [developer.dataiku.com](https://developer.dataiku.com/), Dataiku Academy

---

## Вступ

**Visual ML** — це підсистема Dataiku для побудови, оцінювання та розгортання моделей машинного навчання **без коду** (point-and-click), із можливістю переходу в **Expert mode** для повного контролю та власних алгоритмів. Вона покриває весь спектр: від швидкого прототипу за кілька кліків до глибокої гіперпараметричної оптимізації, deep learning, computer vision та causal ML.

Ключова ідея Dataiku — поєднати **AutoML** (платформа сама обирає feature handling, алгоритми та hyperparameters) з **повним контролем**: незалежно від того, чи ви обрали *Automated Machine Learning*, чи *Expert mode*, ви **завжди** маєте доступ до всіх налаштувань.

Цей документ розкриває:

- **The Lab** і **Visual analysis** — простір експериментів;
- типи задач: **Prediction** (binary classification, multiclass, regression), **Clustering**, та спеціалізовані (time series, causal, computer vision);
- **AutoML modes/templates** (quick prototypes, interpretable, high performance, expert);
- **Feature handling**, алгоритми, train/test policy, hyperparameter search, метрики;
- результати та **інтерпретованість** (ROC, confusion matrix, feature importance, Shapley, partial dependence, subpopulation, individual explanations, what-if, fairness);
- розгортання моделі у **Flow як Saved Model**, версії, retrain, Score/Evaluate recipes, **Model Evaluation Stores** і drift (коротко).

> Перед читанням бажано знати концепцію **Flow**, **datasets** і **recipes** ([02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md)) та **Prepare recipe** ([05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md)). Деталі продакшн-розгортання (API node, bundles, deployment) — у [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md). Програмний доступ до ML — у [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md).

---

## Концепції

### The Lab і Visual Analysis (Базово)

**The Lab** — це експериментальний простір, прив'язаний до **датасета**. Він не захаращує Flow і не створює постійних виходів, поки ви свідомо щось не задеплоїте. Лаб має дві вкладки:

- **Script** — візуальна підготовка даних (той самий редактор, що й Prepare recipe; деталі — у [05](05-data-preparation-prepare-recipe.md));
- **Models** — побудова ML-моделей.

**Visual Analysis** — це контейнер у Lab, що об'єднує Script + один або кілька ML-tasks на одному датасеті. Один аналіз може містити декілька **ML tasks** (наприклад, кілька prediction-задач з різними цілями).

Робочий цикл стандартний для Dataiku:

1. **Design** — налаштовуєте задачу (target, features, алгоритми, train/test).
2. **Train** — тренуєте; отримуєте один або кілька trained models (**sessions**).
3. **Explore / порівняння** — аналізуєте результати, метрики, інтерпретованість.
4. **Deploy** — найкращу модель розгортаєте у Flow як **Saved Model** (зелений ромб).

> The Lab = дизайн і експерименти (на семплі / повних даних). Flow = продакшн-артефакт (Saved Model), який можна перетреновувати, скорити та оцінювати в автоматизованих сценаріях.

### Типи ML-задач (Базово)

При створенні моделі з датасета (**Action panel → Lab → ...** або **Models → Create your first model**) Dataiku пропонує типи задач:

| Задача | Тип | Опис |
|---|---|---|
| **Prediction** | supervised | прогноз цільової змінної: *binary classification*, *multiclass classification*, *regression* (тип визначається автоматично за meaning/значеннями target) |
| **Clustering** | unsupervised | групування рядків без міток (K-means, DBSCAN, …) |
| **Time Series Forecasting** | supervised | прогноз часових рядів (окремий тип, не «звичайна» регресія) |
| **Causal Prediction** | causal | оцінка ефекту treatment на outcome (uplift / CATE) |
| **Image classification** / **Object detection** | computer vision | класифікація зображень / детекція об'єктів |

**Тип prediction** визначається автоматично:

- target — **категоріальний з 2 класами** → *binary classification*;
- target — **категоріальний з >2 класами** → *multiclass classification*;
- target — **числовий** → *regression*.

### Saved Model — зв'язок Lab → Flow (Базово)

**Saved Model** — це Flow-артефакт (зелений ромб), що містить навчену модель **разом з усім lineage**: hyperparameters, feature preprocessing, code environment, метаданими — усе необхідне для аудиту, retrain і deployment.

Коли ви тиснете **Deploy** на моделі в Lab, Dataiku додає у Flow **два** об'єкти:

1. **Train recipe** (зелений круглий рецепт) — на відміну від звичайних рецептів, бере **датасет на вхід і видає модель** (Saved Model); дозволяє перетреновувати з тими самими налаштуваннями на нових даних;
2. **Saved Model** (зелений ромб) — вихід цього Train recipe.

> Деталі версіонування та lifecycle — нижче в розділі «Розгортання у Flow».

---

## AutoML modes / templates (Базово → Просунуто)

При створенні **Prediction** чи **Clustering** ви обираєте між **Automated Machine Learning** і **Expert mode**. AutoML, своєю чергою, має кілька **templates** (стилів).

### AutoML Prediction templates

| Template | Філософія | Що включає |
|---|---|---|
| **Quick Prototypes** | швидкість + різноманіття | швидко тренує декілька різних моделей, щоб дати початкове уявлення про задачу і вирішити, чи варто рухатися далі |
| **Interpretable Models** | white-box, зрозумілість | **decision trees** і **linear models** (logistic/linear regression); зазвичай швидке тренування; акцент на поясненні драйверів прогнозу |
| **High Performance** | максимальна якість | переважно **tree-based** моделі (Random Forest, XGBoost, LightGBM, GBT) з **глибоким hyperparameter optimization search**; найкраща якість ціною інтерпретованості та часу тренування |

> Незалежно від обраного template, Dataiku автоматично робить розумні дефолти: train/test split, feature handling (категоріальні/текстові змінні, missing values, rescaling), а також **semi-automatic feature generation** (interactions, polynomial) для виявлення нелінійних зв'язків. Усі ці дефолти можна змінити вручну.

### AutoML Clustering

Для clustering AutoML-режим теж пропонує швидкий старт із розумними дефолтами (типово **K-means** із автоматичним підбором k та обробкою змінних). Через Expert/повний контроль доступні всі алгоритми кластеризації (див. нижче).

### Expert mode (Просунуто)

**Expert mode** дає повний контроль і додаткові можливості, недоступні в простому AutoML:

- **Deep Learning** — власні архітектури на **Keras / TensorFlow** (ви пишете функцію побудови мережі);
- **Custom Python models** — будь-який estimator зі scikit-learn-сумісним API;
- **Custom models / plugin algorithms** — алгоритми з плагінів;
- **Ensembling** — ансамблювання кількох моделей (average, voting, stacking тощо);
- повний контроль над усіма feature-handling та optimization-налаштуваннями.

> Важливо: «AutoML» vs «Expert» — це лише **стартова точка** з різними дефолтами. Після створення задачі обидва режими відкривають однаковий, повністю редагований Design-екран.

---

## Дизайн моделі: Design-екран (Базово → Середній)

Design-екран prediction-моделі має вкладки (ліворуч):

- **Target** — цільова змінна, prediction type, **Partitioning** (partitioned models), переоцінка ваг (class weights / sample weights);
- **Train / Test Set** — політика розбиття train/test, k-fold;
- **Metrics** — метрика оптимізації + гіперпараметрична метрика;
- **Features handling** — ролі та обробка кожної змінної;
- **Feature generation** / **Feature reduction**;
- **Algorithms** — вибір алгоритмів та їх hyperparameters;
- **Runtime environment** — code env, контейнери, GPU.

### Train / Test policy (Середній)

DSS пропонує кілька стратегій формування test-набору:

- **Random split** (дефолт) — типово **80/20**. Можна попередньо subsample-ити датасет (наприклад, *Random sampling (fixed number of records)* або *Class rebalancing*).
- **Time ordering** — розбиття за **часовим порядком** (за вказаною змінною), а не випадкове. Критично для часових/впорядкованих даних, щоб уникнути «зазирання в майбутнє».
- **Explicit extracts** — вручну вказати train і test набори (з одного або двох датасетів) з власними правилами фільтрації/семплінгу. Корисно, коли структура даних має бути збережена.
- **K-fold cross-validation** — датасет ділиться на K частин; кожна по черзі є test-набором. Дає **усереднену** оцінку якості з межами похибки (~2× std між фолдами). **Сильно** збільшує час тренування (приблизно ×K).

> **K-fold для оцінки** (на вкладці Train/Test) і **K-fold для hyperparameter search** — це різні речі (див. нижче). Перше оцінює фінальну модель; друге обирає hyperparameters.

### Metrics — метрика оптимізації (Базово)

Ви обираєте **optimization metric**, яка керує і hyperparameter search, і визначенням «найкращої» моделі.

**Classification:** `Accuracy`, `Precision`, `Recall`, `F1 Score`, `Cost Matrix Gain`, `Log Loss`, `Cumulative Lift`, `ROC AUC`, `Average Precision`, `Custom score`.

**Regression:** `EVS` (Explained Variance Score), `MAPE`, `MAE`, `MSE`, `RMSE`, `RMSLE`, `R2 Score`, `Custom score`.

> Для незбалансованих класів `Accuracy` оманлива — оберіть `ROC AUC`, `F1` або `Average Precision`. `Cost Matrix Gain` дозволяє оптимізувати під бізнес-вартість помилок (через cost matrix).

---

## Feature handling (Середній)

Вкладка **Features handling** керує тим, як кожна змінна перетворюється перед подачею в алгоритм. Для кожної змінної задаються **роль** (включена/виключена, або є target), **тип** і метод обробки.

### Типи змінних (variable types/roles)

- **Numerical** — числові;
- **Categorical** — категоріальні;
- **Text** — вільний текст (за замовчуванням **відхиляються** як features, треба ввімкнути);
- **Vector** — готові ембединги/вектори;
- **Image** — зображення (для computer vision).

### Encoding категоріальних (Середній)

- **Dummy-encoding (one-hot)** — окрема бінарна колонка на кожне значення (можна обмежити max категорій / drop rare);
- **Ordinal encoding** — числовий ранг для кожної категорії;
- **Frequency encoding** — кодування частотою появи;
- **Target / Impact encoding** — кодування на основі цільової змінної (*Impact coding*);
- **GLMM encoding** — узагальнене змішане target-encoding;
- **Hashing (feature hashing)** — хешування у фіксовану кількість колонок (для високої кардинальності);
- **Datetime cyclical encoding** — циклічне кодування часових ознак (для категоріальних/datetime).

> **Імпакт/target encoding** дає leakage, якщо застосувати наївно на всіх даних. Dataiku обчислює його коректно в межах train/CV, але будьте уважні з власними реалізаціями.

### Rescaling числових (Базово)

- **No rescaling**;
- **Standard rescaling** (z-score / avgstd) — віднімаємо середнє, ділимо на std;
- **Min-max rescaling** — у діапазон [0, 1].

> Rescaling важливий для алгоритмів, чутливих до масштабу (SVM, KNN, лінійні з регуляризацією, нейромережі). Для дерев (RF, XGBoost) — практично не впливає.

### Impute missing values (Базово)

Стратегії заповнення пропусків (окремо для numerical і categorical):

- **Impute with mean / median / most frequent value** (числові);
- **Impute with a constant value**;
- **Treat as a regular value / Create a separate category** (для категоріальних — окрема категорія «missing»);
- **Drop rows** — видалити рядки з пропусками (обережно — може спотворити вибірку).

### Feature generation (Просунуто)

Вкладка **Feature generation** автоматично створює нові ознаки:

- **Pairwise linear combinations / interactions** — добутки/комбінації пар числових змінних;
- **Polynomial combinations** — поліноміальні члени;
- **Explicit pairwise interactions** — задані вручну взаємодії.

Це допомагає лінійним моделям ловити нелінійності. У High Performance template частина цього вмикається автоматично («semi-automatic feature generation»).

### Feature reduction (Просунуто)

Вкладка **Feature reduction** зменшує розмірність перед тренуванням:

- **Principal Component Analysis (PCA)** — проєкція у простір головних компонент;
- **Correlation with target** — відбір за кореляцією з ціллю;
- **Tree-based** — відбір за важливістю з дерев;
- **Drop / Lasso-based** — відсікання неінформативних ознак.

> Окремо існує **Genetic Algorithm** для feature generation & selection (Expert) і **Self-Supervised Tabular Embeddings** (pretraining + inference recipes).

### Text handling (Середній → Просунуто)

Текстові ознаки (увімкнені вручну в Features handling → «Text variables») векторизуються одним із методів:

- **Count vectorization** — матриця підрахунків слів;
- **TF/IDF vectorization** — count, зважений на inverse document frequency;
- **Hashing trick** (у KB — «Term hashing») — sparse-матриця хешів;
- **Hashing trick + Truncated SVD** — щільна матриця для алгоритмів без підтримки sparse;
- **Text embedding** — щільні семантичні вектори (**лише Python in-memory backend**); джерела: *From the code environment resources* (локальні `sentence-transformers`/`transformers`) або *From the LLM Mesh* (API embedding-моделі — див. [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md));
- **Custom preprocessing** — власна обробка.

Фільтр **Min. rows fraction %** відсікає рідкісні слова. Попереднє очищення тексту — процесором **Simplify text** у Prepare recipe (Normalize text, Stem words, Clear stop words; деталі — [05](05-data-preparation-prepare-recipe.md)).

---

## Алгоритми (Середній → Просунуто)

Visual ML має два ML-engines: **In-memory (Python, scikit-learn-based)** — основний; **MLLib (Spark)** — **deprecated**. Нижче — алгоритми Python-backend з ключовими hyperparameters, які Dataiku показує в UI.

### Prediction — лінійні

- **Ordinary Least Squares** (regression);
- **Ridge Regression** — `Alpha`, `Regularization term`;
- **Lasso Regression** — `Alpha`, `Regularization term`; також **Lasso Path** (`Maximum features`);
- **Logistic Regression** — `Regularization (L1 or L2)`, `C`;
- **Stochastic Gradient Descent (SGD)** — `Loss function`, `Penalty (L1/L2/elastic net)`, `Alpha`, `L1 ratio`, `Iterations`.

### Prediction — дерева й ансамблі

- **Decision Tree** — `Maximum depth`, `Criterion (Gini/Entropy)`, `Minimum samples per leaf`, `Split strategy`;
- **Random Forest** — `Number of trees`, `Feature sampling strategy`, `Maximum depth of tree`, `Minimum samples per leaf`, `Allow sparse matrices`;
- **Extra Random Trees** — як RF, з більшою рандомізацією;
- **Gradient Boosted Trees (GBT)** — `Number of boosting stages`, `Learning rate`, `Loss`, `Maximum depth`, `Minimum samples per leaf`;
- **XGBoost** — `Maximum number of trees`, `Early stopping` + `Early stopping rounds`, `Maximum depth`, `Learning rate`, `L1/L2 regularization`, `Gamma`, `Minimum child weight`, `Subsample`, `Colsample by tree`, `Custom missing value`;
- **LightGBM** — `Maximum number of trees`, `Maximum depth`, `Number of leaves`, `Learning rate`, `Minimum split gain`, `Minimum child weight`, `Minimum leaf samples`, `Colsample by tree`, `L1/L2 regularization`, `Bagging` + `Subsample ratio`, `Early stopping`.

### Prediction — інші

- **Support Vector Machine (SVM)** — `Kernel type`, `C`, `Gamma`, `Tolerance`, `Maximum iterations`;
- **K Nearest Neighbors (KNN)** — `K`, `Distance weighting`, `Neighbor finding algorithm`, `p (Minkowski)`;
- **Artificial Neural Network (MLP)** — `Hidden layer sizes`, `Activation`, `Alpha`, `Solver`, `Max iterations`, `Early stopping`, `Initial Learning Rate`;
- **Deep Neural Network** (Expert) — `Hidden layers`, `Units per layer`, `Learning rate`, `Batch size`, `Dropout`, `L1/L2 regularization`, `Early stopping`;
- **TabICL** (classification, новіше) — in-context tabular learning.

### Deep learning — Keras / TensorFlow (Просунуто)

В **Expert mode → Deep Learning** ви пишете Python-функцію, що будує **Keras**-модель (на TensorFlow). Dataiku керує preprocessing, train/test, callbacks, метриками й деплоєм як для будь-якої visual-моделі. Текст у DL-backend обробляється через `TokenizerProcessor`. Потребує відповідний code env і (бажано) GPU. **Обмеження:** для Keras-моделей недоступні Shapley feature importance, individual explanations, Model Document Generator.

### Spark MLlib (deprecated)

MLLib-engine історично давав розподілене тренування на Spark, але **deprecated**. Для великих даних сьогодні рекомендується Python-backend із семплінгом/контейнерами або distributed hyperparameter search.

### Custom Python models (Просунуто)

В Expert mode можна підключити будь-який estimator зі scikit-learn-сумісним інтерфейсом (`fit`/`predict`), або власний алгоритм із плагіна. Підтримує grid/random/bayesian search над вашими hyperparameters.

### Clustering — алгоритми

- **K-means** — `Number of clusters`, `Seed`;
- **Mini-batch K-means**;
- **Gaussian Mixture** — `Number of mixture components`, `Max Iterations`;
- **Agglomerative Clustering** — `Number of clusters`;
- **Spectral Clustering** — `Affinity measure`, `Gamma`, `Coef0`;
- **DBSCAN** — `Epsilon`, `Min. Sample ratio`;
- **HDBSCAN** — `Min. cluster ratio`;
- **Interactive Clustering** — `Number of Pre-clusters`, `Number of clusters`;
- **Isolation Forest** — для anomaly detection (`Number of trees`, `Contamination`).

---

## Hyperparameter search (Просунуто)

Dataiku підтримує **три стратегії** пошуку гіперпараметрів (для кожного алгоритму ви задаєте grid значень або діапазони):

- **Grid search** — перебирає **усі** комбінації заданих значень. Для кожного hyperparameter задаєте список значень або діапазон («5 values equally spaced between 30 and 80», «8 values logarithmically spaced between 1 and 1000»);
- **Random search** — випадково обирає N комбінацій із простору й зупиняється;
- **Bayesian search** — стартує як random, але навчає допоміжну предиктивну модель простору пошуку й **скеровує** наступні спроби в найперспективніші зони — швидше досягає хорошого набору.

### Search cross-validation (Просунуто)

Стратегія крос-валідації **всередині пошуку** (як оцінюється кожна комбінація):

- **Simple train/test split** (`SHUFFLE`) — швидко;
- **K-fold cross-validation** (`KFOLD`) — N фолдів; надійніше, але повільніше;
- **Time series** (`TIME_SERIES_KFOLD`, `TIME_SERIES_SINGLE_SPLIT`) — для часових даних;
- **Custom** (`CUSTOM`) — власна логіка.

Додатково керується **бюджетом пошуку**: ліміт ітерацій (для random/bayesian), ліміт часу (search time), **parallelism** (паралельні тренування, у т.ч. розподілений search на контейнерах/Kubernetes). Під час тренування DSS показує **графік еволюції найкращого CV-score** в реальному часі.

> Логіка: hyperparameter search обирає найкращі hyperparameters за **search CV** → фінальна модель перетреновується й оцінюється на **окремому test-наборі** (з вкладки Train/Test). Не плутайте ці два рівні валідації.

---

## Результати та інтерпретованість (Середній → Просунуто)

Після тренування клік по моделі відкриває **Result** (model report) з лівою навігацією: **Summary**, performance-чарти, **Explainability**-панелі, метрики, та окреме меню **Views** (плагінні звіти).

### Classification — performance (Базово)

| Панель | Що показує |
|---|---|
| **Confusion matrix** | TP/TN/FP/FN; за замовчуванням на **optimal threshold**; зміна cut-off оновлює матрицю й метрики **наживо** |
| **ROC curve** | TPR vs FPR; чим крутіша на початку — тим краще; джерело **AUC** |
| **Decision chart** | метрики (precision, recall, F1) як функція **усіх** значень threshold |
| **Density chart** | розподіл передбачених імовірностей за фактичним класом — наскільки модель розділяє класи |
| **Lift charts** | cumulative gains / lift — ефективність таргетингу |
| **Cost matrix** | задані gain/cost на кожну клітинку, оптимізація threshold під бізнес-вартість (**Cost Matrix Gain**) |
| **Detailed metrics** | таблиця всіх метрик |

**Threshold / cut-off optimization:** більшість моделей дають неперервний score; DSS обчислює confusion matrix для багатьох threshold і автоматично обирає за вибраною метрикою. **Threshold-independent** метрики (AUC) vs **threshold-dependent** (precision, recall, F1, що звітуються на оптимальному cut-off; його можна змінити вручну).

### Regression — performance (Базово)

- **Scatter plot** (predicted vs actual), **error distribution / error deciles**;
- метрики: `EVS`, `MAPE`, `MAE`, `MSE`, `RMSE`, `RMSLE`, `R2 Score`, `Custom Score`.

### Feature importance (Середній)

Панель **Feature importance** з двома методами:

- **Shapley feature importance** (дефолт, model-agnostic) — на основі **approximated Shapley values**; працює для будь-якої моделі на **Python backend**. Три візуалізації: **Absolute feature importance** (bar chart за абсолютною важливістю), **Feature effects** (по семплах), **Feature dependence** (Shapley vs значення фічі). **Не підтримується** для Keras/CV; авто-обчислення пропускається при text embeddings, **kNN/SVM**, або **>50 features**;
- **Gini feature importance** (лише tree-моделі: scikit-learn trees, LightGBM, XGBoost) — нормований внесок у зменшення критерію розбиття; завжди додатний, сума = 1; **переоцінює** числові фічі з багатьма унікальними значеннями;
- для **лінійних** моделей — **regression coefficients** (порівнюйте на rescaled-змінних).

### Shapley values (Просунуто)

Dataiku використовує **approximated Shapley values** як універсальний рушій пояснень — і для глобальної важливості (вище), і для **individual explanations** (нижче). Лише Python backend.

### Partial dependence plots (PDP) (Просунуто)

Панель **Partial dependence** (обчислюється явно, за вашим запитом). X — обрана фіча; Y — ступінь partial dependence (для classification — імовірність класу через log-odds; для regression — передбачене значення). Сірі бари показують частоту значень — обережно інтерпретуйте розрідженні зони.

### Subpopulation analysis (Просунуто)

Панель **Subpopulation analysis** перевіряє, чи **різниться якість моделі між підгрупами** (інструмент справедливості/bias). Обираєте одну змінну (категоріальну → рядок на значення; числову → bins). Таблиця порівнює загальну якість vs кожну підгрупу (ROC AUC, accuracy, precision, recall). Вибір підгрупи дає її density chart і confusion matrix.

### Individual explanations (Просунуто)

Панель **Individual explanations** — локальні пояснення per-record. Два методи: **Shapley** (точніше, повільніше) і **ICE-based** (швидше; сума пояснень ≠ різниці prediction−avg). UI: розподіл імовірностей зі слайдерами для записів на «низькому» й «високому» кінцях → картки (зелені бари праворуч = позитивний вплив, червоні ліворуч = негативний). У **Score recipe** опція **Output explanations** додає колонку `explanations` (JSON: feature → influence). **Тільки** Python backend; **не** для Keras/CV; обчислення в 10–1000× повільніше за просте скорення.

### What-if / interactive scoring (Середній)

Панель **What if?** — інтерактивний симулятор: ліворуч задаєте значення features (dropdown/слайдери, можна **ignore features** = missing), праворуч — оновлений прогноз + внески фіч. Підтримує binary/multiclass/regression. Є **Comparator** для іменованих сценаріїв; публікується на dashboards для стейкхолдерів. Розширено — **counterfactual / actionable recourse** («Explore Neighborhood» для класифікації, «Optimize Outcome» для регресії).

### Fairness report (Просунуто)

**Model Fairness Report** (Views → Model Fairness Report) — надається **плагіном** (Tier 2 support). Налаштування: **Sensitive column**, **Sensitive group** (референс), **Positive outcome**. Чотири метрики: **Demographic Parity** (рівний Positive Rate), **Equalized Odds** (рівні TPR і FPR), **Equality of Opportunity** (рівні TPR), **Predictive Rate Parity** (рівні PPV). Важливо: неможливо оптимізувати одразу під кілька fairness-метрик — обирайте одну до побудови моделі.

### ML Diagnostics (Середній)

**ML Diagnostics** — попередження під час/після тренування за категоріями: **Dataset Sanity Checks** (малий test, imbalance, MAPE із нулями), **Overfitting Detection** (чисті листя, забагато листків), **Leakage Detection** («Too good to be true?» при AUC >98%, підозріло висока важливість фічі), **Model Checks** (порівняння з dummy-моделлю), **ML Assertions** та ін. Окремо — **Model error analysis** (плагін): тренує Error Tree, що передбачає, де модель помиляється.

---

## Розгортання моделі у Flow (Базово → Просунуто)

### Saved Model і версії (Середній)

- **Deploy** з Lab створює **Train recipe** + **Saved Model** (зелений ромб) у Flow.
- Один Saved Model містить **кілька версій**; кожен (re)train додає нову. **Активна** лише одна — її використовують Retrain/Score/Evaluate за замовчуванням.
- Перегляд/порівняння: подвійний клік по Saved Model → вкладка **Versions** (метрики поряд, звіт кожної версії).
- **Активація версії** — у списку Versions (UI: *Make active / Set as active version*) або через Python (`set_active_version()`).

### Retrain (Середній → Просунуто)

- Виконується **запуском Train recipe** — ті самі налаштування, **нові дані** → **нова версія**.
- **Activate new versions** (налаштування Saved Model): **Automatically** (дефолт — нова версія стає активною) або **Manually** (нову версію не активують автоматично; ви/сценарій активуєте після перевірки).
- Типово автоматизується **Scenarios**: Build/Train → Evaluate → checks → активувати лише якщо краще за поточну. Зміна **дизайну** (алгоритм/фічі/hyperparameters) робиться в **Lab** і повторному Deploy, а не Retrain.

### Score recipe (Базово)

Застосовує Saved Model до **немічених** даних. Вхід: Saved Model (active version) + датасет (ті самі features). Вихід: вхідні дані + **prediction column**, **probability columns** `proba_<class>` (classification), prediction details, опційно **explanations**.

**Scoring engines:** **Local (Python)** (ширша сумісність), **Local (Optimized)** (Java, швидше, автоматично де можливо; *Force original backend* — щоб вимкнути), **Spark**, **SQL (in-database)**, **SQL (Snowflake Java UDF)**. Optimized підтримує RF/GBT/LightGBM/XGBoost/Extra Trees/Decision Trees/лінійні/SGD/NN; **не** — Deep NN, Naive Bayes, KNN, SVM, custom. Spark/SQL — лише в Score recipe (не API node).

### Evaluate recipe (Середній)

Оцінює версію Saved Model на **міченому** датасеті. Вхід: Saved Model (версія, дефолт active) + labelled dataset. До трьох виходів: **Model Evaluation Store (MES)**, **output dataset** (фічі + prediction + correctness), **metrics dataset** (лише метрики, рядок на запуск). Підтримує **Custom Evaluation Metrics** (Python). Обмеження: лише non-partitioned Classification/Regression/Time Series; **не** CV/Deep Learning. Використовує **Python**-engine (можливі дрібні відмінності з Optimized scorer).

### Standalone Evaluation recipe (Просунуто)

Для моделей, що **не** треновані в DSS Visual ML і не імпортовані з MLflow — тобто **зовнішніх передбачень без Saved Model**. Вхід — один датасет, що вже містить передбачення. Вихід — **тільки MES**. Конфіг (моделі-об'єкта немає): **Prediction type**, **Prediction column**, опційно **Labels column** (можна *Don't compute perf*), **Model classes**, **Probability mappings**. Без ground truth result-екрани недоступні, але **input data drift** і **prediction drift** працюють.

### Model Evaluation Stores і drift (Просунуто, коротко)

**MES** накопичує **Model Evaluations** у часі (рядок на запуск Evaluate). Кожна ME — контейнер метрик версії моделі на eval-датасеті + drift. Три типи drift (порівняння поточної ME з **reference** = тренувальні дані версії):

- **Input Data Drift** — глобальний drift score через domain classifier (random forest відрізняє reference від нових даних) + **univariate** drift по кожній фічі + **Feature Drift Importance**;
- **Prediction Drift** — зсув розподілу передбачень (ground truth **не** потрібен);
- **Performance Drift** — зміна фактичної якості (ground truth **потрібен**).

До метрик/drift кріпляться **Checks**, що вмикаються у Scenarios (алерти, авто-retrain).

> Повні деталі MES, drift, API node, bundles та deployment — у [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md).

---

## Спеціалізовані можливості (Просунуто)

### Time Series Forecasting

Окремий тип задачі (**не** регресія). Time column має meaning *Datetime with zone / no zone / Date only*. Конфіг: **Time variable**, **Target variable(s)**, **Time series identifier columns** (множинні ряди у long-форматі), **Forecast horizon**, **Gap**, **Quantiles**.

Моделі: baseline (**Trivial identity**, **Seasonal naive**, **NPTS**), statistical (**Seasonal trend / STL**, **AutoARIMA**, **Prophet**), deep learning через GluonTS/PyTorch (**Simple Feed Forward**, **DeepAR**, **Transformer**, **MQ-CNN**), ML (**Random Forest**, **XGBoost**, **Ridge**). Resampling пропусків (linear/quadratic/mean/…). Оцінка — time-based split + опційно **K-Fold (backtesting)**; метрики **MASE, MAPE, sMAPE, MAE, MSE, RMSE, MSIS, MAQL, MWQL, ND**. Потрібен code env *Visual Machine Learning and Timeseries Forecasting* (CPU/GPU).

> Увага: «ARIMA», «N-BEATS», «Feed forward» — назви зі **старого deprecated-плагіна**; у нативному Visual ML це **AutoARIMA**, **Simple Feed Forward**; **N-BEATS відсутній**.

### Causal Prediction / uplift (Просунуто)

Окремий тип (з DSS 12.0): передбачає **CATE** (ефект treatment на outcome), а не сам outcome. Конфіг: **Outcome** (числовий → causal regression; бінарний категоріальний → causal classification), **Treatment** (з **Control value**; підтримує multi-valued). Алгоритми: **meta-learners** (**S-learner**, **T-learner**, **X-learner**) і **Causal Forest**. Оцінка: **Uplift curve**, **Qini curve**, **AUUC**, **Qini coefficient**, **Net uplift**, **Treatment Randomization Test**, **Positivity Analysis**. **K-fold не підтримується**; несумісне з MLflow, ensembling, export, MES, Document Generator, SQL/Spark/Java scoring. Потрібен **Full Designer** + code env *Visual Causal Machine Learning*.

### Computer Vision (Просунуто)

Нативні **Image classification** і **Object detection** на **PyTorch** через transfer learning. Вхід: managed folder з зображеннями (jpg/jpeg/png) + датасет (рядок = зображення, колонка-шлях + target). Object detection target — JSON `[{ "bbox":[x,y,w,h], "category":"label" }]` (origin top-left). Pretrained: класифікація — **EfficientNet B0/B4(дефолт)/B7**; детекція — **Faster R-CNN (ResNet-50-FPN)**; контроль через **Number of retrained layers**. **Data augmentation**: Color, Horizontal/Vertical flip, Rotation, Crop. GPU рекомендований (CUDA). Пояснення — **heatmap** (фокус пікселів). Потрібен спец code env (Administration → Code envs → Internal envs setup).

### NLP

У Visual ML текст — через векторизацію (TF/IDF, count, hashing, embeddings; вище). Поза Visual ML — **NLP-модуль** (Language Detection, Named Entity Extraction, Sentiment Analysis, Translation, Summarization, OCR, Speech Recognition/Whisper, Text Embedding) і **LLM Mesh / Generative AI** (RAG, prompt studios, LLM-рецепти) — деталі у [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md).

### MLflow models import (Просунуто)

Імпорт зовнішніх MLflow-моделей (**лише `pyfunc` flavor**; **не** R/Spark; MLflow ≥ 2.0) як Saved Models. Джерела: Python API (шлях/managed folder), Experiment Tracking, Databricks registry. Ключове:

```python
saved_model = project.create_mlflow_pyfunc_model(name, prediction_type)  # REGRESSION/BINARY_CLASSIFICATION/MULTICLASS/OTHER
v = saved_model.import_mlflow_version_from_path("v1", model_dir, "code-env")
v.set_core_metadata(target_column_name, class_labels=..., get_features_from_dataset="eval_ds")
v.evaluate("eval_ds")  # активує performance-вкладку
```

Після імпорту: scoring recipe, API node, версії, Model Comparisons, MES, drift, Govern Node. **Experiment Tracking** — хостинг MLflow Tracking API (`project.setup_mlflow(...)`, autologging, **Experiments** у меню Machine Learning, кнопка **Deploy** на run).

### Model Document Generator (Середній)

Генерує **.docx**-документацію навченої моделі з `.docx`-шаблону: **Actions → Export model documentation** → дефолтний/власний шаблон → Export → Download. Шаблони per prediction-type. **Не** для ensemble, Keras/CV, MLflow, plugin-алгоритмів. Потребує активної graphical export library на інстансі. Python: `generate_documentation(...)`.

### Partitioned models (Просунуто)

**Одна модель на партицію** замість глобальної. Вмикається на **Target → Partitioning**. Кожна sub-модель окремо вчиться, має власний grid search, threshold, calibration. Scoring: **Partitioned** або **Partition dispatch** (SQL-engine не підтримує dispatch). **Тільки** prediction + Python backend; clustering/Deep Learning/CV **не** підтримуються.

---

## Приклади (how-to)

### Приклад 1. Швидкий prediction-прототип (Базово)

1. Відкрийте датасет → **Lab → AutoML Prediction** (або **Models → Create**).
2. Оберіть **target**; DSS визначить тип (binary/multiclass/regression).
3. Template — **Quick Prototypes** → **Create**.
4. **Train**. Перегляньте результати: ROC/AUC (classification) або scatter/R² (regression).
5. Найкращу модель → **Deploy** у Flow як Saved Model.

### Приклад 2. Інтерпретована модель з поясненнями (Середній)

1. Lab → **AutoML Prediction** → template **Interpretable Models**.
2. У **Features handling** перевірте encoding/rescaling; вимкніть нерелевантні фічі.
3. **Train** → **Feature importance** (Shapley) → **Partial dependence** для топ-фіч.
4. **Subpopulation analysis** по чутливій змінній → переконайтеся у відсутності disparity.
5. **What if?** — перевірте поведінку на крайових кейсах.

### Приклад 3. High Performance + hyperparameter search (Просунуто)

1. Lab → **AutoML Prediction** → **High Performance** (XGBoost/LightGBM/RF).
2. **Algorithms** → увімкніть діапазони hyperparameters; **Search strategy = Bayesian**, search CV = **K-fold (5)**, ліміт ітерацій/часу, parallelism (контейнери).
3. **Train** → стежте за графіком еволюції CV-score.
4. Перевірте **ML Diagnostics** (leakage/overfit) → **Deploy**.

### Приклад 4. Продакшн-цикл: Score + Evaluate + retrain (Просунуто)

1. **Deploy** модель → у Flow з'являється Saved Model.
2. **Score recipe**: Saved Model + нові дані → scored dataset (engine: Optimized).
3. **Evaluate recipe**: Saved Model + labelled dataset → **MES** + metrics dataset.
4. **Scenario**: періодично Train recipe (нова версія) → Evaluate → **Check** (AUC ≥ поріг, drift ≤ поріг) → активувати нову версію лише якщо пройшла (**Activate new versions = Manually**).

---

## Best practices

- **Починайте з Quick Prototypes** — швидко зрозумійте складність задачі, перш ніж вкладатися в High Performance.
- **Уникайте data leakage:** не включайте фічі, недоступні на момент прогнозу; для часових даних — **Time ordering** split; стережіться target/impact encoding поза CV; реагуйте на «Too good to be true?» діагностику.
- **Обирайте правильну метрику:** для незбалансованих класів — ROC AUC / F1 / Average Precision, не Accuracy; для бізнес-вартості — Cost Matrix Gain.
- **Валідуйте чесно:** K-fold для надійної оцінки; не плутайте search-CV (вибір hyperparameters) з фінальним test-набором.
- **Відтворюваність:** фіксуйте seed, версіонуйте код env, зберігайте налаштування в Saved Model (lineage), документуйте через Model Document Generator.
- **Design vs deployment:** експерименти — в Lab; продакшн — Saved Model + Train/Score/Evaluate recipes у сценаріях. Не редагуйте «бойову» модель наживо.
- **Інтерпретуйте перед деплоєм:** Feature importance + PDP + Subpopulation analysis + (за потреби) Fairness report.
- **Моніторте drift:** Evaluate recipe → MES → Checks → авто-алерти/retrain. Навіть без ground truth використовуйте input data drift і prediction drift.
- **Rescaling — для scale-sensitive алгоритмів** (SVM/KNN/лінійні/NN); для дерев не критично.
- **Контролюйте активацію версій:** у продакшні — **Manually**, активуйте лише після перевірки метрик/drift.

---

## Типові помилки

- **Accuracy на незбалансованих даних** — модель, що завжди передбачає мажоритарний клас, має «високу» accuracy; ML Diagnostics попереджає про imbalance.
- **Data leakage через майбутні/похідні від target фічі** — нереалістично високий AUC; реагуйте на leakage-діагностику.
- **Випадковий split для часових даних** — завищує якість; вмикайте Time ordering.
- **Плутанина search-CV і test** — оцінка якості за search-CV занижено-оптимістична; дивіться на окремий test-набір.
- **Очікування Shapley/explanations для Keras/CV** — недоступні; ці методи лише для Python backend tabular-моделей.
- **Score vs Evaluate розбіжності** — Evaluate на Python-engine, Score може бути на Optimized; дрібні відмінності — норма.
- **Забутий feature-engineering у Score** — нові дані мають пройти **ті самі** Prepare-кроки, що й train (інакше schema/розподіли не збігаються).
- **Авто-активація гіршої версії** — при `Activate new versions = Automatically` retrain мовчки замінює модель; у продакшні ставте Manually.
- **Очікування кількох fairness-метрик одночасно** — математично неможливо оптимізувати під усі; оберіть одну до побудови.
- **Text як feature «з коробки»** — текстові змінні за замовчуванням відхиляються; увімкніть і оберіть векторизацію.
- **Ігнорування code env для спец-задач** — time series / causal / computer vision потребують окремих code envs (інакше тренування не стартує).

---

## Перехресні посилання

- [02-projects-flow-core-concepts.md](02-projects-flow-core-concepts.md) — Flow, datasets, recipes, lineage.
- [03-datasets-connections-partitioning.md](03-datasets-connections-partitioning.md) — partitioning (для partitioned models).
- [04-visual-recipes.md](04-visual-recipes.md) — Join/Group/Window для feature engineering перед ML.
- [05-data-preparation-prepare-recipe.md](05-data-preparation-prepare-recipe.md) — Prepare recipe, Simplify text, Lab Script, meanings.
- [06-coding-recipes-code-envs-notebooks.md](06-coding-recipes-code-envs-notebooks.md) — code environments, notebooks (для custom/Keras/spec-задач).
- [08-llm-generative-ai-mesh.md](08-llm-generative-ai-mesh.md) — LLM Mesh embeddings, NLP, Generative AI.
- [09-scenarios-automation-metrics-checks.md](09-scenarios-automation-metrics-checks.md) — Scenarios, metrics, checks (авто-retrain, drift-алерти).
- [10-mlops-deployment-api-node.md](10-mlops-deployment-api-node.md) — API node, bundles, deployment, повні деталі MES/drift.
- [11-programmatic-api-python-client.md](11-programmatic-api-python-client.md) — Python API для ML (DSSMLTask, Saved Models, MLflow).
- [13-governance-security-collaboration.md](13-governance-security-collaboration.md) — Govern Node, fairness/responsible AI у governance.
- [16-glossary.md](16-glossary.md) — глосарій ML-термінів.

### Джерела

- [Machine learning — DSS docs](https://doc.dataiku.com/dss/latest/machine-learning/index.html)
- [Automated machine learning](https://doc.dataiku.com/dss/latest/machine-learning/auto-ml.html)
- [In-memory Python algorithms](https://doc.dataiku.com/dss/latest/machine-learning/algorithms/in-memory-python.html)
- [Features handling](https://doc.dataiku.com/dss/latest/machine-learning/features-handling/index.html)
- [Train/Test settings](https://doc.dataiku.com/dss/latest/machine-learning/supervised/settings.html)
- [Advanced models optimization](https://doc.dataiku.com/dss/latest/machine-learning/advanced-optimization.html)
- [Model results & explainability](https://doc.dataiku.com/dss/latest/machine-learning/supervised/results.html)
- [Model fairness report](https://doc.dataiku.com/dss/latest/machine-learning/supervised/model-fairness-report.html)
- [ML diagnostics](https://doc.dataiku.com/dss/latest/machine-learning/diagnostics.html)
- [Models lifecycle](https://doc.dataiku.com/dss/latest/machine-learning/models-lifecycle.html)
- [Scoring engines](https://doc.dataiku.com/dss/latest/machine-learning/scoring-engines.html)
- [Model evaluations & drift](https://doc.dataiku.com/dss/latest/mlops/model-evaluations/index.html)
- [Knowledge Base — ML & Analytics](https://knowledge.dataiku.com/latest/ml-analytics/index.html)
