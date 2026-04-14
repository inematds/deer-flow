# Agentes autônomos — rodar sozinhos sem abrir o navegador

Este documento cobre as formas de fazer os agentes do DeerFlow **rodarem
por conta própria**, sem você estar na frente da UI. Para o uso manual
do dia a dia, veja [`usar-agentes.md`](./usar-agentes.md).

---

## "Sozinho" tem dois significados

Antes de qualquer coisa, vale separar dois eixos distintos:

1. **Autonomia dentro de uma conversa** — você dá uma tarefa e o agente
   executa vários passos sem você guiar cada um. Isso está coberto em
   [`usar-agentes.md`](./usar-agentes.md) na seção de Plan Mode + Subagents.

2. **Ser acionado sozinho** — o agente roda sem você abrir o navegador
   (cron, webhook, canal de mensageria, chamada de API). Este documento
   trata **deste** caso.

---

## Caminhos disponíveis

O DeerFlow **não tem scheduler nativo** (não existe um dashboard de
agendamentos dentro dele), mas oferece quatro pontos de entrada para
disparar agentes externamente:

| Caminho                 | Para que serve                                        | Complexidade |
|-------------------------|-------------------------------------------------------|--------------|
| **Canal IM**            | Conversar pelo celular / outra máquina em tempo real  | Baixa        |
| **REST API**            | Cron, webhook, GitHub Actions, n8n, Zapier            | Baixa        |
| **Python Client**       | Embutir em outro serviço Python (ETL, filas, bots)    | Média        |
| **Skill evolution**     | Agente que cria e modifica suas próprias skills       | Alta / experimental |

---

## 1. Canais IM — conversar pelo celular

O caminho mais fácil para "agente trabalhando sozinho enquanto você faz
outra coisa". Canais suportados:

- **Telegram** — o mais simples, só precisa de bot token via `@BotFather`
- **Slack** — Socket Mode
- **Feishu / Lark**
- **WeChat** (via iLink)
- **WeCom**

### Como funciona

1. Você habilita o canal no `config.yaml` com as credenciais do bot
2. O gateway do DeerFlow inicia um cliente que fica escutando mensagens
3. Qualquer mensagem que o bot receber vira uma conversa no DeerFlow
4. O agente responde diretamente no chat do Telegram / Slack / etc.

### Comandos built-in (funcionam em qualquer canal)

| Comando    | O que faz                       |
|------------|---------------------------------|
| `/new`     | Começa uma conversa nova        |
| `/status`  | Info da thread atual            |
| `/models`  | Lista os modelos disponíveis    |
| `/memory`  | Mostra a memória do agente      |
| `/help`    | Ajuda                           |

Mensagens sem prefixo são tratadas como chat normal.

### Amarrar um canal a um agente e modo específicos

Exemplo para Telegram forçando o agente `pesquisador` com Plan Mode e
subagents sempre ligados:

```yaml
channels:
  langgraph_url: http://langgraph:2024
  gateway_url: http://gateway:8001

  session:
    assistant_id: lead_agent
    context:
      thinking_enabled: true
      is_plan_mode: true        # plan mode sempre ligado
      subagent_enabled: true    # subagents sempre ligados

  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: ["123456789"]    # trave no seu user_id
    session:
      assistant_id: pesquisador     # este canal sempre usa "pesquisador"
```

Token do bot no `.env`:

```bash
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
```

### Passo a passo: Telegram (mais rápido)

1. Abra um chat com `@BotFather` no Telegram
2. `/newbot` → dê um nome e um username
3. Copie o token retornado
4. Adicione no `.env`:
   ```bash
   TELEGRAM_BOT_TOKEN=<seu token>
   ```
5. Edite o `config.yaml` → seção `channels.telegram.enabled: true`
6. Para descobrir seu `user_id` do Telegram, mande `/start` para
   `@userinfobot` e cole o número em `allowed_users`
7. Reinicie o gateway:
   ```bash
   docker restart deer-flow-gateway
   ```
8. Mande uma mensagem para o seu bot — ele deve responder

### Por que amarrar a um custom agent?

Porque o contexto no celular costuma ser diferente do contexto na mesa.
Exemplo:

- Canal **Telegram** → agente `briefings` (respostas curtas, formato
  bullet, sem código, otimizado para leitura em tela pequena)
- Canal **Slack do time** → agente `team-assistant` (respostas em PT-BR,
  sem jargão, educado)
