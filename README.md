# orca-superpowers-skills

Combinação de ferramentas para usar superpowers e orquestração no Orca ADE.

## Skills

| Skill | Descrição |
|---|---|
| [orca-superpowers-orchestrator](orca-superpowers-orchestrator/SKILL.md) | Orquestrador central que despacha Planner → Gate humano → Executor → Reviewer em terminais do Orca, com TUI obrigatória e aviso de conclusão via CLI |
| [orca-superpowers-review-loop](orca-superpowers-review-loop/SKILL.md) | Malha de review reprovado: Fixer corrige só os itens apontados → Re-review, com limite de 3 iterações e escala humana |
| [orca-superpowers-merge](orca-superpowers-merge/SKILL.md) | Finalização auditável pós-aprovação: testes reais → gate da estratégia de merge → merge/push → card do worktree e limpeza de terminais |

## Instalação

Copie a pasta da skill para o diretório de skills do agente:

```powershell
# Claude Code / Antigravity (Gemini) / Codex / OpenCode (agents)
$skills = "orca-superpowers-orchestrator", "orca-superpowers-review-loop", "orca-superpowers-merge"
foreach ($skill in $skills) {
  Copy-Item -Recurse -Force $skill "$env:USERPROFILE\.claude\skills\$skill"
  Copy-Item -Recurse -Force $skill "$env:USERPROFILE\.gemini\config\plugins\superpowers\skills\$skill"
  Copy-Item -Recurse -Force $skill "$env:USERPROFILE\.codex\skills\$skill"
  Copy-Item -Recurse -Force $skill "$env:USERPROFILE\.agents\skills\$skill"
}
```
