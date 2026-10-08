---
name: dev-defaults
description: Padrões de desenvolvimento compartilhados pelo time para criar aplicações e sites do mesmo jeito — escolha de stack (front-end, back-end, CSS/UI), banco de dados, monorepo ou repositórios separados, estrutura de pastas, componentização, UI/UX, autorização, segurança, testes, documentação, comentários e heurísticas de trabalho. Use sempre que for iniciar um projeto ou site, escolher framework, banco ou estrutura de repositório, criar módulo de backend, endpoint, migration, tela, componente, modal ou formulário, escrever README/changelog/documentação, revisar código de alguém, ou quando a pergunta for "qual o padrão?", "onde eu coloco isso?", "qual stack usar?", "vale monorepo?", "como documento isso?" — mesmo que a pessoa não cite padrões explicitamente.
---

# dev-defaults

Os padrões que o time usa para que qualquer pessoa — ou qualquer sessão do Claude — crie
aplicações e sites do mesmo jeito. Cada regra vem com o **porquê**: é ele que permite decidir
bem nos casos que este guia não previu. Copiar a forma sem entender o motivo produz código que
parece certo e falha na operação.

Os padrões são neutros de domínio. Exemplos (pedidos, unidades, vagas) são ilustrativos —
troque pelo vocabulário do projeto. Decisões específicas da empresa ficam separadas em
`references/empresa.md` e **valem por cima** das gerais quando o projeto é da casa.

## Como usar

Este arquivo resume cada decisão. O detalhe e os exemplos estão nas referências — leia a que
corresponde ao que você vai fazer, não todas:

| Vou... | Leia |
|---|---|
| iniciar projeto, escolher framework, CSS, back-end | `references/stack.md` |
| escolher banco, modelar, escrever migration ou SQL | `references/banco-de-dados.md` |
| decidir monorepo × repositórios, ou mudar contrato entre front e back | `references/repositorios.md` |
| criar tela, componente, modal, hook de dados, tabela | `references/frontend.md` |
| desenhar a interface, textos, estados, acessibilidade | `references/ui-ux.md` |
| criar módulo, endpoint, guard, integração, autenticação | `references/backend.md` |
| escrever README, docs, changelog, commit, PR | `references/documentacao.md` |
| conduzir a tarefa (analisar, validar, entregar) | `references/heuristicas.md` |
| trabalhar num projeto da empresa | `references/empresa.md` |

Quando um padrão daqui conflitar com uma convenção **já estabelecida** no projeto em que você
está, siga o projeto e aponte a divergência — refatorar por conta própria no meio de outra
tarefa mistura mudanças e dificulta a revisão.

## 1. Stack

**Front-end — Next.js é o padrão** quando o projeto precisa do que ele já resolve: rotas,
renderização no servidor, middleware de sessão, SEO, API junto do front. Para tela única,
painel estático ou ferramenta interna simples, **TypeScript puro com Vite** é mais honesto:
menos dependência, menos manutenção.

**CSS e componentes — shadcn/ui quando há UI de produto** (formulário, tabela, modal, select,
toast). O ganho não é estético: são componentes acessíveis que moram no repositório. **Tailwind
puro** quando não há esses componentes. Nunca duas bibliotecas de componentes no mesmo app.

**Back-end — NestJS é o padrão** para backend próprio: módulos, injeção de dependência, guards
e validação padronizados, então quem chega encontra a mesma estrutura em todo domínio. Fuja
dele em dois casos, e diga qual se aplica: o ecossistema não cobre o requisito, ou outro
caminho tem custo **operacional** menor (route handlers do Next para backend pequeno, função
serverless para webhook, Fastify para dois endpoints).

**TypeScript sempre**, nas duas pontas. A pergunta que decide qualquer peça da stack: *o que
custa mais caro daqui a um ano — adotar isto ou não ter adotado?*

## 2. Banco de dados

**PostgreSQL + Prisma é o padrão**: relacional, transações sérias, JSON quando precisa, e
schema versionado com migrations revisáveis em PR. Redis entra para cache, filas, locks e
limites de requisição — nunca como fonte de verdade. SQLite só para ferramenta local ou teste.
Banco de documentos só com motivo concreto que o Postgres não atenda (raro).

Regras que mais evitam incidente:

