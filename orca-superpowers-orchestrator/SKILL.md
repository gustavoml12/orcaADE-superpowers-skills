---
name: orca-superpowers-orchestrator
description: Use when orchestrating an end-to-end development cycle in Orca ADE where a dedicated orchestrator agent dispatches planner, executor, and reviewer agents across terminal sessions
---

# Orca Superpowers Orchestrator

## Overview

Coordena o ciclo completo de desenvolvimento dentro do **Orca ADE**, operando no mesmo workspace/branch ativo através de um **Agente Orquestrador Central**. 

Diferente do handoff em cadeia simples, o Orquestrador atua estritamente como **hub supervisor e despachante**: ele não escreve código de produção, não redige especificações e não faz code review diretamente. Ele gerencia o fluxo em 3 processos distintos acionados via **Orca CLI**:

1. **Processo 1: Planejador (Planner):** Executa `superpowers:brainstorming` e `superpowers:writing-plans` para gerar a spec técnica e o plano com checkboxes.
2. **Processo 2: Executor (Executor):** Assume após o gate de aprovação humana, executando o plano via `superpowers:executing-plans` com TDD estrito e commits atômicos.
3. **Processo 3: Revisor (Reviewer):** Audita a suíte de testes e o `git diff` via `superpowers:requesting-code-review` e `superpowers:verification-before-completion`.

