# Instalação do DeerFlow — passo a passo

Histórico completo do processo de instalação, com contexto e troubleshooting.
Para um guia curto "só os comandos", veja [`resumo-instalacao.md`](./resumo-instalacao.md).

**Cenário alvo:**
- Gateway mode (sem LangGraph Platform license)
- LLM local via Ollama na máquina host
- Uso pessoal, single-user
- Sem exposição pública / sem auth adicional

---

## 1. Pré-requisitos

### Software

| Ferramenta      | Versão mínima |
|-----------------|---------------|
| Docker + Compose| recente       |
| Python          | 3.12+         |
| Node.js         | 22+ (24+ recomendado) |
| pnpm            | 10+           |
| uv              | 0.10+         |
| nginx           | disponível no host (o make doctor confere) |
| Ollama          | instalado e rodando |

Verifique com:

```bash
docker --version
docker compose version
groups | grep docker       # seu user tem que estar no grupo docker
ollama --version
```

### Hardware

- **16 GB+ RAM** no servidor
- **GPU** com VRAM suficiente pros modelos escolhidos
  (ou memória unificada estilo Grace Blackwell)
- **40 GB+ SSD** livre (imagens Docker + dados)

### Rede

- Ollama escutando em `0.0.0.0:11434` (para que o container do gateway
  consiga alcançar via `host.docker.internal`). Confirme com:

  ```bash
  ss -tlnp | grep 11434
  ```

  Deve aparecer `0.0.0.0:11434`, não só `127.0.0.1`.

---

## 2. Clonar o repositório

```bash
cd ~/projetos
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow
```

---

## 3. Baixar os modelos no Ollama

Antes de configurar o DeerFlow, garanta que os modelos que você vai referenciar
no `config.yaml` já estão disponíveis no Ollama:

```bash
ollama list
```

Os modelos usados no `config.yaml` deste setup:

- `qwen3.5:35b-a3b` — MoE, principal, com thinking
- `qwen-agentic:latest` — heavy-lift 72B
- `deepseek-r1:14b` — reasoning specialist
- `llama3.2:latest` — fast, usado para title/summarization

Se algum não aparecer no `ollama list`, baixe com:

```bash
ollama pull qwen3.5:35b-a3b
ollama pull deepseek-r1:14b
ollama pull llama3.2:latest
# etc.
```

---

## 4. Criar os arquivos de configuração

### 4.1. `config.yaml` (raiz do projeto)

Copie do template e edite, ou use direto o arquivo já preparado neste setup.
Pontos-chave do `config.yaml` para este cenário:

- `models:` — 4 modelos via `langchain_ollama:ChatOllama`, todos com
  `base_url: http://host.docker.internal:11434`
- Primeiro modelo da lista é o **default** (aqui: `qwen3-moe`)
- `reasoning: true` e `supports_thinking: true` nos modelos que suportam
  (qwen3, deepseek-r1)
- `num_ctx: 32768` nos modelos principais (Ollama default é 2048 — sempre
  suba explicitamente)
- `title.model_name: llama3.2-fast` — usa modelo pequeno para títulos
- `summarization.model_name: llama3.2-fast` — idem
- `sandbox.use: deerflow.sandbox.local:LocalSandboxProvider` com
  `allow_host_bash: true` (host = container gateway, então é isolado pelo
  próprio Docker)
- `memory.enabled: true` — long-term memory ligada
- `checkpointer.type: sqlite` — persistência entre restarts
- `agents_api.enabled: true` — **importante**, senão a UI de agentes quebra
  (ver seção de troubleshooting)

### 4.2. `.env` (raiz do projeto)

Pode ficar vazio — neste setup não há API keys de LLM porque Ollama é local.
Só crie o arquivo para o `make doctor` não reclamar:

```bash
touch .env
```

Se depois quiser ligar observabilidade, adicione LangSmith ou Langfuse aqui.

### 4.3. `frontend/.env`

Copiar do template:

