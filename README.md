# skills-mj

Minhas skills e subagentes do Claude Code, num lugar só.

## Plugin `metodo`

- **Método Spec → Plan → Build:** `/projeto` (orquestradora), `/spec`, `/visao`, `/usuarios`, `/telas`, `/dados`, `/plan`, `/build`
- **Trilha de frontend:** `/frontend`, `/frontend-visual`, `/frontend-testes`, `/frontend-performance`
- **CRM:** `/crm-planner` — planeja funis, regras, permissões e automações no Agendor ou no Salesforce Sales Cloud (modo rápido ou completo com subagentes; bases de conhecimento em `plugins/metodo/skills/crm-planner/referencias/`)
- **Subagentes:** `plan-visao`, `plan-usuarios`, `plan-telas`, `plan-dados`, `build-onda`, `build-revisor`, `crm-processo`, `crm-governanca`, `crm-automacao`, `crm-revisor`
- **Modelos** dos documentos: `plugins/metodo/skills/projeto/modelos/` (a `/projeto` copia para `docs/modelos/` quando faltar)

## Instalar no computador (vale para todos os projetos)

No Claude Code, digite:

```
/plugin marketplace add marcojuniorrj-web/skills-mj
/plugin install metodo@skills-mj
```

Feche e abra o Claude Code. Para conferir, digite `/projeto`.

## Atualizar

Depois de mudar algo neste repositório:

```
/plugin marketplace update skills-mj
```

## Usar nas sessões na nuvem de um repositório

Coloque no `.claude/settings.json` do repositório:

```json
{
  "extraKnownMarketplaces": {
    "skills-mj": { "source": { "source": "github", "repo": "marcojuniorrj-web/skills-mj" } }
  },
  "enabledPlugins": { "metodo@skills-mj": true }
}
```

## Como editar uma skill

Cada skill é uma pasta em `plugins/metodo/skills/<nome>/` com um `SKILL.md`. Ao mudar, suba a versão em
`plugins/metodo/.claude-plugin/plugin.json` e em `.claude-plugin/marketplace.json`.

Para o claude.ai (sem subagentes), compacte a pasta de cada skill num zip separado e envie em
Configurações → Skills. A descrição da skill não pode ter `<` ou `>`.
