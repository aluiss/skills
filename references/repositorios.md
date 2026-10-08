# Monorepo ou repositórios separados

Leia ao iniciar um projeto, ao separar um serviço, ou ao mudar um contrato entre front e back.

## Quando cada um

**Monorepo quando:**
- projeto pequeno, ou front e back que sobem juntos;
- não há backend separado (só route handlers);
- existe código compartilhado real — tipos, validação, design system.

Ganho: uma PR, uma versão, tipos compartilhados sem publicar pacote, refactor cruzado em um
commit.

**Repositórios separados quando:**
- os deploys são independentes (times ou cadências diferentes);
- um serviço precisa reiniciar ou escalar sem afetar o outro;
- há fronteira de acesso clara entre os códigos.

Na dúvida num projeto novo e pequeno, comece em monorepo: separar depois é mais barato que
juntar.

## Estrutura de monorepo

```
apps/
├── web/
└── api/            # quando existir
packages/
├── types/          # contratos compartilhados entre front e back
├── ui/             # design system, se mais de um app consome
└── config/         # eslint, tsconfig, tailwind compartilhados
```

`packages/types` existe por um motivo concreto: em repositórios separados o tipo do front é
escrito à mão espelhando o backend, e as pontas divergem em silêncio. No monorepo esse risco é
evitável — aproveite.

Não crie `packages/utils` genérico. Pacote sem dono vira depósito; mantenha o utilitário dentro
do app até aparecer um segundo consumidor real.

## A regra do deploy parcial

Com repositórios separados, **toda mudança de contrato tem que sobreviver a um deploy parcial**
— front novo com API velha e vice-versa. Em monorepo a disciplina fica mais barata, mas não
desaparece: versões antigas da tela continuam abertas no navegador de alguém.

- Campo de entrada novo entra **opcional**, com o comportamento antigo preservado quando ausente;
  o caminho antigo só sai depois que a outra ponta estiver em produção.
- **Rota nova** em vez de assinatura alterada, quando a mudança quebraria o cliente atual.
- Campo removido da resposta: confira que o front não lê mais antes de tirar.
- **Ao entregar, diga o que depende de qual deploy**: "API e web precisam subir juntos" ou
  "sem o deploy do web, o comportamento X continua o antigo". Migration que precisa vir antes
  também é dita.

## Versionamento e branches

- Branch principal sempre publicável; trabalho em branches curtas, integradas por PR.
- Uma PR, um assunto. Atualização de dependência maior, refactor e feature vão separados.
- Tag/versão por release quando o projeto publica para usuários (casando com o changelog).
