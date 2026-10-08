# Front-end

Leia ao criar uma tela, um componente, um modal, um hook de dados ou uma tabela. Os exemplos
usam Next.js (App Router) + shadcn/ui + react-query; os princípios valem para qualquer stack
equivalente. Decisões de aparência, texto e acessibilidade estão em `ui-ux.md`.

## Índice

1. Estrutura de rotas e o papel da página
2. shadcn/ui e Tailwind: quando cada um
3. Camada de dados: cliente HTTP → hook → cache
4. Tipos e validação
5. Componentização na prática
6. Modais
7. Tabelas e paginação
8. Navegação e permissão
9. Feedback ao usuário
10. Estado lembrado e telas que se atualizam sozinhas

## 1. Estrutura de rotas e o papel da página

```
app/(private)/<area>/<sub>/page.tsx
```

Um layout por aplicação (ou por grupo de rotas quando o shell muda de verdade), não um por área.
A página contém a tela: estado, chamadas de dados, composição. Ela pode ser longa; o que não pode
é esconder regra de negócio dentro de JSX repetido.

Página de índice de área redireciona para a primeira rota que o usuário pode ver, em vez de
renderizar um conteúdo próprio que ninguém mantém.

Rota dinâmica: converta o parâmetro uma vez no topo (`const id = Number(params.id)`). Rota que lê
query string precisa de fronteira de `Suspense` — exporte uma casca com `<Suspense>` e um
componente de conteúdo com a lógica.

## 2. shadcn/ui e Tailwind: quando cada um

**shadcn/ui quando houver UI de produto**: formulário, tabela, modal, select, toast, popover. O
motivo não é aparência — são componentes acessíveis (foco, teclado, ARIA) que ficam no seu
repositório, então você adapta sem esperar release de terceiro.

**Tailwind puro quando o projeto é simples**: landing page, painel de leitura, ferramenta de tela
única. Instalar um sistema de componentes para estilizar poucas divs é custo sem retorno.

Regras de convivência:

- `components/ui/` é a base gerada pelo shadcn. Ajuste quando precisar (é seu código), mas evite
  regra de domínio ali — ela pertence a `components/custom/`.
- Não misture uma segunda biblioteca de componentes. Dois sistemas de design no mesmo app
  significam dois jeitos de fazer modal e nenhum padrão.
- Tokens de cor e espaçamento no tema do Tailwind, não hardcoded por componente: tema escuro e
  ajuste de marca passam a ser uma mudança, não uma varredura.

## 3. Camada de dados: cliente HTTP → hook → cache

**Um cliente HTTP só** (`core/api.ts`), com base URL, credenciais e o interceptor de renovação de
sessão. Criar outra instância ou usar `fetch` à mão em uma tela específica é como a autenticação
começa a falhar só naquela tela.

**Hook de domínio** expõe apenas funções que chamam a API e devolvem os dados:

```ts
export function usePedidos() {
  async function listar(query: PedidoQuery = {}): Promise<PedidoListDTO> {
    const response = await api.get('/api/pedidos', { params: query })
    return response.data
  }
  async function criar(data: CriarPedidoDTO): Promise<PedidoDTO> {
    const response = await api.post('/api/pedidos', data)
    return response.data
  }
  return { listar, criar }
}
```

**Cache na tela**, com chave contendo tudo que muda o resultado:

```ts
const { data, isPending } = useQuery({
  queryKey: ['pedidos', unidadeId, page, search],
  queryFn: () => listar({ unidadeId, page, search }),
  enabled: !!unidadeId,
})
```

Mutação invalida as chaves afetadas — inclusive as indiretas (alterou o item? invalide também o
resumo do dia). Manter o cache visível na tela que o usa evita dado velho que ninguém sabe de
onde veio.

## 4. Tipos e validação

Quando front e back são repositórios separados, os tipos do front espelham os DTOs à mão em
`models/<dominio>.dto.ts`. Enums viram union de literais:

```ts
export type StatusPedido = 'ABERTO' | 'EM_ANDAMENTO' | 'CONCLUIDO';
```

Campo opcional no backend é opcional aqui. Divergência silenciosa entre as pontas só aparece em
runtime, geralmente na frente do usuário. Em monorepo, publique esses tipos em `packages/types` e
elimine a cópia.

Schemas de formulário (zod) ficam em `validations/<dominio>.ts`, com mensagens no idioma do
usuário. Use o tipo inferido do schema no formulário; declarar um tipo paralelo à mão gera
incompatibilidade difícil de ler.

## 5. Componentização na prática

