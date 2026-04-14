# Setup local — documentação pessoal

Docs desta instalação do DeerFlow (Gateway mode + Ollama local + uso pessoal).

| Arquivo                                                        | O que tem                                                       |
|----------------------------------------------------------------|-----------------------------------------------------------------|
| [`resumo-instalacao.md`](./resumo-instalacao.md)               | Versão curta — comandos do começo ao fim, arquivos de config, comandos do dia a dia |
| [`instalacao-passo-a-passo.md`](./instalacao-passo-a-passo.md) | Walk-through completo com contexto, troubleshooting de todos os erros encontrados |
| [`criar-agentes.md`](./criar-agentes.md)                       | Como criar custom agents (UI, filesystem, via lead agent), estrutura do SOUL.md, recomendações |
| [`usar-agentes.md`](./usar-agentes.md)                         | Uso manual dos agentes: seletor, toggles (Plan Mode / Subagents / Thinking), uploads, memória |
| [`agentes-autonomos.md`](./agentes-autonomos.md)               | Rodar agentes sem abrir o navegador: canais IM, REST API, cron, Python client |
| [`modelos-remotos.md`](./modelos-remotos.md)                   | Usar Codex CLI e Claude Code como provedores de LLM (credenciais auto-detectadas) |

**Por onde começar:**
- Reinstalar do zero → `resumo-instalacao.md`
- Entender o que foi feito e por quê → `instalacao-passo-a-passo.md`
- Criar agentes especializados → `criar-agentes.md`
- Usar os agentes no dia a dia → `usar-agentes.md`
- Fazer agentes rodarem sozinhos (Telegram, cron, API) → `agentes-autonomos.md`
- Habilitar Claude / GPT via CLIs já autenticadas → `modelos-remotos.md`
