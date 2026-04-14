# Resumo — instalação rápida deste setup

Versão curta para reinstalar ou recriar o ambiente em outra máquina.
Para o passo a passo com contexto e troubleshooting, veja
[`instalacao-passo-a-passo.md`](./instalacao-passo-a-passo.md).

**Cenário:** Gateway mode + Ollama local + uso pessoal.

---

## Comandos (do começo ao fim)

```bash
# 1. pré-requisitos — confirmar
docker --version
ollama --version
ss -tlnp | grep 11434        # Ollama precisa estar em 0.0.0.0:11434

# 2. clonar
cd ~/projetos
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow

# 3. baixar modelos no Ollama
ollama pull qwen3.5:35b-a3b
ollama pull qwen-agentic:latest      # se quiser a versão 72B
ollama pull deepseek-r1:14b
ollama pull llama3.2:latest

# 4. configuração
cp config.example.yaml config.yaml   # depois editar (ver seção abaixo)
touch .env                           # vazio — sem API keys neste setup
cp frontend/.env.example frontend/.env

# 5. instalar dependência do Ollama no backend
cd backend && uv add langchain-ollama && cd ..

# 6. verificar
make doctor

# 7. subir
make up-pro

# 8. acessar
# http://localhost:2026
```

---

## Arquivos de configuração — onde estão

| Arquivo                                     | Para que serve                                      |
|---------------------------------------------|-----------------------------------------------------|
| `config.yaml` (raiz)                        | Configuração principal: modelos, tools, sandbox, memória, subagents, checkpointer |
| `.env` (raiz)                               | Secrets / API keys e tracing (LangSmith, Langfuse)  |
| `frontend/.env`                             | Variáveis de ambiente do Next.js (defaults funcionam) |
| `backend/pyproject.toml`                    | Dependências Python do backend (inclui `langchain-ollama`) |
| `backend/uv.lock`                           | Lock file do uv                                     |
| `backend/.deer-flow/memory.json`            | Memória de longo prazo global                       |
| `backend/.deer-flow/USER.md`                | Perfil global — injetado em todos os agentes        |
| `backend/.deer-flow/agents/{nome}/SOUL.md`  | Personalidade de um custom agent                    |
| `backend/.deer-flow/agents/{nome}/config.yaml` | Modelo/skills específicos de um custom agent     |
| `backend/.deer-flow/checkpoints.db`         | Estado das conversas (SQLite)                       |
| `backend/.deer-flow/threads/{id}/`          | Arquivos de cada thread                             |
| `skills/public/`                            | Skills que vêm com o DeerFlow                       |
| `skills/custom/`                            | Skills customizadas (gitignored)                    |
| `docker/docker-compose.yaml`                | Composição dos containers em produção               |

---

## Configurações-chave do `config.yaml`

As seções mais importantes para este cenário:

```yaml
# --- Modelos Ollama ---
models:
  - name: qwen3-moe                        # primeiro = default
    use: langchain_ollama:ChatOllama
    model: qwen3.5:35b-a3b
    base_url: http://host.docker.internal:11434
    num_ctx: 32768
    reasoning: true
    supports_thinking: true
  # ... outros modelos

# --- Sandbox (local dentro do container) ---
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
  allow_host_bash: true

# --- UI de agentes customizados (obrigatório) ---
agents_api:
  enabled: true

# --- Memória e persistência ---
memory:
  enabled: true
  storage_path: memory.json

checkpointer:
  type: sqlite
  connection_string: checkpoints.db

# --- Title e summarization usando modelo rápido ---
title:
  enabled: true
  model_name: llama3.2-fast

summarization:
  enabled: true
  model_name: llama3.2-fast
```

---

## Comandos do dia a dia

```bash
# subir
make up-pro

# parar
make down

# logs ao vivo
docker compose logs -f gateway

# reiniciar só o gateway (após editar config.yaml)
docker restart deer-flow-gateway

# atualizar DeerFlow
git pull && make up-pro

# backup dos dados pessoais
tar czf deerflow-backup-$(date +%F).tgz \
  config.yaml .env \
  backend/.deer-flow \
  skills/custom
```

---

## Modelos remotos opcionais (Codex CLI / Claude Code)

DeerFlow pode usar suas credenciais já autenticadas de **Codex CLI** e
**Claude Code** como provedores adicionais — sem API key no `.env`.
O `docker-compose.yaml` já monta `~/.codex` e `~/.claude` nos containers
como read-only, então é só adicionar os modelos no `config.yaml`:

```yaml
- name: claude-sonnet
  display_name: Claude Sonnet 4.6 (Claude Code)
  use: deerflow.models.claude_provider:ClaudeChatModel
  model: claude-sonnet-4-6
  max_tokens: 16384
  enable_prompt_caching: true
  supports_thinking: true
  supports_vision: true

- name: gpt-codex
  display_name: GPT-5 (Codex CLI)
  use: deerflow.models.openai_codex_provider:CodexChatModel
  model: gpt-5
  reasoning_effort: medium
  supports_thinking: true
```

Reinicie o gateway após editar:

```bash
docker restart deer-flow-gateway
```

Detalhes, troubleshooting e como forçar modelo por custom agent: veja
[`modelos-remotos.md`](./modelos-remotos.md).

---

## Observações importantes

- **Ollama em 0.0.0.0:11434** — se estiver só em `127.0.0.1`, o container
  do gateway não alcança. Configurar via
  `systemctl edit ollama` → `[Service]` → `Environment="OLLAMA_HOST=0.0.0.0:11434"`.

- **`agents_api.enabled: true`** — sem isso, a UI em `/workspace/agents`
  quebra com 403 em loop. Aviso de segurança do template só se aplica se
  você expor o gateway sem auth para terceiros.

- **`host.docker.internal`** — funciona dentro dos containers do DeerFlow
  no Linux porque os compose files já declaram
  `extra_hosts: ["host.docker.internal:host-gateway"]`.

- **`make dev` vs `make up-pro`** — `make dev` roda no host sem Docker
  (bom para dev), `make up-pro` roda em containers Docker (produção).
  Para "no ar", use `make up-pro`.

- **Acesso LAN** — o nginx escuta em `0.0.0.0:2026`, então qualquer
  IP da máquina na LAN alcança. Para acesso de outra máquina na rede,
  use `http://IP-DO-SERVIDOR:2026`.

- **Warning do bash no make doctor** — pode ignorar. O "host" do
  LocalSandboxProvider é o container gateway, isolado pelo Docker.

---

## Onde encontrar mais

- [`instalacao-passo-a-passo.md`](./instalacao-passo-a-passo.md)
  — walk-through completo com troubleshooting
- [`criar-agentes.md`](./criar-agentes.md) — como criar custom agents
- [`modelos-remotos.md`](./modelos-remotos.md)
  — Codex CLI e Claude Code como provedores de LLM
- `docs/CONFIGURATION.md` — referência oficial dos campos do `config.yaml`
  (em `backend/docs/`)
- `docs/ARCHITECTURE.md` — arquitetura interna do DeerFlow
  (em `backend/docs/`)
- `README.md` — documentação upstream do projeto
