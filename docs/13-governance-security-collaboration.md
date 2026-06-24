# 13. Governance, Security та Collaboration

> Документ про те, як у **Dataiku** керують **доступом, ідентичністю, ізоляцією та аудитом**, як організовано **enterprise governance** через **Dataiku Govern** (окрема Govern node), як забезпечується **Responsible AI** і **data governance**, і як команди **співпрацюють** усередині інстансу (wiki, discussions, workspaces, sharing). Це «адмінсько-керівничий» зріз платформи: тут перетинаються ролі **admin**, **governance manager** і **звичайного користувача**. Версії DSS **13.x** і **14.x** (актуально на 2025–2026). Технічні/UI/код терміни залишені в **оригіналі (англійською)**.

---

## Зміст

1. [Вступ: чотири різні речі, які легко переплутати](#1-вступ-чотири-різні-речі)
2. [Концепції: модель безпеки Dataiku зверху вниз](#2-концепції-модель-безпеки)
3. [Users & groups: authentication і provisioning](#3-users--groups)
4. [LDAP / Active Directory](#4-ldap--active-directory)
5. [SSO: SAML та OAuth2/OIDC. SCIM](#5-sso-saml-oidc-scim)
6. [User profiles (ліцензійні тіри)](#6-user-profiles)
7. [Permissions model: global, project, connection](#7-permissions-model)
8. [Exposed/Shared objects та workspaces](#8-shared-objects-та-workspaces)
9. [User Isolation Framework (UIF)](#9-user-isolation-framework-uif)
10. [Audit: audit trail, EventServer, CRU](#10-audit)
11. [Dataiku Govern: node, blueprints, governed items](#11-dataiku-govern)
12. [Sign-offs, workflows та deployment approvals](#12-sign-offs-workflows-deployment-approvals)
13. [Model Risk Management (MRM)](#13-model-risk-management)
14. [Responsible AI: fairness, model cards, explainability, guardrails](#14-responsible-ai)
15. [Data governance: catalog, lineage, tags, PII, data quality](#15-data-governance)
16. [Collaboration: wiki, discussions, to-do, version control](#16-collaboration)
17. [Security hardening (високорівнево)](#17-security-hardening)
18. [Приклади: типові сценарії налаштування](#18-приклади)
19. [Best practices](#19-best-practices)
20. [Типові помилки](#20-типові-помилки)
21. [Перехресні посилання](#21-перехресні-посилання)

---

## 1. Вступ: чотири різні речі

Найчастіша концептуальна помилка новачка — змішати в одну купу поняття, які Dataiku тримає **строго окремо**. Запам'ятайте цю рамку, і 80% теми стане зрозумілою (Базово):

| Поняття | Питання, на яке відповідає | Де налаштовується |
|---|---|---|
| **Authentication** | «Хто ти?» (перевірка особи) | Login sources: Local, LDAP, SSO, PAM, Custom |
| **Provisioning** | «Звідки беруться записи user/group у DSS?» | User Suppliers: SSO, LDAP, Azure AD, Custom |
| **Authorization** | «Що тобі дозволено робити?» | Global permissions + Project permissions + Connection permissions |
| **Isolation** | «Під якою OS/compute-ідентичністю реально виконується код?» | **User Isolation Framework (UIF)** |

І ще одна важлива теза одразу: **user profile — це ліцензійний тір, а НЕ механізм безпеки.** Те, що користувачу дозволено, визначають **groups + permissions**, ніколи не profile.

Цей документ читається разом із:
- `01-overview-architecture.md` — типи нод (Design / Automation / API / Deployer / **Govern**), варіанти розгортання;
- `07-machine-learning-visual-ml.md` — деталі fairness, explainability, subpopulation analysis, model document generator (тут — governance-зріз);
- `08-llm-generative-ai-mesh.md` — LLM Mesh та guardrails (тут — лише як елемент Responsible AI);
- `09-scenarios-automation-metrics-checks.md` — metrics, checks, Data Quality (тут — як інструмент data governance);
- `10-mlops-deployment-api-node.md` — Project/API Deployer (тут — як Govern гейтить деплой через sign-off);
- `11-programmatic-api-python-client.md` — `GovernClient`, керування правами через API.

---

## 2. Концепції: модель безпеки

### 2.1 Ноди й де що живе (Базово)

Governance/security «розмазані» по нодах. Корисна мапа:

- **Design node** — тут живуть: connections і їхні permissions, code envs, Data Catalog, Data Lineage, Tags, Wiki, Data Quality, pseudonymization, LLM Mesh guardrails, тренування моделей (fairness/explainability).
- **Deployer node** — гейтить деплой dev→prod (читає sign-off-статус із Govern).
- **Automation / API node** — продакшен-виконання; теж шлють audit-події.
- **Govern node** — **окрема** нода: artifacts, blueprints, sign-offs, business initiatives, model/bundle registry, MRM.

### 2.2 Принцип кумулятивності прав (Базово)

У Dataiku **немає «заборонних» (deny) permissions**. Права лише **додаються**: підсумкові права користувача = об'єднання прав усіх його груп. Не можна «відняти» право в конкретного користувача всередині групи — треба перебудувати членство в групах. Це робить дизайн груп критично важливим.

### 2.3 Все крутиться навколо груп (Базово)

Майже всі дозволи в DSS призначаються **групам, не окремим користувачам**. Project permissions, connection permissions, global permissions — усе на рівні group. Тому базова дисципліна: продумана матриця груп (наприклад `analysts`, `data-scientists`, `prod-admins`, `dashboard-viewers`), а не ручна роздача прав по одному.

---

## 3. Users & groups

### 3.1 Де це в UI (версійна примітка)

У 14.x security-UI розведено на дві зони (старі доки можуть писати інакше — є обидва стилі):
- **Administration > Settings > Security & Audit > «User login & provisioning»** — конфіг авторизаційних джерел (підрозділи **LDAP**, **SSO**, **Azure AD**).
- **Administration > Security > Users / Groups** — самі записи користувачів і груп, мапінг group→profile.

### 3.2 Local users (Базово)

Створюються в **Administration > Security > Users** через **New User**: admin задає login, display name, password, **user profile**, членство в groups. Пароль зберігається локально (хешований, salted-PBKDF2, незворотно).

> **Завжди тримайте хоча б один Local admin-акаунт.** Це ваш «break-glass» доступ, який працює навіть якщо SSO/LDAP «ляже». Універсальний fallback — форма логіну за URL **`/login/`**.

### 3.3 Authentication sources та provisioning (Середній)

**Authenticators** (хто перевіряє пароль): Local, SSO (SAML/OIDC), LDAP, PAM, Custom.

**User Suppliers** (звідки тягнуться записи user/group): SSO, LDAP, **Azure AD** (важливо: **не може автентифікувати** — лише supplier, обов'язково в парі з SSO), PAM (тільки username), Custom (plugin).

Провіжинінг буває:
- **Login-time provisioning / resync** — користувач/групи створюються чи оновлюються під час логіну;
- **On-demand** — кнопки **SYNC ALL** / **SYNC** у Administration > Security > Users (або через public API).

**Порядок джерел за замовчуванням:** `Local > LDAP > Azure AD > SSO > PAM > Custom`.

> **Критичне правило «прилипання» (source stickiness):** щойно DSS-користувача прив'язано до певного джерела, DSS синхронізуватиме його **тільки через це джерело**. Помилково прив'язаний user — це головний біль для подальшого чищення.

---

## 4. LDAP / Active Directory

### 4.1 Увімкнення (Середній)

**Administration > Settings > User login & provisioning > LDAP > «Enable»** (валідація — кнопкою **Test**). DSS використовує **simple bind**.

Ключові поля (точні назви):
- **«LDAP server URL»** — `ldap[s]://HOST[:PORT]/BASE` (часта помилка — забути `BASE`).
- **«Bind DN»**, **«Bind password»** — сервісний акаунт для пошуку.
- **«Use TLS»** — StartTLS (для `ldap://`).
- **«User filter by username»** — з плейсхолдером `{USERNAME}`.
- **«Display name attribute»**, **«Email attribute»**.
- **«Enable group support»**, **«Group filter by username»** (`{USERNAME}`/`{USERDN}`), **«Group name attribute»**, **«Groups restriction»**.

### 4.2 Active Directory — типові значення (Середній)

| Поле | Значення для AD |
|---|---|
| User filter | `(&(objectClass=user)(sAMAccountName={USERNAME}))` |
| Display name | `displayName` |
| Email | `mail` |
| Group filter | `(&(objectClass=group)(member={USERDN}))` |
| Group name | `cn` |
| Username attr | `sAMAccountName` |
| Membership attr | `member` |

**Nested AD groups** (вкладені групи) вимагають спеціального OID-фільтра (Просунуто):
```
(&(objectClass=group)(member:1.2.840.113556.1.4.1941:={USERDN}))
```

### 4.3 Як працює мапінг group→DSS (Середній)

- LDAP-користувач потрапляє в DSS-групу, якщо є членом відповідної LDAP-групи.
- Членство **оновлюється на кожному логіні**.
- Кожній LDAP-групі мапиться **user profile**; при членстві в кількох групах **виграє найпривілейованіший profile**.

**Gotchas:** `ldap://` без TLS шле креденшали відкритим текстом; `ldaps://` потребує запису в Java **truststore**, якщо сертифікат не від відомого CA; у LDAP-користувачів немає редагованого пароля в DSS.

---

## 5. SSO (SAML, OIDC), SCIM

### 5.1 Загальне (Просунуто)

**Settings > Security & Audit > User login & provisioning > SSO > «Enable»**, далі вибір **«SAML»** або **«OIDC»**.

### 5.2 SAML

Ключові поля: **«IdP Metadata XML»**, **«SP entity ID»**, **«SP ACS URL»** (= `BASE_URL/dip/api/saml-callback`), вибір login-атрибута, **«Login remapping rules»** (Java regex), **«Sign requests»** (PKCS#12 keystore).

> **Після будь-якої зміни SAML-конфігу треба перезапустити DSS.** ACS URL має точно збігатися з тим, що зареєстровано в IdP (тільки HTTPS).

### 5.3 OAuth2 / OIDC

Ключові поля: **«Well-known URL»** (`/.well-known/openid-configuration`), **«Issuer»**, **«Authorization endpoint»**, **«Token endpoint»**, **«JWKs URI»**, **«Token endpoint auth method»**, **«Client ID»**, **«Client secret»**, **«Scope»** (мінімум `openid`, плюс `email`/`profile`), **«Identifier claim key»**, **«Login remapping rules»**.

Redirect URI для реєстрації в IdP: `BASE_URL/login/openid-redirect-uri/`.

**Gotchas:** scope обов'язково містить `openid`; Azure не створює client secret автоматично; невідповідність redirect-URI — найчастіша причина збою; **SSO-користувачів не можна синхронізувати on-demand** (тільки під час логіну).

### 5.4 SCIM — важливе уточнення (Просунуто)

> **Self-managed DSS 13.x/14.x НЕ має нативного SCIM 2.0 server.** Okta/Entra не можуть «пушити» users/groups на SCIM-endpoint.

Документований еквівалент — **pull-based User Supplier**: SSO supplier, LDAP supplier, **Azure AD User Supplier** (найближчий аналог — тягне users/groups з Entra ID), Custom supplier. На marketplace є плагін **«Azure Active Directory Sync»**. Керований **Dataiku Cloud** має власний вбудований auto-provisioning (це не те саме, що self-hosted SCIM). Жорсткі вимоги до SCIM — через Dataiku support.

---

## 6. User profiles

**User profile** — це **ліцензійне** поняття (один на користувача), **не контроль безпеки** (Базово).

Поточні profiles у DSS 14 (точні назви): **Full Designer**, **Data Designer**, **Advanced Analytics Designer**, **Governance Manager**, **AI Consumer**, **AI Access User**, **Technical Account**.

Легасі-назви (старі ліцензії): Designer, Data Scientist, Data Analyst, Reader, Explorer, Platform Admin.

Призначається вручну в Administration > Security > Users або автоматично через group→profile (виграє найпривілейованіший).

**Gotchas:** profile — не межа прав; набір назв залежить від ліцензії; profile, призначений через LDAP/SSO-мапінг, перезапишеться на наступному resync (правте мапінг, не користувача); вичерпання «місць» (seats) блокує логін.

---

## 7. Permissions model

### 7.1 Global group permissions (Середній)

Налаштовуються в **Administration > Security > Groups > [group]**. Точні назви:

- **«Administrator»** — автоматично вмикає **все** (обережно: тихо обирає всі прапорці).
- **«Create projects»**, **«Create projects using macros»**, **«Create projects using templates»**, **«Write in root project folder»**.
- **«Create workspaces»**, **«Publish on workspaces»**.
- **«Create Data Collections»**, **«Publish to Data Collections»**.
- **«Write isolated code»** (потребує UIF — код виконується з impersonated-правами).
- **«Write unisolated code»** (виконується з UNIX-привілеями `dssuser` — фактично **обхід** контролю, давати вкрай ощадливо).
- **«Create active Web content»**.
- **«Manage all code envs»**, **«Create code envs»** (у 14.x єдину «Manage code envs» розбили на дві).
- **«Manage all clusters»**, **«Create clusters»**.
- **«Develop plugins»**, **«Edit lib folders»**, **«Create personal connections»**, **«View indexed Hive connections»**, **«Manage user-defined meanings»**, **«Create published API services»**, **«Create published projects»**.

> Окремого global-права **«Manage connections» НЕ існує**: керування connections = прапорець Administrator + per-connection security + «Create personal connections».

### 7.2 Project-level permissions (Базово→Середній)

Налаштовуються в **Project > Security > Permissions** (на рівні груп; нова група починає без доступу). Точні назви + залежності:

| Permission | Що дає / що тягне за собою |
|---|---|
| **Admin** | Будь-яка дія в проєкті, зміна permissions і owner, створення bundles. Імплікує все. |
| **Edit permissions** | Бачити й редагувати permissions (не можна видати право, якого сам не маєш). Імплікує Read dashboards. |
| **Read project content** | Бачити Flow, читати datasets і recipes. Імплікує Read dashboards. |
| **Write project content** | Читати/писати конфіг і дані, створювати datasets/recipes. Імплікує Read+Write dashboards **і** Run scenarios. |
| **Export datasets** | Кнопка Download для вмісту dataset. |
| **Run scenarios** | Запускати scenarios у проєкті. |
| **Read dashboards** | Тільки читати дашборди (бачать лише об'єкти з Authorized objects). |
| **Write dashboards** | Створювати власні дашборди. |
| **Manage authorized objects** | Керувати, які об'єкти доступні dashboard-only users. |
| **Manage shared objects** | Керувати, які об'єкти доступні іншим проєктам (DSS≤13: «Manage exposed elements»). |
| **Publish on workspaces** | Шарити дашборди/datasets у workspaces. |
| **Publish to Data Collections** | Публікувати datasets у Data Collections. |
| **Execute app** | Запускати Dataiku applications / app-as-recipe. |

**Gotchas:** «Run scenarios» **сама по собі майже марна** (потрібен ще scenario «Run as» + Read project content); «Write project content» тихо приносить ще й scenario-run і dashboard-write — будьте свідомі обсягу; «Edit permissions» не дозволяє ескалацію (не видасте те, чого не маєте).

### 7.3 Connection permissions (Середній→Просунуто)

Налаштовуються в **Administration > Connections > [connection] > Security**:

- **«Details readable by»** — які групи можуть читати **незашифровані деталі connection, включно з credentials** (доступно лише з коду). Це чутливе право.
- Сторона **«usable by»** — connection «freely usable by» обраними групами: створювати/переглядати/будувати datasets і слати код. Фільтрує список connections при створенні dataset; **не** обмежує читання вже визначених datasets.
- Режим credentials: **«Global»** (один сервісний акаунт) vs **«Per-user»** (кожен вводить свої; потребує UIF).

> **Ключове застереження:** ці захисти **повноцінні лише з увімкненим UIF.** Без UIF усе виконується під одним Unix-користувачем, тому «Details readable by» стає радше косметикою.

**Gotchas:** доступ дається тільки **групі**, ніколи окремому користувачу; «Details readable by» торгує безпекою на продуктивність Spark на HDFS/S3 (без нього Spark іде повільним шляхом).

---

## 8. Shared objects та workspaces

### 8.1 Три різні «sharing», які плутають (Середній)

Dataiku має три окремі поверхні шарингу зі схожими словами:

1. **Shared objects** (project → project, для білдерів) — `Project > Security > Shared objects`. Шарити можна datasets, managed folders, saved models, eval stores, agents/agent tools (**не** recipes/wiki/SQL notebooks). Потрібне право **«Manage shared objects»** на **source**-проєкті. Доступ **read-only** у проєкті-приймачі. Два режими: **«Managed sharing»** (явні цільові проєкти, governance-friendly) і **«Quick sharing»** (будь-який reader тягне об'єкт — широко). DSS≤13 називав це «Exposed objects/elements».
2. **Authorized objects** (project → dashboard-only users) — «Manage authorized objects». Які об'єкти бачать користувачі лише з Read dashboards.
3. **Workspaces** (project → бізнес-споживачі) — окремий канал.

### 8.2 Workspaces (Середній)

**Workspace** — канал споживання для бізнес-користувачів (без Flow, без recipes). Можна шарити: dashboards, datasets, webapps, wiki articles, Dataiku Applications, external links, Stories. Ролі: **Admin / Contributor / Member** (видимість пласка незалежно від ролі).

> **Головна пастка — дворівневий дозвіл:** щоб шарити у workspace, потрібні **обидва** права «Publish on workspaces» — і **global**, і **project-level**. Зовнішні запрошення потребують налаштованого SMTP.

---

## 9. User Isolation Framework (UIF)

Загальна складність — **Просунуто** (потрібен root і, як правило, свіжий інстанс).

### 9.1 Навіщо UIF (Середній — концепція)

UIF (раніше **Multi-User-Security**) — «набір механізмів для ізоляції коду, який контролює користувач, щоб гарантувати **traceability** і **неможливість** ворожому користувачу атакувати `dssuser`».

**Без UIF** усе виконується під єдиним host-акаунтом **`dssuser`**, що дає два ризики:
1. немає Hadoop-traceability «хто що зробив»;
2. користувач із правом виконувати код може **запустити довільний код як `dssuser`** і змінити конфіг DSS — тобто обійти авторизацію DSS.

У secure-режимі «DSS impersonates the end-user» і виконує весь user-controlled код під ідентичністю, відмінною від `dssuser`.

### 9.2 Як ідентичність тече до compute (Просунуто)

- **Local code (Python/R/Shell):** механізм **`sudo`** («Local code isolation» — обов'язковий базовий шар).
- **Hadoop/Spark on YARN:** Hadoop **proxy user** + Kerberos **delegation tokens**; підтримується тільки **`yarn-client`**. Hive через HiveServer2 + **Ranger** (HiveServer2 impersonation має бути **вимкнено**).
- **Kubernetes:** звичайні workloads ізольовані контейнером (impersonation не потрібен); **Spark on K8s** — sudo для `spark-submit` + per-user K8s credentials, або «managed Spark on Kubernetes» з тимчасовими service accounts.
- **Identity mapping rules** (Administration > Settings > Security & Audit): one-to-one, single/fixed, або rule-based Java regex з `$1`/`$2`. LDAP рекомендований, щоб DSS/UNIX/Hadoop-ідентичності збігалися.

### 9.3 Per-user vs global credentials (Середній→Просунуто)

- **Global** — один сервісний акаунт (shared key/keypair, user/password або AWS AssumeRole).
- **Per-user** — кожен вводить особисті креденшали (user/password, cloud key або OAuth2).

Admin вмикає в `Administration > Connections > [connection] > Connections credentials > Per-user`. Користувач задає свої в `Profile > Credentials`. Креденшали зберігаються зашифрованими. Одразу після перемикання на per-user «Test» не спрацює (ще немає креденшалів).

### 9.4 Secure code execution (Просунуто)

Керується двома global-правами:
- **«Write isolated code»** — код виконується з impersonated-правами (доступне лише з UIF).
- **«Write unisolated code»** — виконується з UNIX-привілеями `dssuser`, може міняти конфіг/security DSS — **фактично обхід**, давати вкрай ощадливо.

Споріднене: **«Write unsafe code»** гейтить custom Python UDFs / partition functions у Prepare.

### 9.5 Передумови й увімкнення (Просунуто)

Передумови: підтримка **ACL** на ФС data dir; **root**-доступ; валідний UNIX-акаунт/shell для кожного impersonated-користувача; DSS-групи з відповідними UNIX-групами; для Hadoop — Kerberos + keytab для `dssuser` + Ranger.

> **Критична пастка:** **не накочуйте UIF на вже заповнений інстанс.** Офіційно «highly not recommended» — «start with an empty DSS instance». Вимкнення/downgrade UIF **не підтримується**. Налаштовуйте UIF на свіжому інстансі під час інсталяції.

Увімкнення:
```bash
./bin/dss stop
# як root:
./bin/dssadmin install-impersonation dssuser
```
Конфіг: `/etc/dataiku-security/INSTALL_ID/security-config.ini`, секція `[users] allowed_user_groups` = UNIX-групи, яким дозволено impersonation. Членство в `allowed_user_groups` потрібне для local Python/R, visual ML і Spark-об'єктів; **не** потрібне для SQL-engine visual recipes, SQL recipes і простих Prepare recipes.

---

## 10. Audit

### 10.1 Audit trail: що логується (Базово→Просунуто)

Audit trail логує дії користувачів із **user id, timestamp, IP address, authentication method**. Кожна подія — JSON-повідомлення з обов'язковим **topic** (категорія, задається кодом DSS) і опціональним **routing key**.

Шість audit-**topics**:

| Topic | Опис |
|---|---|
| `generic` | Аплікативний аудит більшості подій (login/logout теж тут) |
| `generic-failure` | Більшість невдач |
| `authfail` | Усі **невдачі** authentication & authorization |
| `apinode-query` | Запити/відповіді API node |
| `compute-resource-usage` | Використання compute-ресурсів |
| `apicalls` | Сирий лог API-викликів (дебаг) |

Приклади `msgType`: `login`, `logout`, `dataset-read-data`, `flow-job-start/done`, `scenario-run`, `project-export-download`, `dataset-export`, (API) `prediction-query`, `sql-query`.

> **Критичне обмеження UI-в'ю** (Administration > Security > Audit trail): показує **лише останні 1000 подій** і скидається при рестарті DSS. **Ніколи** не використовуйте його як compliance-запис.

### 10.2 EventServer: централізація (Середній→Просунуто)

**DSS Event Server** — легкий HTTP-сервер поруч із Design/Automation node (не окремий тип ноди). Приймає audit-повідомлення з кількох нод і централізує їх.

**Audit dispatcher** на кожній ноді маршрутизує події (за topic + routing key) на **targets**:
1. **Log4J target** (за замовчуванням) — локальні файли; будь-який appender (напр. syslog) для експорту.
2. **EventServer target** — HTTP POST на віддалений Event Server (стандартний шлях крос-машинної централізації).
3. **Kafka target** — у Kafka-topic (**experimental, Tier 2**).

Налаштування: приймач — `Administration > Settings > Event Server`; ноди-відправники додають EventServer target із URL+auth key; API nodes конфігуруються авто через API Deployer.

**Сховище:** пише **тільки JSON-файли** у filesystem-like connection (Filesystem, S3, Azure Blob, GCS), партиціоновано як `/topic/routing-key/YYYY/mm/dd/HH`. **Dataset не створюється автоматично** — будуєте самі. Установка: `./bin/dssadmin install-event-server`. Не HA, не горизонтально масштабований.

### 10.3 Compute Resource Usage (CRU) (Просунуто)

DSS автоматично шле **Compute Resource Usage** на topic `compute-resource-usage` (lifecycle `-start` → `-update` → `-complete`). Типи: `LOCAL_PROCESS`, `SINGLE_K8S_JOB`, `SPARK_K8S_JOB`, `SQL_CONNECTION`, `SQL_QUERY`. Контексти: jobs, ML training, notebooks, web apps, API deployments.

**Вбудованого адмін-дашборду немає.** Періодичний K8s-репортинг — у `Administration > Settings > Misc` (потрібен рестарт). Для аналітики — KB-Solution **«Dataiku Resource Usage Monitoring»** (вимагає **DSS 14.2+**, **не** на Dataiku Cloud).

### 10.4 Login / auth audit та retention (Середній)

Окремого `login.log` немає: успіх `login`/`logout` → `generic`; невдачі → `authfail`. Захоплюється `authUser`, `msgType`, `timestamp`, `mdc.user`, `origAddress` (source IP). API node має окремі логери `dku.apinode.audit.auth` і `dku.apinode.audit.admin`.

**Retention за розміром, не за часом:** дефолт `${DIP_HOME}/run/audit/audit.log`, ротація на **100 MB**, **20** історичних файлів (~2.1 GB). Налаштовується appender `AUDITFILE` у `resources/logging/dku-log4j.properties` + рестарт.

> **Best practice:** для compliance відвантажуйте події off-box (Event Server → durable S3/Azure/GCS з власною lifecycle-політикою), а не покладайтесь на локальну ротацію.

На Dataiku Cloud audit авто-доступний через S3-connections `customer-audit-log` і `dku-audit-log`.

---

## 11. Dataiku Govern

### 11.1 Що це за нода (Базово→Середній)

**Govern node** — окрема нода, «централізоване місце для керування governance аналітики, моделей і агентів на enterprise-рівні», подібна до Automation/Deployer node. Ліцензується окремо як продукт **Dataiku Govern** зі **Standard** та **Advanced License** (Advanced відкриває **Action Designer**, **Blueprint Designer**, **Custom Pages Designer**, **GenAI Registry**, скриптові налаштування).

Govern зв'язується двома шляхами:
1. Design/Automation ноди **публікують** у Govern projects, Saved Models, versions, bundles;
2. **Deployer перевіряє governance-статус** перед деплоєм.

> Ключова теза з доків: «Без approval від sign-off-процесу в Govern node, packages не можуть рухатись з development у production environment.»

### 11.2 Governable items та реєстрація (Базово→Середній)

«Governing» додає об'єкту **governance-шар**, перетворюючи його на Govern project / model / model version / bundle. Шар містить fields, metrics, attachments, workflows, sign-offs. Вкладки Govern-айтема: **Overview** (description, Sponsor, scope, qualification rating, schedule, reviewers, attachments), **Workflow**, **Sign-offs**, **Attachments**, **Timeline**, **Source Objects**, **Deployments**.

Незаговернені айтеми авто-синкаються на сторінку **Governable items** («інбокс» придатних до governance об'єктів: Projects, Bundles, Saved Models, Saved Model versions, GenAI Items). Реєстрація: клік на **gavel icon** у рядку → модал **Govern** / **Hide**.

> **Критичне правило ієрархії:** «An item can be governed only if its parents are governed.» Model version вимагає, щоб модель і проєкт були заговернені раніше. **Project — верхній рівень.**

### 11.3 Registries (Середній)

- **Model Registry** — ієрархічний список усіх моделей з підключених нод **незалежно від governance-статусу** (status, Model metrics, deployment metrics, drift, governance state).
- **Bundle Registry** — аналогічно для bundles.
- **GenAI Registry** (Advanced) — fine-tuned LLMs, Agents, Augmented LLMs + versions.

Верхня навігація Govern: Home → Governable Items → Model Registry → GenAI Registry → Bundle Registry → Business Initiatives → Governed Projects. Waffle-меню: Timeline, Action/Blueprint/Custom Pages Designer, Governance Settings, Roles and Permissions, Administration.

### 11.4 Blueprints (Базово — використання дефолтів / Просунуто — дизайн)

**Blueprint** — контейнер із одним чи кількома **blueprint versions** для одного типу governed-об'єкта. Blueprint version — це власне **шаблон**: визначає, які **fields** збираються/показуються і який вигляд має **workflow**.

Три категорії: **Govern blueprints** (стандартні), **Dataiku blueprints** (locked, синкаються з Design/Deployer), **Custom blueprints**.

Lifecycle версії: **Draft → Active → Archived** (+ Delete). **Застосувати можна тільки Active.**

Дефолтні blueprints (по одному на стандартний об'єкт): **Business initiatives**, **Projects**, **Bundles**, **Models**, **Model versions** (DSS 14 додає **GenAI items**).

Будуються з: **Fields** (Number, Boolean, Text, Category, Date, **Reference**, File, Time-series, JSON; кожне з «Is Required»/«Is a list»), **Views**, **Workflow** steps, **Actions** (Python-кнопки), **Hooks** (CREATE/UPDATE/DELETE). Інструмент — **Blueprint Designer** (Advanced).

### 11.5 Govern data model (Середній — концепція)

- **Artifact** = екземпляр Blueprint version (центральний факт; кожен Govern-айтем — artifact).
- **Reference fields** — механізм, що **зв'язує artifacts** між собою (напр. Business Initiative посилається на Governed Projects).

Ієрархія сутностей:
```
Business Initiative          (групує проєкти спільною бізнес-метою)
   └── Governed Project       (верхній governable-айтем)
         ├── Governed Saved Model
         │      └── Governed Model Version   (потребує модель + проєкт заговернені)
         └── Governed Bundle                  (потребує проєкт заговернений)
```

**Business initiatives** — чисто Govern-об'єкт (не синкається), групує governed-проєкти. Один-до-багатьох: ініціатива тримає багато проєктів, кожен проєкт — рівно в одній ініціативі.

---

## 12. Sign-offs, workflows, deployment approvals

### 12.1 Sign-off механіка (Середній)

Workflow — послідовність steps (напр. «In Progress», «Under Review», qualification-стадія). У стандартних шаблонах **governed model versions і governed bundles мають обов'язковий sign-off** у кроці «Review».

Sign-off складається з **двох частин**:

1. **Feedback** — кілька слотів; **опціональний**; призначається Roles, Users, Groups або Global API keys. Статуси відгуку: **Approved**, **Minor issue**, **Major issue**. Можна **delegate**.
2. **Final Approval** — рівно **один** слот. Тільки **Final Approver** робить **Approve / Reject / Abandon**. **Саме цей статус перевіряє Deployer.**

Точні стани sign-off: `NOT_STARTED`, `WAITING_FOR_FEEDBACK`, `WAITING_FOR_APPROVAL`, `APPROVED`, `REJECTED`, `ABANDONED`.

**Reviewers** визначаються один раз на рівні **governed-project** («Sign-off reviewers and approvers») і успадковуються model versions/bundles.

> **Часта помилка:** думати, що Feedback гейтить прогрес. **Гейтить лише Final Approval.** І reviewers конфігуруються на рівні проєкту, не per-model.

### 12.2 Deployment approvals: declarative gate (Середній→Просунуто)

Deployer питає в Govern статус **Final Approval** і застосовує **Governance Policy**, налаштовану **per deployment infrastructure**. Три опції (точні UI-лейбли):

1. **«Prevent the deployment of unapproved packages.»** → блок із **Error** (API: `PREVENT`).
2. **«Warn and ask for confirmation before deploying unapproved packages.»** → попередження, можна продовжити (`WARN`).
3. **«Always deploy without checking.»** → деплоїть завжди; **це default** (`NO_CHECK`).

Параметр: **`govern_check_policy`** = `PREVENT` / `WARN` / `NO_CHECK`.

Навігація: **Deployer → Infrastructures → (вибрати) → Settings → Governance Policy**. Різні stages (dev/test/prod) можуть мати різні політики.

### 12.3 Custom pre-deployment hook (Просунуто)

У налаштуваннях Project Deployer infrastructure можна додати Python-hook, що гейтить на **конкретному** sign-off-кроці (за `step_id`), а не лише на глобальному Final Approval:

```python
import dataikuapi
gc = dataikuapi.GovernClient(host, apikey)
signoff_def = (gc.get_artifact(gov_bundle_id)
                 .get_signoff(pre_prod_signoff_step_id)
                 .get_definition().get_raw())
if signoff_def.get('status') != 'APPROVED':
    return HookResult.error('Pre-prod sign-off not approved')
return HookResult.success("Pre-prod sign-off is approved")
```

> **Часта помилка:** залишити infra на дефолтному `NO_CHECK` — тоді governance-гейт фактично нічого не робить.

Деталі деплою dev→prod, bundles, infrastructures — у `10-mlops-deployment-api-node.md`.

---

## 13. Model Risk Management

**Dataiku Solution for Model Risk Management (MRM)** — готове рішення (solutions catalog) для критичних банківських/фінансових моделей, що централізує MRM-фреймворк **у Dataiku Govern** (Просунуто):

- єдиний уніфікований workflow для data scientists, model-risk/validation teams та аудиторів;
- кастомізовані approval-workflows, що **збирають stakeholder sign-offs на кожній стадії перед деплоєм**;
- повна traceability, searchable auditable log;
- усі історичні версії моделей зберігаються для ex-post аудитів.

Явно орієнтований на **Federal Reserve SR 11-7** (independent validation, risk-tiering за критичністю). Будується на стандартних sign-off + Governance Policy — це не окремий «движок».

> **Застереження:** за замовчуванням **жодне Govern-поле не є обов'язковим**, тож примус збору доказів валідації потребує custom-шаблонів/hooks (Advanced License).

---

## 14. Responsible AI

> Глибокі деталі fairness/explainability — у `07-machine-learning-visual-ml.md`; LLM-guardrails — у `08-llm-generative-ai-mesh.md`. Тут — governance-зріз.

### 14.1 Дві різні «fairness»-фічі, які плутають (Середній)

| | Subpopulation analysis | Model Fairness Report |
|---|---|---|
| Тип | **Вбудована** | **Plugin** (треба встановити) |
| Що міряє | *Performance*-метрики по підгрупах (ROC AUC, Accuracy, Precision, Recall) | Формальні *fairness*-метрики |
| Навігація | model Result > Subpopulation analysis | model page > Views > Model Fairness Report |

Чотири метрики Fairness Report: **Demographic Parity** (рівний Positive Rate), **Equalized Odds** (рівні TPR і FPR), **Equality of Opportunity** (рівний TPR), **Predictive Rate Parity** (рівний PPV). Не можна оптимізувати кілька одночасно (impossibility theorem) — обирайте одну за «Fairness Tree».

### 14.2 Model Document Generator (model cards) (Базово→Просунуто)

Генерує **Word (.docx)** «model card» для тренованої моделі. Навігація: trained model > **Actions** > **Export model documentation** > вибрати template > Export > Download (потрібна активована graphical export library).

Дефолтні шаблони: **Regression, Binary classification, Multiclass classification, Clustering, Time series forecasting**. Захоплює 150+ плейсхолдерів: призначення/методологію, feature importance/effects, алгоритм + hyperparameters, train/test strategy, performance metrics, confusion matrices, ROC/curves тощо. Custom-шаблони: синтаксис `{{placeholder.name}}`, умовні `{{if}}…{{endif}}`.

> Не підтримує ensemble, Keras, computer vision, MLflow-imported та plugin-алгоритми.

### 14.3 What-if / Interactive scoring (Базово)

model Result > **Interactive scoring**: **Compute view** (крутиш features → live prediction + explanations) і **Comparator view** (порівняння side-by-side). Тільки Python/Keras backends; це якісний sensitivity-tool, не причинний аналіз.

### 14.4 Explainability (Середній)

- **Global feature importance:** **Shapley (SHAP)** для більшості Python-моделей; **Gini** для деревних.
- **Partial Dependence Plots (PDP)**.
- **Individual prediction explanations:** **Shapley** (внески сумуються до різниці прогнозу, важче) vs **ICE** (швидше, але в нелінійних не сумується). На масштабі — через Score recipe «Output explanations» (додає `_explanations` JSON-колонку).

### 14.5 LLM Mesh guardrails (Середній→Просунуто)

Safety-шар LLM Mesh: **PII Detection** (на Microsoft Presidio; дії Reject/Placeholder/Hash/Removal/Partial masking), **Prompt Injection Detection**, **Toxicity Detection** (OpenAI moderation або local HuggingFace), **Forbidden Terms**, **Topics Boundaries**, **Bias Detection**, **Response Format Checker**, **Custom**. Конфігуруються на рівні connection (instance-wide) та/або об'єкта (per recipe/agent) — стекаються. Детально — `08-llm-generative-ai-mesh.md`.

---

## 15. Data governance

### 15.1 Data Catalog (Базово→Середній)

«Центральне місце для шарингу й пошуку datasets по організації». Категорії: **Data Collections** (куровані групи; ролі Admin/Contributor/Reader), **Datasets & Indexed Tables** (всі DSS-datasets + **Indexed External Tables** — сирі таблиці connection, проіндексовані адміном), **Most Used Datasets** (v14), **Connections Explorer**, **AI Search** (v14, семантичний LLM-пошук). Споріднене: **Feature Store** (реєстр **Feature Groups**). Навігація: waffle > Data Catalog.

> Indexed External Tables з'являються лише після того, як admin проіндексує connection; AI Search тихо падає на keyword-пошук, якщо AI Services вимкнено.

### 15.2 Data Lineage / where-used (Середній→Просунуто)

**Column-level Data Lineage** трекає трансформації колонок між datasets/проєктами. **Тільки direct lineage**. **Upstream** = root-cause; **Downstream** = impact/where-used. Авто для visual recipes/prepare; code recipes + pivot/split/unfold падають на **name-based matching** (позначено іконкою-питанням). У **DSS 14 Data Lineage переїхав** у власну секцію **Operations & Monitoring** (старий Catalog-URL v13 404-иться).

> Не довіряйте сліпо lineage code-recipes (name-based heuristic). Режим «Manual lineage only» прибирає авто-lineage для рецепта.

### 15.3 Metadata, tags (Базово→Просунуто)

Вбудовано: label, description, checklists, tags, `customFields`. **Custom Fields визначаються як plugin-компонент** (`custom-fields/custom-fields.json`) — **окремого екрана Administration > Custom Fields НЕ існує** (часта плутанина).

**Tags — дві системи:**
1. **Project/object tags** — free-text + color, на льоту (право «Write project content»);
2. **Global tag categories** — instance-wide governed-значення `category:tag`, у `Administration > Settings > Global tag categories` (потрібен Administrator; підтримує merge + «Apply to»).

> Free tags плодять дублі (`prod`/`Prod`) — ліки це global categories. Імпорт проєкту без потрібної global-категорії деградує теги до project-level, поки admin її не відтворить.

### 15.4 Business glossary — уточнення (Базово)

> **У Dataiku НЕМАЄ фічі з назвою «Business Glossary».** Найближчі замінники: Data Catalog + Data Collections + Tags + Wiki (на Design node), або Business Initiatives + custom Blueprints (на Govern node).

### 15.5 Sensitive data / PII (Середній)

- **Column Pseudonymization** processor у Prepare («Pseudonymize text»): SHA-256/512/MD5 + Pepper + per-row Salt (псевдонімізація зворотна, анонімізація — ні).
- **LLM Mesh PII Detector** (Presidio) — див. 14.5.
- **Kiji Privacy Proxy** — окремий open-source шлюз маскування PII (не плутати з LLM Mesh PII Detection).

> **Важливе обмеження:** DSS **не має** нативного column-level masking, dynamic masking by user/group чи row-level security — це треба пушити в базу (напр. Snowflake masking policies).

### 15.6 Data Quality vs Metrics & Checks (Середній)

- **Metrics** — вимірювальний шар (probes рахують метрики, авто-історизуються). **Не deprecated.**
- **Checks** — легасі-порогова логіка (OK/ERROR/WARNING/EMPTY; ERROR валить білд). **Для datasets замінено на Data Quality rules.** Лишається для managed folders, saved models, model evaluation stores.
- **Data Quality rules** (з **DSS 12.6**, тільки datasets) — ~34 типи правил, статуси OK/Warning/Error/Empty/Not Computed, моніторинг на рівні dataset/project/instance. **Data Quality Templates** — багаторазові набори правил (**застереження: без access control — будь-який автентифікований користувач може їх керувати**).

Правило вибору: datasets → Data Quality rules; folders/models/eval stores → Checks; Metrics — основа обох. Детально — `09-scenarios-automation-metrics-checks.md`.

---

## 16. Collaboration

### 16.1 Wiki (Базово)

Кожен проєкт wiki-enabled. Статті в **Markdown** (+ HTML/CSS, LaTeX), ієрархічна таксономія, attachments (файли або DSS-об'єкти). Object-link макрос має **зворотний** порядок: `(label)[dataset:PROJECT_KEY.name]`; wiki-лінки `[[PROJECT_KEY.Article Name]]`. **Global wiki** — промоут через Settings > Wiki.

> Object-link макрос — дужки спершу (на відміну від звичайного Markdown); PROJECT_KEY чутливий до регістру.

### 16.2 Discussions (Базово)

Threaded-коментарі на об'єктах (projects, datasets, recipes, notebooks, models, dashboards, workspaces). Markdown + **@mentions**. Email потребує **трьох умов**: admin SMTP «Notification emails» channel + email у профілі користувача + увімкнено в My Settings. **Resolve** ховає тред.

### 16.3 To-do lists (Базово)

**Один спільний project-level чек-лист** на home-сторінці проєкту. **Без** per-user-призначення, due dates, пріоритетів, нотифікацій, крос-проєктного в'ю. Тонка фіча — «простий спільний чек-лист».

### 16.4 Version control та real-time co-edit (Середній)

> **У DSS НЕМАЄ Google-Docs-стилю co-editing і НЕМАЄ per-object edit-locks для recipes/datasets.** Модель — **warn + last-write-wins, на базі Git**.

Проєкти — справжні Git-репозиторії (Auto/Explicit commit; merge requests). Notebooks показують conflict-warning при одночасному редагуванні («behavior undefined»); notebook лишається в пам'яті, поки не вивантажений (File > Close and Halt; адмін бачить у Administration > Monitoring > Background tasks — це моніторинг, не lock). Datasets залокати не можна.

### 16.5 Code Studios collaboration (Середній)

Персональні контейнерні web-IDE (VS Code, JupyterLab, RStudio, Streamlit/Dash/Gradio/Voila), кожен — окремий Kubernetes pod. **Single-user by design**: «you can't collaborate in real-time on Code Studios». Інший користувач, що відкриває чужий, отримує «Not authorized: Not owner of the Code Studio». Колаборація — через shared project libraries (`project-lib-versioned`, Git-versioned) або зовнішній Git. Деталі — `06-coding-recipes-code-envs-notebooks.md`.

---

## 17. Security hardening

### 17.1 Passwords, secrets (Середній)

- **Дві схеми шифрування:** login-паролі — **salted-PBKDF2** (незворотно); connection-credentials — **AES-CTR + HMAC-SHA256** (відновлювані). Master key у data dir, опційно в **AWS Secrets Manager / Azure Key Vault / GCP Secret Manager** (HashiCorp Vault — **не** документована нативна інтеграція).
- **User secrets:** особисті зашифровані креденшали в `Profile > My Account > Other credentials`; з коду — `client.get_auth_info(with_secrets=True)`. Глобальних/спільних secrets немає.

> Креденшали в параметрах recipe/dataset **не шифруються** — використовуйте **presets**. «Details readable by» оголює незашифровані креденшали.

### 17.2 Network (Середній→Просунуто)

HTTPS двома шляхами: вбудований nginx-cert або (краще) **reverse proxy з TLS-термінацією**. **WebSocket-підтримка обов'язкова** (nginx 1.3.13+/Apache 2.4.5+ mod_proxy_wstunnel) — без неї DSS ламається. DSS **не можна** перемапити на non-root URL sub-path. Hardening — у `general-settings.json` (`secureCookies`, `sameSiteNoneCookies`, `forceSingleSessionPerUser`, session timeouts) та `install.ini` (CSP, X-Frame-Options, HSTS).

> Увімкнення secure/SameSite cookies без HTTPS ламає логін.

### 17.3 Code env safety (Просунуто)

Без UIF усе — під єдиним `dssuser` (нуль ізоляції). З UIF — impersonation (див. розділ 9). **Containerized execution** виштовхує recipes/ML/notebooks/webapps на Kubernetes, кожен workload ізольований від DSS-engine. Code envs — `Administration > Code Envs > Permissions` (дефолт «Usable by all» — зніміть для least privilege). Після апгрейду перебудовуйте container-images: `./bin/dssadmin build-container-exec-code-env-images --all`. Деталі — `06-coding-recipes-code-envs-notebooks.md`.

---

## 18. Приклади

### 18.1 Базова матриця груп для dev-інстансу (Базово)

| Group | Global perms | Project perms (типово) |
|---|---|---|
| `analysts` | Create projects | Write project content (у власних проєктах) |
| `data-scientists` | Create projects, Create code envs, Write isolated code | Write project content |
| `dashboard-viewers` | — | Read dashboards (на обраних проєктах) |
| `prod-admins` | Administrator | Admin |

### 18.2 Connection із per-user credentials до Snowflake (Просунуто)

1. Увімкнути UIF (на свіжому інстансі).
2. `Administration > Connections > snowflake-prod > Connections credentials > Per-user`.
3. Для OAuth2: «Auth Type» = «OAuth», заповнити client id/secret/authorization+token endpoints/scope (`SESSION:ROLE-ANY` + `offline_access` для refresh token).
4. Кожен користувач задає свої креденшали у `Profile > Credentials`.

### 18.3 Governance-гейт деплою в прод (Середній)

1. На Govern node заговернити Project → Saved Model → Model Version.
2. Призначити reviewers на рівні governed-project.
3. Запросити Review; Final Approver ставить **Final Approval = Approved**.
4. На Deployer: prod-infrastructure → Settings → Governance Policy = **«Prevent the deployment of unapproved packages.»**
5. Тепер деплой неапрувленого package блокується з Error.

---

## 19. Best practices

### Least privilege (Базово)
- Видавайте **мінімально необхідне**: не «Administrator» там, де достатньо «Create projects».
- «Write unisolated code» / «Write unsafe code» — лише вузькому колу довірених; це фактично обхід безпеки.
- Code envs — знімайте дефолт «Usable by all».
- Connection «Details readable by» — лише групам, яким реально треба читати креденшали з коду.

### Separation of duties dev/prod (Середній)
- **Фізично розділяйте** Design (dev) і Automation/API (prod) ноди; ніколи не пушіть «сирий» проєкт у прод (тільки bundles через Deployer — `10-...`).
- Різні групи-адміни для dev і prod; різні credentials (per-user або окремі service accounts).
- Prod-infrastructure → Governance Policy = **Prevent**, не дефолтний NO_CHECK.

### Governance workflow (Середній→Просунуто)
- Починайте з governance MLOps-потоку (моделі/bundles), потім розширюйте на проєкти.
- Reviewers — на рівні governed-project (успадковуються), не per-model.
- Розрізняйте Feedback (необов'язково, не гейтить) і Final Approval (гейтить).
- Для регульованих індустрій зробіть обов'язкові поля валідації через custom blueprints/hooks (інакше Govern нічого не примушує).

### Audit-моніторинг (Середній→Просунуто)
- **Не покладайтесь** на UI-в'ю (1000 подій, скидається при рестарті) для compliance.
- Розгорніть **Event Server** і відвантажуйте в durable S3/Azure/GCS з власною lifecycle-політикою.
- Локальна ротація — за розміром (100 MB × 20), не за часом; для довшого retention потрібен off-box.
- Моніторте `authfail` topic для виявлення brute-force / misconfig.

### Identity (Базово→Середній)
- Тримайте хоча б один Local admin (break-glass через `/login/`).
- Через LDAP/SSO синхронізуйте групи, мапте group→profile (виграє найпривілейованіший).
- Пам'ятайте про source stickiness — продумайте прив'язку джерела ДО масового provisioning.

---

## 20. Типові помилки

- **Плутати authentication, provisioning, authorization та isolation.** Це чотири різні шари; profile ≠ permission.
- **Думати, що user profile обмежує права.** Це ліцензійний тір; права — лише groups+permissions.
- **Чекати «deny»-permissions.** Їх немає: права кумулятивні, керуйте членством у групах.
- **Видавати права окремому користувачу.** Майже все призначається групам; connection-доступ — тільки групі.
- **«Run scenarios» без Read project content і scenario Run-as.** Право саме по собі майже марне.
- **Не усвідомлювати, що «Write project content» тягне scenario-run + dashboard-write.**
- **Покладатись на connection permissions без UIF.** Без UIF «Details readable by» — косметика; усе під одним `dssuser`.
- **Видавати «Write unisolated code» широко.** Це обхід DSS-авторизації (код як `dssuser`).
- **Накочувати UIF на заповнений інстанс.** Highly not recommended; downgrade не підтримується — ставте на свіжому.
- **Не перезапустити DSS після зміни SAML-конфігу.** Зміни не застосуються; ACS URL має точно збігатись.
- **Очікувати нативний SCIM на self-managed.** Його немає — User Suppliers (Azure AD = supplier, не authenticator).
- **Загубити Local admin при поломці SSO.** Завжди тримайте break-glass-акаунт і знайте `/login/`.
- **Використовувати UI audit-в'ю як compliance-запис.** Останні 1000 подій, скидається при рестарті — централізуйте через Event Server.
- **Думати, що Event Server створює dataset.** Він пише лише JSON-файли; dataset будуєте самі.
- **Залишити Governance Policy на NO_CHECK.** Тоді sign-off-гейт нічого не блокує.
- **Плутати Feedback із Final Approval.** Гейтить деплой тільки Final Approval.
- **Governити child раніше за parent.** Заблоковано: спершу проєкт, потім модель, потім версія.
- **Очікувати «Business Glossary».** Такої фічі немає — Catalog/Tags/Wiki або Business Initiatives.
- **Шукати екран «Administration > Custom Fields».** Custom Fields — це plugin-компонент.
- **Плутати Subpopulation analysis (вбудована, performance) із Model Fairness Report (plugin, fairness).**
- **Класти креденшали в параметри recipe/dataset.** Вони не шифруються — використовуйте presets.
- **Шарити у workspace лише з одним рівнем «Publish on workspaces».** Потрібні обидва — global і project-level.
- **Чекати real-time co-edit / object lock.** Модель — last-write-wins на Git; Code Studios single-user.
- **Сподіватись на native column/row-level masking у DSS.** Немає — робіть у базі (напр. Snowflake policies).
- **Увімкнути secure/SameSite cookies без HTTPS.** Зламає логін.

---

## 21. Перехресні посилання

- `01-overview-architecture.md` — типи нод (Design/Automation/API/Deployer/**Govern**), варіанти розгортання, Kubernetes/elastic AI.
- `02-projects-flow-core-concepts.md` — проєкти, project export vs bundle, Git-версіонування проєкту.
- `03-datasets-connections-partitioning.md` — connections (основа connection permissions та per-user credentials).
- `06-coding-recipes-code-envs-notebooks.md` — code envs та їхні permissions, isolated/unisolated code, Code Studios.
- `07-machine-learning-visual-ml.md` — повні деталі fairness, subpopulation analysis, explainability, what-if, model document generator.
- `08-llm-generative-ai-mesh.md` — LLM Mesh та guardrails (PII/toxicity/prompt-injection) як частина Responsible AI.
- `09-scenarios-automation-metrics-checks.md` — metrics, checks, Data Quality rules (інструмент data governance).
- `10-mlops-deployment-api-node.md` — Project/API Deployer, bundles, infrastructures (Govern гейтить деплой через sign-off).
- `11-programmatic-api-python-client.md` — `dataikuapi`, `GovernClient`, керування правами/користувачами з коду.
- `12-dashboards-charts-webapps-reporting.md` — dashboards, Authorized objects, workspaces для бізнес-споживачів.
- `16-glossary.md` — терміни: UIF, impersonation, sign-off, blueprint, artifact, governance policy, audit topic, EventServer, guardrail, fairness, lineage.

---

> Джерела: doc.dataiku.com/dss/latest (розділи `security/`, `user-isolation/`, `operations/audit-trail/`, `governance/`, `machine-learning/model-document-generator`, `generative-ai/guardrails/`, `data-catalog/`, `data-lineage/`, `collaboration/`, `workspaces/`), knowledge.dataiku.com (admin-configuring, govern, mlops-o16n), developer.dataiku.com (GovernClient, Govern hooks/deployments), dataiku.com (Govern product, MRM solution, Responsible AI). DSS 13.x/14.x, перевірено червень 2026.
