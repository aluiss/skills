# Backend

Leia ao criar um módulo, um guard, uma migration, uma integração externa, ou ao mexer em algo que
várias pessoas escrevem ao mesmo tempo. Os exemplos usam NestJS + Prisma; os princípios valem
para qualquer framework com injeção de dependência e ORM.

## Índice

1. Anatomia de um módulo
2. DTOs e validação
3. Autorização: papel × escopo
4. Guards e onde a checagem realmente cabe
5. Concorrência: claim atômico
6. Parâmetro de negócio no banco
7. Integração externa e sincronização
8. Datas e normalização de texto
9. Auditoria
10. Migrations
11. Erros
12. Segurança e autenticação
13. Consultas pesadas e cache

## 1. Anatomia de um módulo

```
src/<dominio>/
├── <dominio>.module.ts
├── <dominio>.controller.ts     # rotas, documentação, guards — sem regra de negócio
├── <dominio>.service.ts        # regra de negócio e acesso a dados
├── dto/                        # entrada validada
├── entities/                   # tipos de consulta/resposta
├── guards/                     # autorização específica do domínio
└── <dominio>.<aspecto>.spec.ts
```

Um domínio pode ter mais de um service quando as responsabilidades têm naturezas diferentes
(integração externa × leitura paginada × regra de negócio). Isso é melhor que um service de duas
mil linhas onde ninguém acha nada.

Rota estática antes de rota paramétrica no mesmo controller (`GET /disponibilidade` antes de
`GET /:id`), senão o roteador casa o parâmetro primeiro e a rota específica nunca executa.

## 2. DTOs e validação

Validação declarativa na entrada, com descrição que explica a **regra**, não o tipo — ela vira a
documentação de quem consome a API.

```ts
export class UpdateLimitesDto {
  /** Dias até os quais o cliente é considerado novo. */
  @IsInt() @Min(1) @Max(36500)
  diasNovo: number;
}
```

Dois pontos que costumam morder:

- **Booleano vindo de querystring**: com conversão implícita, `Boolean('false')` é `true` e o
  filtro passa a mentir. Transforme lendo o valor cru e comparando com `'true'`.
- **Body em array** frequentemente escapa do pipe de validação. Se existir rota de criação em
  lote, repita a barreira dentro do service — regra que só existe no DTO some no lote.

## 3. Autorização: papel × escopo

Papel (`ADMIN`, `USER`) diz **o que a pessoa é**. Escopo (setor, unidade, cliente, projeto) diz
**sobre o que ela manda**. A maior parte das decisões reais é de escopo:

- quem opera uma unidade age nela mesmo com papel baixo;
- quem tem papel alto em outra unidade não deve agir ali;
- um papel global (dono/superusuário) passa por cima de tudo.

Concentre isso em helpers reutilizáveis — algo como `assertScopeAccess(actor, escopoId)` para
mutações e um `scopeWhere(actor)` que devolve o filtro das listagens. Regra de escopo espalhada
em `if` dentro de cada método diverge em semanas.

Cuidado com o identificador do usuário: se o sistema separa a credencial (login/sessão) da pessoa
(usuário de domínio), as FKs de negócio apontam para a **pessoa**. Misturar os dois ids gera
vínculo silenciosamente errado.

## 4. Guards e onde a checagem realmente cabe

Guard resolve bem quando a autorização depende de um dado disponível na requisição:

```ts
@Injectable()
export class UnidadeGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const user = ctx.switchToHttp().getRequest().user;
    if (!user) throw new ForbiddenException('Usuário não autenticado.');
    if (user.role === 'OWNER') return true;
    if (user.unidadeId !== Number(ctx.switchToHttp().getRequest().params.unidadeId)) {
      throw new ForbiddenException('Você não tem acesso a esta unidade.');
    }
    return true;
  }
}
```

**Quando o guard não serve:** rota aninhada (`PATCH /itens/:itemId`) não carrega o escopo. Aí a
checagem vai no service, resolvendo o escopo pelo recurso pai:

```ts
private async getItemDoEscopo(itemId: number, actor: AuthUser) {
  const item = await this.prisma.itens.findUnique({
    where: { id: itemId },
    include: { pedido: { select: { unidadeId: true } } },
  });
  if (!item) throw new NotFoundException('Item não encontrado.');
  assertScopeAccess(actor, item.pedido.unidadeId);
  return item;
}
```