```bash
cp frontend/.env.example frontend/.env
```

Todas as variáveis do template são comentadas — os defaults (via nginx proxy)
funcionam para este cenário.

---

## 5. Instalar dependência do Ollama no backend

O DeerFlow não traz `langchain-ollama` por padrão. Adicione:

```bash
cd backend
uv add langchain-ollama
cd ..
```

Isso atualiza `backend/pyproject.toml` e `backend/uv.lock`. O build Docker
subsequente já vai incluir o pacote na imagem.

**Nota:** essa instalação vai para a venv local do host, o que satisfaz o
`make doctor`. A imagem Docker é buildada do zero a partir do `pyproject.toml`
quando você roda `make up-pro`, então também pega a dependência.

---

## 6. Verificar o setup com `make doctor`

```bash
make doctor
```

Esperado:

```
System Requirements
  ✓ Python, Node.js, pnpm, uv, nginx

Configuration
  ✓ .env found
  ✓ frontend/.env found
  ✓ config.yaml found
  ✓ config.yaml version (v7)
  ✓ config.yaml loadable
  ✓ models configured (4 model(s))

LLM Provider
  ✓ langchain-ollama installed

Web Capabilities
  ✓ web search configured (DuckDuckGo)
  ✓ web fetch configured (Jina AI Reader)

Sandbox
  ✓ sandbox configured (Local sandbox)
  ! bash compatibility (host bash enabled on LocalSandboxProvider)
      → Use container sandbox for stronger isolation when bash is required

Status: Ready
```

O warning do bash pode ser ignorado neste cenário: o "host" do
LocalSandboxProvider é o container gateway, isolado pelo próprio Docker.

---

## 7. Subir em produção (Gateway mode)

```bash
make up-pro
```

Isto:

1. Builda as imagens Docker (5–15 min na primeira vez)
2. Sobe 3 containers:
   - `deer-flow-frontend` (Next.js production server, porta interna 3000)
   - `deer-flow-gateway` (FastAPI + agent runtime embutido, porta 8001)
   - `deer-flow-nginx` (reverse proxy, porta **2026** — única exposta ao host)

Acompanhe os logs em outro terminal:

```bash
docker compose logs -f gateway
```

Quando aparecer `Application startup complete`, o backend está pronto.

---

## 8. Testar o acesso

Local:

```bash
curl http://localhost:2026/api/models
```

Deve retornar JSON com os 4 modelos configurados.

No navegador: **http://localhost:2026**

Da LAN (a partir de outra máquina): **http://IP-DO-SERVIDOR:2026**
(o nginx está em `0.0.0.0:2026`, então qualquer IP da máquina funciona.)

---

## 9. Troubleshooting — erros que aconteceram

### 9.1. `Error: langchain-ollama not installed` no `make doctor`

**Causa:** DeerFlow não traz a lib do Ollama por padrão.

**Fix:**
```bash
cd backend && uv add langchain-ollama
```

### 9.2. `frontend/.env found` com `!` (warning)

**Causa:** a mensagem do doctor é enganosa — o `!` significa **não encontrado**.

**Fix:**
```bash
cp frontend/.env.example frontend/.env
```

### 9.3. UI de agentes quebra com "client-side exception"

**Sintoma:** ao abrir `/workspace/agents` no frontend, página quebra
com erro de cliente. Logs do nginx mostram loop infinito de
`GET /api/agents HTTP/1.1" 403`.

**Causa:** `agents_api.enabled` estava `false` no `config.yaml`. O frontend
tenta listar agentes, toma 403, entra em retry e estoura exceção no React.

**Fix:** editar `config.yaml`:

```yaml
agents_api:
  enabled: true
```

Reiniciar o gateway:

```bash
docker restart deer-flow-gateway
```

Depois um **hard refresh** no navegador (`Ctrl+Shift+R`) para limpar o estado
quebrado do React.

### 9.4. Gateway não conecta no Ollama

**Sintoma:** chat dá erro de "connection refused" ou "model not found".

