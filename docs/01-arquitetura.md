# Arquitetura

## Visão geral

```
                    ┌────────────────────────┐
                    │  WhatsApp (celular)    │
                    └──────┬──────────┬──────┘
          Exportar conversa│          │ dispositivo vinculado
             (.txt / .zip) │          │
                           ▼          ▼
                   ┌────────────┐  ┌──────────────┐
   usuário ──────► │ Import API │  │    WAHA      │  container separado,
   (upload)        └─────┬──────┘  │ (client WA)  │  sem lógica de IA
                         │         └──┬────────▲──┘
                         │   webhook  │        │ HTTP (sendText)
                         ▼            ▼        │
                ┌──────────────────────────────┴───┐
                │              api (FastAPI)       │
                │  imports · webhooks · admin ·    │
                │  search · MCP (/mcp)             │
                └──────────────┬───────────────────┘
                               │ grava mensagens + enfileira jobs
                               ▼
                ┌──────────────────────────────────┐
                │   PostgreSQL + pgvector          │
                │   dados · vetores · FTS · fila   │
                └──────────────┬───────────────────┘
                               │ consome jobs
                               ▼
                ┌──────────────────────────────────┐
                │           worker                 │
                │ segmentos · embeddings ·         │
                │ exemplos · router · agentes      │
                └───────┬──────────────────┬───────┘
                        │                  │
                        ▼                  ▼
               ┌────────────────┐  ┌──────────────────┐
               │ Ollama (CPU)   │  │ Gemini API       │
               │ só embeddings  │  │ geração (LLM)    │
               └────────────────┘  └──────────────────┘

  Claude Code ──(MCP, HTTP localhost)──► api /mcp   (ferramentas de leitura)
```

### Princípios

1. **Dados brutos são a fonte da verdade.** Segmentos, embeddings e exemplos são derivados e podem ser apagados e reconstruídos a qualquer momento a partir de `messages`.
2. **Persistir primeiro, processar depois.** Receber uma mensagem nunca espera por embedding ou LLM. Todo processamento derivado passa pela fila de jobs.
3. **Retrieval determinístico, geração mínima.** O Response Agent não usa tool calling: o código busca o contexto e faz uma única chamada ao LLM. Só o Personal Agent usa tool calling.
4. **Nenhum LLM envia mensagem.** O envio para contatos só acontece pela máquina de estados de aprovação, acionada por um comando do usuário.
5. **Provedores trocáveis.** O LLM e o modelo de embeddings ficam atrás de interfaces pequenas, configuradas por variáveis de ambiente.

## Containers (`docker-compose.yml`)

| Serviço | Imagem | Função | Porta (host) |
|---|---|---|---|
| `postgres` | `pgvector/pgvector:pg17` | Banco, vetores, busca textual e fila de jobs | `127.0.0.1:5432` |
| `ollama` | `ollama/ollama` | Embeddings locais (`bge-m3`) na CPU | não exposta |
| `waha` | `devlikeapro/waha:gows` (versão Core, gratuita, engine GOWS) | Client do WhatsApp: sessão, eventos, envio, histórico | `127.0.0.1:3000` (painel/API) |
| `api` | build local | FastAPI: imports, webhook do WAHA, APIs admin, busca, servidor MCP | `127.0.0.1:8000` |
| `worker` | mesmo build do `api`, outro comando | Consome a fila de jobs | nenhuma |

- **Todas as portas ficam presas em `127.0.0.1`.** Nada é exposto na rede.
- O `api` roda `alembic upgrade head` ao iniciar.
- O `api` garante na inicialização a configuração da sessão do WAHA (`PUT /api/sessions/{WAHA_SESSION}`: webhook, eventos, HMAC e `ignore.status`). O webhook é configurado **por sessão**, não por variável de ambiente do WAHA (ver [fase-0-resultados.md](fases/fase-0-resultados.md)).
- Volumes: `pgdata`, `ollama_models`, `waha_sessions` e `media` (mídias extraídas dos imports).
- O modelo de embeddings é baixado pelo próprio app na inicialização (`POST /api/pull` do Ollama) se ainda não existir.

## Stack (fechada)