```
┌────────────────────────────────────────────────────────────────────────┐
│               Mesmo Workspace / Worktree Ativo no Orca                 │
│                                                                        │
│                      [AGENTE ORQUESTRADOR]                             │
│   (Pergunta ao usuário a ordem/distribuição dos 4 agentes:             │
│    Claude Code, Codex, AGY, OpenCode)                                  │
│                                │                                       │
│    1. orca terminal create     ▼                                       │
│       ┌────────────────────────────────────────────────┐               │
│       │ [Processo 1: PLANNER (ex: Claude Code)]        │               │
│       │ Executa: brainstorming + writing-plans         │               │
│       │ Produz: docs/superpowers/specs/ & plans/       │               │
│       └──────────────────────┬─────────────────────────┘               │
│                              ▼                                         │
│    2. GATE DE APROVAÇÃO: Orquestrador valida plano com usuário         │
│                              │                                         │
│    3. orca terminal create   ▼                                         │
│       ┌────────────────────────────────────────────────┐               │
│       │ [Processo 2: EXECUTOR (ex: Codex)]             │               │
│       │ Executa: executing-plans (TDD estrito)         │               │
│       │ Marca checkboxes [x] no plano                  │               │
│       └──────────────────────┬─────────────────────────┘               │
│                              ▼                                         │
│    4. orca terminal create   ▼                                         │
│       ┌────────────────────────────────────────────────┐               │
│       │ [Processo 3: REVIEWER (ex: AGY / OpenCode)]    │               │
│       │ Executa: requesting-code-review + testes reais │               │
│       │ Produz: docs/superpowers/reviews/ ou aprovação │               │
│       └──────────────────────┬─────────────────────────┘               │
│                              ▼                                         │
│    5. Orquestrador consolida resultado e reporta ao usuário            │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Agentes Suportados e Matriz de Invocação

O Orquestrador suporta qualquer permutação dos 4 agentes instalados no sistema. **Abrir a TUI interativa é OBRIGATÓRIO em toda etapa** — todo agente nasce como TUI visível (`--focus`), e não existe fallback para modo não-interativo (`-p`/`exec`/`run`): se a TUI não abrir, o fluxo é interrompido e o usuário é avisado, jamais se recorre ao modo batch. O prompt é injetado depois, via `orca terminal send`, apenas após a TUI confirmar pronta (`tui-idle`):

| Agente | CLI | Comando de Abertura Interativa (via Orca CLI) |
|---|---|---|
| **Claude Code** | `claude` | `claude --dangerously-skip-permissions` |
| **Codex CLI** | `codex` | `codex --dangerously-bypass-approvals-and-sandbox` |
| **Antigravity CLI** | `agy` | `agy --dangerously-skip-permissions` |
| **OpenCode CLI** | `opencode` | `opencode` |

Padrão de injeção do prompt em qualquer passo:

```bash
orca terminal create --worktree active --title "<titulo>" --focus --command "<AGENTE_INTERATIVO>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json   # só envie se satisfied: true
# satisfied: false → repita o wait uma vez com timeout maior; ainda assim false → PARE e avise o usuário (nunca use -p/exec/run)
orca terminal send --terminal <handle> --text "<PROMPT>" --enter --json
```

---

## Quando Usar

- Ao iniciar o desenvolvimento de features, refatorações ou correções complexas no Orca ADE.
- Quando se deseja supervisão centralizada por um agente neutro sem poluir o contexto com código ou arquivos de implementação.
- Quando o usuário deseja alternar papéis entre Claude, Codex, AGY e OpenCode (ex: Codex planejando, OpenCode executando, AGY revisando).

### Quando NÃO Usar
- Para tarefas atômicas triviais de um único arquivo (use diretamente um agente no terminal).
- Em ambientes fora do Orca ADE onde a CLI `orca` não está disponível.

---

## Protocolo Operacional do Orquestrador

### Passo 0: Coleta de Requisito e Alocação dos Papéis

O Orquestrador recebe o requisito inicial e pergunta ao usuário qual agente executará cada papel (ou adota a sugestão se o usuário confirmar):

> **Sugestão Padrão Recomendada:**
> - **Planner:** Claude Code (`claude`)
> - **Executor:** Codex (`codex`)
> - **Reviewer:** Antigravity (`agy`)

Se o usuário desejar ordem personalizada (ex.: Planner=OpenCode, Executor=Claude Code, Reviewer=Codex), o Orquestrador parametriza as variáveis de execução:
- `$PLANNER_CLI`
- `$EXECUTOR_CLI`
- `$REVIEWER_CLI`

#### Identificação do Orquestrador (obrigatória antes do Passo 1)

Cada agente precisa saber **para quem** avisar a conclusão. Antes de disparar qualquer etapa, o Orquestrador resolve o próprio handle de terminal e o injeta em todos os prompts como `<ORQ_HANDLE>`:

```bash
orca terminal show --json        # sem --terminal = terminal ativo = o do orquestrador → campo "handle"
```

Se `terminal show` falhar (orquestrador fora de um terminal Orca), não siga: o aviso via CLI não teria destino. Informe o usuário e pare.

**Contrato de notificação (vale para Planner, Executor e Reviewer):** o agente só é considerado concluído quando envia, do próprio terminal, exatamente:

```bash
orca terminal send --terminal <ORQ_HANDLE> --text "ETAPA <planner|executor|reviewer> CONCLUÍDA | status: <OK|FALHA> | resumo: <3 frases> | plano: <caminho-do-plano ou ->" --enter --json
```

`<ORQ_HANDLE>` é substituído pelo handle real no momento da injeção do prompt. A mensagem chega automaticamente como entrada na sessão do Orquestrador; o aviso é obrigatório mesmo em caso de falha (`status: FALHA`).

---

### Passo 1: Disparo do Processo de Planejamento (Planner)

O Orquestrador não cria o plano diretamente. Ele cria uma aba no Orca para o Planner:

```bash
# TUI obrigatória: --focus + agente interativo; sem tui-idle satisfied não há Passo 1
orca terminal create --worktree active --title "1-Planner" --focus --command "<PLANNER_CLI>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
# satisfied: false → repita o wait com timeout maior; ainda false → PARE e avise o usuário (proibido recorrer a modo não-interativo)
orca terminal send --terminal <handle> --text "<TEMPLATE_DO_PROMPT_ABAIXO>" --enter --json
```

#### Template de Prompt injetado no Planner:
```text
Use as skills superpowers:brainstorming e superpowers:writing-plans.
REQUISITO DO USUÁRIO:
<REQUISITO_INICIAL>

DIRETRIZES:
1. Explore a arquitetura e elabore a especificação técnica em docs/superpowers/specs/YYYY-MM-DD-<feature>-design.md.
2. Escreva o plano de tarefas executável com TDD em docs/superpowers/plans/YYYY-MM-DD-<feature>.md com checkboxes '- [ ]' e testes explícitos por tarefa.
3. Ao concluir a criação dos dois arquivos e comitá-los, avise o orquestrador (obrigatório, mesmo em falha):
   orca terminal send --terminal <ORQ_HANDLE> --text "ETAPA planner CONCLUÍDA | status: OK | resumo: <3 frases> | plano: docs/superpowers/plans/<nome>.md" --enter --json
