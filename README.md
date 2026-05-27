# Sales Agent

Agente de análise de vendas Shopify construído com CrewAI. Responde perguntas em linguagem natural sobre pedidos, produtos e receita, e gera relatórios mensais em Markdown — tudo a partir dos seus dados reais do Shopify.

## Como funciona

Três agentes em pipeline sequencial:

```
fetch_orders ──→ save_orders_to_db   →   group_by_*   →   save_report
(ShopifyTool)      (DBTool)            (AnalysisTool)    (ReportTool)
      │               │                      │                │
      └───────────────┘                      │                │
         DataFetcherAgent           SalesAnalystAgent   ReportWriterAgent
```

- **DataFetcherAgent** — busca pedidos na API Shopify e persiste no PostgreSQL
- **SalesAnalystAgent** — agrupa e analisa os dados com pandas
- **ReportWriterAgent** — formata e salva o relatório em `reports/`

Os agentes só decidem *quando* e *como* chamar as tools. Toda lógica de negócio fica nas tools.

## Pré-requisitos

- Python 3.11+
- [uv](https://docs.astral.sh/uv/)
- Docker

## Setup

**1. Clone e instale as dependências**

```bash
git clone <repo>
cd primeiro_agente
uv sync
```

**2. Configure as variáveis de ambiente**

```bash
cp .env.example .env
```

Preencha o `.env`:

| Variável | Descrição |
|---|---|
| `SHOPIFY_SHOP_DOMAIN` | Ex: `minha-loja.myshopify.com` |
| `SHOPIFY_ACCESS_TOKEN` | Token do Private App (Admin → Apps → Develop apps) |
| `OPENAI_API_KEY` | Chave da API OpenAI |
| `DATABASE_URL` | Já apontado para o Docker local por padrão |
| `LLM_MODEL` | Modelo a usar (padrão: `gpt-4o-mini`) |

**3. Suba o banco e aplique as migrations**

```bash
docker compose up -d
uv run alembic upgrade head
```

## Uso

**Pergunta direta**

```bash
uv run sales-agent ask "Quais produtos mais venderam no mês passado?"
uv run sales-agent ask "Top regiões por receita" --start 2025-04-01 --end 2025-04-30
```

**Modo interativo** (loop de perguntas)

```bash
uv run sales-agent interactive
```

**Relatório mensal** (salva em `reports/`)

```bash
uv run sales-agent report --month 2025-04
```

## Estrutura do projeto

```
src/sales_agent/
├── tools/       # Funções @tool — únicas que tocam API, banco ou disco
├── agents/      # Definição de role/goal/backstory/tools (sem instanciação)
├── factory/     # AgentFactory + CrewFactory
├── models/      # Pydantic models para boundary da Shopify API
├── db/          # ORM models, session factory, OrderRepository
├── config/      # settings.py via pydantic-settings
└── cli/         # Comandos ask / report / interactive
alembic/         # Migrations — nunca editar versions/ manualmente
reports/         # Relatórios gerados
tests/
```

## Testes

```bash
# Todos os testes
uv run pytest

# Teste específico
uv run pytest tests/test_analysis_tool.py -v
```

Os testes de tools usam dados fixos (sem API real). Mocks HTTP via `pytest-httpx`.

## Stack

| Camada | Tecnologia |
|---|---|
| Orquestração de agentes | CrewAI |
| LLM | OpenAI GPT-4o-mini (configurável via `LLM_MODEL`) |
| API Shopify | httpx com cursor-based pagination |
| Banco de dados | PostgreSQL 16 + SQLAlchemy 2 + Alembic |
| Análise de dados | pandas |
| CLI | Typer + Rich |
| Gerenciador de pacotes | uv |
