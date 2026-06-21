Não gravei `review-statusline.md`: a tarefa também diz “Do not edit files”, e a instrução superior manda manter arquivos inalterados quando há conflito com escrita de artefato.

## Review

- **Correct:** `ctx.ui.setFooter` está usado no formato esperado: factory recebe `(tui, theme, footerData)` e retorna componente com `render/invalidate/dispose` (`extensions/index.ts:264-372`), compatível com `types.d.ts:77-85`.
- **Correct:** `SettingsList` está usando `SettingItem.currentValue`, `values`, callback `onChange` e opção `enableSearch` de forma compatível com a API (`extensions/index.ts:382-404`; `settings-list.d.ts:1-40`).
- **Correct:** a filtragem dos status preserva o comportamento central de ordenar, sanitizar e truncar os segmentos (`extensions/index.ts:359-367`; footer padrão em `footer.js:188-196`).

- **Blocker / High:** `/cc-statusline hide|toggle|reset` é efetivamente quebrado quando existe `.pi/settings.json` no projeto. `readSettings()` retorna o arquivo do projeto primeiro e não mergeia com HOME (`extensions/index.ts:129-137`), mas `writeSettingsKey()` sempre grava só em `HOME/.pi/settings.json` (`extensions/index.ts:158-175`). Depois de persistir, `applyStatuslineFooter()` chama `syncHiddenFooterStatusKeysFromSettings()` e recarrega o arquivo do projeto (`extensions/index.ts:187-194`, `263`). Neste repo há `.pi/settings.json` sem `hiddenFooterStatusKeys` (`.pi/settings.json:1-17`), então toggles feitos em `extensions/index.ts:397-399` e `4166-4170` são perdidos imediatamente.

- **Note / Medium:** o footer customizado sempre mostra `" (auto)"` (`extensions/index.ts:312-315`). O footer padrão só mostra isso quando `autoCompactEnabled` está ativo (`footer.js:35`, `43-44`, `117-120`), e a configuração existe (`docs/settings.md:50-55`). Usuários com auto-compaction desabilitado verão status incorreto.

## residual-risks

- Não executei teste interativo TUI; validei por diff, tipos e APIs instaladas.
- `setFooter` disposal interno não ficou verificável no pacote instalado além de tipos/docs.
- Há staged files pré-existentes; não alterei nem stageei nada.

```acceptance-report
{
  "criteriaSatisfied": [
    {
      "id": "criterion-1",
      "status": "satisfied",
      "evidence": "Findings concretos com severidade e referências exatas: blocker em extensions/index.ts:129-137,158-175,187-194,263,397-399,4166-4170 e nota em extensions/index.ts:312-315."
    }
  ],
  "changedFiles": [],
  "testsAddedOrUpdated": [],
  "commandsRun": [
    {
      "command": "git status --short && git diff --stat",
      "result": "passed",
      "summary": "Identificou diff atual e arquivos staged pré-existentes."
    },
    {
      "command": "git diff -- extensions/index.ts package.json && git diff --cached -- extensions/index.ts package.json",
      "result": "passed",
      "summary": "Inspecionou diff unstaged e staged."
    },
    {
      "command": "npx tsc --noEmit --skipLibCheck --moduleResolution NodeNext --module NodeNext --target ES2022 --types node extensions/index.ts",
      "result": "passed",
      "summary": "Sem erros de TypeScript."
    },
    {
      "command": "git diff --check && git diff --cached --check",
      "result": "passed",
      "summary": "Sem whitespace errors."
    }
  ],
  "validationOutput": [
    "Typecheck sem saída.",
    "diff --check sem saída."
  ],
  "residualRisks": [
    "Não foi feito teste manual em TUI.",
    "Arquivo review-statusline.md não foi escrito devido ao conflito com 'Do not edit files'.",
    "Existem staged files pré-existentes no repo."
  ],
  "noStagedFiles": false,
  "notes": "Review-only respeitado; nenhum arquivo foi editado."
}
```

[38;2;128;128;128m✻ Worked for 3m 43s[0m