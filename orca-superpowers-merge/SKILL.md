---
name: orca-superpowers-merge
description: Use when a feature branch in Orca ADE was approved by code review and the merge, push, workspace status update and terminal cleanup must run as a gated, auditable step across Orca terminal sessions
---

# Orca Superpowers Merge

## Overview

Transforma o "pós-merge manual" (consolidação do `orca-superpowers-orchestrator`) numa **etapa auditável** no **Orca ADE**: um agente **Merger** dedicado roda a suíte real, aplica a estratégia de merge escolhida pelo usuário, faz push, atualiza o card do worktree e limpa os terminais — sempre com prova de conclusão.

Contrato da família, idêntico às demais skills:

- **TUI interativa OBRIGATÓRIA** (`--focus` + `terminal wait --for tui-idle`), sem fallback para `-p`/`exec`/`run`.
- **Aviso de conclusão é a única prova de fim**: `ETAPA merger CONCLUÍDA | status: <OK|FALHA> | ...` enviado ao `<ORQ_HANDLE>`.
- **O agente desta skill não faz merge nem roda testes** — ele despacha, valida gates e reporta.

```
   [AGENTE DA SKILL — coordenador, não codifica]
                │
  Passo 0: review aprovado? <ORQ_HANDLE> resolvido? ── NÃO ──▶ PARE
                │ SIM
                ▼
  Passo 1: estratégia de merge perguntada ao USUÁRIO (gate humano)
                │  merge | squash | fast-forward
                ▼
  Passo 2: terminal create (Merger) → tui-idle → send
        ┌─────────────────────────────────────────────┐
        │ Merger: suíte real → finishing-a-branch →   │
        │ merge → push → worktree set (status/comment)│
        └─────────────────────────────────────────────┘
                │ aviso chega?
        ┌───────┴────────┐
     status: OK      status: FALHA
        ▼                ▼
   Passo 3: encerra   Passo 4: NENHUM push/merge foi feito;
   terminais e        leia o terminal, informe o usuário e pare
   reporta ao usuário
```

---

## Quando Usar

- Logo após um review **aprovado** (`Ready to merge`), inclusive como passo final do `orca-superpowers-orchestrator` ou do `orca-superpowers-review-loop`.
- Quando o usuário pede para "finalizar", "merge da branch", "fechar a tarefa" com verificação de testes antes.

### Quando NÃO Usar

- Review reprovado ou pendências abertas → `orca-superpowers-review-loop`.
- Trabalho ainda em implementação → `orca-superpowers-orchestrator`.
- Sem aprovação de review → pare; merge sem auditoria está fora do escopo.

---

## Protocolo

### Passo 0: Pré-condições (tudo verificado aqui, não pelo Merger)

1. Existe aprovação de review (`Ready to merge`) no relatório de `docs/superpowers/reviews/` ou aviso do Reviewer. **Sem ela, pare.**
2. Nenhuma pendência/blocker declarada pelo usuário.
3. Resolver o destino dos avisos:

```bash
orca terminal show --json        # campo "handle" = <ORQ_HANDLE>
```

Falhou → pare e avise o usuário.

### Passo 1: Gate Humano de Estratégia

Pergunte ao usuário e **nunca escolha sozinho**:

> "Qual estratégia de merge para `<branch>` → `<branch-base>`? (1) `merge` — commit de merge, histórico completo (recomendado quando houver múltiplos commits revisados) · (2) `squash` — um commit limpo · (3) `fast-forward` — só se não houver commits novos na base."

Sem resposta afirmativa, **não dispare o Merger**.

### Passo 2: Despacho do Merger

```bash
orca terminal create --worktree active --title "merge-Merger" --focus --command "<MERGER_CLI>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
# satisfied: false → repita UMA vez com timeout maior; ainda false → PARE e avise o usuário
orca terminal send --terminal <handle> --text "<TEMPLATE_MERGER>" --enter --json
```

#### Template injetado no Merger:
```text
Use as skills superpowers:verification-before-completion e superpowers:finishing-a-development-branch.
APROVAÇÃO DE REVIEW: <caminho-do-relatorio-ou-aviso-reviewer>
ESTRATEGIA DE MERGE APROVADA PELO USUARIO: <merge|squash|fast-forward>

REGRAS RÍGIDAS (nesta ordem, sem atalhos):
1. Rode a suíte de testes real do projeto e guarde a saída completa.
2. Se QUALQUER teste falhar: NÃO faça merge, NÃO faça push, envie status: FALHA com a saída e encerre.
3. Só com tudo verde: aplique a estratégia escolhida para <branch-base>.
4. Conflito ou erro de git: NÃO resolva por conta própria; envie status: FALHA e encerre.
5. Push apenas depois do merge local concluído.
6. Atualize o card do worktree:
   orca worktree set --worktree active --workspace-status completed --comment "merged <branch> -> <branch-base> (<estrategia>)" --json
7. Avise o orquestrador (obrigatório, mesmo em falha):
   orca terminal send --terminal <ORQ_HANDLE> --text "ETAPA merger CONCLUÍDA | status: <OK|FALHA> | resumo: <3 frases> | testes: <passou|falhou> | merge: <hash-ou->" --enter --json
8. Só depois do envio, encerre a execução.
```

Aguarde `ETAPA merger CONCLUÍDA`. Sem aviso em 10 minutos → `orca terminal read --terminal <handle> --limit 50 --json` antes de decidir. Ausência de aviso nunca conta como conclusão.

### Passo 3: Encerramento (após `status: OK`)

1. Feche os terminais das etapas concluídas (nunca o terminal do coordenador):

```bash
orca terminal list --worktree active --json
orca terminal close --terminal <handle-da-etapa> --json
```

2. Reporte ao usuário: hash do merge, estratégia usada, saída de testes resumida, worktree em `completed`.

### Passo 4: Falha (`status: FALHA`)

1. **Nenhum push/merge foi feito** — confirme lendo o terminal do Merger (`orca terminal read`).
2. Informe o usuário com a causa (teste quebrou / conflito / discordância).
3. Se a falha foi por teste, redirecione para `orca-superpowers-review-loop` ou ao Executor.
4. Nunca tente o merge pela terceira vez sem nova aprovação humana.

---

## Comandos Úteis do Orca ADE

```bash
orca terminal show --json                                              # <ORQ_HANDLE> — ANTES do Passo 2
orca terminal create --worktree active --title "<t>" --focus --command "<agente-interativo>" --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
orca terminal send --terminal <handle> --text "<prompt>" --enter --json
orca terminal read --terminal <handle> --limit 50 --json
orca terminal list --worktree active --json
orca terminal close --terminal <handle> --json
orca worktree set --worktree active --workspace-status completed --comment "<status>" --json
orca worktree current --json
```

---

## Red Flags - Erros Críticos

- **NUNCA inicie o Merger sem aprovação de review** registrada.
- **NUNCA escolha a estratégia de merge sozinho** — é gate humano, sempre.
- **NUNCA faça merge ou push no terminal do coordenador**: ele não codifica nem versiona.
- **NUNCA avance sem `ETAPA merger CONCLUÍDA`** no `<ORQ_HANDLE>`; leia o terminal antes de concluir.
- **NUNCA aceite `status: FALHA` como "quase lá"**: sem testes verdes não há merge, não há push.
- **NUNCA resolva conflito de git por conta própria** dentro desta skill — escalada ao usuário.
- **NUNCA feche o terminal do coordenador** na limpeza do Passo 3.
- **TUI é OBRIGATÓRIA**: `--focus` + `tui-idle` antes do `send`; duas falhas → interrompa e avise. **NUNCA** use `-p`/`exec`/`run`.