- **Migration escrita para ser lida**, com comentário do porquê; coluna nova em tabela com
  dados entra nullable ou com default, e o código trata o nulo como "comportamento antigo".
- **Fuso explícito**: `timestamptz`, e conversão consciente para o fuso de negócio. Data com
  semântica de dia (agenda, vencimento) é comparada por dia, nunca por instante local.
- **SQL cru em arquivos `.sql`** carregados pelo código, não em strings espalhadas.
- **Produção é leitura por padrão** — toda escrita é mostrada antes e confirmada.
- **Nunca mude o default de uma coluna existente sem olhar os dados**: um default antigo nunca
  usado pode estar preenchido em todas as linhas e virar regra no dia em que o código passar a
  lê-lo.

## 3. Monorepo ou repositórios separados

**Monorepo** quando o projeto é pequeno, front e back sobem juntos, não há backend separado ou
existe código compartilhado real (tipos, validação, design system). Ganho: uma PR, uma versão,
tipos sem cópia manual.

**Repositórios separados** quando os deploys são independentes (times ou cadências
diferentes), um serviço precisa escalar ou reiniciar sem o outro, ou há fronteira de acesso.

A consequência que precisa ser dita em voz alta: **toda mudança de contrato tem que sobreviver
a um deploy parcial** — front novo com API velha e vice-versa. Campo novo entra opcional, rota
nova em vez de assinatura alterada, e a entrega diz o que depende de qual deploy.

## 4. Estrutura de pastas

O princípio em todo cenário: **agrupar por domínio, não por tipo de arquivo.** `pedidos/` com
controller, service, DTO e teste juntos é mais fácil de navegar e de apagar do que
`controllers/`, `services/` e `dtos/` com um pedaço de cada domínio em cada pasta.

```
api/                                   web/
├── prisma/schema.prisma + migrations  ├── app/(private)/<area>/<sub>/page.tsx
└── src/                               ├── components/ui/          # shadcn
    ├── common/   # guards, decorators ├── components/custom/<area>/
    ├── sql/      # SQL cru + loader   ├── components/custom/modals/<area>/
    └── <dominio>/                     ├── core/api.ts             # cliente HTTP único
        ├── <dominio>.module.ts        ├── hooks/<dominio>.ts
        ├── <dominio>.controller.ts    ├── lib/                    # helpers puros
        ├── <dominio>.service.ts       ├── models/<dominio>.dto.ts # tipos do back
        ├── dto/  entities/  guards/   └── validations/<dominio>.ts
        └── <dominio>.<aspecto>.spec.ts
```

Monorepo: `apps/web`, `apps/api`, `packages/types` (contratos), `packages/ui` (se mais de um
app consome), `packages/config`. Next sem backend próprio: `app/api/<dominio>/route.ts` como
borda HTTP e a regra em `server/<dominio>/service.ts`. Projeto simples: `src/{ui,lib,styles}`
— não invente camadas que ninguém pediu.

## 5. Front-end

A página é o orquestrador: estado, dados e composição. Extraia componente quando o bloco se
repete, tem estado próprio ou a página ficou difícil de ler — não por contagem de linhas.

- **Um cliente HTTP só**; hooks de domínio expõem só as funções que chamam a API; o cache
  (react-query) fica visível na tela, com chave contendo tudo que muda o resultado.
- **Modal = um arquivo por ação**: recebe `open`/`onOpenChange` e os dados, faz a própria
  submissão, invalida e fecha no sucesso, **não fecha no erro**.
- **Formulário** com react-hook-form + zod, schema em `validations/`, espelhando a regra do
  backend — que continua sendo a fonte de verdade.
- **Esconder botão é UX; autorização é do backend.**

Detalhe em `references/frontend.md`.

## 6. UI/UX

A interface serve a quem opera, muitas vezes horas por dia. Prioridade nesta ordem: **dado
correto e legível → ação óbvia → consistência → beleza.**

- Todo componente que carrega dados tem os **quatro estados**: carregando, vazio, erro e com
  dados. O vazio e o erro dizem o que aconteceu e o que fazer.
- **Feedback para toda ação**, repassando a mensagem do backend; botão desabilitado durante a
  submissão (clique duplo vira registro duplicado).