4. Só depois do envio, encerre a execução.
```

O Orquestrador aguarda a notificação do Planner chegar ao seu terminal (a mensagem injetada entra como nova entrada na sessão dele) e confere a geração do arquivo em `docs/superpowers/plans/`. Se não houver aviso em 10 minutos, leia o terminal do Planner (`orca terminal read --terminal <handle_planner> --json`) antes de qualquer conclusão — ausência de aviso nunca conta como conclusão.

---

### Passo 2: Gate Obrigatório de Aprovação Humana

Assim que o plano é gerado:
1. O Orquestrador localiza o plano mais recente em `docs/superpowers/plans/`.
2. Apresenta ao usuário:
   - Resumo da arquitetura proposta.
   - Lista resumida de tarefas e arquivos impactados.
3. **Pausa obrigatória**: O Orquestrador pergunta explicitamente ao usuário:
   > *"O plano foi elaborado em `docs/superpowers/plans/<nome>.md`. Deseja aprovar o início da implementação pelo Executor (`<EXECUTOR_CLI>`)?"*
4. **NUNCA** dispare o Executor antes da confirmação afirmativa do usuário ("Sim", "Aprovado", etc.).

---

### Passo 3: Disparo do Processo de Implementação (Executor)

Após aprovação humana, o Orquestrador abre uma nova aba para o Executor:

```bash
# TUI obrigatória: --focus + agente interativo; sem tui-idle satisfied não há Passo 3
orca terminal create --worktree active --title "2-Executor" --focus --command "<EXECUTOR_CLI>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
# satisfied: false → repita o wait com timeout maior; ainda false → PARE e avise o usuário (proibido recorrer a modo não-interativo)
orca terminal send --terminal <handle> --text "<TEMPLATE_DO_PROMPT_ABAIXO>" --enter --json
```

#### Template de Prompt injetado no Executor:
```text
Use a skill superpowers:executing-plans.
Consulte o plano em docs/superpowers/plans/<NOME_DO_PLANO>.md.

REGRAS RÍGIDAS DE EXECUÇÃO:
1. Execute cada tarefa do plano estritamente em TDD (escreva o teste, veja falhar, implemente o código, veja passar).
2. Faça commit atômico por tarefa no formato: git commit -m "feat/fix: <resumo>".
3. Atualize o arquivo do plano marcando a checkbox correspondente com '- [x]'.
4. Ao concluir todas as tarefas e garantir que a suíte de testes passa, avise o orquestrador (obrigatório, mesmo em falha):
   orca terminal send --terminal <ORQ_HANDLE> --text "ETAPA executor CONCLUÍDA | status: <OK|FALHA> | resumo: <3 frases> | plano: <caminho-do-plano>" --enter --json
5. Só depois do envio, encerre.
```

O Orquestrador aguarda o aviso do Executor no seu terminal e confere que todas as checkboxes principais do plano estão marcadas com `[x]`. Se não houver aviso em 10 minutos, leia o terminal do Executor antes de qualquer conclusão.

---

### Passo 4: Disparo do Processo de Auditoria (Reviewer)

Com a implementação concluída e commitada, o Orquestrador aciona o Revisor:

```bash
# TUI obrigatória: --focus + agente interativo; sem tui-idle satisfied não há Passo 4
orca terminal create --worktree active --title "3-Reviewer" --focus --command "<REVIEWER_CLI>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
# satisfied: false → repita o wait com timeout maior; ainda false → PARE e avise o usuário (proibido recorrer a modo não-interativo)
orca terminal send --terminal <handle> --text "<TEMPLATE_DO_PROMPT_ABAIXO>" --enter --json
```

#### Template de Prompt injetado no Revisor:
```text
Use as skills superpowers:requesting-code-review e superpowers:verification-before-completion.
Avalie o git diff da branch contra a spec em docs/superpowers/specs/ e o plano em docs/superpowers/plans/<NOME_DO_PLANO>.md.