| Área | Escolha |
|---|---|
| Linguagem | Python 3.12 |
| Dependências | `uv` (`pyproject.toml` + `uv.lock`) |
| API | FastAPI + Uvicorn |
| Configuração | `pydantic-settings`, lida do `.env` (com `.env.example` versionado) |
| ORM | SQLAlchemy 2.x **async** + `asyncpg` |
| Migrations | Alembic |
| Vetores | `pgvector` (extensão) + pacote Python `pgvector` |
| Busca textual | Full-text search do Postgres (`portuguese` + `unaccent`) |
| Fila | Tabela `jobs` no Postgres com `SELECT … FOR UPDATE SKIP LOCKED` (implementação própria, sem Redis/Celery) |
| HTTP client | `httpx` (async) |
| LLM | SDK `openai` apontando para o endpoint compatível com OpenAI do Gemini |
| Embeddings | Ollama `/api/embed`, modelo `bge-m3` (1024 dimensões) |
| MCP | SDK oficial `mcp` (FastMCP), transporte *streamable HTTP* montado em `/mcp` no `api` |
| Testes | `pytest` + `pytest-asyncio`, Postgres de teste via compose |
| Lint/format | `ruff` |
| Logs | `logging` padrão com saída JSON (`python-json-logger`) |

## Configuração (`.env`)

```dotenv
# Banco
DATABASE_URL=postgresql+asyncpg://app:app@postgres:5432/whatsapp_agent

# Fuso horário usado para interpretar os horários do export (que não trazem fuso)
APP_TIMEZONE=America/Sao_Paulo

# Embeddings
EMBEDDING_BASE_URL=http://ollama:11434
EMBEDDING_MODEL=bge-m3
EMBEDDING_DIM=1024

# LLM (qualquer endpoint compatível com OpenAI)
LLM_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/
LLM_API_KEY=...
LLM_MODEL=gemini-2.5-flash          # conferir modelos gratuitos disponíveis na época
LLM_REQUESTS_PER_MINUTE=8           # abaixo do limite do plano gratuito
LLM_REQUESTS_PER_DAY=200

# WAHA
WAHA_BASE_URL=http://waha:3000
WAHA_API_KEY=...
WAHA_SESSION=default
WAHA_WEBHOOK_HMAC_KEY=...

# Identidade do usuário (o self chat chega com os dois IDs; ambos também vêm em `me` de cada evento)
SELF_JID=                          # telefone, ex.: 5511999999999@c.us
SELF_LID=                          # LID, ex.: 123456789012345@lid
SELF_DISPLAY_NAMES=Paulo           # nomes com que o usuário aparece nos exports, separados por vírgula

# Imports
IMPORT_MAX_UPLOAD_MB=500
IMPORT_MAX_UNCOMPRESSED_MB=2000
```

## Provedores de IA

### Embeddings: local

