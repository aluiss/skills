# Documentação

Leia ao criar um repositório, planejar uma feature grande, documentar uma API, escrever o
changelog, um commit ou uma PR. A regra geral: **documente o que o código não conta** — por que
foi feito assim, como rodar, o que mudou para quem usa. O resto o código já diz.

## Índice

1. O que documentar e onde
2. README do repositório
3. Planos e decisões (`docs/`)
4. Documentação da API
5. Changelog para o usuário
6. Mensagens de commit
7. Pull requests
8. Comentários no código

## 1. O que documentar e onde

| Para quem | O quê | Onde |
|---|---|---|
| Quem vai rodar o projeto | Como instalar, configurar e publicar | `README.md` |
| Quem vai mexer no código | Por que uma decisão foi tomada; plano de feature grande | `docs/<assunto>-plano.md` |
| Quem consome a API | Rotas, regras, respostas de erro | OpenAPI/Swagger gerado do código |
| Quem usa o sistema | O que mudou na vida dela | Changelog exibido na aplicação |
| Quem revisa | O que a mudança faz e como testar | Descrição da PR |
| Quem lê aquela linha | Por que ela existe | Comentário no código |

Documento fora do repositório (wiki, chat) desatualiza em silêncio. Mantenha perto do código e
revise junto com ele.

## 2. README do repositório

Todo repositório tem um. Modelo mínimo:

```markdown
# <Nome>

<O que é, em uma ou duas frases, e quem usa.>

## Requisitos
- Node <versão> (ver `.nvmrc`), PostgreSQL <versão>, Redis (opcional: <para quê>)

## Rodando localmente
1. `npm install`
2. Copie `.env.example` para `.env` e preencha (tabela abaixo)
3. `npx prisma migrate deploy` (se houver banco)
4. `npm run dev`

## Variáveis de ambiente
| Variável | Para quê | Exemplo |
|---|---|---|

## Scripts
| Comando | O que faz |
|---|---|

## Publicação
<Como sobe, em que ordem (migration antes?), o que depende de outro repositório.>

## Estrutura
<Uma linha por pasta principal, só se não for a estrutura padrão do dev-defaults.>
```

- **`.env.example` versionado**, com todas as variáveis e valores falsos; o `.env` real nunca.
- Quando algo depende de outro repositório (front ↔ API), diga aqui.

## 3. Planos e decisões (`docs/`)

Feature grande, integração nova ou mudança de arquitetura começa por um plano em
`docs/<assunto>-plano.md`, escrito **antes** de codificar e atualizado ao implementar:

```markdown
# <Assunto> — plano

> Status: PLANEJAMENTO (data) | IMPLEMENTADO em <data> (o que falta: …)
> Objetivo: <uma frase>

## Levantamento
<O que existe hoje, com números da base real; achados que condicionam o desenho.>

## Decisões necessárias
### A. <pergunta>
- A-i — <opção> (recomendado): <por quê>
- A-ii — <opção>: <consequência>
**DECIDIDO: A-i** (<data>, <quem>)

## Desenho
## Fases
## Implementação (preenchido ao implementar)
```

O plano registra **o porquê** e **as alternativas descartadas** — é o que impede a mesma
discussão de voltar seis meses depois. Decisão pequena de arquitetura que não merece plano vira
um parágrafo no README ou um comentário no código.

## 4. Documentação da API

Gerada do próprio código (`@nestjs/swagger`), para não divergir:

- Toda rota com `@ApiOperation({ summary, description })`. A **descrição conta a regra**: quem
  pode chamar, o que muda no sistema, efeitos colaterais.
- `@ApiResponse` para cada status relevante (200/201, 400, 403, 404, 409, 429, 503) dizendo
  **quando** ele acontece.
- DTOs com doc-comment por campo explicando a regra, não o tipo ("Id do USUÁRIO, não do
  login").
- Rota pública, rota restrita a um papel e rota com efeito irreversível ficam explícitas na
  descrição.

## 5. Changelog para o usuário

Lista de versões exibida na aplicação, a mais recente no topo, com **SemVer honesto**:

| Mudança | Versão |
|---|---|
| Correção sem mudar o que a pessoa faz | PATCH (`7.3.0 → 7.3.1`) |
| Funcionalidade nova | MINOR (`7.3.1 → 7.4.0`) |
| Quebra de contrato / mudança de fluxo que exige reaprender | MAJOR (`7.4.0 → 8.0.0`) |

Cada item tem um tipo (**novidade**, **melhoria**, **correção**, **mudança**) e é escrito do
ponto de vista de quem usa:

- Bom: "Agora é possível deixar em branco os campos de seleção."
- Ruim: "refactor: update select handler".

Diga **onde** fica ("Em Operação › Atendimentos › …"), **o que** a pessoa ganha, e quando algo
some ou muda de lugar, para onde foi. Enquanto a versão não foi publicada, itens novos entram
na mesma versão.

## 6. Mensagens de commit

**Conventional Commits + gitmoji**, resumo em inglês, no imperativo ou descritivo curto:

```
<tipo>(<Escopo>): :<gitmoji>: <Resumo>
```

| Tipo | Quando | Gitmoji comuns |
|---|---|---|
| `feat` | funcionalidade nova | `:sparkles:` |
| `fix` | correção de bug | `:bug:` |
| `refactor` | mudança sem alterar comportamento (ou ajuste de UI/regra) | `:recycle:` `:lipstick:` `:wrench:` |
| `perf` | desempenho ou usabilidade | `:zap:` `:children_crossing:` |
| `docs` | documentação, changelog | `:memo:` |
| `test` | testes | `:white_check_mark:` |
| `chore` | build, dependências, configuração | `:arrow_up:` `:wrench:` |
| `db` / `feat` com migration | schema e dados | `:card_file_box:` |

Escopo: `Back-end`, `Front-end` ou `Full` (monorepo, quando a mudança atravessa as duas
pontas). Exemplos:

```
feat(Back-end): :sparkles: Technician monitoring
fix(Front-end): :bug: Protocol filter on infra tab
docs(Front-end): :memo: Update changelog
```

Corpo do commit (opcional) explica **o porquê** quando não é óbvio. Um commit, um assunto.

## 7. Pull requests

Descrição com:

- **O que** muda e **por quê** (link para o plano ou a conversa que originou).
- **Como testar**: passos, dados de exemplo, e o que foi verificado (testes, typecheck, chamada
  real).
- **Deploy**: migration? depende do deploy de outro repositório? variável de ambiente nova?
- Capturas de tela quando muda interface.

Uma PR, um assunto. Atualização de dependência maior, refactor e feature vão separados.

## 8. Comentários no código

Regras completas no `SKILL.md` (§10). Resumo: comente o **porquê**, registre o incidente que
originou a linha, doc-comment em exportado de uso não óbvio, nada de código comentado, e mude o
comentário junto com a regra.
