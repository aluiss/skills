# Banco de dados

Leia ao escolher banco, modelar tabelas, escrever migration ou SQL cru, integrar com base de
terceiro ou mexer em dados de produção.

## Índice

1. Escolha do banco
2. Modelagem e nomes
3. Migrations
4. Datas e fuso horário
5. SQL cru e bases de terceiros
6. Desempenho
7. Transações
8. Dados de produção

## 1. Escolha do banco

| Necessidade | Escolha | Por quê |
|---|---|---|
| Dado de negócio de qualquer aplicação | **PostgreSQL** | Relacional, transações sérias, índices parciais, JSON quando precisa, maduro em todo provedor |
| Cache, sessão, lock distribuído, limite de requisição, fila | **Redis** | Rápido e com expiração nativa — nunca a fonte de verdade |
| Ferramenta local, protótipo, teste | **SQLite** | Zero infraestrutura |
| Busca textual pesada, analytics | Postgres primeiro (`tsvector`, índices); serviço dedicado só quando medir que não basta | Cada banco a mais é backup, monitoramento e credencial a mais |

Banco de documentos (Mongo e afins) só com motivo concreto que o Postgres não atenda —
"o dado não tem esquema" quase sempre significa "o esquema ainda não foi pensado".

Acesso pela aplicação: **Prisma** (schema versionado, tipos gerados). Um banco por aplicação;
duas aplicações escrevendo na mesma tabela é contrato implícito que ninguém documenta.

## 2. Modelagem e nomes

- **Tabelas e colunas em `snake_case` no banco**, campos em `camelCase` no código (`@map` /
  `@@map` no Prisma). O banco é lido em SQL cru por gente que não conhece o ORM.
- **Chave primária** inteira autoincremental ou UUID — escolha uma por projeto. Id que vem de
  sistema externo **não** é a chave primária: guarde em coluna própria com índice único
  (`elleven_id`, `external_id`). Quando o sistema externo renumera, a sua base não quebra.
- **FK de verdade** no banco, com `onDelete` pensado (cascata só quando o filho não faz
  sentido sem o pai).
- **Enum** para conjunto fechado e estável; **tabela de cadastro** quando a gestão cria novos
  valores pela tela.
- **`created_at` / `updated_at`** em toda tabela de negócio. Exclusão lógica (`deleted_at`)
  quando há histórico ou auditoria a preservar; e então **todo** filtro de listagem considera
  a coluna.
- **Unicidade condicional** = índice único parcial (`WHERE col IS NOT NULL`), porque `NULL` não
  conflita em `UNIQUE` comum.
- **Colunas mutuamente exclusivas** ganham `CHECK` no banco, além da validação no DTO: o banco
  é a última defesa contra escrita feita fora da aplicação.

## 3. Migrations

Toda alteração de schema é uma migration versionada e revisada em PR. Escreva à mão quando o
gerador não expressa a intenção, sempre com comentário do porquê:

```sql
-- Turno do atendimento. A capacidade de um turno num dia é o número de slots criados com ele.
-- Registros anteriores ficam NULL ("sem turno") e mantêm o comportamento antigo.
ALTER TABLE "slots" ADD COLUMN "turno" "Turno";
CREATE INDEX "slots_dia_turno_idx" ON "slots"("data", "turno");
```

- **Coluna nova em tabela com dados entra nullable (ou com default)**, e o código trata o nulo
  como comportamento antigo. Obrigatória de cara quebra o deploy ou exige backfill inventado.
- **Antes de passar a usar uma coluna existente, olhe os dados.** Um default que nunca foi lido
  pode estar preenchido em todas as linhas: passar a respeitá-lo vira regra para todo mundo de
  uma vez. Zere ou ajuste os dados na mesma migration.
- **Migration de dados** (corrigir, preencher, apagar) separada da de schema quando possível, e
  com o recorte explícito no comentário.