- Frontend web na mesa → agente `coder` / `pesquisador` (respostas
  longas, com blocos de código e relatórios completos)

---

## 2. REST API — cron, webhooks, integração

O gateway expõe rotas que criam um run sem precisar de UI.

### Endpoints

- `POST /api/runs/stream` — cria um run e devolve SSE (Server-Sent Events)
- `POST /api/runs/wait` — cria um run e bloqueia até terminar, retornando
  o resultado final
- `POST /api/runs/cancel/{run_id}` — cancela um run em andamento

### Estrutura do body

```json
{
  "input": {
    "messages": [
      {"role": "user", "content": "faça X"}
    ]
  },
  "config": {
    "configurable": {
      "thread_id": "meu-thread-recorrente",
      "assistant_id": "pesquisador",
      "is_plan_mode": true,
      "subagent_enabled": true,
      "thinking_enabled": true,
      "model_name": "claude-opus"
    }
  }
}
```

Campos relevantes do `configurable`:

| Campo                 | Tipo   | Função                                         |
|-----------------------|--------|------------------------------------------------|
| `thread_id`           | string | Se omitido, cria thread nova. Se igual entre chamadas, preserva memória da conversa |
| `assistant_id`        | string | Nome do agente (`lead_agent` ou custom agent)  |
| `model_name`          | string | Força um modelo específico                     |
| `is_plan_mode`        | bool   | Liga Plan Mode (TodoList)                      |
| `subagent_enabled`    | bool   | Liga delegação para subagents                  |
| `thinking_enabled`    | bool   | Liga thinking explícito do modelo              |
| `recursion_limit`     | int    | Limite de recursões (default ~100)             |

### Exemplo: `curl` blocking

```bash
curl -X POST http://localhost:2026/api/runs/wait \
  -H "Content-Type: application/json" \
  -d '{
    "input": {"messages": [{"role": "user", "content": "Liste os 5 arquivos mais recentes em /mnt/user-data/workspace"}]},
    "config": {
      "configurable": {
        "thread_id": "teste-api-001",
        "assistant_id": "lead_agent",
        "is_plan_mode": false,
        "subagent_enabled": false
      }
    }
  }'
```

### Exemplo: cron para briefing diário

```cron
# crontab -e
# Todo dia às 7h: gerar briefing usando o agente "pesquisador"
0 7 * * * curl -s -X POST http://localhost:2026/api/runs/wait \
  -H "Content-Type: application/json" \
  -d '{"input":{"messages":[{"role":"user","content":"Gere o briefing diário: principais notícias de IA nas últimas 24h, com links"}]},"config":{"configurable":{"thread_id":"briefing-diario","assistant_id":"pesquisador","is_plan_mode":true,"subagent_enabled":true}}}' \
  > /tmp/briefing-$(date +\%Y\%m\%d).json
```

Depois outro script lê o JSON, extrai o markdown e te manda por email
/ Telegram / S3 / onde quiser.

### Como usar `thread_id` fixo a favor

Se o `thread_id` for o mesmo entre chamadas, a **memória e o histórico da
conversa são preservados**. Você pode ter:

- Thread `briefing-diario` — acumula contexto ao longo do tempo e
  lembra o que já cobriu
- Thread `monitor-servidor` — acumula informações sobre incidentes e
  padrões do servidor
- Thread `ideias-projeto-X` — um "caderno" temático que o agente mantém

Como o DeerFlow tem sumarização automática, thread muito longa não
estoura contexto — ela é resumida periodicamente pelo `llama3.2-fast`
(configurado como `summarization.model_name`) sem consumir quota dos
modelos caros.

### Triggers externos possíveis

| Origem                | Como conecta no DeerFlow              |
|-----------------------|---------------------------------------|
| Cron do Linux         | `crontab -e` + `curl` para `/api/runs/wait` |
| Systemd timer         | Service file disparando um script     |
| Kubernetes CronJob    | Mesmo padrão em cluster               |
| GitHub Actions        | Workflow `on: schedule` + `curl` ou action http |
| n8n                   | HTTP Request node                     |
| Zapier / Make         | Webhook de saída                      |
| Monitor de arquivo    | Script com `inotifywait` + `curl`     |
| Fila (RabbitMQ, SQS)  | Worker consumindo a fila e chamando a API |

---

## 3. Python Client embutido

