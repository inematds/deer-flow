# Como usar os agentes no dia a dia

Este documento cobre o **uso manual** dos agentes na interface web.
Para **criar** novos custom agents, veja
[`criar-agentes.md`](./criar-agentes.md). Para fazer os agentes rodarem
sozinhos (canais IM, cron, API externa), veja
[`agentes-autonomos.md`](./agentes-autonomos.md).

---

## Agente vs. modelo — não confunda

Esses dois conceitos são independentes e você pode misturá-los livremente:

- **Agente** = persona. É o SOUL.md + skills permitidas + prompt. Define
  **quem** está respondendo (como pensa, que regras segue, que estilo usa).

- **Modelo** = LLM que executa. Define **quem é o cérebro** da resposta
  (Qwen, Claude, GPT, etc.).

Combinações possíveis:

| Agente            | Modelo             | Resultado                                        |
|-------------------|--------------------|--------------------------------------------------|
| `coder`           | `claude-opus`      | Code reviewer com o modelo mais capaz            |
| `coder`           | `qwen-agentic-72b` | Code reviewer 100% local, zero custo externo     |
| `pesquisador`     | `claude-sonnet`    | Research com boa qualidade e custo moderado      |
| `escritor`        | `qwen3-moe`        | Redação rápida, local                            |
| `escritor`        | `gemma4-31b`       | Redação em PT-BR, geração de texto fluente       |

Se o custom agent tem `model:` declarado no próprio `config.yaml` (em
`backend/.deer-flow/agents/{nome}/config.yaml`), esse valor **sobrepõe**
o dropdown do frontend — útil quando você quer garantir "este agente
sempre usa Opus".

---

## Selecionar agente e modelo na UI

No frontend (`http://localhost:2026`):

1. Clique em **New Chat** ou abra uma conversa existente
2. No cabeçalho da conversa, use o **seletor de agente** para escolher
   `lead_agent` (default) ou um custom agent que você criou
3. No **seletor de modelo** (separado), escolha a LLM
4. Mande a mensagem

---

## Os três toggles que mudam tudo

A interface expõe três flags que transformam o DeerFlow de "chatbot
reativo" em "agente que trabalha sozinho por muito tempo":

| Toggle                     | O que faz                                                                                           | Quando ligar                              |
|----------------------------|-----------------------------------------------------------------------------------------------------|-------------------------------------------|
| **Thinking**               | Ativa raciocínio explícito do modelo (Qwen3, DeepSeek-R1, Claude, GPT-5 — os que suportam)          | Problemas complexos, matemática, código   |
| **Plan Mode**              | Liga o `TodoListMiddleware` — agente quebra a tarefa em todos, marca `in_progress`, executa, marca `done`, pega o próximo | Tarefas com 3+ etapas                     |
| **Subagents** (task tool)  | Habilita a tool `task()` — lead agent delega para sub-workers que rodam em paralelo                  | Tarefas paralelizáveis                    |

**Regra prática**: para tarefas triviais (pergunta simples, tradução
curta) deixe tudo desligado — mais rápido e barato. Para tarefas
pesadas (pesquisa, análise de múltiplos arquivos, relatório), ligue os
três.

---

## Anatomia de uma sessão "agente trabalhando sozinho"

Exemplo real. Você digita:

> *"Pesquise as 3 principais mudanças no direito tributário brasileiro
> em 2026, gere um relatório com fontes e salve como `tributario.md`."*

Com **Plan Mode + Subagents ligados**, o lead agent vai:

1. **Planejar** (TodoList interno):
   ```
   ☐ Pesquisar reforma tributária 2026
   ☐ Pesquisar mudanças em ICMS
   ☐ Pesquisar mudanças em imposto de renda
   ☐ Compilar e salvar relatório
   ```

2. **Delegar** — chama `task()` três vezes em paralelo, cada uma com um
   subagent `general-purpose` rodando `web_search` + `web_fetch` nos
   tópicos separados.

3. **Esperar** os 3 subagents retornarem (até 15–30 min cada).

4. **Compilar** — junta os resultados, escreve o markdown, chama
   `write_file`.

5. **Apresentar** — chama `present_files` para te mostrar
   `tributario.md`.

