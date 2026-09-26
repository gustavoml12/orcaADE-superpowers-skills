# Skill Review Loop (Orca Superpowers Review Loop)

Repositório da skill `orca-superpowers-review-loop` para malha de correção e reavaliação de code review no **Orca ADE**.

## Conteúdo
- [SKILL.md](SKILL.md): Definição completa da skill de malha review → correção → re-review.

## Visão Geral
O agente da skill atua como coordenador, sem codificar nem revisar:
1. **Passo 0**: Localiza o relatório reprovado em `docs/superpowers/reviews/`, confere que há itens concretos e resolve `<ORQ_HANDLE>` (`orca terminal show`).
2. **Passo 1 (Fixer)**: Despacha o corretor via `orca terminal create` com `superpowers:receiving-code-review` + `superpowers:executing-plans`, estritamente sobre os itens apontados.
3. **Passo 2 (Reviewer)**: Despacha o reavaliador com `superpowers:requesting-code-review` + `superpowers:verification-before-completion`; anexa `## Iteração N` ao mesmo relatório.
4. **Passo 3**: Aprovado → fim (segue para `orca-superpowers-merge`); reprovado e iteração < 3 → nova rodada; = 3 → escala ao usuário com `status: FALHA`.

## Garantias
- TUI interativa obrigatória em 100% dos turnos (`--focus` + `tui-idle`), sem fallback `-p`/`exec`/`run`.
- Prova de conclusão apenas pelo aviso `ETAPA <fixer|reviewer> CONCLUÍDA` enviado ao `<ORQ_HANDLE>`.
- Limite duro de 3 iterações totais por ciclo, com escala humana obrigatória.