Se você está escrevendo um serviço Python e quer chamar o DeerFlow como
**biblioteca** (sem HTTP):

```python
from deerflow.client import DeerFlowClient

client = DeerFlowClient()

# Chat simples
response = client.chat(
    "Analise este CSV e me dê insights",
    thread_id="job-analise-42",
)
print(response)

# Streaming
for event in client.stream(
    "Faça X",
    thread_id="job-analise-42",
):
    if event.type == "messages-tuple":
        print(event.content, end="", flush=True)
```

Útil para:

- Processar eventos Kafka / RabbitMQ e disparar agentes
- Rodar como parte de um pipeline de ETL
- Embutir num bot Discord / WhatsApp custom (que não é canal nativo)

**Requer**: rodar dentro do ambiente Python do DeerFlow (precisa do
`config.yaml` acessível e das deps instaladas). Não dá pra usar de fora
do repositório sem construir um pacote próprio.

---

## 4. Skill evolution (experimental)

Flag no `config.yaml`:

```yaml
skill_evolution:
  enabled: true
  moderation_model_name: null   # null = usa o modelo default
```

Quando ligada, o agente pode **criar e modificar skills** em
`skills/custom/` por conta própria. É o nível mais alto de autonomia —
o agente aprende e amplia suas próprias capacidades ao longo do tempo.

**Por que fica desligado por padrão**: um agente com acesso a escrever
seus próprios prompts pode ficar estranho rápido. Um prompt ruim que ele
mesmo escreveu pode levar ele a escrever prompts ainda piores. Tem um
modelo de moderação opcional (`moderation_model_name`) que escaneia
skills antes de gravar, mas ainda é experimental.

**Recomendação**: se quiser brincar, ligue numa sessão isolada (talvez
até num clone do repositório), não em produção.

---

## Limites reais do "sozinho"

Sendo honesto sobre o que o DeerFlow **não** faz:

### Não tem scheduler nativo

Você precisa de cron / systemd / Kubernetes CronJob externos. É
intencional: deixa a responsabilidade onde o Linux já faz bem.

### Não tem triggers de evento nativos

Se você quer "quando chegar um email X, dispare o agente Y", precisa
escrever o script que escuta emails e chama a API. Exceção: **canais IM**
são triggers de mensagem prontos.

### Recursion limit e timeouts são reais

- **Recursion limit** default ~100 (configurável por run). Proteção
  contra custo descontrolado.
- **Subagent timeout** default 900s (15 min) para `general-purpose`,
  300s (5 min) para `bash`. No `config.yaml` deste setup subimos
  `general-purpose` para 1800s.
- **Circuit breaker** do LLM: depois de 5 falhas consecutivas, o provider
  é marcado como morto por 60s.

Ou seja, "sozinho por 10 minutos" é normal. "Sozinho por 6 horas" exige
quebrar em múltiplos runs ou subir os limites.

---

## Receita prática em 3 fases

### Fase 1 — Uso manual (imediato)

1. Crie 2–3 custom agents via UI (`pesquisador`, `coder`, `escritor`)
   — veja [`criar-agentes.md`](./criar-agentes.md)
2. Na aba de chat, **sempre ligue Plan Mode + Subagents** para tarefas
   não-triviais
3. Use `claude-opus` quando quiser qualidade máxima, `qwen3-moe` (Ollama)
   no dia a dia

### Fase 2 — Telegram bot (1h de trabalho)

1. Crie um bot no `@BotFather`
2. Configure token no `.env`, habilite `channels.telegram` no `config.yaml`
3. Trave `allowed_users`
4. Amarre o canal ao agente desejado
5. Reinicie o gateway
6. Converse pelo celular

### Fase 3 — Jobs recorrentes via cron (opcional)

Para tarefas diárias (briefing de mercado, review de PRs, monitoramento),
crie um cron chamando `/api/runs/wait` com um `thread_id` fixo e um prompt
parametrizado. Saída em `/tmp/` ou encaminhada por script.

---

## Próximos passos

- [`criar-agentes.md`](./criar-agentes.md) — como criar custom agents
- [`usar-agentes.md`](./usar-agentes.md) — uso manual no dia a dia
- [`modelos-remotos.md`](./modelos-remotos.md) — habilitar Claude / GPT
- [`instalacao-passo-a-passo.md`](./instalacao-passo-a-passo.md) — setup completo
