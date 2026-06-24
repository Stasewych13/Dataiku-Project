# Dataiku-Project

База знань і робоче середовище для роботи з платформою **Dataiku** (Dataiku DSS / Dataiku Cloud).

Репозиторій містить три складові:

1. **`docs/`** — вичерпна українськомовна база знань про весь функціонал Dataiku та best practices.
   Технічні терміни, назви елементів UI, рецептів і код залишені в оригіналі (English).
2. **`.claude/skills/dataiku/`** — Claude Code skill для роботи з Dataiku: інструкції, чеклісти,
   довідкові матеріали, які Claude підвантажує під час виконання задач у Dataiku.
3. **`MEMORY.md`** — жива пам'ять про фактичний процес роботи в Dataiku. Оновлюється під час
   реалізації проєкту: коли фактичний стан у програмі відрізняється від загальних знань — фіксуємо тут.

## Структура `docs/`

| Документ | Тема |
|----------|------|
| `01-overview-architecture.md` | Огляд платформи та архітектура (ноди, deployment) |
| `02-projects-flow-core-concepts.md` | Проєкти, Flow, базові концепції |
| `03-datasets-connections-partitioning.md` | Datasets, connections, partitioning |
| `04-visual-recipes.md` | Visual recipes (Group, Join, Window, …) |
| `05-data-preparation-prepare-recipe.md` | Prepare recipe, processors, формули |
| `06-coding-recipes-code-envs-notebooks.md` | Code recipes, code environments, notebooks |
| `07-machine-learning-visual-ml.md` | Visual ML / AutoML, моделі, оцінювання |
| `08-llm-generative-ai-mesh.md` | LLM Mesh та Generative AI |
| `09-scenarios-automation-metrics-checks.md` | Scenarios, автоматизація, metrics, checks |
| `10-mlops-deployment-api-node.md` | MLOps, deployment, bundles, API node |
| `11-programmatic-api-python-client.md` | Програмний доступ: Python client, REST API |
| `12-dashboards-charts-webapps-reporting.md` | Dashboards, charts, webapps, звітність |
| `13-governance-security-collaboration.md` | Governance, безпека, колаборація |
| `14-plugins-applications-extensibility.md` | Plugins, applications, розширюваність |
| `15-best-practices.md` | Зведені best practices |
| `16-glossary.md` | Глосарій термінів |

## Як користуватись

- Читай `docs/` як довідник — кожен файл самодостатній і має перехресні посилання.
- У Claude Code викликай skill: `/dataiku` (коли працюєш над задачами в Dataiku).
- Перед роботою звіряйся з `MEMORY.md` — там зафіксований реальний процес.

> Джерела: офіційна документація Dataiku (doc.dataiku.com), Knowledge Base (knowledge.dataiku.com),
> Developer Guide (developer.dataiku.com), Dataiku Academy. Актуально для DSS 13.x/14.x (2025–2026).