- Ollama na CPU com `bge-m3`: multilíngue, 1024 dimensões, bom em português.
- É o único modelo que vê **todo** o histórico, por isso fica local.
- A dimensão é fixa na coluna `vector(1024)`. Trocar de modelo exige migration e re-embedding (ver [02-modelo-de-dados.md](02-modelo-de-dados.md#troca-de-modelo-de-embeddings)).
- Interface:

```python
class EmbeddingProvider(Protocol):
    model: str
    dim: int
    async def embed(self, texts: list[str]) -> list[list[float]]: ...
```

### Geração: Gemini (plano gratuito)

- Acesso pelo endpoint compatível com OpenAI, usando o SDK `openai` com `base_url` configurável. Trocar para Groq, OpenRouter ou Ollama é só mudar o `.env`.
- **Privacidade (decisão consciente do usuário):** no plano gratuito, o Google pode usar os dados enviados para melhorar seus produtos. O sistema minimiza o que envia: só o recorte necessário a cada chamada, nunca o histórico em massa.
- **Limites do plano gratuito:** todo acesso passa por um *rate limiter* (limite por minuto e contador diário persistido no banco). Uma resposta `429` gera *retry* com *backoff* exponencial, reagendando o job. Estourado o limite diário, o job é adiado para o dia seguinte e o usuário é avisado no self chat (a partir da Fase 4).
- Toda chamada é registrada em `llm_calls` (modelo, tokens, latência, status).
- Interface:

```python
class ChatProvider(Protocol):
    async def complete(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        temperature: float = 0.7,
        purpose: str = "",          # gravado em llm_calls
    ) -> ChatResult: ...
```

## WhatsApp: WAHA

- **O WAHA e qualquer biblioteca desse tipo são clientes não oficiais.** Isso viola os termos de uso do WhatsApp e existe risco de o número ser banido. Mitigações: nenhum envio automático, envios só após aprovação, volume baixo e nenhuma mensagem em massa.
- O WAHA é tratado como um **adaptador isolado**. O resto do sistema conversa com ele por duas interfaces:
  - **Entrada:** `POST /api/v1/webhooks/waha`. O `webhook_parser` converte o payload do WAHA num objeto interno `IncomingEvent`, e o resto do código nunca vê o formato do WAHA.
  - **Saída:** `WhatsAppGateway.send_text(chat_jid, text, reply_to=None, message_id=None) -> sent_message_id`. O `message_id` pode ser gerado antes do envio (`WhatsAppGateway.new_message_id()`), o que permite reconhecer o eco do webhook sem depender da ordem de chegada.
- Assim, se um dia o WAHA for trocado por um serviço próprio com whatsmeow, só o adaptador muda.
- **O webhook valida a assinatura HMAC** do WAHA e rejeita requisições sem assinatura válida.

## Agentes

| | Personal Agent | Response Agent |
|---|---|---|
| Canal | Self chat | Conversas individuais de contatos autorizados |
| Quem aciona | Mensagem do usuário no self chat | Mensagem recebida de contato autorizado |
| Tool calling | Sim (ferramentas de leitura) | **Não**: pipeline fixo, uma chamada ao LLM |
| Pode enviar para terceiros | **Nunca** | **Nunca diretamente**; só gera sugestão |
| Fase | 4 | 5 |

**Message Router (Fase 5):** regra determinística, sem LLM, que decide o destino de cada mensagem recebida:

```
mensagem recebida (depois de persistida)
  ├── enviada pelo bot (🤖 / outbound_messages)         → ignorar
  ├── self chat, digitada pelo usuário                 → Personal Agent / comando de aprovação
  ├── conversa individual, from_me (usuário respondeu) → invalidar sugestão pendente daquele chat
  ├── conversa individual, contato autorizado e automação ligada
  │                                                    → Response Agent (com debounce)
  └── qualquer outro caso (grupos incluídos)           → só armazenar
```

## Ferramentas (camada `tools/`)

As ferramentas são **funções Python comuns** numa camada de serviço. Elas são usadas de três formas:

1. diretamente pelo código (pipeline do Response Agent);
2. como *function calling* do Personal Agent;
3. expostas via MCP para clientes externos (Claude Code).

| Ferramenta | Tipo | Fase |
|---|---|---|
| `search(query, contact?, conversation?, since?, until?, sender?, limit?)` | leitura | 1 |
| `get_conversation(conversation_id, since?, until?, limit?)` | leitura | 1 |
| `list_conversations(query?)` | leitura | 1 |
| `get_contact(query)` | leitura | 1 |
| `get_messages_around(message_id, before?, after?)` | leitura | 1 |
| `get_response_examples(text, contact?, limit?)` | leitura | 3 |
| `get_memories(query)` | leitura | 6 |
| `save_memory(content)` | escrita | 6 (ver regra na fase) |

**Não existe ferramenta `send_message`.** O envio é uma operação interna da máquina de aprovação e não é exposto nem ao LLM nem ao MCP.

## Estrutura do código

```
.
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml / uv.lock
├── .env.example
├── alembic.ini
├── docs/
├── app/
│   ├── config.py                 # Settings (pydantic-settings)
│   ├── main.py                   # FastAPI app, monta rotas e /mcp
│   ├── logging.py
│   ├── db/
│   │   ├── models.py             # modelos SQLAlchemy
│   │   ├── session.py
│   │   └── migrations/           # Alembic
│   ├── api/                      # rotas FastAPI (uma por arquivo)
│   │   ├── health.py
│   │   ├── imports.py
│   │   ├── contacts.py
│   │   ├── conversations.py
│   │   ├── search.py
│   │   ├── webhooks.py
│   │   └── replies.py
│   ├── whatsapp/                 # adaptador WAHA
│   │   ├── gateway.py            # send_text
│   │   └── webhook_parser.py     # payload WAHA → IncomingEvent
│   ├── importers/
│   │   └── android_export/
│   │       ├── archive.py        # validação e extração segura do zip
│   │       ├── parser.py         # .txt → ParsedMessage
│   │       └── service.py        # orquestra o import
│   ├── ingestion/
│   │   ├── messages.py           # persistência + deduplicação
│   │   └── contacts.py           # resolução/vínculo de contatos
│   ├── processing/
│   │   ├── turns.py
│   │   ├── segments.py
│   │   └── response_examples.py
│   ├── ai/
│   │   ├── embeddings.py
│   │   ├── chat.py
│   │   └── rate_limit.py
│   ├── search/
│   │   └── hybrid.py
│   ├── tools/                    # ferramentas de leitura (funções puras sobre o banco)
│   ├── mcp_server.py
│   ├── agents/
│   │   ├── router.py
│   │   ├── personal.py
│   │   ├── response.py
│   │   └── prompts/              # prompts em arquivos .md versionados
│   ├── approval/
│   │   ├── state_machine.py
│   │   └── commands.py
│   └── worker/
│       ├── queue.py              # enqueue / claim / complete / fail
│       ├── runner.py             # loop do worker
│       └── handlers.py           # kind → função
├── scripts/                      # utilitários de linha de comando (avaliação, reprocessamento)
└── tests/
    ├── fixtures/                 # exports ANONIMIZADOS e payloads de webhook de exemplo
    └── ...
```

## Fila de jobs

- A tabela `jobs` é definida em [02-modelo-de-dados.md](02-modelo-de-dados.md#jobs).
- O worker faz *claim* com `SELECT … WHERE status='queued' AND run_after <= now() ORDER BY priority, id FOR UPDATE SKIP LOCKED LIMIT 1`.
- Falhas usam *retry* com *backoff* exponencial até `max_attempts`; depois disso o job fica com status `failed` e o erro registrado.
- Jobs presos em `running` há mais de N minutos (worker morreu) voltam para `queued`.
- **Handlers devem ser idempotentes**, porque um job pode rodar mais de uma vez.
- Prioridades: jobs de tempo real (router, agentes) têm prioridade maior que jobs de backfill (embeddings de import).

## Segurança e privacidade

- **Tudo local.** As únicas saídas para fora da máquina são as chamadas à API do Gemini (recortes de conversa) e o WhatsApp via WAHA.
- **Portas** presas em `127.0.0.1`.
- **Webhook** do WAHA autenticado por HMAC.
- **API admin** sem autenticação própria no MVP, protegida por só escutar em localhost. O endpoint `/mcp` também.
- **Upload de ZIP:** limite de tamanho, bloqueio de *zip slip* (caminhos absolutos ou com `..`), limite de tamanho descomprimido e de número de arquivos (proteção contra *zip bomb*), extensões de mídia numa lista permitida, e nada extraído é executado.
- **Segredos** só no `.env`, que fica no `.gitignore`.
- **Fixtures de teste** anonimizadas: nunca versionar conversas reais.
- **Backup:** `pg_dump` documentado no README do projeto.
- **LGPD:** uso pessoal e não econômico (art. 4º, I), mas os dados são de terceiros. Por isso não se compartilha nada, não se expõe nada na rede e o que vai para APIs externas é o mínimo.

## Decisões fechadas

| Decisão | Definição | Motivo |
|---|---|---|
| Backend | Python 3.12 + FastAPI | — |
| Containers | Docker Compose, 5 serviços | — |
| Banco | PostgreSQL 17 + pgvector | Um banco só para dados, vetores, busca textual e fila |
| Fila | Tabela no Postgres (`SKIP LOCKED`) | Evita Redis no MVP |
| Client WhatsApp | WAHA Core, engine GOWS, container separado | Pronto, com HTTP e webhooks, e isolado do resto; GOWS é leve (~430 MiB) e guarda o histórico (Fase 0) |
| Fonte histórica | Export do WhatsApp **Android** (`.txt`/`.zip`) | O usuário usa Android |
| Embeddings | `bge-m3` via Ollama, **local**, CPU | Privacidade (vê todo o histórico) e não há GPU |
| LLM | Gemini free via endpoint OpenAI-compatible | Não há GPU; o provedor é trocável |
| Unidade de busca | **Segmentos** de conversa, não mensagens isoladas | Mensagens curtas ("ok", "kkk") geram vetores inúteis |
| Busca | Híbrida: vetorial + full-text, fundidas por RRF | Termos exatos (nomes, siglas) exigem busca lexical |
| Response examples | Pares de **turnos**, extraídos de forma determinística, só de conversas individuais | Rajadas de mensagens e ambiguidade em grupos |
| Response Agent | Pipeline fixo, sem tool calling | Previsibilidade e economia de cota |
| Personal Agent | Tool calling só com ferramentas de leitura | — |
| MCP | Adaptador sobre a camada `tools/`, em `/mcp` | Permite usar o Claude Code como cliente |
| Envio | Só pela máquina de aprovação; nenhuma ferramenta de envio | Segurança |
| Grupos | Armazenados e pesquisáveis; fora do Response Agent e dos exemplos | Ambiguidade de quem responde a quem |
| Fine-tuning | Fora do escopo | — |
| Mídias | Arquivos guardados em volume; sem transcrição/OCR no MVP | — |

## Fora do escopo (MVP)

- Transcrição de áudio, OCR e descrição de imagens.
- Envio automático sem aprovação.
- Response Agent em grupos.
- Interface web (a administração é pela API/Swagger e pelo self chat).
- Múltiplos usuários ou múltiplas sessões do WhatsApp.
- Import de exports de iOS (`_chat.txt`). O parser deve ficar isolado para permitir isso no futuro.
