# 08. LLM Mesh та Generative AI

> Документ про **generative AI підсистему Dataiku** — від єдиного шлюзу до LLM (**LLM Mesh**) до повного циклу LLMOps: prompt engineering, RAG/Knowledge Banks, візуальні LLM-рецепти, агенти, Dataiku Answers, fine-tuning, оцінювання, guardrails та контроль витрат. Версії DSS **13.x** і **14.x** (актуально на 2025–2026). Технічні/UI/код терміни залишені в **оригіналі (англійською)**.

---

## Зміст

1. [Вступ: навіщо LLM Mesh](#1-вступ-навіщо-llm-mesh)
2. [Концепція LLM Mesh: єдиний governance-шлюз](#2-концепція-llm-mesh)
3. [LLM connections та підтримувані провайдери](#3-llm-connections-та-провайдери)
4. [Prompt Studios та Prompt recipe](#4-prompt-studios-та-prompt-recipe)
5. [LLM-візуальні рецепти: Classify / Summarize / переклад / генерація](#5-llm-візуальні-рецепти)
6. [Embeddings, Knowledge Banks та RAG](#6-embeddings-knowledge-banks-та-rag)
7. [Document extraction (обробка документів)](#7-document-extraction)
8. [Agents та agentic AI](#8-agents-та-agentic-ai)
9. [Dataiku Answers та чат-інтерфейси](#9-dataiku-answers)
10. [Fine-tuning LLM](#10-fine-tuning-llm)
11. [Оцінювання LLM та RAG (Evaluation)](#11-оцінювання-llm-та-rag)
12. [Guardrails, PII та moderation](#12-guardrails-pii-та-moderation)
13. [Контроль витрат та моніторинг використання](#13-контроль-витрат-та-моніторинг)
14. [LLM Mesh API та Starter Kit](#14-llm-mesh-api-та-starter-kit)
15. [Best practices](#15-best-practices)
16. [Типові помилки](#16-типові-помилки)
17. [Перехресні посилання](#17-перехресні-посилання)

> **Add-on.** Частина можливостей нижче входить лише в платний add-on **Advanced LLM Mesh** (custom quotas, custom guardrails, RAG guardrails, Topics Boundaries, `Fine-tune` recipe, `Evaluate LLM`/`Evaluate Agent` recipes). Це позначено в тексті як **(Advanced LLM Mesh)**.

---

## 1. Вступ: навіщо LLM Mesh

Коли організація починає вбудовувати LLM у продукти, швидко з'являється «зоопарк»: різні команди звертаються напряму до OpenAI, Azure, Anthropic, Bedrock, кожна зі своїми API-ключами, без єдиного нагляду за витратами, без контролю над тим, які дані йдуть у зовнішні моделі, без аудиту й без захисту від PII-витоків чи prompt injection. Це не масштабується і небезпечно.

**LLM Mesh** — це відповідь Dataiku на цей хаос: **спільний backbone (шлюз) для enterprise generative AI**. Замість того щоб кожен код чи рецепт «знав» конкретного провайдера й тримав власний ключ, усі звертаються до LLM через **LLM connections**, налаштовані адміністратором. LLM Mesh додає зверху governance: permissioning, аудит, моніторинг витрат і блокування за квотами, кешування, guardrails (PII, toxicity, prompt injection тощо). (Базово)

Над цим шлюзом Dataiku будує повний інструментарій LLMOps:
- **Prompt Studio** і **Prompt recipe** — prompt engineering від експерименту до batch-продакшену;
- візуальні **LLM-рецепти** (`Classify text`, `Summarize text`) — типові NLP-задачі без навчання моделі;
- **Knowledge Banks** + **Embed** рецепти — RAG (Retrieval-Augmented Generation) на ваших документах;
- **AI Agents** (visual і code) + **Agent Tools** — агентні застосунки;
- **Dataiku Answers** — готовий enterprise чат-застосунок з RAG;
- **Fine-tune** та **Evaluate** рецепти — адаптація й оцінювання моделей.

Ідея, що проходить наскрізно: **усе generative AI у Dataiku говорить до моделей через LLM Mesh**, тому governance, безпека й контроль витрат застосовуються однаково — і до Prompt recipe, і до агента, і до Dataiku Answers.

---

## 2. Концепція LLM Mesh

**LLM Mesh** — це абстрактний рівень («mesh») між **споживачами LLM усередині Dataiku** (рецепти, студії, агенти, чат-застосунки, код) і **провайдерами LLM** (зовнішні API та локальні моделі). (Базово)

### 2.1 Що дає LLM Mesh

- **Unified secure gateway.** Єдина точка доступу до багатьох провайдерів. Код і рецепти посилаються на **LLM ID**, а не на конкретний ключ/endpoint — модель можна замінити, не переписуючи pipeline. (Базово)
- **Access permissioning.** Доступ до кожної LLM connection керується правами; не всі користувачі бачать усі моделі.
- **Audit & query tracing.** Усі запити до LLM логуються та трасуються — хто, коли, яким проєктом, скільки токенів.
- **Cost monitoring & blocking.** Облік витрат і квоти, що можуть **блокувати** запити при перевищенні бюджету (розділ [13](#13-контроль-витрат-та-моніторинг)).
- **Guardrails.** PII detection, toxicity, prompt injection, topics boundaries, custom — на рівні connection чи usage-time (розділ [12](#12-guardrails-pii-та-moderation)).
- **Response caching.** Однакові запити можуть повертатися з кешу — економія коштів і латентності.
- **Rate limiting / throttling.** Автоматичне обмеження частоти запитів per-model/per-provider (розділ [13](#13-контроль-витрат-та-моніторинг)).

### 2.2 Концепція «virtual LLM»

Важлива для розуміння всієї підсистеми ідея: **багато об'єктів Dataiku самі стають «LLM» у LLM Mesh**. (Середній рівень)

- Звичайна модель провайдера — це LLM.
- **Retrieval-Augmented LLM** (звичайна модель + Knowledge Bank, розділ [6](#6-embeddings-knowledge-banks-та-rag)) — теж з'являється як вибирана модель.
- **AI Agent** стає **«virtual LLM»** з ID на кшталт `agent:...` і доступний усюди, де працює LLM Mesh.

Завдяки цьому той самий **Prompt Studio / Prompt recipe / LLM Mesh API** однаково викликає і «голу» модель, і RAG-augmented модель, і цілого агента — інтерфейс єдиний.

---

## 3. LLM connections та провайдери

**LLM connection** налаштовує **адміністратор** в `Administration > Connections > New connection`. Тут вказують тип провайдера, credentials (API key / service account / посилання на існуючу cloud-connection), а також **права** — хто з користувачів/груп може використовувати цю connection. (Базово)

### 3.1 Підтримувані провайдери та можливості

Кожен провайдер підтримує різний набір **capabilities**: text/chat completion, embeddings, image generation, reranking. Орієнтовна матриця (DSS 14; набір зростає від версії до версії):

| Provider (connection type) | Capabilities | Що потрібно |
|---|---|---|
| **OpenAI** | text/chat, image generation (DALL·E 3), embeddings | OpenAI API key |
| **Azure OpenAI** | text/chat, image generation, embeddings | Azure account, deployed service + model deployments, API key |
| **Microsoft Foundry** (Azure AI Foundry) | text/chat, image generation, embeddings, reranking | Azure account, deployed model, endpoint, API key |
| **Azure LLM** | chat, embeddings | Azure ML / Azure AI Studio serverless endpoint |
| **Anthropic** | text/chat (Claude family) | Anthropic API key |
| **AWS Bedrock** | text/chat, embeddings, image generation, reranking | AWS account з доступом до Bedrock; S3 connection |
| **AWS SageMaker** | text completion, summarization | існуюча SageMaker connection |
| **Google Vertex AI** | chat (Gemini), image generation | service account key / OAuth, роль Vertex AI Service Agent |
| **Cohere** | text models | Cohere API key |
| **Mistral AI** | text, embeddings | Mistral AI API key |
| **Databricks Mosaic AI** | text, embeddings | існуюча Databricks Model Deployment connection |
| **Snowflake Cortex** | chat models | існуюча Snowflake connection |
| **NVIDIA NIM** | chat, multimodal, embeddings, tool calling | NIM API key або self-hosted сервіс |
| **Stability AI** | image generation | Stability AI API key |
| **HuggingFace (local)** | text/chat, embeddings (локально на GPU) | self-hosted модель, GPU; код-середовище на рівні connection |

> **Anthropic Claude** доступний двома шляхами: напряму через **Anthropic** connection або через **AWS Bedrock** (родина Claude). Вибір залежить від того, де вашій організації зручніше з погляду білінгу/комплаєнсу.

> **Локальні / self-hosted моделі.** HuggingFace-моделі, що працюють **на GPU всередині вашої інфраструктури**, — ключовий варіант, коли дані **не можна** відправляти у зовнішні API. На такі connection можна спрямувати чутливі дані (напр. PII) через guardrails (розділ [12](#12-guardrails-pii-та-moderation)).

### 3.2 Capabilities та обмеження

- Не всі провайдери дають усі capabilities — напр. **Stability AI** лише image generation, **Cohere** — переважно text/reranking.
- Деякі enterprise-функції LLM Mesh **не застосовуються** до окремих провайдерів. Зокрема **cost quotas та rate limiting не діють** для **SageMaker, Databricks Mosaic AI, Snowflake Cortex** і **локальних HuggingFace** моделей (деталі — розділ [13](#13-контроль-витрат-та-моніторинг)).
- Для **agentic** сценаріїв і **tool calling** потрібна модель з підтримкою function/tool calling (Anthropic Claude, OpenAI/Azure OpenAI, Mistral, Vertex Gemini, Databricks Mosaic AI, Microsoft Foundry, Snowflake Cortex, NVIDIA NIM тощо) — розділ [8](#8-agents-та-agentic-ai).

---

## 4. Prompt Studios та Prompt recipe

### 4.1 Prompt Studio (Базово)

**Prompt Studio** — це **project-level об'єкт**, повноцінне середовище для **prompt engineering**: проєктування, тестування, ітерування й порівняння промптів і моделей **перед** тим, як виносити їх у Flow. Один Prompt Studio може містити кілька промптів; студію також можна використовувати як чат-інтерфейс до моделі.

**Режими роботи:**
- **Managed mode** — описуєте задачу простою мовою, Dataiku генерує фінальний промпт.
- **Advanced mode** — пишете промпт вручну, з placeholders і повним контролем параметрів.
- **Chat mode** — багатоходова розмова (multi-turn).
- **Prompt without inputs** — разові експерименти (не деплояться у recipe).

**Складники промпту:**
- **Prompt text** — основні інструкції.
- **Inputs / Placeholders** — змінні, прив'язані до колонок датасету синтаксисом **`{{column_name}}`**; під час виконання підставляються значення рядка.
- **Examples** — пари вхід/вихід для zero-shot / single-shot / few-shot prompting.
- **Settings** — параметри генерації: підтверджені **Temperature** і **Max tokens** (наявність Top-p як окремого контролу не підтверджена).
- **Structured outputs / JSON** — промпт можна налаштувати повертати JSON для подальшої автоматизації.

**Тестування та порівняння.** Placeholders мапляться на колонки обраного датасету, і ви ганяєте промпт на семпл-рядках реальних даних. Студія дає порівнювати **кілька промптів** і **кілька LLM (API або локальних) пліч-о-пліч**, бачачи виходи одночасно. Для формального оцінювання якості є окремий інструментарій (розділ [11](#11-оцінювання-llm-та-rag)).

### 4.2 Prompt recipe (Базово)

Коли промпт готовий, його **зберігають як Prompt recipe** — і він стає вузлом у Flow. **Prompt recipe застосовує промпт до кожного рядка вхідного датасету й пише результат у вихідний датасет** (batch-обробка). Це міст від експерименту до продакшену.

- **Створення:** або з Prompt Studio (save prompt-as-recipe), або з панелі **Actions** датасету в розділі **GenAI recipes**.
- **Шаблони:** та сама підстановка `{{column_name}}` для кожного рядка.
- **Output:** нова вихідна колонка (текст або JSON) **плюс error-колонка**, що ловить помилки окремих рядків (один «поганий» рядок не валить весь job).
- **Ціль (LLM target):** через LLM Mesh можна спрямувати на три типи — **Standard LLM**, **Retrieval-Augmented LLM** (RAG) або **AI Agent** (агентна обробка кожного рядка).
- **Cost / quota:** керується централізовано через LLM Mesh (кеш, квоти, аудит), а не на рівні рецепта.

> **Переклад і вільна генерація тексту** в Dataiku роблять саме через **Prompt recipe** — окремих візуальних рецептів `Translate` чи `Generate` **немає** (розділ [5](#5-llm-візуальні-рецепти)).

---

## 5. LLM-візуальні рецепти

Крім універсального Prompt recipe, Dataiku має **спеціалізовані no-code LLM-рецепти** для типових NLP-задач — без навчання власної моделі, одразу на LLM Mesh. (Базово)

| Recipe (точна назва) | Що робить | Ключові вихідні колонки |
|---|---|---|
| **Classify text** | Класифікує текст у категорії через LLM, без навчання/розмітки. Класи task-specific (напр. sentiment) або задані користувачем; zero/single/few-shot; багатомовність. | `predicted_class` / `llm_prediction`, опціонально `explanation` |
| **Summarize text** | Підсумовує текст; розумно чанкує довгий вхід і збирає зв'язний підсумок. Опційно — мова виходу й обмеження довжини. | `summarized_text`, `llm_error_message` |
| **Prompt** | Вільний промпт на рядок (розділ [4](#4-prompt-studios-та-prompt-recipe)). Використовується і для **перекладу**, і для **генерації**. | конфігуровний output + error-колонка |

**Класифікація і підсумовування** — найшвидший спосіб отримати цінність від LLM: задаєте колонку з текстом, обираєте модель з LLM Mesh, (для класифікації) задаєте класи — і отримуєте результат по всьому датасету.

> Для **перекладу** немає окремого рецепта: створюєте Prompt recipe з інструкцією на кшталт «Translate the following text to Ukrainian: `{{text}}`». Те саме — для будь-якої кастомної генерації.

---

## 6. Embeddings, Knowledge Banks та RAG

**RAG (Retrieval-Augmented Generation)** дає «голій» LLM знання про **вашу** конкретну предметну область: при запиті з вашого корпусу автоматично вибираються найрелевантніші фрагменти й **додаються в запит** до моделі, щоб вона синтезувала відповідь на основі цього контексту — з посиланнями на джерела. (Базово)

### 6.1 Embed рецепти

Текст векторизують **два візуальні Embed-рецепти**; обидва на виході дають **Knowledge Bank**:

- **Embed dataset recipe** (since v12.3+) — вхід: **структурований датасет** з текстом у колонці. Обираєте text column для корпусу; опційно metadata columns (показуються в джерелах) і колонку **Document-Level Security**; для smart-update методів потрібен **Document unique ID**.
- **Embed documents recipe** (since v13.4+, multimodal у 14.x) — вхід: **managed folder з документами** (PDF, PPTX/PPT, DOCX/DOC, ODT/ODP, TXT, MD, PNG, JPG, HTML) + опційний metadata-датасет. Рецепт **витягує контент** (текст та/або зображення), **чанкує** і векторизує. Єдиний шлях до **multimodal Knowledge Bank** (текст + зображення).

**Chunking.** Довгий текст автоматично ріжеться на чанки під ліміт embedding-моделі; structured-extraction рушій ділить контент за заголовками на змістовні секції перед чанкуванням.

**Embedding model.** Обидва рецепти потребують **Embedding LLM** — embedding-модель, виставлену через LLM connection. Конкретні назви моделей не зашиті в продукт — залежать від обраного провайдера. Multimodal RAG додатково потребує text-completion модель із підтримкою зображень (VLM) на етапі генерації.

### 6.2 Knowledge Banks (Базово)

**Knowledge Bank** — це сховище векторизованого корпусу, оптимізоване під high-dimensional semantic search. Створюється автоматично при запуску Embed-рецепта. Має **три ключові функції**:

1. **Search** — семантичний пошук по корпусу (перевірити якість retrieval/чанкінгу).
2. **Retrieval-Augmented LLM** — комбінація Knowledge Bank + LLM, що дає відповіді з джерелами; доступна в **Prompt Studio**, **Prompt recipe** і **LLM Mesh API**.
3. **Knowledge Bank Search Tool** — програмний запит для агентів і автоматизацій (розділ [8](#8-agents-та-agentic-ai)).

**Managed vs unmanaged.** Managed Knowledge Bank — vector store, керований DSS; **unmanaged** — вказівник на вже наявний зовнішній vector store. У 14.x додано unmanaged-підтримку **pgvector** і **Vertex AI Search**.

### 6.3 Vector stores

Knowledge Bank бекендиться vector store. (Середній рівень)

- **Без окремого setup (за замовчуванням):** **Chroma (ChromaDB)** — дефолт, без налаштування; **Milvus (local)**; **Qdrant**; **FAISS**.
- **Сторонні (потрібна попередньо налаштована connection):** **Azure AI Search**, **Elasticsearch**, **OpenSearch** (вкл. AWS, серверлес), **Milvus (remote)**, **Pinecone**, **pgvector** (PostgreSQL), **Vertex AI Vector Search**.

Особливості: метрику схожості (Cosine / Euclidean / Dot Product) можна обирати для Chroma, Milvus Local, Qdrant, FAISS; **smart-update методи (Smart sync, Upsert) не підтримуються** для Pinecone та AWS OpenSearch serverless. Методи оновлення Knowledge Bank: **Smart sync / Upsert / Overwrite / Append**.

### 6.4 Механіка RAG та налаштування retrieval (Середній рівень)

Усередині Knowledge Bank ви обираєте підлеглу LLM і налаштовуєте **search settings** — і отримуєте **Retrieval-Augmented LLM**, що з'являється у списку моделей у Prompt Studio / Prompt recipe / LLM Mesh API. Кнопка в Knowledge Bank може одразу відкрити Prompt Studio з уже вибраною RAG-моделлю.

**Search settings:**
- **Similarity-based (semantic) search** — за метрикою vector store.
- **Hybrid search** — поєднує семантику й keyword-пошук; підтримується лише **Azure AI Search, Elasticsearch, Milvus**.
- **Diversity enhancement (MMR)** — Maximal Marginal Relevance (баланс релевантності й різноманіття).
- **top-k** — кількість документів для retrieval (точна UI-мітка/дефолт у доках чітко не зафіксовані — *під уточнення*).
- **Threshold filtering** — поріг score, що відсікає слабкі збіги.

**Filtering:** static (фіксована підмножина), dynamic (фільтр змінюється per query), agent-inferred (агент сам формує фільтр з промпту).

**Reranking:** native (Azure AI Search → Semantic Ranker; Elasticsearch / Milvus → RRF) або model-based (HuggingFace, напр. BGE-Reranker; Bedrock / Microsoft Foundry — Cohere Rerank).

**Генерація:** підтримуються Custom Generation Prompt і налаштування подачі джерел (Sources).

### 6.5 RAG guardrails (Advanced LLM Mesh)

Окремо від загальних guardrails, **RAG guardrails** перевіряють, що **обраний контекст і відповідь мають сенс** щодо запиту й корпусу; діють лише на Retrieval-Augmented LLM. Дві LLM-as-a-judge метрики:
- **Relevancy** — наскільки відповідь адресує запит;
- **Faithfulness** — наскільки відповідь узгоджена з витягнутим контекстом.

Адмін задає **мінімальні пороги**; якщо відповідь нижче порога — **reject** запит або **замінити** на пояснювальне/fallback-повідомлення. Потрібні допоміжна LLM (для оцінки) та embedding-модель; оцінка йде **наживо** на RAG-виходах.

> **14.x novelties:** **GraphRAG** і **Graph Search agent tool**, **Automated RAG Optimization**, multimodal Embed documents, unmanaged Knowledge Banks на pgvector / Vertex AI Search.

---

## 7. Document extraction

Перш ніж документ потрапить в embedding, його контент треба **витягти**. Два шляхи: всередині **Embed documents** рецепта (екстракція відбувається внутрішньо) або окремий рецепт **Extract document content** (вхід — managed folder, вихід — **датасет** витягнутого контенту). (Середній рівень)

**Рушії текст-екстракції:**
- **Raw text extraction** — на основі фізичного layout; дуже швидко; підтримує OCR для сканованих PDF.
- **Structured text extraction** — багатокроковий; ділить за заголовками на секції, автоматично чанкує; **потребує інтернету для PDF** (тягне layout-моделі з HuggingFace).

**OCR-рушії:** **Tesseract** (передвстановлений у Dataiku cloud; для кастомних деплоїв — ручна установка tessdata), **EasyOCR** (без конфігурації, але дуже повільний на CPU — рекомендований GPU), **AUTO** (Tesseract → fallback на EasyOCR).

**VLM strategy** — сторінки документа конвертуються в зображення й батчами йдуть у **Vision LLM** для підсумків/екстракції (захоплює графіку й таблиці). На етапі запиту відповідні сторінки-зображення подаються прямо в контекст VLM.

**Extract fields** (режим Extract document content) — VLM зі **structured output**: ви описуєте схему (ім'я поля, опис, тип: string/integer/number/date/boolean/array), і модель витягує структуровані поля з документів.

---

## 8. Agents та agentic AI

**AI Agents** (вперше — DSS **13.4**; у DSS 14 винесені в окрему секцію документації) — це LLM, що **самостійно обирає й викликає інструменти (tools)** для виконання задачі. (Середній → досвідчений)

### 8.1 Типи агентів

- **Simple Visual Agent** (no-code) — `Flow > +Other > Generative AI > Simple Visual Agent`. Налаштовуєте, (а) які **Tools** доступні агенту й (б) опційні інструкції. Потребує LLM з підтримкою **tool/function calling**. Модель сама обирає інструменти за їхнім описом + JSON-schema (native tool calling) і повертає **sources**.
- **Structured Visual Agent** (DSS 14, advanced) — послідовність **блоків** для прозорої, контрольованої, детермінованої логіки. Блоки: Agentic Loop, LLM Request, Manual/Mandatory Tool Call, **Delegate to Another Agent**, Routing, For Each, Parallel, Reflection, Set State / Scratchpad, Context Compression, Generate Text Output / Artifact, Python Code, **Long-Term Memory**, Semantic Feedback Management. **Human-in-the-loop approval** додано у 14.4.0.
- **Code Agent** — `Flow > Generative AI > Code Agent`; Python 3.10+. Реалізуєте клас-нащадок **`BaseLLM`** з `dataiku.llm.python`, метод **`process(self, query, settings, trace)`**. Глибока інтеграція з LangChain/LangGraph (Managed Tools → LangChain Structured Tools; агент/LLM → `as_langchain_chat_model()` → `DKUChatModel`).
- **External Agents** (DSS 14) — обгортають агентів Snowflake Cortex, Databricks, AWS Bedrock, Google Vertex AI як Dataiku-агентів.

### 8.2 Agent Tools (Managed Tools)

Інструменти створюються як **reusable project-об'єкти** через меню **Generative AI > Agent Tools**. Підтверджені типи (групи Query Data / Perform Actions / Orchestration / Integration / Custom):

- **Knowledge Bank Search** (RAG-retrieval; **повертає документи, але не генерує відповідь** — це робить агент), **Dataset Lookup**, **Dataset Append**, **SQL Question Answering**, **Semantic Model Query**, **Model Predict**;
- **API Endpoint Tool**, **Query an LLM / Agent** (multi-agent delegation), **Image Generation**, **Send Message** (email/Slack/Teams), **Generate Artifact**, **Calculator**, **Optimization Tool**, **Google Search**, **Google Calendar**;
- інтеграції: **Jira, Salesforce, ServiceNow, Snowflake Cortex Analyst/Search, Databricks Genie/Vector Search**;
- **Local MCP / Remote MCP** (підтримка **Model Context Protocol** з DSS **14.2.0**), **Custom / Inline Code Tool**, plugin-tools.

Кожен tool має ім'я, опис для LLM і JSON-Schema вхід. Tools викликаються з Python API, конвертуються в **LangChain Structured Tools** або обираються у Visual Agent.

### 8.3 Запуск, спостережуваність, multi-agent

- Кожен агент стає **«virtual LLM» у LLM Mesh** (ID `agent:...`) і доступний усюди — Prompt Studio, Prompt recipe, чат-UI, LLM Mesh API.
- **Tracing/observability:** агент продукує повний **Trace** (вкладені spans/events); **Trace Explorer** webapp показує Explorer / Timeline / Tree з входами, виходами, тривалістю, usage. Поряд: **Agent Evaluation** (14.3.0), **Agent Review** (14.4.0), **Agent Logging** (14.5.0).
- **Multi-agent:** через tool **Query an LLM / Agent**, блок **Delegate to Another Agent**, code-агентів, що кличуть інших агентів, та протокол **A2A (Agent-to-Agent)**.

### 8.4 Agent Connect → Agent Hub

- **Agent Connect** (DSS 13.4.0+) — уніфікований чат-інтерфейс і **роутер** над кількома GenAI-сервісами (Dataiku Answers, LLM Mesh Agent, Augmented LLM). **Deprecated**, замінений на Agent Hub.
- **Agent Hub** (DSS 14.2+) — наступник у вигляді **visual webapp plugin** («enterprise agent control plane»): централізована бібліотека курованих агентів, multi-agent оркестрація в одній розмові, «My Agents» для кінцевих користувачів, прозорість відповідей (sources/activities/downloads), governance над доступними LLM/tools/агентами.

---

## 9. Dataiku Answers

**Dataiku Answers** — це **готовий, масштабований enterprise web-застосунок** для LLM-чату й RAG, що постачається як **plugin** з plugin store (не частина core DSS). Повністю кастомізований, responsive (mobile), працює поверх LLM Mesh. (Базово → середній)

**Версії:** потребує DSS ≥ 12.4.1; Answers ≥ 2.6.1 — DSS ≥ 14.2.

**Ключові можливості:**
- **Chat UI** з multi-turn розмовою та історією (history dataset на SQL, індексація, опція permanent-delete).
- **Три методи retrieval:** **No Retrieval** (лише LLM), **Knowledge Bank Retrieval** (Pinecone, ChromaDB, Qdrant, Azure AI Search, ElasticSearch) і **Dataset Retrieval** (генерує SQL до структурованих даних — PostgreSQL, Snowflake, MS SQL Server, BigQuery, Databricks, Oracle 12.2+). Режим «Let Answers decide» — smart-роутинг.
- **Source citations** (з ID, цитатами, метаданими) і metadata-фільтрація.
- **Завантаження файлів** (images, PDF, DOCX, JSON, Python, HTML, JS, Markdown, PPTX, CSV) з авто-підсумовуванням чи retrieval; ліміт розміру (дефолт 15 MB).
- **Feedback collection:** thumbs up/down на кожне повідомлення + окремий загальний feedback-датасет (для рев'ю та оцінювання).
- **Image generation** (опційно; DALL·E 3, Stable Diffusion, Imagen) з тижневими лімітами на користувача.
- **Streaming responses**, sample prompts, кешування, окремі LLM для генерації заголовків і «Decisions Generation» (SQL).

**Адмін/деплой:** plugin усередині Dataiku-проєкту; потрібні **History dataset (SQL)** і **User profile dataset (SQL)**; брендинг через title/subheading/example questions/custom CSS/logo. Логування розмов і feedback — для моніторингу й оцінювання. Є **Answers API** (POST `/api/ask`, JSON або SSE-стрімінг).

> Dataiku Answers — найшвидший спосіб віддати кінцевим користувачам **production-ready RAG-чат** без власної фронтенд-розробки.

---

## 10. Fine-tuning LLM

**Fine-tune recipe** (візуальний; **Advanced LLM Mesh**) дотреновує модель і реєструє результат для використання в LLM Mesh. (Досвідчений)

- **Підтримувані моделі:** OpenAI, Azure OpenAI, AWS Bedrock і **локальні HuggingFace** моделі.
- **Формат датасету:** дві обов'язкові колонки — **prompt** (вхід) і **completion** (бажаний вихід), без пропусків; опційна колонка **system message** (per-row контекст, може мати пропуски). Можна додати **validation dataset** для трекінгу loss.
- **Гіперпараметри:** дефолт «Auto»; перемикається в «Explicit» для ручного тюнінгу.
- **Локальні HuggingFace:** використовується код-середовище з рівня connection; контейнер задається в налаштуваннях рецепта; **GPU настійно рекомендований**.
- **Деплой:** fine-tuned моделі Azure OpenAI / AWS Bedrock треба задеплоїти перед інференсом (DSS може керувати деплоями автоматично).
- **Code-based fine-tuning:** Python-рецепт зі сніпетами або повністю кастомним кодом; єдина жорстка вимога — вихід у форматі **safetensors**.
- **Важливе обмеження:** fine-tune recipe **не застосовує** connection-level guardrails.

> **LoRA/PEFT/QLoRA** — згадуються в **Developer Guide** (Python-приклади з HF `transformers` + `peft`/`LoraConfig`) як рекомендований memory-efficient підхід для локального fine-tuning, але що саме **візуальний** рецепт використовує під капотом — **офіційно не задокументовано** (*під уточнення*).
>
> **Версії:** DSS 13.1.0 — «Managed LLM fine-tuning» введено.

---

## 11. Оцінювання LLM та RAG

**Evaluate LLM** і **Evaluate Agent** рецепти (**Advanced LLM Mesh**) оцінюють single-LLM, RAG-системи чи multi-step агентні пайплайни. Приймають датасет з колонками **input, output, context, ground-truth**. (Досвідчений)

- **Input presets:** **Prompt Recipe** (оцінити виходи Prompt recipe) і **Dataiku Answers** (аналіз conversation-history датасетів); типи задач — «Question answering» чи «Other LLM Evaluation Task».
- **Non-LLM метрики:** **BLEU**, **ROUGE** (exact word-matching для перекладу/підсумовування), **BERTScore** (внутрішні HF embedding-моделі, детерміновано).
- **LLM-as-a-judge метрики** (потребують completion-LLM; рахуються по рядках, тоді усереднюються): **Answer Correctness, Answer Relevancy, Answer Similarity, Context Precision, Context Recall, Faithfulness**. Технічно — вторинна LLM як проксі людської оцінки.
- **Setup:** код-середовище з пресетом **«Agent and LLM Evaluation»** (Python ≥ 3.9); embedding-LLM і completion-LLM з LLM Mesh; дефолти — `Administration > Settings > LLM Mesh > Evaluation recipes`.
- **Custom metrics:** Python через функцію `evaluate`, що повертає floats / масиви.
- **Outputs:** результати в **evaluation store** (метрики, summary-таблиці, row-by-row); **Model comparisons** — side-by-side між версіями/промптами/LLM.

> **Версії:** DSS 13.2.0 — «LLM evaluation recipe»; 13.4.0 — вбудовані faithfulness/relevancy + multimodal context.

---

## 12. Guardrails, PII та moderation

Запити/відповіді проходять через **послідовний guardrails-pipeline**; кожен крок може перевіряти query, response або обидва. (Середній → досвідчений)

**Рівні конфігурації:**
- **Connection-level** — на LLM connection; діють на всі запити через неї (напр. дозволити PII-обробку лише на локальних моделях).
- **Usage-time / recipe-level** — у візуальних LLM-рецептах (напр. Prompt recipe); діють лише на той рецепт.
- **Agent-level** — при побудові агентів.

**Вбудовані guardrails:**
1. **PII Detection** (на базі **Microsoft Presidio**) — **34 entity types** (credit cards, emails, phones, SSN, driver's licenses, medical licenses для US/UK/Spain/Italy/Singapore/Australia). П'ять дій: **reject**, **replace with placeholder** (напр. `<PERSON>`), **pseudonymize via hash**, **strip/remove**, **partial mask**. Потребує provisioning Python-env + інтернет для завантаження моделей (**air-gapped не підтримується**).
2. **Prompt Injection Detection** — блокує спроби перевизначити поведінку моделі.
3. **Toxicity Detection** — токсична мова в query/response.
4. **Topics Boundaries** (**Advanced LLM Mesh**) — LLM-as-a-judge для дозволених/заборонених тем; дії: block with error, audit only, decline politely. («Forbidden terms» згадується в інтро як вбудований guardrail, але фактично злитий у Topics Boundaries — *під уточнення*).
5. **Bias Detection** — дискримінаційні патерни.
6. **Custom Guardrails** (**Advanced LLM Mesh**; DSS 13.4+) — plugin-компонент типу **LLM Guardrail** (`guardrail.json` + `guardrail.py`, клас-нащадок `BaseGuardrail`, метод `process()`). Можуть переписати query/response, попросити LLM повторити з контекстом, додати запис у trace/audit або заблокувати/перенаправити.

**RAG guardrails** (Advanced LLM Mesh) — розділ [6.5](#65-rag-guardrails-advanced-llm-mesh).

> **Патерн безпеки даних:** чутливі дані (PII) спрямовують на **локальні HuggingFace** моделі через connection-level guardrails, а зовнішні API лишають для нечутливого. PII detection дозволяє маскувати/псевдонімізувати перед відправкою у зовнішню модель.

---

## 13. Контроль витрат та моніторинг

### 13.1 Cost control / quotas (Середній)

Налаштовується в `Administration > Settings > LLM Mesh > Cost control`. Кожна **quota** задає **scope** (за провайдером / проєктом / connection / користувачем), **quota amount** (макс. вартість у **USD**), **blocking behavior** (зупиняти запити при вичерпанні), **reset period** (календарний місяць чи rolling N днів) і опційні **email alerts**. Один запит може матчити кілька quota — інкрементуються всі. **Fallback Quota** покриває запити, що не матчать жодної.

> **Custom quotas — Advanced LLM Mesh.** Базова ліцензія дає лише єдину Fallback Quota (одна квота на всі connections).

**Не покриваються квотами:** SageMaker, Databricks Mosaic AI, Snowflake Cortex; **локальні HuggingFace** не можна таргетувати квотами.

**Response caching** (LLM connection > Usage control) знижує вартість, повертаючи відповіді на ідентичні запити з кешу.

### 13.2 LLM Cost Guard

**Pre-built dashboard** для адмінів — трекінг usage і cost по сервісах/провайдерах. Тогл за user / connection / LLM type / cache status / context type / project key. Оцінки вартості — на основі семплу промптів/відповідей, застосованого до публічного «sticker» pricing.

> ⚠️ Цифри LLM Cost Guard — **оцінкові** (sticker-price sampling), не точний білінг провайдера.

### 13.3 Rate limiting / throttling

Ліміти в **requests per minute (RPM)** (TPM/токенні ліміти в доках не згадані — *під уточнення*). При перевищенні запити **автоматично throttling (затримуються)**; якщо обслужити в розумний строк не вдається — fail з rate-limiting помилкою. Scope — **per model / per provider**, глобально по всіх проєктах/connections/користувачах. Конфіг — `Administration > Settings > LLM Mesh`. **Не діє** на SageMaker, Databricks Mosaic AI, Snowflake Cortex, локальні HuggingFace (для local rate-limit додано в 14.6.0).

> **Версії:** DSS 13.4.0 — «LLM cost blocking & alerting»; DSS **14.0.0** — «**Automatic throttling for LLM connections**».

---

## 14. LLM Mesh API та Starter Kit

### 14.1 LLM Mesh API (Досвідчений)

Програмний (Python) доступ до моделей у provider-agnostic стилі. Загальний патерн:

```python
import dataiku
client = dataiku.api_client()
project = client.get_default_project()
# LLM_ID — звичайна модель, "retrieval:..." (RAG) або "agent:..." (агент)
llm = project.get_llm(LLM_ID)
resp = (llm.new_completion()
           .with_message("Summarize this text: ...")
           .execute())
print(resp.text)
```

Той самий API однаково викликає «голу» модель, **Retrieval-Augmented LLM** і **agent** — бо всі вони «LLM» у Mesh. Підтримуються streaming, structured output, multimodal (зображення у вході), tool calling. У коді можна також отримати `as_langchain_chat_model()` для інтеграції з LangChain/LangGraph (розділ [8](#8-agents-та-agentic-ai)).

### 14.2 LLM Starter Kit

**LLM Starter Kit** (`EX_LLM_STARTER_KIT`) — завантажуваний sample-проєкт, що демонструє end-to-end GenAI: prompt engineering, RAG з Knowledge Banks, оцінювання LLM. Потребує Python 3.10 код-env з OpenAI, Anthropic, Cohere, AI21, LangChain, Transformers, Sentence-Transformers, FAISS, BERT-Score. Доступний через Gallery (`gallery.dataiku.com`) і `downloads.dataiku.com`.

---

## 15. Best practices

**Вибір моделі:**
- Підбирайте модель під задачу й бюджет: дешевші/менші моделі для класифікації/підсумовування, потужніші — для складної генерації й агентів. Завдяки LLM Mesh модель легко замінити, не чіпаючи pipeline (звертайтеся за LLM ID).
- Для агентів обирайте модель із надійним **tool/function calling**.
- Порівнюйте кандидатів пліч-о-пліч у **Prompt Studio**, а формально — через **Evaluate LLM** / **Model comparisons**.

**Безпека даних:**
- Чутливі дані (PII) — **на локальні HuggingFace моделі**; зовнішні API — для нечутливого.
- Увімкніть **PII Detection** guardrail (mask/pseudonymize) на connection-рівні перед відправкою у зовнішні моделі.
- Додавайте **Prompt Injection** і **Toxicity** guardrails для user-facing застосунків (Dataiku Answers, агенти).
- Керуйте доступом до connections через права LLM Mesh; усе логуйте (audit/tracing).

**Контроль витрат:**
- Налаштуйте **cost quotas** з blocking + email alerts (per project/user); тримайте **Fallback Quota** як страхувальну сітку.
- Увімкніть **response caching** для повторюваних запитів.
- Моніторте **LLM Cost Guard**, пам'ятаючи, що цифри оцінкові.
- Покладайтеся на **automatic throttling**, щоб не впертися в rate limits провайдера.

**Якість RAG:**
- Перевіряйте retrieval у режимі **Search** Knowledge Bank ще до підключення LLM — погані чанки = погані відповіді.
- Тюньте **chunking**, **top-k**, **hybrid search** і **reranking** під свій корпус.
- Увімкніть **RAG guardrails** (faithfulness/relevancy thresholds), щоб відсікати галюцинації.
- Оцінюйте через **Evaluate LLM/Agent** (Context Precision/Recall, Faithfulness, Answer Correctness); збирайте user feedback у Dataiku Answers як сигнал якості.
- Для документів обирайте правильний extraction engine (structured/VLM) і OCR під формат корпусу.

**Агенти:**
- Починайте з **Simple Visual Agent**; переходьте на **Structured Visual Agent** / **Code Agent**, коли потрібен детермінований контроль чи складна логіка.
- Давайте інструментам **чіткі описи** — LLM обирає tool саме за описом + JSON-schema.
- Використовуйте **Trace Explorer** для дебагу й **Agent Evaluation** для якості.

---

## 16. Типові помилки

- **Прямі виклики провайдерів в обхід LLM Mesh.** Втрачаєте governance, аудит, квоти, кеш, guardrails. Завжди — через LLM connections.
- **Чутливі дані у зовнішні API без guardrails.** PII може витекти. Спершу — PII Detection / локальні моделі.
- **Очікувати рецепти `Translate`/`Generate`.** Їх немає — це робиться через **Prompt recipe**.
- **Ігнорувати error-колонку Prompt recipe.** Окремі рядки падають мовчки; перевіряйте error-колонку.
- **RAG без перевірки retrieval.** Якщо погано працює **Search** Knowledge Bank, жодна LLM не врятує — спершу налаштуйте chunking/embedding/top-k.
- **Плутати Knowledge Bank Search Tool з RAG.** Tool **повертає документи, але не генерує відповідь** — генерацію робить агент.
- **Розраховувати на квоти/rate-limit для всіх провайдерів.** Не діють на SageMaker, Databricks Mosaic AI, Snowflake Cortex, локальні HuggingFace.
- **Сприймати LLM Cost Guard як точний білінг.** Це **оцінка** за sticker-price.
- **Fine-tune без GPU / без add-on.** `Fine-tune` recipe потребує **Advanced LLM Mesh**, а локальний fine-tuning — GPU. Пам'ятайте: fine-tune **не** застосовує connection guardrails.
- **Agent на моделі без tool calling.** Агент не зможе викликати інструменти — оберіть сумісну модель.
- **Забути ввімкнути connection для vector store.** GCS / Elasticsearch / OpenSearch / PostgreSQL потрібно явно дозволити для Knowledge Bank.

---

## 17. Перехресні посилання

- `01-overview-architecture.md` — місце LLM Mesh у платформі (DataOps → MLOps → **LLMOps**), типи нод.
- `02-projects-flow-core-concepts.md` — Flow, рецепти, проєкти (Prompt recipe, Embed recipe, агенти — вузли Flow).
- `03-datasets-connections-partitioning.md` — connections загалом; vector store connections (Elasticsearch, pgvector, Pinecone тощо).
- `04-visual-recipes.md` — візуальні рецепти; `Classify text` / `Summarize text` поряд з рештою.
- `06-coding-recipes-code-envs-notebooks.md` — code envs (для local LLM, Presidio, LLM Starter Kit) та Code Agent.
- `07-machine-learning-visual-ml.md` — класичний ML vs LLM-підхід; evaluation stores і Model comparisons.
- `09-scenarios-automation-metrics-checks.md` — оркестрація Embed/Prompt/Evaluate рецептів у сценаріях, моніторинг.
- `10-mlops-deployment-api-node.md` — деплой RAG/агентів/Prompt recipe у продакшен, API node.
- `11-programmatic-api-python-client.md` — LLM Mesh API детально, `dataiku` client, LangChain.
- `12-dashboards-charts-webapps-reporting.md` — Dataiku Answers і Agent Hub як webapps; Trace Explorer.
- `13-governance-security-collaboration.md` — permissioning connections, аудит, governance над LLM/агентами.
- `14-plugins-applications-extensibility.md` — Dataiku Answers / Agent Hub як plugins; custom guardrails та custom agent tools як plugin-компоненти.
- `15-best-practices.md` — наскрізні best practices.
- `16-glossary.md` — терміни: LLM Mesh, RAG, Knowledge Bank, guardrail, agent, embedding.

---

> **Резюме.** LLM Mesh — це governance-шлюз Dataiku до десятків LLM-провайдерів (OpenAI, Azure OpenAI/Foundry, Anthropic, Bedrock, Vertex AI, Mistral, Cohere, локальні HuggingFace тощо) з єдиними правами, аудитом, квотами, кешем і guardrails. Поверх нього — повний LLMOps-стек: Prompt Studio/recipe, візуальні рецепти `Classify`/`Summarize`, RAG через Embed-рецепти + Knowledge Banks + vector stores, AI Agents з Agent Tools/MCP, готовий чат-застосунок **Dataiku Answers**, а також `Fine-tune` й `Evaluate` рецепти та guardrails (PII/toxicity/injection/RAG) — частина в add-on **Advanced LLM Mesh**.