- **Cadastros que o sistema precisa** (permissões, configurações) entram por migration com
  `ON CONFLICT DO NOTHING`, para rodar igual em todos os ambientes.
- **Nunca edite migration já aplicada** em outro ambiente — crie uma nova.
- Ordem de deploy: migration antes do código que depende dela; e o código novo precisa tolerar
  o schema antigo durante a janela de deploy.

## 4. Datas e fuso horário

- Instantes em **`timestamptz`**. Coluna sem fuso guarda "hora de algum lugar" e cada leitor
  interpreta de um jeito.
- O **fuso de negócio** (ex.: `America/Sao_Paulo`) é explícito no código e no SQL
  (`AT TIME ZONE`), nunca o do processo ou do servidor.
- **Data com semântica de dia** (agenda, vencimento, competência): coluna `date`, ou meia-noite
  UTC normalizada na entrada, comparada por dia. Comparar `Date` local com coluna de data
  desloca o dia em fusos negativos.
- "Hoje" é pelo calendário do fuso de negócio — 22h em São Paulo já é amanhã em UTC.
- Base de terceiro com `timestamp` sem fuso: descubra em qual fuso ela grava e converta na
  própria consulta.

## 5. SQL cru e bases de terceiros

SQL cru é bem-vindo onde é melhor — relatório, agregação pesada, base de terceiro. Mantenha-o
em **arquivos `.sql`** carregados pelo código: dá para ler, versionar e rodar no console durante
uma investigação. Cabeçalho do arquivo explica a origem e as diferenças em relação à consulta
original, se foi adaptada de outra.

- **Parâmetros sempre por placeholder**, nunca concatenação de string.
- **Cuidado com tipos**: drivers devolvem `bigint` como string e `timestamp` no fuso do
  processo; converta na consulta (`::int`, `AT TIME ZONE`).
- **Deduplicação explícita** (`DISTINCT ON`) quando um `JOIN` pode multiplicar a linha.
- **Base de terceiro é leitura**: usuário somente leitura, pool próprio e pequeno, timeout de
  consulta, e cache quando a tela atualiza sozinha (uma consulta por ciclo para todos, não uma
  por pessoa com a tela aberta).
- **Sincronização para espelho local**: lock contra execução dupla, um único carimbo de tempo
  para gravar e podar, só podar quando a origem devolveu linhas, cursor incremental com janela
  de sobreposição. Detalhe em `backend.md`.

## 6. Desempenho

- Índice para toda coluna usada em filtro frequente, junção ou ordenação de listagem.
- Expressão (CASE, COALESCE, função) não usa índice: quando o recorte é por expressão, faça um
  **pré-filtro indexável** (superconjunto) e o corte exato depois.
- Meça com `EXPLAIN ANALYZE` na base real antes de otimizar — o gargalo raramente está onde se
  imagina.
- Paginação no servidor para volumes grandes; nunca buscar tudo para paginar em memória.

## 7. Transações

- Use quando várias escritas precisam acontecer juntas ou nenhuma.
- **Transação interativa tem prazo** (no Prisma, 5 s por padrão). Trabalho por item dentro
  dela — auditoria, leituras extras — estoura o prazo num lote grande. Prefira transação em lote
  (lista de operações) quando as escritas não dependem umas das outras.
- Nada de chamada externa (e-mail, API) dentro da transação: ela segura conexão e trava linhas
  esperando a rede.

## 8. Dados de produção

- **Leitura por padrão.** Investigação usa sessão somente leitura.
- **Toda escrita é apresentada antes**: o comando, o recorte, quantas linhas atinge e o que não
  será tocado — e só roda com confirmação explícita.
- Correção em massa: rode primeiro a contagem, depois a correção com o mesmo `WHERE`.
- Backup/ponto de restauração conhecido antes de migration destrutiva.
- Banco de desenvolvimento é base de teste: campos operacionais podem estar vazios lá e
  preenchidos em produção. Conclusões sobre volume e distribuição se confirmam em produção
  (em leitura).
