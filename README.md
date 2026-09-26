# orca-superpowers-skills

Combinação de ferramentas para usar superpowers e orquestração no Orca ADE.

## Skills

| Skill | Descrição |
|---|---|
| [orca-superpowers-orchestrator](orca-superpowers-orchestrator/SKILL.md) | Orquestrador central que despacha Planner → Gate humano → Executor → Reviewer em terminais do Orca, com TUI obrigatória e aviso de conclusão via CLI |

## Instalação

Copie a pasta da skill para o diretório de skills do agente:

```powershell
# Claude Code / Antigravity (Gemini) / Codex / OpenCode (agents)
$skill = "orca-superpowers-orchestrator"
Copy-Item -Recurse -Force "$skill" "$env:USERPROFILE\.claude\skills\$skill"
Copy-Item -Recurse -Force "$skill" "$env:USERPROFILE\.gemini\config\plugins\superpowers\skills\$skill"
Copy-Item -Recurse -Force "$skill" "$env:USERPROFILE\.codex\skills\$skill"
Copy-Item -Recurse -Force "$skill" "$env:USERPROFILE\.agents\skills\$skill"
```
