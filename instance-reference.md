# Dataiku — довідка по нашому інстансу (instance reference)

> Внутрішня довідка про конкретний інстанс Dataiku нашої команди. Доповнює загальну
> документацію (`docs/`) реальними фактами: версія, доступні connections, можливості.
> **Вантажиться в NotebookLM як окреме джерело — разом із файлами `docs/`.**
> Не містить хостів, URL чи credentials.

**Актуально на:** 2026-06-25 · **версія DSS:** 14.5.1

---

## Платформа

- **Продукт:** Dataiku (раніше Dataiku DSS).
- **Версія:** **14.5.1** — сучасна гілка 14.x; доступні новітні можливості (GenAI / Agent Hub).
- **Розгортання:** self-managed on-prem (на внутрішньому сервері).
- **Робоча нода:** **Design node** (повний UI: Flow, Lab, recipes, побудова проєктів).

## Доступні connections (джерела даних)

Зовнішні джерела, налаштовані в інстансі:

- **Amazon Redshift** — основне аналітичне сховище (тут переважно й лежать робочі дані).
- **SQL** — SQL-бази.
- **Google Sheets** — табличні дані.
- **Jira** — тікети/задачі.
- **File System**, **HTTP / HTTP with Cache** — допоміжні (файли на сервері / дані за URL).

Також доступні вбудовані та внутрішні типи: Upload files, Folder / Files in Folder,
Editable Managed Database, Model Evaluation Store, Feature Group, Metrics, Internal Stats,
Experiments, Dataset from another Project.

## Можливості інстансу

Інстанс **повнофункціональний**:

- **MLOps:** Model Evaluation Store (drift), Feature Group (Feature Store), Experiments (tracking).
- **GenAI:** Agent Hub, LLM Mesh.
- **Програмний доступ:** Public API доступний (Personal API keys, пакет `dataikuapi`) —
  можлива програмна побудова Flow.

## Як це впливає на практику

- Дані переважно в **Redshift** → тримай обчислення в базі (**SQL pushdown**, in-database engine),
  не вантаж усе в пам'ять DSS; для великих обсягів — chunked-обробка або SQL.
- Інші джерела — **Google Sheets** і **Jira**.
- Версія **14.5.1** свіжа → можна спиратися на сучасні фічі (GenAI, Agent Hub, новітні recipes).
- Загальні принципи й how-to — у `docs/` (особливо `docs/15-best-practices.md`).
