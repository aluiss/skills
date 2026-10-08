# Decisões da empresa (NOVAISP)

Leia ao trabalhar em qualquer projeto da NOVAISP. Estas decisões **valem por cima** dos padrões
gerais do `SKILL.md` quando houver diferença. Cada uma tem data e motivo — se a realidade mudou,
atualize aqui em vez de contornar no código.

## Índice

1. Idioma
2. Papéis e autorização
3. Ambientes e dados
4. Fuso horário e datas
5. Sistemas externos
6. Repositórios e deploy
7. Contas e acesso
8. Operação

## 1. Idioma

- **Interface, mensagens de erro, changelog, documentação e respostas ao time em português do
  Brasil.**
- Identificadores no código em inglês; comentários em português.
- Mensagens de commit seguem o padrão de `documentacao.md` (resumo em inglês).

## 2. Papéis e autorização

- Papéis: **Super Admin** (`OWNER`), **Administrador** (`ADMIN`), **Supervisor** (`SUPERVISOR`)
  e **Colaborador** (`USER`). Use esses nomes em toda tela.
- **O Super Admin está sempre acima de todas as regras** e pode fazer qualquer ação (decisão de
  2026-10-02). Toda regra nova de autorização — guard, verificação na tela, filtro de escopo —
  começa liberando o `OWNER`, mesmo quando a regra é "só o departamento X". Se um pedido parecer
  exigir restringir o Super Admin, pergunte antes.
  *Exibição não é autorização*: deixar o Super Admin fora de uma lista de técnicos, por
  exemplo, não conflita com a regra.
- Acesso a páginas e ações é **configurável**: catálogo de recursos (`access_resources`) +
  concessões por papel, departamento, equipe ou pessoa (`access_grants`). Página ou ação nova
  entra no catálogo **por migration**, com as concessões iniciais decididas no plano — e a rota
  da tela é cadastrada, senão fica sem checagem.
- Escopo por setor: supervisor e administrador mandam no próprio departamento e nos que
  supervisionam.
- Rótulos de departamento e equipe variam de grafia entre ambientes ("Área Técnica" ×
  "ÁREA TÉCNICA"): compare sempre normalizado (sem acento, caixa alta).

## 3. Ambientes e dados

- **Produção é somente leitura** para investigação. Qualquer escrita (correção de dados, script,
  sincronização manual) é apresentada e só roda com confirmação explícita.
- O **banco de desenvolvimento é base de teste**: campos operacionais costumam estar vazios lá e
  preenchidos em produção. Conclusões sobre volume, distribuição e comportamento real se
  confirmam em produção, em leitura.
- Migrations são aplicadas no dev durante o desenvolvimento; produção recebe no deploy. Toda
  entrega diz se há migration pendente para produção.
- Ao testar em dev com usuário real de teste, diga o que ficou alterado (senha, registros).

## 4. Fuso horário e datas

- Fuso de negócio: **`America/Sao_Paulo`**. Os bancos da aplicação (dev e prod) estão nesse
  fuso desde 2026-09-29 — SQL cru com `now()` comparado a colunas sem fuso pode deslocar 3 h.
- "Hoje", "vence hoje" e virada de dia são pelo calendário de São Paulo.
- O ERP grava `timestamp` sem fuso em hora de São Paulo: converta na consulta
  (`AT TIME ZONE 'America/Sao_Paulo'`).

## 5. Sistemas externos

- **ERP (Voalle)**: acesso somente leitura — espelho Postgres (consultas em `src/sql/*.sql`) e
  a API de dados da empresa, com chave **por domínio** (cada chave só abre o próprio domínio).
  Ids do ERP nunca são chave primária local (coluna própria, ex.: `elleven_id`).
- Ligação entre pessoas do Workspace e do Voalle por **id** (vínculo gravado), nunca por nome:
  os nomes no Workspace são abreviados e não batem por igualdade.
- E-mail sai pela caixa `naoresponda@` no servidor da empresa. Envio de credencial segue a regra
  de `backend.md` §12 (gravar só depois de entregar).

## 6. Repositórios e deploy

- **API e web são repositórios separados**, publicados independentemente: toda mudança de
  contrato segue a regra do deploy parcial (`repositorios.md`) e a entrega diz se os dois
  precisam subir juntos.
- Changelog visível na aplicação (`web/lib/changelog.ts`), atualizado a cada entrega
  (`documentacao.md` §5).

## 7. Contas e acesso

- Login só com e-mail dos domínios da empresa (`@novaisp.com.br`, `@novanet-rj.com.br`).
- **A operação inteira acessa pelo mesmo IP**: limite de tentativas é por e-mail, com teto alto
  por IP (decisão de 2026-10-05, depois que o limite de 5/min por IP deixava o escritório 15 min
  sem login).
- Senha temporária gerada pelo Super Admin não depende de e-mail e obriga a troca no primeiro
  login.

## 8. Operação

- Telas de operação (Torre, CRC, Área Técnica) ficam abertas o dia todo, às vezes em TV:
  siga `ui-ux.md` §10 (densidade, atualização automática, idade do dado visível).
- A mesma máquina é usada por várias pessoas: preferências e filtros guardados no navegador
  levam o id do usuário na chave.
