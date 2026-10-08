# dev-defaults

Skill do Claude com os padrões de desenvolvimento do time: escolha de stack, banco de dados,
monorepo × repositórios separados, estrutura de pastas, front-end, UI/UX, back-end, segurança,
testes, documentação, comentários e heurísticas de trabalho.

Com ela instalada, o Claude (Claude Code, extensão do VS Code ou claude.ai) segue os mesmos
padrões para qualquer pessoa do time — e serve de referência para quem lê também.

## Estrutura

```
SKILL.md                 # resumo de cada decisão + qual referência ler
references/
├── stack.md             # front, CSS/UI, back, kit de bibliotecas, ferramentas
├── banco-de-dados.md    # escolha, modelagem, migrations, fuso, SQL, produção
├── repositorios.md      # monorepo × multi-repo, deploy parcial, branches
├── frontend.md          # telas, dados, modais, tabelas, navegação, estado lembrado
├── ui-ux.md             # tema, estados, formulários, textos, acessibilidade
├── backend.md           # módulos, autorização, concorrência, segurança, cache
├── documentacao.md      # README, planos, API, changelog, commits, PRs
├── heuristicas.md       # como conduzir a tarefa
└── empresa.md           # decisões específicas da NOVAISP (valem por cima das gerais)
```

## Instalação

**Claude Code / extensão do VS Code** — clone direto na pasta de skills do usuário:

```bash
git clone <url-deste-repositorio> ~/.claude/skills/dev-defaults
```

Reinicie a sessão do Claude Code. Para conferir, pergunte "qual o padrão para criar um módulo
de backend?" — a resposta deve citar o dev-defaults.

**Só num projeto** (todos que clonarem o projeto recebem): adicione como submódulo em
`.claude/skills/dev-defaults` do repositório do projeto.

**claude.ai**: compacte a pasta (`zip -r dev-defaults.zip . -x '.git/*'`) e envie em
Configurações › Capacidades › Skills.

## Atualização

```bash
git -C ~/.claude/skills/dev-defaults pull
```

## Como contribuir

1. Branch nova, mudança, PR para revisão do time.
2. Toda regra nova traz **o porquê** — de preferência o incidente que a motivou. Regra sem
   motivo não se defende no caso que ninguém previu.
3. Decisão específica da empresa vai em `references/empresa.md`, com data.
4. Mantenha o `SKILL.md` curto (resumo + ponteiro); o detalhe vai na referência.
5. Registre a mudança no `CHANGELOG.md` com nova versão.