**Diagnóstico:** testar de dentro do container:

```bash
docker exec deer-flow-gateway curl -s http://host.docker.internal:11434/api/version
```

Se falhar:
- Garanta que `OLLAMA_HOST=0.0.0.0:11434` está setado
  (`systemctl edit ollama` e adicione na seção `[Service]`)
- Ou use o IP da bridge docker: `ip addr show docker0` → troca
  `host.docker.internal` pelo IP (tipo `172.17.0.1`) no `config.yaml`

### 9.5. Docker compose restart falha com `empty section between colons`

**Causa:** o `docker compose restart` pede variáveis de ambiente
(`DEER_FLOW_HOME`, `DEER_FLOW_REPO_ROOT`) que o `make up-pro` define em runtime.

**Fix:** não use `docker compose restart`; use `docker restart` direto:

```bash
docker restart deer-flow-gateway
```

---

## 10. Operação do dia a dia

```bash
make up-pro           # subir
make down             # parar e remover containers
docker compose logs -f gateway        # logs ao vivo
docker restart deer-flow-gateway      # reiniciar só o gateway (após edit de config.yaml)
git pull && make up-pro               # atualizar DeerFlow
```

---

## 11. Backup — o que preservar

| Caminho                        | O que é                                 |
|--------------------------------|------------------------------------------|
| `config.yaml`                  | Config global                            |
| `.env`                         | Secrets (vazio neste setup)              |
| `backend/.deer-flow/memory.json` | Memória de longo prazo do usuário     |
| `backend/.deer-flow/USER.md`   | Perfil global injetado em todos os agentes |
| `backend/.deer-flow/agents/`   | Todos os custom agents (SOUL.md + config) |
| `backend/.deer-flow/threads/`  | Arquivos e outputs de cada conversa      |
| `backend/.deer-flow/checkpoints.db` | Estado das conversas (SQLite)       |
| `skills/custom/`               | Skills customizadas                      |

Backup rápido:

```bash
tar czf deerflow-backup-$(date +%F).tgz \
  config.yaml .env \
  backend/.deer-flow \
  skills/custom
```

---

## 12. (Opcional) Habilitar Claude Code e Codex CLI como LLMs

O DeerFlow pode usar as suas credenciais já autenticadas de **Codex CLI** e
**Claude Code** como provedores adicionais de LLM, sem API key separada no
`.env`. Útil para ter modelos de altíssima qualidade disponíveis em paralelo
ao Ollama local.

O `docker/docker-compose.yaml` já monta `~/.codex` e `~/.claude` nos containers
como read-only — então é só:

1. Garantir que as CLIs estão autenticadas no host
   (`~/.codex/auth.json` e `~/.claude/.credentials.json` existem)
2. Adicionar os blocos `models:` correspondentes no `config.yaml`
   (ver `modelos-remotos.md` para exemplos prontos)
3. Reiniciar o gateway: `docker restart deer-flow-gateway`

Depois o frontend lista os novos modelos no dropdown, e você pode forçar
modelos específicos por **custom agent** via o `config.yaml` per-agent
(ex: um agente `code-reviewer` sempre usa Claude Opus).

Detalhes completos, troubleshooting e tradeoffs: veja
[`modelos-remotos.md`](./modelos-remotos.md).

---

## 13. Próximos passos sugeridos

- **Criar custom agents** — veja [`criar-agentes.md`](./criar-agentes.md)
- **Habilitar Claude/GPT via CLIs** — veja [`modelos-remotos.md`](./modelos-remotos.md)
- **Ligar Langfuse** para observabilidade (self-hosted, 2 containers extras)
- **Plugar um canal IM** (Telegram é o mais simples — só precisa de bot token)
- **Trocar sandbox para Docker isolado** por thread
  (`deerflow.community.aio_sandbox:AioSandboxProvider`)
- **Adicionar Tavily** para web search de qualidade
  (tier grátis de 1000 calls/mês)