- **Destrutivo pede confirmação** dizendo exatamente o que será afetado.
- **Tokens de tema**, modo escuro funcionando, responsivo até celular, foco visível, contraste
  AA, nada comunicado só por cor.
- **Textos no idioma do usuário**, no vocabulário da operação, sem jargão técnico.

Detalhe em `references/ui-ux.md`.

## 7. Back-end

- **Controller fino** (HTTP → service); regra de negócio no service, onde o teste alcança.
- **Autorização pelo escopo do dado, não só pelo papel** — papel diz o que a pessoa é, escopo
  diz sobre o que ela manda. Helpers reutilizáveis, não `if` espalhado.
- **Escrita concorrente usa claim atômico**: condição no `where` do update e a contagem como
  prova; conflito (409) para quem perdeu.
- **Parâmetro de negócio mora no banco** (linha de configuração), não em constante.
- **Erro com mensagem para o usuário final**; erro do ORM traduzido para significado (409, 404).
- **Segurança**: limite de tentativas por usuário (e-mail), não só por IP; segredo nunca em log,
  resposta ou trilha de auditoria; credencial gravada só depois de entregue.

Detalhe em `references/backend.md`.

## 8. Documentação

Documente o que o código não conta: **por que** foi feito assim, **como** rodar e **o que**
mudou para quem usa.

- **README por repositório**: o que é, como rodar, variáveis de ambiente, como publicar.
- **Plano antes de feature grande** (`docs/<assunto>-plano.md`) com levantamento, decisões
  tomadas e as em aberto — e status atualizado quando implementar.
- **API documentada no próprio código** (OpenAPI/Swagger) com descrições de regra, não de tipo.
- **Changelog para o usuário**, escrito do ponto de vista de quem usa, com SemVer honesto.
- **Commits e PRs** dizendo o porquê; o diff já diz o quê.

Detalhe e modelos em `references/documentacao.md`.

## 9. Testes

Ao lado do código, um arquivo por aspecto quando o domínio cresce. O nome descreve a **regra**,
não o método:

```ts
it('recusa agendar quando o horário acabou de ser ocupado por outro pedido', ...)
```

Teste o que tem regra: cálculo, autorização, concorrência, transformação de dados. Regra pura
em função pura (fácil de testar sem subir framework). Comentário no teste registra o incidente
que o originou. Teste que fixa comportamento errado vira armadilha: ao mudar uma regra,
atualize a asserção e avise. Em desenvolvimento, exercite também a chamada real — rota, guard e
serialização só aparecem nela.

## 10. Comentários

Comente o **porquê**, nunca o quê. O código diz o que faz; o que ele não conta é a restrição
escondida, o bug que aquela linha evita, a decisão de negócio por trás da condição.

```ts
// findFirst, e não findUnique: o findUnique compara o rótulo por igualdade exata, e em produção
// ele está gravado com outra caixa — a consulta não achava o registro e barrava o time inteiro.
```

- **Comentário registra o incidente**: sem ele, a próxima pessoa "limpa" a linha e o bug volta.
- **Doc-comment (`/** */`)** em função, tipo e campo exportados cujo uso não é óbvio pelo nome;
  vira dica no editor e documentação da API.
- **Idioma**: comentários no idioma do time; identificadores em inglês.
- **Sem código comentado** (o histórico está no Git) e **sem comentário que repete o código**.
- **TODO** só com contexto: o que falta, por quê, e quem/quando decide.
- Comentário desatualizado é pior que nenhum — ao mudar a regra, mude o comentário.

## 11. Heurísticas de trabalho

Valem tanto quanto as convenções de código. Detalhe em `references/heuristicas.md`.

1. **Pedido de análise é análise** — diagnóstico, evidência e opções; implementar só quando
   pedido ou confirmado.
2. **Confirme decisões de produto antes de codificar**, já trazendo o contexto investigado.
3. **Valide contra dados reais** (em leitura) antes e depois — diagnóstico com número.
4. **Mudou regra de cálculo? Mostre o delta** entre a regra antiga e a nova sobre os dados reais.
5. **Produção é leitura por padrão**; escrita é apresentada e aguarda confirmação.
6. **Limpe o que criou para testar** e diga que limpou.
7. **Nunca afirme verificação que não fez** — diga o que foi checado e o que ficou de fora.
8. **Corrija o próprio diagnóstico em voz alta** quando ele estava errado.
