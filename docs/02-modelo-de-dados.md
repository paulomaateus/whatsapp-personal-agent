# Modelo de dados

Convenções:

- **PKs** são `bigint GENERATED ALWAYS AS IDENTITY`.
- **Datas** são sempre `timestamptz`, gravadas em UTC.
- **Enums** são colunas `text` com `CHECK`, não tipos `ENUM` do Postgres, para facilitar migrations.
- Toda tabela tem `created_at timestamptz NOT NULL DEFAULT now()`. As tabelas que sofrem alteração têm também `updated_at`.
- **Cada fase cria apenas as tabelas listadas nela.**

Extensões (Fase 1): `vector` e `unaccent`.

---

## Fase 1: histórico, busca e fila

### `contacts`

Uma pessoa.

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| display_name | text NOT NULL | Nome exibido; editável |
| phone_number | text NULL | Só dígitos, com DDI (`5511999999999`) |
| is_self | bool NOT NULL DEFAULT false | Exatamente um contato com `true` (índice único parcial) |
| is_authorized | bool NOT NULL DEFAULT false | Usado a partir da Fase 5 |
| automation_enabled | bool NOT NULL DEFAULT false | Usado a partir da Fase 5 |
| notes | text NULL | |
| created_at, updated_at | | |

Regra: `automation_enabled = true` exige `is_authorized = true` (`CHECK`).

### `contact_identifiers`

As várias formas como um mesmo contato aparece. Isso resolve o problema de o export trazer nomes e o tempo real trazer JID/LID.

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| contact_id | bigint FK → contacts ON DELETE CASCADE | |
| kind | text | `JID` \| `LID` \| `PHONE` \| `EXPORT_NAME` |
| value | text | Normalizado: JID/LID em minúsculas; `EXPORT_NAME` com `trim` e sem caracteres invisíveis |
| created_at | | |

`UNIQUE (kind, value)`.

### `conversations`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| kind | text | `INDIVIDUAL` \| `GROUP` \| `SELF` |
| title | text NOT NULL | Nome do contato ou do grupo |
| chat_jid | text NULL UNIQUE | Preenchido quando a conversa aparece no tempo real (`…@c.us`, `…@g.us`, `…@lid`) |
| contact_id | bigint FK → contacts NULL | Obrigatório em `INDIVIDUAL` e `SELF`; `NULL` em `GROUP` |
| last_message_at | timestamptz NULL | |
| created_at, updated_at | | |

### `conversation_participants`

| Coluna | Tipo |
|---|---|
| conversation_id | FK → conversations |
| contact_id | FK → contacts |

PK: `(conversation_id, contact_id)`.

### `imports`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| original_filename | text | |
| file_sha256 | text | Não é único: reimportar é permitido e idempotente |
| status | text | `processing` \| `completed` \| `failed` |
| conversation_id | FK → conversations NULL | |
| stats | jsonb | Contagens (linhas lidas, mensagens novas, duplicadas, ignoradas, mídias, erros de parse) |
| error | text NULL | |
| created_at, finished_at | | |

### `messages`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| conversation_id | FK → conversations | |
| sender_contact_id | FK → contacts NULL | `NULL` para mensagens de sistema |
| from_me | bool NOT NULL | Enviada pela conta do usuário (pelo usuário **ou** pelo bot) |
| source | text | `EXPORT` \| `REALTIME` \| `HISTORY_SYNC` |
| external_id | text NULL | ID do WhatsApp (tempo real / history sync) |
| export_hash | text NULL | ID sintético do export (ver Fase 1) |
| message_type | text | `TEXT` \| `IMAGE` \| `VIDEO` \| `AUDIO` \| `DOCUMENT` \| `STICKER` \| `LOCATION` \| `CONTACT` \| `POLL` \| `SYSTEM` \| `DELETED` \| `OTHER` |
| content | text NULL | Texto ou legenda |
| sent_at | timestamptz NOT NULL | |
| timestamp_precision | text | `minute` (export Android) \| `second` |
| reply_to_message_id | FK → messages NULL | Só disponível no tempo real |
| is_edited | bool DEFAULT false | |
| media_path | text NULL | Caminho relativo no volume `media` |
| import_id | FK → imports NULL | |
| metadata | jsonb DEFAULT '{}' | Dados brutos úteis que não viraram coluna |
| tsv | tsvector | Mantido por trigger: `to_tsvector('portuguese', unaccent(coalesce(content,'')))` |
| created_at, updated_at | | |

Índices e restrições:

- `UNIQUE (conversation_id, external_id) WHERE external_id IS NOT NULL`
- `UNIQUE (conversation_id, export_hash) WHERE export_hash IS NOT NULL`
- `(conversation_id, sent_at)`
- `GIN (tsv)`

> Sobre o nome `source`: a especificação antiga chamava de `BACKUP`. O nome certo é `EXPORT`, porque o arquivo exportado não é um backup do WhatsApp.

### `conversation_segments`

Unidade de busca semântica. É **derivada** e pode ser recriada a partir de `messages`.

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| conversation_id | FK → conversations ON DELETE CASCADE | |
| first_message_id, last_message_id | FK → messages | |
| start_at, end_at | timestamptz | |
| message_count | int | |
| text | text | Texto renderizado (formato na Fase 1) |
| content_hash | text | sha256 de `text`. Se o texto não mudou, não refaz o embedding |
| tsv | tsvector | Mantido por trigger (igual a `messages`) |
| embedding | vector(1024) NULL | `NULL` até o job de embedding rodar |
| embedding_model | text NULL | |
| created_at, updated_at | | |

Índices: `HNSW (embedding vector_cosine_ops)`, `GIN (tsv)`, `(conversation_id, start_at)`.

### `jobs`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| kind | text | Ex.: `build_segments`, `embed_segments`, `build_response_examples`, `route_message`, `run_personal_agent`, `generate_reply` |
| payload | jsonb | |
| status | text | `queued` \| `running` \| `done` \| `failed` |
| priority | smallint DEFAULT 100 | Menor = mais urgente. Tempo real: 10; backfill: 100 |
| attempts | int DEFAULT 0 | |
| max_attempts | int DEFAULT 5 | |
| run_after | timestamptz DEFAULT now() | Usado para *backoff*, *debounce* e adiamento por cota |
| dedupe_key | text NULL | `UNIQUE WHERE status IN ('queued','running')`: evita jobs duplicados na fila |
| locked_at | timestamptz NULL | |
| last_error | text NULL | |
| created_at, updated_at | | |

Índice: `(status, priority, run_after)`.

---

## Fase 2: tempo real

### `webhook_events`

Log bruto dos eventos do WAHA, para depuração e reprocessamento.

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| event_type | text | |
| payload | jsonb | |
| status | text | `received` \| `processed` \| `ignored` \| `error` |
| error | text NULL | |
| received_at | timestamptz | |

Retenção: eventos processados com mais de 30 dias são apagados por um job diário.

---

## Fase 3: exemplos, avaliação e chamadas ao LLM

### `response_examples`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| conversation_id | FK → conversations ON DELETE CASCADE | Só conversas `INDIVIDUAL` |
| contact_id | FK → contacts | O interlocutor |
| context_message_ids | bigint[] | Turnos anteriores |
| incoming_message_ids | bigint[] | Turno recebido |
| response_message_ids | bigint[] | Turno de resposta do usuário |
| context_text | text | |
| incoming_text | text | |
| response_text | text | |
| response_delay_seconds | int | Da última mensagem recebida à primeira resposta |
| origin | text | `HISTORY` \| `FEEDBACK` (Fase 6) |
| weight | real DEFAULT 1.0 | Exemplos vindos de feedback pesam mais |
| embedding | vector(1024) NULL | Embedding de `context_text + incoming_text` |
| embedding_model | text NULL | |
| created_at | | |

`UNIQUE (conversation_id, (response_message_ids[1]))`, com índice por expressão, para a extração ser idempotente. Índice HNSW no embedding.

### `llm_calls`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| purpose | text | `eval`, `personal_agent`, `response_agent`, … |
| provider_base_url | text | |
| model | text | |
| request | jsonb | Mensagens enviadas (para depuração) |
| response | jsonb NULL | |
| status | text | `ok` \| `error` \| `rate_limited` |
| prompt_tokens, completion_tokens | int NULL | |
| latency_ms | int | |
| error | text NULL | |
| created_at | | |

O contador diário do *rate limiter* é `count(*)` em `llm_calls` no dia. Retenção de `request`/`response`: 90 dias; depois disso esses campos viram `NULL`.

### `eval_runs` / `eval_items`

| `eval_runs` | Notas |
|---|---|
| id, created_at | |
| config | jsonb: modelo, prompt, k exemplos, tamanho da amostra |
| status | `running` \| `completed` |
| summary | jsonb: métricas agregadas |

