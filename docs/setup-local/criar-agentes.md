# Como criar agentes no DeerFlow

DeerFlow tem **três conceitos diferentes** que se chamam "agent". Na prática, o
que você vai querer criar em 99% dos casos é um **custom agent**. Este
documento explica os três e mostra como criar cada um.

---

## 1. Lead Agent — o agente principal

É o agente que sempre roda quando você começa uma conversa. Você não "cria" —
você **configura**. Toda a parte do `config.yaml` na raiz do projeto
(modelos, tools, middleware, memória, summarization) define como ele se
comporta.

Para mudar o "espírito" global do lead agent (instruções que valem para
qualquer agente), edite:

```
backend/.deer-flow/USER.md
```

Esse arquivo é injetado no system prompt de **todos** os agentes
(lead + custom). Serve para coisas tipo:

> *"Eu sou o Nelson, trabalho com X, prefiro respostas em PT-BR,
> sempre cite fontes quando fizer pesquisa."*

---

## 2. Custom Agent — o que você cria 99% das vezes

É um agente com **personalidade própria** (SOUL.md), modelo opcional, skills
restritas e prompt customizado. Continua rodando dentro do lead agent do
DeerFlow — você não está construindo um agente do zero, está criando uma
**persona especializada**.

Cada custom agent vive em:

```
backend/.deer-flow/agents/{nome-do-agente}/
├── config.yaml       # modelo, skills permitidas, tools
├── SOUL.md           # personalidade / identidade
├── USER.md           # opcional — descrição pro usuário
└── memory.json       # memória per-agent
```

### Pré-requisito: `agents_api` habilitado

Para usar a UI web (`/workspace/agents`), a API de agentes precisa estar ligada
no `config.yaml`:

```yaml
agents_api:
  enabled: true
```

Se estiver `false`, o frontend toma 403 em loop ao tentar listar agentes e a
UI quebra com "client-side exception". Depois de mudar, reinicie o gateway:

```bash
docker restart deer-flow-gateway
```

### Jeito 1 — Via UI (mais fácil)

1. Abra o frontend do DeerFlow no navegador
2. Vá em **`/workspace/agents`**
3. Clique em **New Agent**
4. Preencha:
   - **Nome** — identificador único, minúsculo, sem espaços
     (ex: `pesquisador-legal`, `code-reviewer`, `escritor-ptbr`)
   - **SOUL.md** — a "alma" do agente: quem é, como age, quais as regras.
     Exemplo completo abaixo
   - **Modelo** (opcional) — força um modelo específico
     (ex: `qwen-agentic-72b` pra tarefas pesadas). Deixe em branco pra
     herdar o modelo da conversa
   - **Skills** (opcional) — lista de skills permitidas. Vazio = todas
     as skills habilitadas globalmente
5. Salve

Depois, ao começar uma conversa, selecione o agente no dropdown — o SOUL.md
dele é injetado no system prompt junto com o lead_agent prompt.

### Jeito 2 — Na mão, via filesystem

Útil para versionar no git (backup) ou automatizar. Crie os arquivos
diretamente:

```bash
mkdir -p backend/.deer-flow/agents/meu-agente
```

Exemplo de `SOUL.md`:

```markdown
# Pesquisador Legal

Você é um assistente jurídico especializado em direito brasileiro.

## Valores
- Sempre cite fontes (lei, artigo, decisão judicial)
- Nunca invente jurisprudência
- Quando não souber, diga "não sei" e sugira onde procurar

## Estilo
- Respostas objetivas em PT-BR
- Linguagem formal mas acessível
- Estrutura: Conclusão → Fundamentação → Fontes
```

Exemplo de `config.yaml` do agente (opcional — tudo é opcional):

```yaml
# backend/.deer-flow/agents/meu-agente/config.yaml
model: qwen-agentic-72b        # força modelo específico (senão herda)
skills:                         # null = todas / [] = nenhuma / lista = só essas
  - deep-research
  - systematic-literature-review
```

Depois reinicie o gateway para ele ler o novo diretório:

```bash
docker restart deer-flow-gateway
```

### Jeito 3 — Pedindo pro próprio agente criar

O DeerFlow tem uma tool built-in chamada `setup_agent_tool`. Você pode pedir
ao lead agent em linguagem natural:

