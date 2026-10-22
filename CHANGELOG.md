# Changelog

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
