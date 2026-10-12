# orca-superpowers-skills

Combinação de ferramentas para usar superpowers e orquestração no Orca ADE.

## Skills

| Skill | Descrição |
|---|---|
| [orca-superpowers-orchestrator](orca-superpowers-orchestrator/SKILL.md) | Orquestrador central que despacha Planner → Gate humano → Executor → Reviewer em terminais do Orca, com TUI obrigatória e aviso de conclusão via CLI |
| [orca-superpowers-review-loop](orca-superpowers-review-loop/SKILL.md) | Malha de review reprovado: Fixer corrige só os itens apontados → Re-review, com limite de 3 iterações e escala humana |
| [orca-superpowers-merge](orca-superpowers-merge/SKILL.md) | Finalização auditável pós-aprovação: testes reais → gate da estratégia de merge → merge/push → card do worktree e limpeza de terminais |

| [orca-superpowers-handoff](orca-superpowers-handoff/SKILL.md) | Cadeia de handoff sequencial entre agentes no mesmo worktree, com TUI obrigatória e aviso de conclusão via CLI |

## Instalação

Copie a pasta da skill para o diretório de skills do agente:

```powershell
# Claude Code / Antigravity (Gemini) / Codex / OpenCode (agents)
$skills = "orca-superpowers-orchestrator", "orca-superpowers-review-loop", "orca-superpowers-merge", "orca-superpowers-handoff"
$targets = "$env:USERPROFILE\.claude\skills", "$env:USERPROFILE\.gemini\config\skills", "$env:USERPROFILE\.codex\skills", "$env:USERPROFILE\.agents\skills", "$env:USERPROFILE\.config\opencode\skills"
foreach ($skill in $skills) {
  foreach ($target in $targets) {
    Copy-Item -Recurse -Force $skill "$target\$skill"
  }
}
```