Durante tudo isso você fica **olhando o progresso no frontend**. Não
precisa dar OK em cada passo — é o que a maioria das pessoas quer dizer
com "agente funcionando sozinho".

---

## Quando o agente para e pede confirmação

O DeerFlow tem um middleware chamado `ClarificationMiddleware` que
intercepta chamadas à tool `ask_clarification`. O agente **só pausa e
espera** você se ele próprio julgar que não pode continuar sem mais
input — por exemplo:

- *"Você quer salvar como PDF ou Markdown?"*
- *"Qual das duas versões do arquivo devo usar?"*
- *"O diretório X não existe — posso criar?"*

Fora disso, ele segue até uma destas condições:

- completar a tarefa
- atingir o limite de recursões (`recursion_limit`, default ~100)
- estourar timeout de subagent (15–30 min)
- `LoopDetectionMiddleware` detectar loop de tool calls

---

## Anexos e uploads

No frontend, você pode anexar arquivos ao chat:

- **Imagens** (PNG, JPG, WebP) — só são enviadas ao modelo se o
  **modelo suporta vision** (`supports_vision: true` no `config.yaml`).
  Nos modelos deste setup: `claude-sonnet`, `claude-opus`, `gemma4-31b`.
- **PDFs**, **Word**, **PowerPoint**, **Excel** — convertidos
  automaticamente para markdown via `markitdown` antes de entrar no
  contexto do agente.
- **Texto** (txt, md, csv, json, yaml, log) — passa direto.

Os arquivos ficam em `backend/.deer-flow/threads/{thread_id}/user-data/uploads/`
do ponto de vista do host e em `/mnt/user-data/uploads/` de dentro do
sandbox.

---

## Memória de longo prazo

Se `memory.enabled: true` no `config.yaml` (que é o caso deste setup), o
agente acumula **fatos sobre você** entre conversas:

- Preferências ("respostas em PT-BR", "sempre cite fontes")
- Contexto de trabalho ("trabalha com direito tributário", "usa Linux")
- Metas em andamento ("está escrevendo um curso sobre IA")

Você pode ver e editar a memória pela UI (há uma página `/workspace/memory`
ou similar) ou direto no arquivo `backend/.deer-flow/memory.json`.

**Cuidado**: memória não é registro literal — é uma síntese que o LLM
extrai das conversas. Ela pode conter interpretações erradas. Revise
periodicamente.

---

## Trocar modelo no meio da conversa

Sim, você pode. Troca o modelo no dropdown, manda a próxima mensagem, e
o novo modelo pega a conversa de onde parou (o histórico é o mesmo). Útil
para:

- Começar com `qwen3-moe` (rápido) para entender o problema
- Trocar para `claude-opus` quando precisar de precisão na resposta final

A memória e os arquivos da thread são compartilhados — nada se perde.

---

## Referência rápida

| Você quer                                  | Use                                                 |
|--------------------------------------------|-----------------------------------------------------|
| Trocar agente na conversa atual            | Dropdown de agente no topo do chat                  |
| Trocar modelo na conversa atual            | Dropdown de modelo no topo do chat                  |
| Agente executar várias etapas sozinho      | **Plan Mode** + **Subagents** ligados               |
| Agente pausar para perguntar algo          | Ele chama `ask_clarification` (automático)          |
| Forçar modelo por agente                   | `model:` no `backend/.deer-flow/agents/{nome}/config.yaml` |
| Mandar uma imagem                          | Anexar no chat (modelo precisa de `supports_vision`) |
| Mandar um PDF / Word / PPT / Excel         | Anexar — conversão para markdown é automática       |
| Ver/editar a memória                       | Página Memory no frontend ou `backend/.deer-flow/memory.json` |

---

## Próximos passos

- Para rodar os agentes sem abrir o navegador (Telegram, API, cron),
  veja [`agentes-autonomos.md`](./agentes-autonomos.md).
- Para criar agentes novos, veja [`criar-agentes.md`](./criar-agentes.md).
- Para habilitar mais modelos (Claude, GPT), veja
  [`modelos-remotos.md`](./modelos-remotos.md).
