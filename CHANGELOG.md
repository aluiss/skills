# Changelog

## 1.2.0 — 2026-11-10

- UI/UX (`references/ui-ux.md`): regras que faltavam, a partir de uma comparação com o catálogo
  ui-patterns.com e as heurísticas de Nielsen. Ordenação por coluna em tabelas longas,
  "Desfazer" em ações reversíveis no lugar de confirmação, quando usar (e não usar) modal e
  edição no lugar, marcação de campo obrigatório, formato estruturado e busca em lista longa,
  estado vazio com próximo passo, "Excluir" × "Remover", tooltip não substitui `aria-label`,
  texto mínimo de 12 px. Tabela agrupada por dia ou com ordem manual fica fora da ordenação por
  coluna.
- Nova seção "Referências externas": a norma é o documento + Nielsen + WCAG; o ui-patterns.com é
  catálogo de consulta, sem os padrões persuasivos. Filtro continua acima da tabela, não no
  cabeçalho da coluna.

## 1.1.0 — 2026-10-22

- Front-end: novo padrão "Página dividida em seções" (`references/frontend.md`, seção 5). Cards
  de KPI, filtros e tabela viram componentes próprios; a página só orquestra (consultas,
  mutações, modais, estado compartilhado). Inclui o contrato das seções e onde ficam tipos e
  helpers puros (`lib/<area>/<tela>.ts`).

## 1.0.0 — 2026-10-06

- Primeira versão compartilhada. Substitui a skill pessoal `padroes-engenharia`, absorvendo todo
  o conteúdo dela.
- Novas referências: `banco-de-dados.md`, `ui-ux.md`, `documentacao.md`, `repositorios.md` e
  `empresa.md`.
- Novas regras de segurança (limite de tentativas por e-mail, credencial gravada só depois de
  entregue, troca obrigatória de senha na API), cache de consultas externas, telas de operação
  ao vivo, kit padrão de bibliotecas e padrão de commits (Conventional Commits + gitmoji).
