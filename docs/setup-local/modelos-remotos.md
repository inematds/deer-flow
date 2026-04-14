# Modelos remotos via Codex CLI e Claude Code

O DeerFlow pode usar suas credenciais já autenticadas de **Codex CLI** e
**Claude Code** como providers de LLM, sem precisar de API key separada no
`.env`. Isso roda **em paralelo** aos modelos locais do Ollama — você mantém
Ollama como default e usa os remotos quando precisar de mais qualidade.

---

## Quando usar cada um

| | Ollama local | Codex / Claude Code |
|---|---|---|
| Qualidade | Boa (qwen3.5 MoE, 70B dense) | Muito superior em planejamento e tool-use |
| Velocidade | Depende da GPU | Tipicamente mais rápido que 70B local |
| Custo | Zero | Consome quota da assinatura ChatGPT / Claude |
| Privacidade | 100% local | Sai para OpenAI / Anthropic |
| Internet | Não precisa | Precisa |
| Rate limits | Só da GPU | Limites por dia/semana da conta |

**Recomendação**: mantenha `qwen3-moe` (Ollama) como default e habilite
Codex/Claude como opções secundárias para tarefas onde o salto de qualidade
vale o custo. Troque no dropdown do frontend ou force por **custom agent**.

---

## Pré-requisitos

- **Codex CLI** instalado e autenticado no host
  ```bash
  codex auth login
  ls ~/.codex/auth.json   # confirma que existe
  ```

- **Claude Code** instalado e autenticado no host
  ```bash
  claude setup-token   # ou o método de login apropriado
  ls ~/.claude/.credentials.json   # confirma que existe
  ```

---

## Como o DeerFlow encontra as credenciais

**Você não precisa configurar montagem de volume** — o
`docker/docker-compose.yaml` já declara:

```yaml
volumes:
  - type: bind
    source: ${HOME}/.claude
    target: /root/.claude
    read_only: true
  - type: bind
    source: ${HOME}/.codex
    target: /root/.codex
    read_only: true
```

Ou seja, os diretórios `.claude` e `.codex` do seu home viram read-only
dentro dos containers `deer-flow-gateway` e `deer-flow-langgraph`. Os
providers auto-detectam as credenciais nesses caminhos.

Prioridade de fallback do `ClaudeChatModel`:

1. `ANTHROPIC_API_KEY` (env var no `.env`)
2. `CLAUDE_CODE_OAUTH_TOKEN` ou `ANTHROPIC_AUTH_TOKEN` (env var)
3. `CLAUDE_CODE_CREDENTIALS_PATH` (env var apontando para um arquivo)
4. `~/.claude/.credentials.json` (auto)

O `CodexChatModel` lê de `~/.codex/auth.json` direto.

---

## Configuração no `config.yaml`

Adicione os modelos desejados na seção `models:`. Exemplos funcionais:

### Claude Sonnet 4.6 via Claude Code

```yaml
- name: claude-sonnet
  display_name: Claude Sonnet 4.6 (Claude Code)
  use: deerflow.models.claude_provider:ClaudeChatModel
  model: claude-sonnet-4-6
  max_tokens: 16384
  enable_prompt_caching: true     # cache reduz custo em conversas longas
  supports_thinking: true
  supports_vision: true
```

### Claude Opus 4.6 via Claude Code

```yaml
- name: claude-opus
  display_name: Claude Opus 4.6 (Claude Code)
  use: deerflow.models.claude_provider:ClaudeChatModel
  model: claude-opus-4-6
  max_tokens: 16384
  enable_prompt_caching: true
  supports_thinking: true
  supports_vision: true
```

### GPT-5 via Codex CLI

```yaml
- name: gpt-codex
  display_name: GPT-5 (Codex CLI)
  use: deerflow.models.openai_codex_provider:CodexChatModel
  model: gpt-5
  reasoning_effort: medium        # low / medium / high
  supports_thinking: true
```

Depois de editar, reinicie o gateway para recarregar a configuração:

```bash
docker restart deer-flow-gateway
```

Confirme que os modelos aparecem na API:

```bash
curl -s http://localhost:2026/api/models | python3 -m json.tool
```

---

## Dica: rotear por custom agent

Combine com **custom agents** para forçar um modelo específico por persona,
sem ter que lembrar de trocar o dropdown:

```yaml
# backend/.deer-flow/agents/code-reviewer/config.yaml
model: claude-opus     # este agente sempre usa Opus
```

```yaml
# backend/.deer-flow/agents/pesquisador-offline/config.yaml
model: qwen-agentic-72b    # este agente sempre roda local
```

```yaml
# backend/.deer-flow/agents/analista-rapido/config.yaml
model: claude-sonnet       # equilibrio qualidade/custo
```

Assim você controla custo e qualidade **por tipo de tarefa**, não por
clique de dropdown.

---

## Title e summarization continuam locais

O `config.yaml` deste setup aponta `title.model_name` e
`summarization.model_name` para `llama3.2-fast` (Ollama). Isso significa:

- Tarefas administrativas (gerar título da conversa, resumir histórico
  quando o contexto estoura) **não consomem quota** de Claude / Codex
- Só a resposta principal usa o modelo que você selecionou no dropdown

Esse design mantém o custo baixo mesmo quando você usa Claude Opus no chat.

---

## Troubleshooting

### `Authentication failed` / `invalid token`

Tokens expiraram ou foram rotacionados no host. Re-autentique:

```bash
claude setup-token       # ou método equivalente do Claude Code
codex auth login         # Codex CLI
```

E reinicie o gateway:

```bash
docker restart deer-flow-gateway
```

### `billing header` error no Claude

Esporádico — acontece quando a Anthropic atualiza o header esperado.
O `claude_provider.py` traz um default atualizado, mas você pode forçar
via env:

```bash
# .env
ANTHROPIC_BILLING_HEADER=x-anthropic-billing-header: cc_version=...
```

### `model not found` no GPT-5 (Codex)

O endpoint do Codex CLI só aceita modelos que a sua conta tem acesso.
Se `gpt-5` não funcionar, teste:

- `gpt-5-codex`
- `gpt-5.4`
- `o3-mini`

Edite o campo `model:` no bloco `gpt-codex` do `config.yaml` e reinicie
o gateway.

### Modelo aparece no dropdown mas falha ao enviar mensagem

Checar os logs do gateway:

```bash
docker logs deer-flow-gateway --tail 50
```

Quase sempre é credencial expirada ou problema de DNS/rede saindo do
container para OpenAI / Anthropic.

### Testar conectividade de dentro do container

```bash
docker exec deer-flow-gateway curl -sI https://api.anthropic.com | head
docker exec deer-flow-gateway curl -sI https://chatgpt.com | head
```

Se falhar, o container não está alcançando a internet — checar rede
Docker e DNS.

---

## Referência de credenciais do host

| Arquivo do host            | Montado no container como | Usado por       |
|----------------------------|---------------------------|-----------------|
| `~/.codex/auth.json`       | `/root/.codex/auth.json`  | `CodexChatModel` |
| `~/.claude/.credentials.json` | `/root/.claude/.credentials.json` | `ClaudeChatModel` |

Os dois são montados **read-only** — o DeerFlow nunca escreve nesses
arquivos, só lê.