Não deixe guard escrito e não aplicado: vira código morto que dá falsa sensação de segurança em
revisão. Se ele existe, alguma rota o usa — ou ele sai.

## 5. Concorrência: claim atômico

Duas pessoas agindo sobre o mesmo recurso ao mesmo tempo é rotina, não exceção. Ler o registro,
decidir e escrever faz a segunda sobrescrever a primeira **sem erro nenhum** — o pior tipo de
bug, porque ninguém percebe até alguém reclamar de trabalho perdido.

O padrão é colocar a condição no `where` da atualização e usar a contagem como prova:

```ts
const claimed = await this.prisma.slots.updateMany({
  where: { id, pedidoId: null, status: 'DISPONIVEL' },
  data: { pedidoId, status: 'OCUPADO', updatedAt: now },
});
if (claimed.count === 0) {
  throw new ConflictException('Este horário acabou de ser ocupado — escolha outro.');
}
```

Com várias candidatas, itere tentando o claim em cada uma e só desista no fim: perder uma corrida
não deve virar erro se ainda há alternativa.

Centralize o critério de "disponível" numa função e reutilize-a na reserva, na ocupação e na
contagem. Critérios duplicados divergem, e o contador passa a mostrar um número que a operação
recusa.

## 6. Parâmetro de negócio no banco

Prazo, faixa, limite, percentual — tudo que a operação ajusta sem deploy — é tabela de linha
única (`id` fixo) lida a cada uso, com `upsert` na escrita e registro de quem alterou:

```ts
async update(valores: Limites, actor: AuthUser) {
  if (valores.minimo >= valores.maximo) {
    throw new BadRequestException('O limite mínimo precisa ser menor que o máximo.');
  }
  return this.prisma.configuracao.upsert({
    where: { id: 1 },
    create: { id: 1, ...valores, updatedAt: new Date(), updatedById: actor.usersId },
    update: { ...valores, updatedAt: new Date(), updatedById: actor.usersId },
  });
}
```

Sem linha configurada = defaults embutidos no código. Nunca assuma que a linha existe.

## 7. Integração externa e sincronização

Padrão: conexão direta com a origem, consultas em arquivos `.sql` carregados pelo código, job
periódico, espelho local para leitura rápida e paginada.

Cuidados que custam incidente quando ignorados:

- **Lock** (advisory lock ou equivalente) para não rodar duas sincronizações ao mesmo tempo.
- **Um único carimbo de tempo** gerado no início e usado tanto na gravação quanto na poda do que
  não foi tocado. Comparar contra `now()` do banco em coluna de precisão menor arredonda a fração
  e apaga a rodada inteira — o espelho zera de forma intermitente e difícil de reproduzir.
- **Só pode quando a origem devolveu linhas**, senão uma consulta vazia esvazia o espelho.
- **Cursor incremental com janela de sobreposição**, para não perder registro por diferença de
  relógio entre as pontas.
- Arquivos `.sql` lidos uma vez na inicialização **não recarregam** com hot reload; em
  desenvolvimento, force reinício real ao alterá-los.

## 8. Datas e normalização de texto

- Data com semântica de dia (agenda, vencimento) é normalizada para meia-noite UTC na entrada e
  comparada por dia. Comparar `Date` local com coluna de data desloca o dia em fusos negativos.
- Texto que vira regra (título, rótulo, categoria) passa por uma normalização única: remove
  acento, converte hífen em espaço, colapsa espaços, caixa alta. **As listas de termos comparadas
  passam pela mesma função** — uma entrada acentuada comparada contra texto normalizado nunca
  casa, e o erro fica invisível quando outro termo genérico cobre o caso por acidente.
- Quando duas regras disputam o mesmo texto, resolva por **termo mais específico com veto
  explícito**, não pela ordem das listas:

```ts
if (ehExcecao(texto)) return false;        // veto: a regra mais específica ganha
return TERMOS_NORMALIZADOS.some((t) => normalizado.includes(t));
```

## 9. Auditoria

Registre a ação de domínio, não só o `UPDATE` genérico: a tela de histórico precisa dizer
"reservou", "liberou", "finalizou".

Quando o diff campo a campo importa, prefira uma atualização por linha a uma em lote — o lote
grava uma entrada sem detalhe e o usuário perde rastreabilidade justamente onde ela é mais
necessária. Exclusão merece entrada explícita dizendo o que sumiu e por quê.

