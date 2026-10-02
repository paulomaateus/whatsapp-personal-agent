# WhatsApp Personal Agent — Documentação

Aplicação pessoal, rodando localmente em Docker, que:

1. importa e indexa o histórico de conversas do WhatsApp do usuário;
2. mantém esse histórico atualizado em tempo real;
3. oferece um **Personal Agent** (no chat do usuário consigo mesmo) para pesquisar e analisar conversas;
4. oferece um **Response Agent** que sugere respostas no estilo do usuário para contatos autorizados, **sempre com aprovação humana antes do envio**.

## Como ler esta documentação

| Documento | Conteúdo | Quando ler |
|---|---|---|
| [01-arquitetura.md](01-arquitetura.md) | Componentes, containers, stack, decisões fechadas, estrutura de código, segurança | Sempre, antes de qualquer fase |
| [02-modelo-de-dados.md](02-modelo-de-dados.md) | Schema completo do PostgreSQL, por fase | Antes de criar/alterar migrations |
| [fases/fase-0-spike-whatsapp.md](fases/fase-0-spike-whatsapp.md) | Validar o client do WhatsApp (WAHA) | Primeiro |
| [fases/fase-1-importacao-e-busca.md](fases/fase-1-importacao-e-busca.md) | Import do export Android, segmentos, embeddings, busca híbrida, MCP | — |
| [fases/fase-2-tempo-real.md](fases/fase-2-tempo-real.md) | Ingestão via webhook, vínculo de contatos, deduplicação export × tempo real | — |
| [fases/fase-3-exemplos-e-avaliacao.md](fases/fase-3-exemplos-e-avaliacao.md) | Response examples por turnos + avaliação offline do estilo | — |
| [fases/fase-4-personal-agent.md](fases/fase-4-personal-agent.md) | Personal Agent no chat consigo mesmo | — |
| [fases/fase-5-response-agent-e-aprovacao.md](fases/fase-5-response-agent-e-aprovacao.md) | Router, Response Agent, máquina de estados de aprovação | — |
| [fases/fase-6-feedback-e-memoria.md](fases/fase-6-feedback-e-memoria.md) | Aprendizado com feedback, memórias | Por último |

## Ordem e dependências

```
Fase 0 ──► Fase 1 ──► Fase 2 ──► Fase 3 ──► Fase 5 ──► Fase 6
 spike     import      tempo      exemplos    response   feedback
           + busca     real       + avaliação  agent     + memória
           + MCP        │                        ▲
                        └──────► Fase 4 ─────────┘
                                 personal agent
```

- A **Fase 0** é bloqueante: ela valida premissas sobre o WAHA que as fases 2, 4 e 5 assumem. Os resultados dela devem ser registrados em `docs/fases/fase-0-resultados.md` e, se contradisserem algo desta documentação, **a documentação deve ser atualizada antes de seguir**.
- A **Fase 1** entrega valor sozinha: com o servidor MCP, o Claude Code já consegue pesquisar o histórico.
- A **Fase 3** é um portão de qualidade: se a avaliação offline mostrar que o estilo gerado é ruim, o Response Agent não deve ir para produção até isso ser resolvido.

## Regras para quem implementa (humano ou Claude)

1. **Implementar uma fase por vez**, na ordem acima. Não antecipar tabelas, endpoints ou dependências de fases futuras.
2. **Decisões marcadas como fechadas em [01-arquitetura.md](01-arquitetura.md#decisões-fechadas) não devem ser alteradas** sem atualizar a documentação primeiro.
3. Quando algo não estiver especificado, escolher a opção mais simples que atenda aos critérios de aceite e **registrar a escolha** na seção "Decisões tomadas durante a implementação" do documento da fase.
4. Cada fase só termina quando **todos os critérios de aceite** estiverem cumpridos e cobertos por testes quando indicado.
5. **Nunca** criar caminho de código em que um LLM consiga enviar mensagem do WhatsApp sem passar pela máquina de estados de aprovação (Fase 5).

## Glossário

| Termo | Significado |
|---|---|
| **Export** | Arquivo gerado por *Mais → Exportar conversa* no WhatsApp Android (`.txt`, ou `.zip` com `.txt` + mídias). Não é o backup criptografado do WhatsApp. |
| **JID** | Identificador do WhatsApp: `5511999999999@c.us` (pessoa), `...@g.us` (grupo), `...@lid` (identificador LID, que não expõe o telefone). |
| **Self chat** | A conversa do usuário consigo mesmo. É o canal de controle dos agentes. |
| **Turno** | Sequência de mensagens consecutivas do mesmo remetente numa conversa, próximas no tempo. |
| **Segmento** | Janela de mensagens consecutivas de uma conversa, usada como unidade de busca semântica. |
| **Response example** | Par (contexto + turno recebido → turno de resposta do usuário) extraído do histórico. |
| **Pending reply** | Sugestão de resposta do Response Agent aguardando decisão do usuário. |
| **Mensagem do bot** | Mensagem enviada pelo sistema (não digitada pelo usuário), sempre prefixada com `🤖`. |