Extraia quando o bloco se repete, tem estado próprio ou a página ficou difícil de ler — não por
contagem de linhas. Indireção prematura espalha o que era fácil de acompanhar num arquivo só.

Um sinal confiável de que falta extração: o mesmo `useState` + `useEffect` aparece em duas telas
para controlar a mesma coisa. Um sinal de que a extração foi longe demais: o componente recebe
oito props e metade delas só atravessa para outro componente.

Componente compartilhado entre áreas vive em `components/custom/` na raiz; específico de uma
área, dentro dela. Se um dia dois apps consumirem o mesmo componente, ele sobe para um pacote de
UI — não para uma pasta "shared" que vira depósito.

## 6. Modais

Um arquivo por ação. Contrato: recebe `open`, `onOpenChange` e os dados da linha; tem sua própria
submissão; invalida e fecha no sucesso; no erro, mostra o toast e **não fecha**, para a pessoa
corrigir sem redigitar.

Reset do formulário no efeito de abertura — sem isso, o estado do registro anterior vaza para o
próximo e alguém salva o dado errado.

Fluxo em etapas (escolher → conferir → confirmar) cabe num modal com estado local de passo. Não
adicione biblioteca de wizard por causa de três telas.

## 7. Tabelas e paginação

- Volume pequeno: tabela + filtro e paginação em memória. Simples e suficiente.
- Precisa de filtro por coluna/ordenação: uma tabela de dados (tanstack) com as colunas
  declaradas em arquivo separado.
- **Volume grande** (dezenas de milhares): pagine no servidor, com `page`/`limit` na chave de
  cache. Buscar tudo para paginar em memória trava a tela — e o problema só aparece em produção,
  onde a base é grande.
- Agrupamento (por dia, por categoria) sai de um `useMemo` sobre os dados já carregados; resumos
  do grupo também. Uma chamada por grupo multiplica requisições sem necessidade.

## 8. Navegação e permissão

Centralize o mapa de navegação (áreas → grupos → itens) em um módulo só, com as regras de
visibilidade declaradas por item (papel, feature flag, permissão concedida).

O ponto que morde: se a checagem de acesso resolve o item pela rota, **uma rota não cadastrada
fica sem checagem nenhuma**. Cadastrar a rota é parte de criar a tela, não enfeite do menu.

Esconder item de menu é UX. A autorização real é do backend — a tela que chamar um endpoint
proibido recebe 403 e mostra a mensagem. Nunca trate a checagem do cliente como suficiente.

## 9. Feedback ao usuário

```ts
toast.success('Pedidos', { description: 'Pedido criado com sucesso.' })
toast.error('Pedidos', {
  description: error.response?.data?.message ?? 'Não foi possível criar o pedido.',
})
```

Primeiro argumento é o contexto (módulo), `description` é o que aconteceu. Repasse a mensagem do
backend quando houver: ela é específica e o texto genérico do front descarta essa informação.

Estado de carregamento explícito (skeleton ou spinner) e botão desabilitado durante a submissão —
sem isso, clique duplo vira registro duplicado.

## 10. Estado lembrado e telas que se atualizam sozinhas

**Filtro, aba e preferência de visualização** que a pessoa ajusta todo dia ficam guardados no
navegador, **por usuário** (a chave inclui o id): na operação, a mesma máquina é usada por mais
de uma pessoa, e uma chave global faria o filtro de uma aparecer para a próxima. Leitura e
escrita em `try/catch` — armazenamento bloqueado não pode quebrar a tela. Período de consulta
(data, mês) em geral **não** é lembrado: abrir num mês antigo parece que os dados sumiram.

**Tela de acompanhamento ao vivo** (painel em TV, monitoramento):

- `refetchInterval` igual ao cache do servidor — atualizar mais rápido só repete o mesmo dado;
  `refetchIntervalInBackground: true` para a aba em segundo plano não congelar.
- Relógio local (`setInterval`) para tempos decorridos andarem entre uma consulta e outra.
- Falha numa atualização mantém o último dado na tela, com aviso discreto — não troca a tela
  inteira por erro.
- Mostre a idade do dado ("última alteração às 14:32") e avise quando a fonte parou de mudar;
  dado parado não pode parecer operação parada.
- Opção de tela cheia para uso em monitor.

**Grade de cards de altura variável**: colunas CSS (`columns-*` + `break-inside-avoid`) em vez
de `grid`, que estica a linha inteira até o card mais alto. A ordem passa a correr de cima para
baixo em cada coluna; quando a leitura linha a linha importa mais, use `grid` com
`items-start`.