## 10. Migrations

Regras completas de modelagem, migrations, fuso e SQL em `banco-de-dados.md`. O essencial:
escritas à mão quando o gerador não expressa a intenção, sempre com comentário do porquê:

```sql
-- Turno do atendimento. A capacidade de um turno num dia é o número de slots criados com ele.
-- Registros anteriores ficam NULL ("sem turno") e mantêm o comportamento antigo.
ALTER TABLE "slots" ADD COLUMN "turno" "Turno";
CREATE INDEX "slots_dia_turno_idx" ON "slots"("data", "turno");
```

- Coluna nova em tabela com dados entra **nullable** (ou com default), e o código trata o nulo
  como "comportamento antigo".
- Unicidade condicional = índice único parcial (`WHERE col IS NOT NULL`), porque `NULL` não
  conflita em `UNIQUE` comum.
- Colunas mutuamente exclusivas ganham `CHECK` no banco, além da validação no DTO: o banco é a
  última linha de defesa contra escrita feita fora da aplicação.

## 11. Erros

Exceções com mensagem escrita **para o usuário final**, no idioma dele — ela chega à tela. Diga o
que houve e o que fazer:

```
'Este horário acabou de ser ocupado — escolha outro.'
'Não há horário disponível no período da tarde em 20/09 para esta unidade.'
```

Traduza erro do ORM para significado de negócio: violação de unicidade → conflito (409);
registro não encontrado na atualização → 404. Vazar código de erro técnico para a tela é jogar
o problema no colo de quem não pode resolvê-lo.

## 12. Segurança e autenticação

**Sessão**: token de acesso curto e refresh longo, ambos em **cookie httpOnly** (nunca em
`localStorage`, onde qualquer script injetado lê). Troca de senha ou mudança de perfil invalida
os tokens emitidos antes dela.

**Limite de tentativas em duas camadas**, porque proteger só por IP derruba escritórios
inteiros — em empresa, dezenas de pessoas saem pelo mesmo IP:

- **Por conta**: N senhas erradas seguidas travam a conta por uma janela crescente (5, 15, 60
  min); só um login bem-sucedido zera o contador. Incremento **atômico**, senão tentativas
  paralelas escapam da contagem.
- **Por identificador** (e-mail) no limitador de requisições, com um teto **alto** por IP só
  contra rajada.
- Mensagem de recusa igual para "e-mail não existe" e "senha errada" — a diferença revela quais
  contas existem.

**Senha**:

- Regra de composição conferida no backend (o formulário só espelha).
- Hash com bcrypt; a senha nunca vai para log, resposta (exceto a senha gerada, uma única vez,
  para quem a gerou) ou trilha de auditoria.
- **Credencial gerada só é gravada depois de entregue.** Gravar a senha nova e depois tentar
  enviar o e-mail deixa a pessoa sem acesso quando o envio falha — e cada nova tentativa troca
  a senha de novo. Envie primeiro (ou exiba) e grave em seguida.
- Senha definida por outra pessoa (administrador) obriga a troca no primeiro login, aplicada
  **na API** (guard que recusa tudo exceto trocar a senha, carregar a sessão e sair), não só na
  tela.

**E-mail e serviços externos**: falha no envio de um aviso que é só cortesia não desfaz a
operação principal — registre no log e siga. Antes de culpar o código, teste a credencial do
serviço (ex.: `verify()` do SMTP) sem enviar nada.

**Geral**: `helmet` ligado, CORS só para as origens do front, segredos só em variável de
ambiente (nunca no repositório nem em log), rota pública marcada explicitamente
(`@Public()`), validação global com `whitelist` e `forbidNonWhitelisted`.

## 13. Consultas pesadas e cache

Consulta cara a uma base externa que alimenta tela com atualização automática:

- **Cache no servidor** com TTL igual ao intervalo de atualização da tela, e **requisições
  simultâneas esperando a mesma promessa** — senão cada pessoa com a tela aberta dispara a sua
  consulta.
- **Pool próprio e pequeno**, separado do da sincronização, com timeout de consulta.
- Falha da fonte vira erro claro para a tela (503 com mensagem), e a tela mantém o último dado.
- Lista que muda pouco (catálogo de funcionários, cadastros de outro sistema): cache longo
  (horas) com um botão "atualizar agora".

