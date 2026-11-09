# Plugins convertidos do Claude Code

Conversão do marketplace [devfullcycle/claude-mkt-place](https://github.com/devfullcycle/claude-mkt-place)
(commit `4a5aa4d`) para o formato nativo do OpenCode V2.

O repositório original contém apenas `commands` e `agents` — nenhum hook, script ou
manifest executável — então a conversão é majoritariamente declarativa.

## Origem → destino

| Plugin original | Commands | Agents |
| --- | --- | --- |
| `adrs-management` | `adr-map`, `adr-identify`, `adr-generate`, `adr-link` | `adr-analyzer`, `adr-generator`, `adr-linker` |
| `diagrams-generator` | `c4-generate`, `mermaid-generate` | `c4-diagram-generator`, `mermaid-diagram-generator` |
| `project-analizer` | `generate-architectural-report`, `run-dependency-audit` | `architectural-analyzer`, `component-deep-analyzer`, `dependency-auditor` |
| `development-guidelines` | `generate-development-guideline` | — |

Os arquivos ficam em `.opencode/commands/` e `.opencode/agents/` e são descobertos
automaticamente pelo OpenCode neste projeto.

## O que mudou na conversão

- `model: sonnet` removido: os subagentes herdam o modelo da sessão.
- `name` removido dos agents: no OpenCode o ID vem do nome do arquivo.
- Adicionado `mode: subagent` e `color` convertido de nome para hexadecimal.
- `description` dos agents reescrita de forma concisa (o conteúdo completo permanece no corpo).
- `allowed-tools` removido dos commands: no OpenCode, restrições de ferramenta são
  definidas em `permissions` (globais ou por agent).
- `Task` / `subagent_type` adaptados para a tool `subagent` (`agent: "<id>"`).
- Nomes de tools atualizados: `Read` → `read`, `Write` → `write`, `Edit` → `edit`,
  `Bash` → `shell`, `WebSearch` → `websearch`.
- `TodoWrite` removido (todos ficam desligados por padrão no OpenCode V2).
- `guidelines-generate.md` renomeado para `generate-development-guideline.md` para
  bater com o nome documentado no USAGE original.

## Uso rápido

```text
/adr-map
/adr-identify AUTH DATA
/adr-generate BILLING
/adr-link

/c4-generate docs/features/payment-fdd.md
/mermaid-generate docs/features/auth-fdd.md

/generate-architectural-report
/run-dependency-audit

/generate-development-guideline TypeScript --orm=prisma --web=express --testing=jest
```

Os fluxos de `project-analizer` mencionam MCP servers (Context7, Firecrawl) como
opcionais; eles só serão usados se estiverem configurados no OpenCode.
