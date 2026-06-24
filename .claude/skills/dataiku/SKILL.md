---
name: dataiku
description: Робота з платформою Dataiku (DSS / Dataiku Cloud) — data pipelines/ETL, Flow та recipes (visual + code), підготовка даних (Prepare), Machine Learning / Visual ML / AutoML, LLM Mesh та GenAI, scenarios/автоматизація, metrics/checks/data quality, MLOps та deployment (bundles, API node), програмний доступ (dataiku / dataikuapi / REST API), dashboards/charts/webapps, governance/security, plugins та applications. Викликай цей skill щоразу, коли задача стосується Dataiku: побудова чи дебаг Flow, написання recipe (Python/SQL/R), налаштування scenario, тренування/деплой моделі, робота з connections/datasets, автоматизація, або питання «як це зробити в Dataiku».
---

# Dataiku skill

Цей skill — твій робочий протокол для будь-яких задач у **Dataiku**. Він зв'язує тебе з двома джерелами істини в цьому репозиторії:

1. **`docs/`** — вичерпна база знань про функціонал Dataiku та best practices (16 документів, ~9700 рядків). Це «як має бути» за офіційною документацією (DSS 13.x/14.x).
2. **`MEMORY.md`** — жива пам'ять про **фактичний** процес роботи в Dataiku на цьому інстансі. Це «як є насправді»: конкретні connections, conventions, версія, відомі особливості/відхилення.

> **Золоте правило:** коли загальні знання (`docs/`) суперечать тому, що ти бачиш у реальному інстансі — **істина в інстансі**. Зафіксуй розбіжність у `MEMORY.md` і працюй за фактом.

---

## Обов'язковий робочий цикл

Виконуй для **кожної** Dataiku-задачі:

### 1. Орієнтація (READ FIRST)
- **Прочитай `MEMORY.md`** повністю (він короткий). Звідти ти дізнаєшся: який інстанс, версія, які connections/conventions використовуються, які є відомі особливості. Це визначає, як працювати тут і зараз.
- Визнач, якого домену стосується задача, і відкрий профільний документ із `docs/` (мапу див. нижче). Читай цілеспрямовано — потрібний розділ, а не весь файл.

### 2. Уточнення
- Якщо в `MEMORY.md` бракує критичного факту (напр. назва connection, project key, версія DSS, чи є API node) — **спитай користувача** або, якщо є доступ до інстансу/API, перевір емпірично. Не вигадуй.
- Звіряй припущення з реальністю інстансу перед тим, як писати код/конфіг, що від них залежить.

### 3. Виконання
- Дій за best practices з `docs/15-best-practices.md` + профільного документа.
- Для коду використовуй коректні API (`dataiku` всередині DSS, `dataikuapi` ззовні) — шпаргалка в `references/code-snippets.md`.
- Дотримуйся conventions проєкту, зафіксованих у `MEMORY.md` (naming, зони Flow, теги, цільові connections).

### 4. Запис у пам'ять (WRITE BACK)
Онови `MEMORY.md`, коли:
- виявив, що **факт у програмі відрізняється** від загальних знань `docs/` (відхилення, особливість версії, кастомний плагін, нестандартна поведінка);
- дізнався **новий стійкий факт** про інстанс (нова connection, convention, рішення, що приймалося і чому);
- закрив **open question** або з'явилось нове.

Кожен запис: **дата (YYYY-MM-DD) + факт + джерело/контекст**. Пиши лаконічно. Не дублюй те, що вже є в `docs/` — у `MEMORY.md` лише специфіка цього інстансу/процесу.

---

## Мапа: яка задача → який документ

| Задача | Документ у `docs/` |
|--------|--------------------|
| Що таке ноди, deployment, архітектура | `01-overview-architecture.md` |
| Проєкти, Flow, зони, теги, variables, build, git | `02-projects-flow-core-concepts.md` |
| Datasets, connections, формати, partitioning | `03-datasets-connections-partitioning.md` |
| Visual recipes: Group, Join, Window, Stack, Pivot… | `04-visual-recipes.md` |
| Prepare recipe, processors, meanings, формули (GREL) | `05-data-preparation-prepare-recipe.md` |
| Code recipes (Python/SQL/R/Spark), code envs, notebooks | `06-coding-recipes-code-envs-notebooks.md` |
| ML: Visual ML/AutoML, моделі, оцінювання, інтерпретація | `07-machine-learning-visual-ml.md` |
| LLM Mesh, GenAI, RAG, prompts, agents, Dataiku Answers | `08-llm-generative-ai-mesh.md` |
| Scenarios, triggers, reporters, metrics, checks, data quality | `09-scenarios-automation-metrics-checks.md` |
| MLOps: bundles, deployment, API node, drift, monitoring | `10-mlops-deployment-api-node.md` |
| Програмний доступ: `dataiku`, `dataikuapi`, REST, CLI | `11-programmatic-api-python-client.md` |
| Dashboards, charts, tiles, webapps, експорт, reporting | `12-dashboards-charts-webapps-reporting.md` |
| Users/groups, permissions, UIF, audit, Govern, RAI | `13-governance-security-collaboration.md` |
| Plugins, компоненти, applications, app-as-recipe, reuse | `14-plugins-applications-extensibility.md` |
| Зведені best practices + антипатерни + production checklist | `15-best-practices.md` |
| Глосарій термінів | `16-glossary.md` |

