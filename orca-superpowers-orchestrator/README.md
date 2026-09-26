# Skill Orquestração (Orca Superpowers Orchestrator)

Repositório da skill `orca-superpowers-orchestrator` para orquestração sequencial e centralizada de agentes no **Orca ADE**.

## Conteúdo
- [SKILL.md](SKILL.md): Definição completa da skill para o agente orquestrador.

## Visão Geral
O agente orquestrador opera como coordenador central sem codificar, planejar ou revisar diretamente:
1. **Passo 0**: Pergunta ao usuário a ordem/distribuição entre os 4 agentes instalados (`claude`, `codex`, `agy`, `opencode`).
2. **Passo 1 (Planner)**: Despacha o agente escolhido via `orca terminal create` com `superpowers:brainstorming` e `superpowers:writing-plans`.
3. **Gate de Aprovação**: Valida spec e plano com o usuário.
4. **Passo 2 (Executor)**: Despacha o executor via `orca terminal create` com `superpowers:executing-plans` (TDD estrito).
5. **Passo 3 (Reviewer)**: Despacha o auditor via `orca terminal create` com `superpowers:requesting-code-review` e suíte de testes reais.
6. **Consolidação**: Apresenta parecer final ao usuário.