> *"Cria um novo agente chamado `pesquisador-legal` com SOUL focado em
> direito brasileiro, que sempre cite fontes e nunca invente
> jurisprudência."*

Ele vai gerar o SOUL.md e salvar via tool automaticamente.

---

## 3. Subagent — tarefas delegadas em runtime

Não são agentes "que você cria" — são workers que o lead_agent invoca no meio
de uma conversa complexa via a tool `task(description, prompt, subagent_type)`.

DeerFlow vem com **2 built-in**:

| Nome              | Descrição                                       |
|-------------------|-------------------------------------------------|
| `general-purpose` | Tem todas as tools (menos `task`), faz qualquer coisa |
| `bash`            | Especialista em comandos shell                  |

Ativa com `subagent_enabled: true` no runtime config (ou implicitamente via
plan mode). Limite de **3 subagents concorrentes**, timeout de **15 minutos**
por padrão (configurável em `config.yaml` → `subagents`).

**Criar novos subagents** exige mexer no código do harness em
`packages/harness/deerflow/subagents/builtins/` — não dá pela UI. Só faça isso
se quiser ampliar a capacidade do harness. Para 99% dos casos, os 2 built-in
bastam e você customiza o comportamento via **custom agent + skill**.

---

## Recomendação: por onde começar

Crie **2–3 custom agents** via UI, cada um com SOUL.md bem específico:

1. **`pesquisador`** — SOUL focado em research
   - Citar fontes, usar `deep-research` skill, estrutura Intro/Método/Resultados

2. **`coder`** — SOUL focado em engenharia de software
   - Testes primeiro, perguntar antes de mudar arquitetura
   - Força modelo `qwen-agentic-72b` (ou `claude-opus` se quiser qualidade máxima)

3. **`escritor`** — SOUL focado em redação PT-BR
   - Estilo conversacional, evitar jargão
   - Força modelo `qwen3-moe` (rápido, local)

Depois experimente trocar entre eles no dropdown de agente do workspace e veja
qual comportamento se encaixa melhor pro seu uso.

---

## Forçar modelo específico por agente

O `config.yaml` per-agente aceita um campo `model:` que sobrepõe o dropdown.
Isso é útil para rotear tarefas por custo/qualidade sem precisar lembrar de
trocar o modelo manualmente:

```yaml
# backend/.deer-flow/agents/code-reviewer/config.yaml
model: claude-opus          # tarefas críticas, qualidade máxima
```

```yaml
# backend/.deer-flow/agents/pesquisador-offline/config.yaml
model: qwen-agentic-72b     # sempre local, zero custo externo
```

```yaml
# backend/.deer-flow/agents/analista/config.yaml
model: claude-sonnet        # equilíbrio custo/qualidade
```

O nome usado no `model:` deve bater com o `name:` declarado na seção `models:`
do `config.yaml` principal. Para habilitar os provedores baseados em
Codex CLI e Claude Code, veja
[`modelos-remotos.md`](./modelos-remotos.md).

---

## Dicas de SOUL.md

O SOUL.md é o prompt de sistema do agente. O que funciona bem:

- **Uma identidade clara** nas primeiras linhas ("Você é um X especializado em Y")
- **Regras negativas explícitas** ("Nunca faça Z", "Não use W")
- **Formato de resposta esperado** (lista, markdown, sem código, etc.)
- **Exemplos** de respostas boas e ruins, se o comportamento for sutil

O que **não** ajuda:

- Listas enormes de capacidades genéricas ("você pode fazer X, Y, Z...")
- Instruções contraditórias (o modelo vai escolher uma aleatoriamente)
- Repetir coisas que já estão no lead prompt do DeerFlow

---

## Referência de arquivos

| Arquivo                                    | Serve para                              |
|--------------------------------------------|-----------------------------------------|
| `config.yaml` (raiz)                       | Configuração global (modelos, tools, etc.) |
| `config.yaml` → `agents_api.enabled`       | Liga/desliga API de agentes (precisa ser `true`) |
| `backend/.deer-flow/USER.md`               | Perfil global — injetado em todos os agentes |
| `backend/.deer-flow/agents/{nome}/SOUL.md` | Personalidade do custom agent           |
| `backend/.deer-flow/agents/{nome}/config.yaml` | Modelo/skills específicos do agente |
| `backend/.deer-flow/agents/{nome}/memory.json` | Memória per-agent                    |
