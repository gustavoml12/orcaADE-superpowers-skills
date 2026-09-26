# Skill Merge (Orca Superpowers Merge)

Repositório da skill `orca-superpowers-merge` para finalização auditável de branch no **Orca ADE**.

## Conteúdo
- [SKILL.md](SKILL.md): Definição completa da etapa de merge pós-aprovação.

## Visão Geral
O agente da skill atua como coordenador, sem codificar nem versionar:
1. **Passo 0**: Confere aprovação de review (`Ready to merge`) e resolve `<ORQ_HANDLE>` (`orca terminal show`).
2. **Passo 1**: Gate humano — o usuário escolhe `merge`, `squash` ou `fast-forward`.
3. **Passo 2 (Merger)**: Despacha via `orca terminal create` com `superpowers:verification-before-completion` + `superpowers:finishing-a-development-branch`: suíte real → merge → push → `worktree set --workspace-status completed --comment`.
4. **Passo 3/4**: `status: OK` → fecha terminais da etapa e reporta; `status: FALHA` → nenhum push/merge foi feito, lê o terminal e escala ao usuário.

## Garantias
- TUI interativa obrigatória em 100% dos despachos (`--focus` + `tui-idle`), sem fallback `-p`/`exec`/`run`.
- Prova de conclusão apenas pelo aviso `ETAPA merger CONCLUÍDA` enviado ao `<ORQ_HANDLE>`.
- Teste vermelho, conflito ou falta de aprovação de review bloqueiam merge e push.
