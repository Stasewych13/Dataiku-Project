# 01. Огляд та архітектура Dataiku

> Базовий документ бази знань. Дає цілісну картину того, **що таке Dataiku**, **з яких частин складається інстанс**, **які бувають типи нод** і **як проєкт проходить шлях від розробки до продакшену**. Решта документів поглиблюють окремі підсистеми — див. перехресні посилання в кінці кожного розділу та у фіналі.

---

## Зміст

1. [Вступ: що таке Dataiku](#1-вступ-що-таке-dataiku)
2. [Ключові концепції (короткий словник)](#2-ключові-концепції-короткий-словник)
3. [Архітектура інстансу та типи нод](#3-архітектура-інстансу-та-типи-нод)
4. [Внутрішні компоненти ноди (backend, JEK, FEK, ...)](#4-внутрішні-компоненти-ноди)
5. [Lifecycle: шлях dev → prod через bundles](#5-lifecycle-шлях-dev--prod-через-bundles)
6. [Варіанти розгортання](#6-варіанти-розгортання)
7. [Kubernetes та elastic AI computation](#7-kubernetes-та-elastic-ai-computation)
8. [Editions та ліцензування (високий рівень)](#8-editions-та-ліцензування)
9. [Огляд UI: homepage, навігація, права доступу](#9-огляд-ui)
10. [Best practices розгортання та поділу середовищ](#10-best-practices-розгортання)
11. [Типові помилки](#11-типові-помилки)
12. [Перехресні посилання](#12-перехресні-посилання)

---

## 1. Вступ: що таке Dataiku

**Dataiku** — це **end-to-end платформа для роботи з даними, аналітики, machine learning та generative AI**. Продукт історично називається **DSS** (Data Science Studio); сьогодні компанія позиціонує його як **«The Enterprise AI Platform for Analytics, ML & AI Agents»**. У цій базі знань під словами «Dataiku» і «DSS» мається на увазі та сама платформа (інстанс DSS).

Ключова ідея Dataiku — **єдине середовище, де співіснують візуальний (no-code/low-code) і кодовий (full-code) підходи**, і де над одним проєктом колаборують ролі від бізнес-аналітика до data scientist та ML-інженера. Це відрізняє Dataiku від вузьких інструментів: одна платформа покриває весь життєвий цикл — від підключення до джерел даних і їх підготовки, через візуалізацію, AutoML і кодові рецепти, до деплою моделей у продакшен, моніторингу й governance.

### Філософія платформи

- **Visual + Code в одному Flow.** Один і той самий конвеєр обробки даних (**Flow**) може містити і візуальні рецепти (**visual recipes**), і кодові рецепти на Python/R/SQL/Spark. Аналітик-новачок працює мишкою, досвідчений data scientist пише код — обидва бачать той самий граф залежностей. (Базово)
- **Колаборація як принцип.** Проєкти, Wiki, dashboards, workspaces, коментарі, права доступу — все вбудовано, щоб над одним проєктом могли працювати кілька людей різних ролей. Див. `13-governance-security-collaboration.md`.
- **Відтворюваність і governance.** Flow декларативний: Dataiku знає залежності між датасетами й рецептами та вміє перебудовувати лише те, що змінилося. Деплой у продакшен — версійований (через **bundles**), а корпоративний нагляд забезпечує **Govern node**.
- **End-to-end AI lifecycle.** Платформа покриває **DataOps → MLOps → LLMOps** в одному місці: підготовка даних, ML, generative AI / **LLM Mesh**, побудова застосунків, оркестрація в продакшені та моніторинг.

> Споживчі продукти Dataiku: **самостійно керований DSS** (self-managed, на ваших серверах) та **Dataiku Cloud** (повністю керований SaaS від Dataiku). Деталі — у розділі [6](#6-варіанти-розгортання).

Суміжні документи для занурення: загальна модель роботи з проєктами — `02-projects-flow-core-concepts.md`; джерела даних — `03-datasets-connections-partitioning.md`.

---

## 2. Ключові концепції (короткий словник)

Цей розділ — мінімальний контекст, щоб читати решту документа. Повний термінологічний словник — у `16-glossary.md`.

| Термін | Що це (стисло) |
|---|---|
| **Instance** | Одна інсталяція DSS (одна нода певного типу). Те, до чого ви заходите браузером. |
| **Node** | Роль інстансу: **Design**, **Automation**, **API**, **Deployer**, **Govern**. |
| **Project** | Контейнер усієї роботи: датасети, рецепти, Flow, моделі, dashboards, сценарії. Див. `02-...`. |
| **Dataset** | Логічна таблиця записів зі схемою; може бути external (джерело) або managed (створений DSS). Див. `03-...`. |
| **Recipe** | Крок трансформації: бере вхідні датасети → дає вихідні. Visual або code. Див. `04-...`, `06-...`. |
| **Flow** | Граф залежностей датасетів і рецептів усередині проєкту. Див. `02-...`. |
| **Connection** | Налаштоване підключення до сховища даних (SQL, S3, HDFS, тощо). Див. `03-...`. |
| **Bundle** | Версійований архів проєкту з Design node для деплою на Automation node. Див. розділ [5](#5-lifecycle-шлях-dev--prod-через-bundles). |
| **API service** | Сервіс реального часу (scoring/прогноз/функція), що деплоїться на API node. Див. `10-...`. |
| **Code env** | Ізольоване Python/R-середовище з фіксованим набором пакетів. Див. `06-...`. |

---

## 3. Архітектура інстансу та типи нод

Інстанс Dataiku — це інсталяція DSS, яка виконує одну з **ролей (типів нод)**. У продакшен-розгортанні різні ноди розділяють відповідальність: де **розробляють**, де **виконують у проді**, де **відповідають на запити реального часу**, хто **керує деплоями** і хто **наглядає та узгоджує**.

### 3.1 Design node (Базово)

**Design node** — це **середовище розробки**. Тут аналітики й data scientists:

- підключаються до джерел даних, створюють датасети та рецепти;
- будують **Flow** проєкту;
- тренують і покращують ML-моделі (**Visual ML**, див. `07-...`);
- створюють **API services** і **сценарії** (scenarios);
- збирають **bundles** для деплою в продакшен.

Це місце, де відбувається експеримент, ітерація та основна творча робота. Один Design node зазвичай обслуговує багатьох користувачів і багато проєктів.

### 3.2 Automation node (Базово / Середній)

**Automation node** (інколи зустрічається назва **Execution node**) — це **продакшен-середовище виконання**. Його призначення — **відокремити розробку від продакшену**:

- на нього **деплоять bundles** з Design node;
- він виконує **сценарії за розкладом** (нічні перебудови Flow, оновлення моделей) проти **продакшен-даних**;
- забезпечує **стабільність**: на нього не пушать «сирий» код напряму — лише перевірені, версійовані bundles.

Розділення Design/Automation — фундамент того, що в Dataiku вважається коректним dev→prod процесом. Це робить деплої контрольованими, версійованими й відтворюваними. Деталі сценаріїв і автоматизації — `09-scenarios-automation-metrics-checks.md`.

### 3.3 API node + API Deployer (Середній)

**API node** — це **сервер застосунків реального часу**, що відповідає на **HTTP/REST-запити** (наприклад, scoring моделі для одного запису за мілісекунди). Ключові властивості:

- **горизонтально масштабований** і **highly available** — можна запустити кілька API nodes за load balancer-ом;
- виконує **API services**, які ви створили на Design або Automation node і задеплоїли;
- може працювати як на окремих серверах, так і **на Kubernetes** (для еластичного масштабування).

> Не плутати: **batch-прогноз** (scoring цілого датасету) — це робота для Flow на Design/Automation node; **real-time scoring** одного запису по HTTP — це робота для **API node**. Детально про API node, deployment та versioning — `10-mlops-deployment-api-node.md`.

### 3.4 Deployer (Project Deployer + API Deployer) (Середній)

**Deployer** — це **централізоване місце керування продакшен-деплоями**. Він має два схожі, але окремі компоненти:

- **Project Deployer** — керує деплоєм **bundles** (проєктів) на **Automation nodes**: дозволяє керувати всіма Automation nodes, деплоїти й оновлювати bundles, моніторити стан задеплоєних проєктів.
- **API Deployer** — керує деплоєм **API services** на **API nodes**.

Deployer можна розгорнути двома способами:
- **attached** — «приєднаний» до Design node (простіший варіант для невеликих установок);
- **dedicated / standalone** — окрема нода (рекомендовано, коли у вас кілька Design та/або Automation nodes).

Ключові поняття Deployer-а: **infrastructures** (зареєстровані цільові середовища — групи Automation/API nodes, часто розділені на stages типу *dev/test/prod*) і **deployments** (конкретний факт деплою певної версії bundle/API service у певну infrastructure).

### 3.5 Govern node (Просунуто)

**Govern node** — це **окрема нода для корпоративного AI governance**: централізоване місце для управління нthroughout life cycle аналітики, моделей та AI-агентів на рівні організації. Технічно у Govern node замість стандартного **backend** працює процес **governserver** (див. розділ [4](#4-внутрішні-компоненти-ноди)).

Що дає Govern node:

- **Sign-off / approval workflows.** Без узгодження в Govern node bundle/модель **не може перейти** з dev-середовища у продакшен. Govern блокує деплой на Automation/API node без рев'ю моделі визначеними стейкхолдерами.
- **Налаштовувані, але консистентні шаблони** governance-процесів: чіткі кроки й gates для кожної стадії data/AI-проєкту, призначення стейкхолдерів, нотатки, прикріплення документації.
- **Централізований трекінг** усіх data/AI-проєктів організації в одному місці (прогрес, scope, стейкхолдери).
- **Audit timeline** — хто, що, коли і над якими проєктами робив.
- **Повна life-cycle governance** через DataOps, MLOps і LLMOps.

> Govern node доступний у вищих комерційних edition-ах і **не входить** у Free Edition та (на момент написання) у 14-денний trial. Деталі governance — `13-governance-security-collaboration.md`.

### 3.6 Як ноди взаємодіють (текстова діаграма)

```
            РОЗРОБКА                  КЕРУВАННЯ ДЕПЛОЄМ            ПРОДАКШЕН
        ┌───────────────┐          ┌────────────────────┐     ┌──────────────────┐
        │  DESIGN NODE  │          │     DEPLOYER       │     │  AUTOMATION NODE │
        │  - Flow       │  bundle  │  ┌──────────────┐  │     │  - scenarios     │
        │  - Visual ML  ├─────────►│  │Project Deploy├──┼────►│  - prod-дані     │
        │  - сценарії   │          │  └──────────────┘  │     └──────────────────┘
        │  - API service│          │  ┌──────────────┐  │     ┌──────────────────┐
        │               ├─────────►│  │ API Deployer ├──┼────►│   API NODE(s)    │
        └───────┬───────┘  API svc │  └──────────────┘  │     │  - real-time     │
                │                  └────────────────────┘     │    HTTP scoring  │
                │ (опційно sign-off)         ▲                └──────────────────┘
                ▼                            │
        ┌───────────────┐                    │ approvals / audit
        │  GOVERN NODE  │────────────────────┘
        │  (governserver)│
        └───────────────┘
```

**Сценарій руху:** розробник на Design node будує Flow → пакує його у **bundle** → публікує bundle у **Project Deployer** → (опційно проходить sign-off у **Govern node**) → Deployer деплоїть bundle на **Automation node**, де він виконується за розкладом проти продакшен-даних. Паралельно **API services** через **API Deployer** потрапляють на **API nodes** для real-time запитів.

---

## 4. Внутрішні компоненти ноди

Цей розділ — про те, **що відбувається «під капотом»** одного інстансу DSS. Корисно для адміністраторів і для розуміння логів/моніторингу. (Середній / Просунуто)

### 4.1 supervisord

**supervisord** — менеджер процесів верхнього рівня. Запускається командою `./bin/dss start` і **стартує, перезапускає та моніторить** головні процеси DSS: `nginx`, `backend`, `jupyter` та інші. Якщо якийсь top-level процес впав — supervisord його підніме.

### 4.2 backend (головний процес)

**backend** — **головний Java-процес DSS**. Відповідає за:

- конфігурацію, користувачів, права доступу;
- внутрішній API та веб-UI;
- планування й оркестрацію **сценаріїв** (scheduling).

Це довгоживучий процес з відчутним обсягом Java-пам'яті (heap налаштовується). Логи — `run/backend.log`.

> На **Govern node** замість `backend` працює процес **governserver** — це й робить Govern ноду окремим типом.

### 4.3 JEK — Job Execution Kernel

**JEK (Job Execution Kernel)** — це **процес, що виконує конкретний job** (те, що відбувається, коли ви тиснете **Build** у Flow). Особливості:

- **один JEK на один запущений job** (їх може бути багато одночасно);
- JEK рахує **залежності** та виконує рецепти на **DSS engine**;
- для інших engine-ів і кодових рецептів JEK **запускає дочірні процеси**: Python, R, Spark, SQL тощо;
- DSS **прогріває** (pre-start) JEK-и заздалегідь, щоб пришвидшити старт jobs;
- у командному рядку розпізнається по `DSSJobKernelMain`; PID видно в UI job-а та в моніторингу;
- логи — `jobs/PROJECT/JOB_ID/output.log`.

Завдяки тому, що кожен job — окремий процес, проблема в одному job не валить backend.

### 4.4 FEK — Future Execution Kernel

**FEK (Future Execution Kernel)** — це **воркер-процес, якому backend делегує важку (по пам'яті) роботу**: побудову **семплів даних** (samples), довгі фонові задачі тощо. Технічна суть:

- FEK — **той самий Java-застосунок, що й backend** (ті ж jars/classpath), але запущений з **іншим main-класом**; обидві JVM хостять однакові Java-об'єкти;
- backend ставить задачу через `FutureService`: серіалізує `FutureThread` у JSON і шле в FEK звичайним **REST-викликом**;
- **один FEK обробляє одну задачу за раз**; DSS періодично «вбиває» надто роздуті FEK-и;
- **надійність:** якщо операція спричиняє `OutOfMemoryException`, гине **лише FEK**, а backend живий і повідомляє користувача, що операція з'їла забагато пам'яті;
- DSS тримає **пул FEK-ів**: на старті запускає налаштовану кількість, і щойно один FEK зайнято — одразу стартує новий, щоб у standby завжди були вільні;
- у командному рядку розпізнається по `DSSFutureKernelName`.

> Запам'ятати різницю: **JEK виконує jobs (Build у Flow)**; **FEK обслуговує backend** (семпли, важкі фонові операції UI). Обидва — окремі JVM-процеси, що ізолюють пам'ять від backend.

### 4.5 nginx (reverse proxy)

**nginx** — reverse proxy, через який проходить веб-трафік до DSS. Інсталятор резервує **діапазон портів** (від базового PORT до приблизно PORT+10), бо різні внутрішні компоненти слухають свої порти; nginx відповідає за публічний вхід.

### 4.6 Jupyter (ipython) сервер і kernels

**Jupyter server** (історична назва ipython) керує всіма **notebooks** (Python, R, Scala). Форкається supervisord-ом, логи — `run/ipython.log`. Кожен відкритий notebook має свій **Jupyter kernel** — окремий обчислювальний процес, який **лишається живим**, навіть коли користувач пішов з notebook (це дозволяє асинхронну роботу, але потребує контролю пам'яті). Детально про notebooks і code envs — `06-coding-recipes-code-envs-notebooks.md`.

### 4.7 Code env execution

**Code environments** — ізольовані Python/R-середовища з фіксованими пакетами. Коли рецепт чи notebook потребує певного code env, відповідний процес (дочірній процес JEK-а для рецептів, або Jupyter kernel для notebook-а) запускається в цьому середовищі. Це дає **відтворюваність** (одні й ті самі версії пакетів) та ізоляцію між проєктами/командами. Code envs повністю сумісні з контейнеризованим виконанням (див. розділ [7](#7-kubernetes-та-elastic-ai-computation)). Деталі — `06-...`.

### 4.8 Інші процеси

DSS запускає й інші спеціалізовані процеси: **Spark**-рецепти, **тренування ML-моделей**, **webapps** (Flask, Shiny, Bokeh, Dash, Streamlit — див. `12-dashboards-charts-webapps-reporting.md`). Для production-моніторингу подій існує **EventServer** — централізований приймач аудит-логів/подій з нод (корисний для аудиту й MLOps; деталі — `10-...` і `13-...`).

### 4.9 Runtime / internal databases та DATADIR

DSS зберігає **внутрішні метадані** (конфігурацію, історію jobs, стан тощо) у власних внутрішніх базах. Усе живе всередині **data directory** (далі **DATADIR** або `DATA_DIR`) — окремої від інсталяційного коду директорії, яка містить:

- `config/` — конфігурацію (включно з `config/license.json`);
- `run/` — логи top-level процесів (`backend.log`, `ipython.log`, ...);
- `jobs/` — логи й артефакти виконаних jobs;
- внутрішні бази даних, Python venv, керовані датасети (managed datasets, якщо вони на файловій системі) та інше.

DATADIR має повністю поміщатися на **одному mount** і бути звичайною текою; Dataiku рекомендує закладати **щонайменше 100 GB**. Розділення на **INSTALL_DIR** (код «kit») та **DATA_DIR** (дані/конфіг) — базовий концепт інсталяції (див. розділ [6.1](#61-self-managed-on-prem--cloud-vm)).

> Точний перелік і тип внутрішніх runtime-баз (наприклад, чи це H2/PostgreSQL для тих чи інших підсистем) залежить від версії та конфігурації — **варто звіряти на живому інстансі** (див. фінальні нотатки оркестратору).

---

## 5. Lifecycle: шлях dev → prod через bundles

Це центральний процес експлуатації Dataiku. (Середній)

### 5.1 Що таке bundle

**Bundle** — це **архів, що містить конкретну версію Flow**, побудованого на Design node. У bundle входять:

- метадані проєкту, конфігурація Flow, рецепти, схеми;
- (опційно) **додаткові дані** — наприклад, невеликі довідкові датасети;
- **release notes** (нотатки релізу);
- **shared objects** за потреби.

Bundle **версіюється** — це знімок проєкту в певний момент, придатний для відтворюваного деплою.

### 5.2 Кроки життєвого циклу

```
1. Design  ── будуємо й тестуємо Flow на Design node
2. Bundle  ── пакуємо перевірену версію проєкту в bundle
3. Publish ── публікуємо bundle у Project Deployer
4. Infra   ── (одноразово) налаштовуємо infrastructures: цільові Automation nodes + stages (dev/test/prod)
5. Deploy  ── Deployer розгортає bundle у вибрану infrastructure (Automation node)
6. Monitor ── стежимо за здоров'ям і статусом задеплоєних проєктів
```

Аналогічний цикл для **API services**: створили на Design node → опублікували в **API Deployer** → задеплоїли на **API node(s)**.

### 5.3 Роль stages та Govern

**Infrastructures** у Deployer-і зазвичай розкладені на **stages** (наприклад *Development → Test → Production*). Це дозволяє послідовно «просувати» один і той самий bundle середовищами. Якщо в організації увімкнено **Govern node**, перехід між критичними stages вимагає **sign-off** — без затвердження bundle не потрапить у продакшен.

Детальніше про MLOps, версіонування деплоїв, rollback і моніторинг — `10-mlops-deployment-api-node.md`. Про сценарії, що ганяють Flow у проді, — `09-scenarios-automation-metrics-checks.md`.

---

## 6. Варіанти розгортання

Документація Dataiku виділяє **три основні підходи до встановлення**: **Dataiku Cloud**, **Dataiku Cloud Stacks** і **Custom (self-managed)**. (Середній)

### 6.1 Self-managed (on-prem / cloud VM)

«**Custom Dataiku install**» — ви самі ставите DSS на Linux-сервери (фізичні або VM у будь-якій хмарі) і **самі володієте інфраструктурою**.

- **ОС: лише Linux.** Підтримуються RHEL-сумісні дистрибутиви, Debian/Ubuntu, Amazon Linux, SUSE (SLES). Windows/macOS як серверна ОС **не підтримуються**.
- **Дві директорії:** `INSTALL_DIR` (розпакований «kit» з кодом) та `DATA_DIR` (DATADIR — конфіг, логи, venv, дані; ≥100 GB, один mount).
- **Установка:** розпакувати `dataiku-dss-VERSION.tar.gz`, запустити `installer.sh -d DATA_DIR -p PORT [-l LICENSE_FILE]`. DSS використовує діапазон портів від PORT до ~PORT+10. Інсталятор перевіряє відсутні системні залежності.
- **Сервіс:** `DATA_DIR/bin/dss start` (вручну) або як systemd-сервіс `sudo systemctl start dataiku`. Boot-старт налаштовується скриптом `install-boot.sh`. DSS працює під **окремим non-root користувачем**.

Цей варіант дає максимум контролю й кастомізації; натомість усе адміністрування — на вас.

### 6.2 Dataiku Cloud (SaaS)

**Dataiku Cloud** — **повністю керований SaaS**, який **хостить і адмініструє сама Dataiku**. Користувач «ніколи не думає про сервери, адміністрування чи апгрейди»: інфраструктура, оновлення, бекапи й моніторинг — на боці Dataiku. Тут **elastic AI compute керується автоматично** (вам не треба самим налаштовувати Kubernetes).

> **Іменування (важливо):** продукт запущено у 2021 як **Dataiku Online** (для SMB та стартапів), а **близько березня 2023 перейменовано на Dataiku Cloud**. Тобто **«Dataiku Online» і «Dataiku Cloud» — це той самий SaaS-продукт**; «Online» — стара назва.

### 6.3 Dataiku Cloud Stacks + Fleet Manager

**Dataiku Cloud Stacks** — це «**керований, але у вашій хмарі**» варіант: повністю керована інсталяція Dataiku, **розгорнута у власному cloud-тенанті клієнта** (AWS, Azure, GCP), причому **Dataiku не має доступу до ваших даних**. Це «no-code path» на основі best-practice **blueprints**, розроблених разом із хмарними провайдерами; включає Elastic AI, посилену безпеку, R-підтримку та auto-healing.

Серцем Cloud Stacks є **Fleet Manager (FM)** — центральний компонент, що автоматизує **повний життєвий цикл** інстансів: deploy, upgrade, backup, restore, configure. Ключові поняття FM:

- **Instance** — окрема інсталяція Design / Automation / Deployer ноди; кожен інстанс — на власній VM.
- **Fleet** — набір інстансів, якими керує FM (Design, Execution/Automation, Deployer; зазвичай один Deployer на fleet).
- **Instance template** — багаторазова конфігурація; інстанс завжди стартує з шаблону й лишається з ним пов'язаним.
- **Virtual network** — мережевий контекст інстансів (на Azure — діапазон CIDR /16, DNS/HTTPS).
- **Load balancers** — розподіляють трафік, дають load balancing і high availability (особливо для Automation nodes).
- **FM agent** — софт Dataiku, що працює поруч із DSS на кожному інстансі та виконує адмін-задачі від FM.
- **Setup actions** — кастомні дії, що виконуються під час старту софту.

Сам FM розгортається cloud-нативним шаблоном: **CloudFormation** (AWS), **ARM template** (Azure), еквівалент (GCP). **Provisioning** має дві стадії: (1) cloud-ресурси (VM + data disk, на AWS — окремий **EBS volume**), (2) software startup (агент ставить/оновлює DSS, виконує setup actions). При **deprovisioning** VM зупиняють, а диск (EBS) зберігають. FM має й **Python API** (клієнти для AWS/Azure/GCP).

Детальні адмін-аспекти безпеки/мереж — `13-governance-security-collaboration.md`.

---

## 7. Kubernetes та elastic AI computation

(Середній / Просунуто)

### 7.1 Що таке elastic AI computation

Точний термін Dataiku — **«Elastic AI computation»**. DSS вміє **виносити обчислення з локального DSS-сервера на еластичні кластери на базі Kubernetes**. Навіщо:

- **масштабованість** «локального» коду понад можливості однієї DSS-ноди;
- кращі обчислення через контейнери;
- доступ до **віддалених машин з GPU**, навіть якщо на самій DSS-машині GPU немає.

> На **Dataiku Cloud** це повністю кероване автоматично. Ручне налаштування нижче стосується **self-hosted / Cloud Stacks**.

### 7.2 Що контейнеризується, а що — ні

**Може виконуватись у контейнерах:**

- Python та R **рецепти й notebooks**;
- візуальні рецепти на **DSS engine**;
- рецепти/notebooks/візуальні рецепти на **Spark** (Spark on Kubernetes);
- Python/R-рецепти з плагінів;
- **початкове тренування**, **перетренування** та **оцінювання** in-memory ML-моделей;
- **scoring** ML-моделей (крім «Optimized engine»).

**Лишається на DSS-сервері:** самі **Design/Automation ноди**, веб-UI, оркестрація, метадані й data directory. У контейнери виносяться лише конкретні обчислювальні задачі.

### 7.3 Конфігураційні об'єкти

В **Administration > Settings > Containerized execution**:

- **Image Build Configurations** — як збираються образи (registry URL; для EKS — push у ECR);
- **Containerized Execution Configurations** — runtime: який image build використовувати, **kubectl context** (кластер), обмеження CPU/пам'яті, **Kubernetes namespace** (квоти), дозволи для груп користувачів; кілька конфігів = різні розміри/квоти контейнерів для різних користувачів;
- **Spark Configurations** — окремі налаштування для Spark-контейнерів.

**Base images** — заздалегідь зібрані образи (опційно з R та CUDA/GPU), будуються командами `./bin/dssadmin build-base-image --type container-exec|spark|cde`. **Важливо: після кожного апгрейду DSS усі base images треба перебудувати.**

### 7.4 Managed Kubernetes: EKS / AKS / GKE

DSS інтегрується з керованими Kubernetes-сервісами через **плагіни** (ставите плагін під свого провайдера):

- **Amazon EKS** → registry **ECR**;
- **Azure AKS** → registry **ACR**;
- **Google GKE** → registry **GCR**.

DSS може **створювати й керувати кластерами сам** («managed») або **приєднуватись до наявного** («unmanaged») — у **Administration > Clusters**. Можна задати **default cluster** глобально (з override на рівні проєкту). Для економії — **scenario steps**, що піднімають/гасять кластер за розкладом, і **тимчасові кластери** під конкретний сценарій. Підтримуються також OpenShift, NVIDIA DGX, generic unmanaged Kubernetes та звичайний Docker.

Окремо: **API nodes теж можна розгортати на Kubernetes** для горизонтального масштабування real-time scoring (це інша підсистема, ніж elastic AI computation для рецептів/тренування).

> Точні мінімальні версії Spark і деталі deployment mode для Spark-on-K8s **варто звірити з актуальною сторінкою документації** для вашої версії DSS.

---

## 8. Editions та ліцензування

Без цін — лише суть і структура. (Базово / Середній)

### 8.1 Free Edition

**Free Edition** (історично «Community» / «Free») — безкоштовна on-premise версія. Ліцензія **генерується автоматично**, коли ви вперше заповнюєте форму під час старту продукту, і **не потребує оновлення**. Призначена, щоб спробувати платформу й вчитися. Має **обмеження по можливостях і масштабованості**, яких може не вистачити великим командам (наприклад, відсутні продакшен-ноди й корпоративний governance).

### 8.2 14-денний Free Trial (керований)

Окремо існує **14-денний free trial** на хостингу Dataiku, де доступні майже всі можливості — підготовка даних, візуалізація, AutoML, advanced analytics, MLOps, колаборація — **за винятком Govern та advanced LLM Mesh**.

### 8.3 Комерційні edition-и

Платні edition-и (для self-managed — з **license file** від Dataiku, який кладеться в `DATA_DIR/config/license.json` або вводиться через **Administration → Enter license**) відкривають корпоративні можливості: **Automation node** та **API node** для продакшену, **Deployer**, **Govern node**, розширений **LLM Mesh**, ширшу безпеку та масштабування. Ліцензування Dataiku **role-based**: ролі-будівники (**Designer**) коштують відчутно дорожче за ролі-споживачі (viewer/reader). **Dataiku Cloud** має власні плани (підписка на SaaS).

> Точні назви поточних комерційних рівнів і їх розмежування (які саме фічі в якому рівні) змінюються між роками — **звіряйте з офіційною сторінкою plans-and-features та з вашим license-файлом**. Конкретні ціни в цій базі знань свідомо не наводяться.

### 8.4 Академічна ліцензія

Dataiku надає **безкоштовні академічні ліцензії** студентам/викладачам/дослідникам, включно з доступом до Enterprise-функціональності й навчальних матеріалів.

---

## 9. Огляд UI

(Базово)

### 9.1 Homepage

Після входу користувач потрапляє на **homepage**, яка показує найрелевантніше саме для нього (з урахуванням ролі): data scientist бачить швидкий доступ до поточних проєктів і нещодавніх об'єктів; data consumer — спільні **Workspaces** без прямого доступу до проєктів. Тіло homepage складається з секцій із **projects, folders, dashboards** і **promoted wikis**; об'єкти можна дивитися мозаїкою плиток або списком, фільтрувати пошуком і сортувати.

### 9.2 Навігація: waffle / application menu

У верхньому правому куті — **waffle menu** (меню-«вафля»), що дає доступ до:

- **Projects**, **Workspaces**, **Applications**;
- **Data Catalog** та глобального **items search** (пошук об'єктів усього інстансу);
- спільних активів: **Feature Store**, **shared code**, **wikis**.

Усередині проєкту вгорі є навігаційна панель проєкту з переходами до **Flow**, датасетів, рецептів, **Lab/Visual ML**, dashboards, сценаріїв, налаштувань тощо (детально про навігацію проєкту — `02-projects-flow-core-concepts.md`).

### 9.3 Права доступу (високий рівень)

Доступ у Dataiku багаторівневий:

- **Users & Groups** — користувачі належать до груп; права здебільшого видаються групам.
- **Global permissions** — що користувач/група може робити на рівні інстансу (створювати проєкти, керувати connections, адмініструвати тощо).
- **Project-level permissions** — права на конкретний проєкт (read/write/admin, доступ до dashboards тощо).
- **Connections permissions** — хто може використовувати які підключення до даних.
- **Workspaces** — спосіб дати споживачам доступ до окремих активів без доступу до самого проєкту.

Глибоко про безпеку, ролі, SSO, governance і колаборацію — `13-governance-security-collaboration.md`. Програмний доступ (Python API) до всього цього — `11-programmatic-api-python-client.md`.

---

## 10. Best practices розгортання

(Середній / Просунуто)

1. **Розділяйте середовища Design / Automation / (опційно) API.** Не виконуйте продакшен-навантаження на Design node. Класичний поділ — мінімум **Design** + **Automation**; для real-time додайте **API node(s)**.
2. **Деплойте лише через bundles.** Жодного «правлю продакшен напряму». Версійований bundle через **Project Deployer** = відтворюваність і можливість rollback.
3. **Виносьте Deployer в окрему ноду**, щойно у вас більше одного Design/Automation node. Attached-deployer прийнятний лише для маленьких установок.
4. **Використовуйте stages (dev → test → prod) в infrastructures.** Просувайте той самий bundle середовищами, а не збирайте новий під кожне.
5. **Увімкніть Govern node для критичних деплоїв.** Sign-off гарантує, що нічого не потрапить у прод без рев'ю та аудиту.
6. **Стандартизуйте code environments.** Фіксовані версії пакетів = відтворюваність між Design і Automation. Перебудовуйте **base images після кожного апгрейду DSS**.
7. **Плануйте ємність DATADIR і пам'ять.** ≥100 GB на DATADIR на одному mount; адекватний heap для backend; контроль «довгоживучих» Jupyter kernels, щоб не текла пам'ять.
8. **Для масштабних обчислень — elastic AI computation на Kubernetes.** Виносьте важкі рецепти/тренування в контейнери; гасіть кластери за розкладом для економії.
9. **Узгодьте варіант розгортання з потребами:** Free/самостійний — для навчання й невеликих кейсів; **Cloud Stacks + Fleet Manager** — коли треба керований lifecycle у власній хмарі; **Dataiku Cloud** — коли інфраструктуру хочеться повністю віддати Dataiku.
10. **Резервне копіювання й моніторинг:** бекапте DATADIR/конфіг; використовуйте **EventServer**/аудит для централізованого нагляду (див. `09-...`, `13-...`).

---

## 11. Типові помилки

- **Сплутати Design і Automation як «одну машину для всього».** Розробка й продакшен на одній ноді → нестабільність, відсутність контрольованих релізів.
- **Деплоїти зміни вручну, оминаючи Deployer/bundles.** Втрата версіонування й можливості rollback; розбіжність між середовищами.
- **Плутати batch-scoring і real-time scoring.** Прогноз цілого датасету — це Flow (Design/Automation). Real-time по HTTP одного запису — це **API node**. Не той інструмент → погана продуктивність/архітектура. Див. `10-...`.
- **Думати, що JEK = FEK.** JEK виконує **jobs (Build)**; FEK обслуговує **backend** (семпли, важкі фонові задачі). Це різні ролі.
- **Очікувати Govern або advanced LLM Mesh у Free Edition / trial.** Їх там немає — це функції вищих edition-ів.
- **Забути перебудувати base images після апгрейду DSS.** Контейнеризоване виконання почне падати.
- **Не задати default Kubernetes cluster.** Активності, що мали йти на Kubernetes, **впадуть** без глобального default-кластера.
- **Поставити DSS на Windows-сервер.** Серверна частина — лише Linux.
- **Недооцінити DATADIR.** Замалий/розкиданий по mount-ах data directory → проблеми з даними, логами й продуктивністю.

---

## 12. Перехресні посилання

- `02-projects-flow-core-concepts.md` — проєкти, Flow, базова модель роботи.
- `03-datasets-connections-partitioning.md` — датасети, підключення, партиціонування.
- `04-visual-recipes.md` — візуальні рецепти.
- `05-data-preparation-prepare-recipe.md` — підготовка даних, Prepare recipe.
- `06-coding-recipes-code-envs-notebooks.md` — кодові рецепти, code envs, notebooks.
- `07-machine-learning-visual-ml.md` — Visual ML, тренування моделей.
- `08-llm-generative-ai-mesh.md` — generative AI та LLM Mesh.
- `09-scenarios-automation-metrics-checks.md` — сценарії, автоматизація, metrics/checks.
- `10-mlops-deployment-api-node.md` — MLOps, deployment, API node (поглиблює розділи 3.3–3.4 і 5).
- `11-programmatic-api-python-client.md` — програмний доступ, Python client.
- `12-dashboards-charts-webapps-reporting.md` — dashboards, charts, webapps.
- `13-governance-security-collaboration.md` — governance (поглиблює 3.5), безпека, права доступу (поглиблює 9.3), колаборація.
- `14-plugins-applications-extensibility.md` — плагіни, applications, розширюваність.
- `15-best-practices.md` — наскрізні best practices (поглиблює розділ 10).
- `16-glossary.md` — повний словник термінів.

---

### Джерела (офіційні)

- DSS Concepts — https://doc.dataiku.com/dss/latest/concepts/index.html
- API Node & API Deployer — https://doc.dataiku.com/dss/latest/apinode/index.html
- Production deployments and bundles — https://doc.dataiku.com/dss/latest/deployment/index.html
- Understanding and tracking DSS processes — https://doc.dataiku.com/dss/latest/operations/processes.html
- How Dataiku Sandboxes Java (FEK/backend) — https://blog.dataiku.com/how-dataiku-sandboxes-java-to-avoid-heaps-of-garbage
- Installation index — https://doc.dataiku.com/dss/latest/installation/index.html
- Custom Linux install — https://doc.dataiku.com/dss/latest/installation/custom/initial-install.html
- Cloud Stacks / Fleet Manager — https://doc.dataiku.com/dss/latest/installation/cloudstacks-aws/concepts.html
- Containers / Elastic AI computation — https://doc.dataiku.com/dss/latest/containers/index.html
- AI Governance (Govern node) — https://doc.dataiku.com/dss/latest/governance/index.html
- DSS license — https://doc.dataiku.com/dss/latest/operations/license.html
- Homepage / UI — https://doc.dataiku.com/dss/latest/concepts/homepage/index.html
- Plans and features — https://www.dataiku.com/product/plans-and-features/

> Орієнтовано на актуальні версії DSS (13.x/14.x, 2025–2026). UI-шляхи й деякі деталі можуть відрізнятися між версіями.