REQUISITOS DA AUDITORIA:
1. Execute a suíte de testes automatizados do projeto e colete a saída real.
2. Inspecione a qualidade de código, padrões de projeto e possíveis regressões.
3. Se houver falhas críticas ou importantes: salve o relatório em docs/superpowers/reviews/YYYY-MM-DD-<feature>-review.md e detalhe os problemas.
4. Se aprovado: emita o relatório final de aprovação ('Ready to merge').
5. Em ambos os casos, avise o orquestrador (obrigatório, mesmo em falha):
   orca terminal send --terminal <ORQ_HANDLE> --text "ETAPA reviewer CONCLUÍDA | status: <OK|FALHA> | resumo: <3 frases> | plano: <caminho-do-relatorio-ou->" --enter --json
6. Só depois do envio, encerre.
```

O Orquestrador aguarda o aviso do Reviewer no seu terminal antes de consolidar. Se não houver aviso em 10 minutos, leia o terminal do Reviewer antes de qualquer conclusão.

---

### Passo 5: Consolidação e Encerramento pelo Orquestrador

O Orquestrador só consolida após receber o aviso do Reviewer (`ETAPA reviewer CONCLUÍDA`) no seu terminal — aviso é a prova de conclusão, não a ausência de output. Depois do aviso:
- **Se Aprovado:** Notifica o usuário com o resumo dos testes bem-sucedidos e instrução para merge da branch.
- **Se Reprovado / Pendências:** Aponta ao usuário o relatório gerado em `docs/superpowers/reviews/` e oferece a opção de reacionar o Executor para corrigir as pendências.

---

## Comandos Úteis do Orca ADE

```bash
# Handle do próprio orquestrador (destino dos avisos de conclusão) — rodar ANTES do Passo 1
orca terminal show --json

# Consultar worktree atual
orca worktree current --json

# Criar aba de terminal com a TUI do agente — OBRIGATÓRIO: sempre interativo e com --focus
orca terminal create --worktree active --title "<titulo>" --focus --command "<agente-interativo>" --json

# Aguardar a TUI do agente ficar pronta antes de enviar o prompt (obrigatório)
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json

# Injetar o prompt no agente interativo
orca terminal send --terminal <handle> --text "<prompt>" --enter --json

# Listar terminais abertos
orca terminal list --worktree active --json

# Inspecionar saída de um terminal específico
orca terminal read --terminal <handle> --limit 50 --json

# Aguardar encerramento de um terminal
orca terminal wait --terminal <handle> --for exit --timeout-ms 300000 --json
```

---

## Red Flags - Erros Críticos

- **NUNCA execute código ou testes no agente Orquestrador:** o orquestrador apenas despacha os processos dedicados via Orca CLI.
- **NUNCA dispare o Executor sem aprovação humana expressa do plano gerado pelo Planner.**
- **NUNCA permita que o Executor pule a marcação das checkboxes no plano:** o Orquestrador depende do plano para validar a conclusão antes de chamar o Reviewer.
- **NUNCA considere o Review concluído sem a execução dos comandos reais de teste do projeto.**
- **NUNCA crie novos worktrees separados para cada agente:** todos os agentes devem operar no mesmo worktree ativo (`--worktree active`).
- **TUI é OBRIGATÓRIA em 100% das etapas:** todo `terminal create` de agente usa comando interativo + `--focus` + `wait --for tui-idle` antes do envio do prompt. **NUNCA** exista fallback para `-p`/`exec`/`run`: se a TUI não abrir (duas tentativas de `tui-idle` com `satisfied: false`), interrompa o fluxo e avise o usuário.
- **NUNCA avance de etapa sem o aviso explícito do agente:** toda injeção de prompt deve conter o `<ORQ_HANDLE>` real e o comando de notificação; sem `ETAPA <nome> CONCLUÍDA` no terminal do Orquestrador, a etapa está indeterminada — leia o terminal do agente antes de decidir.
- **NUNCA se esqueça de resolver o handle do orquestrador (`orca terminal show`) antes do Passo 1:** sem ele os agentes não têm destino para o aviso.