Швидка довідка без занурення в документ — `references/cheatsheet.md`.
Готові патерни коду (`dataiku` / `dataikuapi`) — `references/code-snippets.md`.
Програмна побудова Flow через API (проєкти/datasets/recipes/scenarios) — `references/programmatic-flow.md`.

---

## Ключові принципи Dataiku (тримай у голові)

- **Visual + code співіснують.** Visual recipes для типових трансформацій (читабельність, engine-pushdown), code recipes — коли visual не вистачає. Не переписуй у код те, що чисто робиться візуально.
- **Pushdown — це продуктивність.** Коли дані в SQL/Spark connection, тримай обчислення там (in-database / Spark engine), не тягни все в пам'ять DSS. Перевіряй engine рецепта.
- **Не вантаж усе в pandas.** Для великих даних — `iter_dataframes()` / chunked `get_writer()`, або взагалі SQL-pushdown.
- **Ідемпотентність.** Recipes і scenarios мають давати той самий результат при повторному запуску (overwrite, не сліпий append).
- **Розділяй середовища.** Design (розробка) → Automation (прод-запуски) → API (real-time scoring) через bundles. Не «розробляй у проді».
- **Відтворюваність через code environments.** Фіксуй залежності; не покладайся на base env для проду.
- **Секрети — не в коді.** Credentials через connections / per-user / `get_secret()`, не хардкод api_key/паролів.
- **Least privilege + audit.** Права — на групи, не на окремих; розділення обов'язків dev/prod.

Повний список і антипатерни — `docs/15-best-practices.md`.

---

## Програмна робота з живим інстансом (Режим A / Режим B)

Коли задача — **створити чи змінити об'єкти в Dataiku програмно** (побудувати Flow, datasets, recipes, scenarios), є два режими:

- **Режим A — `dataikuapi` ЗЗОВНІ DSS.** Claude напряму з робочого пристрою під'єднується до сервера й створює об'єкти. Потрібен мережевий доступ до DSS-хоста (підтверджено для цього інстансу — див. `MEMORY.md`). Патерни — `references/programmatic-flow.md`.
- **Режим B — пакет `dataiku` ВСЕРЕДИНІ DSS.** Claude генерує код, який вставляється й запускається в notebook / code recipe всередині Dataiku. Обходить мережу/політики; код біжить на сервері. Патерни — `references/code-snippets.md` (розділ 1).

**Обов'язкові запобіжники для будь-якого режиму A:**
- ❗ **API key — лише з env var `DSS_API_KEY`, НІКОЛИ не в repo/skill/коді** (навіть приватний repo).
- ❗ Спершу **тестовий проєкт**, не прод — ключ має твої права і може видаляти/перезаписувати.
- ❗ **Ідемпотентність** — патерн «перевір-чи-існує-перед-створенням», щоб повторний запуск не дублював і не падав.
- ❗ on-prem → ймовірно **self-signed TLS** (`verify=False` з обережністю або CA банку).
- ❗ Код виконується з **реальним робочим Dataiku** — не подавай невірифіковані API-виклики; сумнівні сигнатури звіряй (у `references/programmatic-flow.md` такі місця позначені `# ⚠️ звірити...`).

Концептуальна повна довідка по API — `docs/11-programmatic-api-python-client.md`.

## Коли НЕ вистачає інформації

- Багато сторінок офіційної документації версієзалежні — точні UI-підписи, шляхи й набір фіч можуть відрізнятись між 13.x і 14.x та залежати від ліцензії/плагінів. Якщо точність критична — **звір на живому інстансі**, а не покладайся сліпо на `docs/`.
- Якщо знайшов розбіжність — це сигнал оновити `MEMORY.md` (саме для цього він існує).