| `eval_items` | Notas |
|---|---|
| id | |
| eval_run_id | FK |
| response_example_id | FK |
| generated_text | |
| real_text | |
| similarity | real: cosseno entre os embeddings da resposta real e da gerada |
| human_choice | text NULL: `REAL` \| `GENERATED` \| `UNSURE` (teste cego) |
| human_score | smallint NULL: 1–5, "soa como eu?" |

---

## Fase 4: Personal Agent

### Alterações
- `messages.sent_by_bot bool NOT NULL DEFAULT false`
- Nova tabela `app_state(key text PK, value jsonb, updated_at)`: estado global, como `agents_paused`.

### `outbound_messages`

Toda mensagem enviada **pelo sistema**. Serve para o webhook reconhecer e ignorar as próprias mensagens do bot, evitando loop.

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| whatsapp_message_id | text NULL UNIQUE | Retornado pelo WAHA; `NULL` enquanto o envio não confirma |
| chat_jid | text | |
| text | text | |
| purpose | text | `AGENT_ANSWER` \| `SUGGESTION` \| `NOTICE` \| `APPROVED_REPLY` |
| pending_reply_id | FK NULL | A partir da Fase 5 |
| created_at | | |

### `agent_runs`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| agent | text | `personal` \| `response` |
| trigger_message_id | FK → messages | |
| status | text | `running` \| `completed` \| `failed` |
| steps | jsonb | Chamadas de ferramenta (nome, argumentos, resumo do resultado) |
| output_text | text NULL | |
| error | text NULL | |
| created_at, finished_at | | |

---

## Fase 5: aprovação

### `pending_replies`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| short_code | text | Código curto (`A3`, `B7`), único entre as pendências ativas |
| conversation_id | FK → conversations | |
| contact_id | FK → contacts | |
| trigger_message_id | FK → messages | Última mensagem recebida considerada |
| status | text | `GENERATING` \| `AWAITING_APPROVAL` \| `REVISING` \| `SENDING` \| `CLOSED` (ver Fase 5) |
| current_revision | int DEFAULT 0 | |
| final_text | text NULL | Texto efetivamente enviado |
| sent_whatsapp_message_id | text NULL | |
| resolution | text NULL | `APPROVED` \| `APPROVED_AFTER_CORRECTION` \| `REJECTED` \| `SUPERSEDED_BY_MANUAL_REPLY` \| `SUPERSEDED_BY_NEW_MESSAGE` \| `EXPIRED` \| `SEND_FAILED` \| `GENERATION_FAILED` |
| manual_reply_message_ids | bigint[] NULL | Quando o usuário respondeu direto pelo celular |
| expires_at | timestamptz | |
| created_at, updated_at | | |

Índice único parcial: no máximo **uma** pendência ativa por conversa.

### `reply_revisions`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| pending_reply_id | FK ON DELETE CASCADE | |
| revision | int | 1, 2, 3… |
| instruction | text NULL | Correção pedida pelo usuário (`NULL` na revisão 1) |
| text | text | Sugestão gerada |
| retrieval | jsonb | IDs de mensagens, exemplos e memórias usados no prompt |
| llm_call_id | FK → llm_calls | |
| suggestion_whatsapp_message_id | text NULL | ID da mensagem `🤖` no self chat, usado para identificar *reply/quote* |
| created_at | | |

`UNIQUE (pending_reply_id, revision)`.

---

## Fase 6: memória

### `memories`

| Coluna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| content | text | Frase curta e autocontida |
| origin | text | `USER_EXPLICIT` \| `AGENT_PROPOSED` |
| status | text | `ACTIVE` \| `PENDING_CONFIRMATION` \| `ARCHIVED` |
| contact_id | FK NULL | Quando a memória é sobre um contato |
| usable_in_replies | bool NOT NULL DEFAULT false | Só com `true` a memória pode entrar no prompt do Response Agent |
| source_message_id | FK → messages NULL | |
| embedding | vector(1024) NULL | |
| embedding_model | text NULL | |
| created_at, updated_at | | |

---

## Troca de modelo de embeddings

Todas as colunas `embedding` usam a mesma dimensão (`EMBEDDING_DIM`). Para trocar de modelo:

1. Criar uma migration que altera a dimensão das colunas (`ALTER … TYPE vector(N)`) e recria os índices HNSW.
2. Zerar `embedding` e `embedding_model`.
3. Enfileirar os jobs de re-embedding.

A busca **sempre** filtra `embedding_model = settings.EMBEDDING_MODEL`, para nunca comparar vetores de modelos diferentes.
