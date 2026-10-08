# Stack

Leia ao iniciar um projeto, ao avaliar adicionar uma biblioteca, ou quando alguém propuser
trocar uma peça da stack. A pergunta central é sempre a mesma: **o que vai custar mais caro
daqui a um ano — adotar isto, ou não ter adotado?**

## Índice

1. Front-end
2. CSS e componentes
3. Back-end
4. Kit padrão de bibliotecas
5. Ferramentas e versões
6. Antes de adicionar uma dependência

## 1. Front-end

| Situação | Escolha | Por quê |
|---|---|---|
| Rotas, SSR/SSG, middleware de sessão, SEO, API junto do front | **Next.js** (App Router) | Já vem resolvido e testado; reconstruir à mão é trabalho recorrente para sempre |
| App de produto com várias telas e autenticação | **Next.js** | Estrutura de rotas e convenções poupam decisão a cada tela nova |
| Site institucional / landing com SEO | **Next.js** (estático quando der) | SEO e imagens otimizadas sem esforço |
| Tela única, painel estático, ferramenta interna simples | **TypeScript + Vite** | Menos dependência, build instantâneo |
| SPA pesada sem servidor | Next estático ou Vite + React | Ambos servem; decida pelo que o time mantém melhor |

Não migre um projeto simples para Next por antecipação: migrar quando a necessidade aparecer
custa menos que manter complexidade nunca usada.

## 2. CSS e componentes

| Situação | Escolha |
|---|---|
| Formulários, tabelas, modais, selects — UI de produto | **shadcn/ui** sobre **Tailwind** |
| Poucas telas, sem componentes interativos complexos | **Tailwind puro** |
| Projeto sem framework com formulário/tabela/modal | **shadcn/ui** — a necessidade é a mesma |

O que decide não é o tamanho do projeto, é a presença de componentes interativos:
acessibilidade de foco, teclado e ARIA é cara de fazer à mão e é exatamente o que o shadcn
entrega. Como o código fica no repositório, ajustar não depende de release de terceiro.

Nunca duas bibliotecas de componentes no mesmo app — dois jeitos de fazer modal, nenhum padrão.
CSS-in-JS, Bootstrap, Material e afins ficam fora: disputam com o Tailwind e quebram a
consistência do tema.

## 3. Back-end

**NestJS é o padrão** para um backend próprio: módulos, injeção de dependência, guards, pipes de
validação e documentação vêm padronizados. O ganho maior é previsibilidade — quem chega
encontra a mesma estrutura em todo domínio.

Escolha outra coisa em dois casos — e diga qual se aplica:

**1. O ecossistema não cobre o requisito.** Runtime específico, restrição da plataforma de
deploy, processamento que pede outra linguagem, biblioteca que só existe fora do Node.

**2. Outro caminho tem custo operacional menor.** Custo aqui é implantar, monitorar, autenticar
e manter — não quantidade de código:

| Necessidade | Alternativa | Quando compensa |
|---|---|---|
| Backend pequeno de um app Next | **Route handlers do Next** | Evita um segundo serviço; a regra continua em `server/<dominio>/`, fora do handler |
| Webhook, tarefa agendada, endpoint isolado | **Função serverless** | Não há estado nem estrutura a compartilhar |
| Serviço mínimo (um ou dois endpoints) | **Fastify** | A estrutura do Nest não se paga em dois endpoints |
| Processamento pesado/streaming | Serviço na linguagem adequada | O gargalo é o runtime, não o framework |

Sinais de que o Nest se paga: vários domínios, papéis e escopos de autorização, integrações
externas, jobs agendados, mais de uma pessoa mexendo. Sinais de que não: um endpoint, um
consumidor, ciclo de vida curto.

## 4. Kit padrão de bibliotecas

Uma escolha por necessidade. Antes de trazer outra para a mesma função, troque a padrão no
projeto inteiro ou não troque — duas bibliotecas para a mesma coisa é o pior dos mundos.

**Front-end**

| Necessidade | Padrão |
|---|---|
| Dados do servidor / cache | **@tanstack/react-query** (não `swr` em paralelo) |
| Cliente HTTP | **axios**, instância única em `core/api.ts` |
| Formulários | **react-hook-form** + **zod** (`@hookform/resolvers`) |
| Tabelas de dados | **@tanstack/react-table** |
| Ícones | **lucide-react** |
| Notificações | **sonner** (via shadcn) |
| Datas | **date-fns** |
| Gráficos | **recharts** (via shadcn charts) |
| Tema claro/escuro | **next-themes** |
| Arrastar e soltar | **@dnd-kit** |
| Testes de ponta a ponta | **Playwright** |

**Back-end**

| Necessidade | Padrão |
|---|---|
| ORM / migrations | **Prisma** |
| Validação de entrada | **class-validator** + **class-transformer** (DTOs) |
| Validação de regra reutilizada / env | **zod** |
| Autenticação | **@nestjs/jwt** + **passport-jwt**, token em cookie httpOnly |
| Senhas | **bcrypt** |
| Limite de requisições | **@nestjs/throttler** (com Redis quando houver mais de uma instância) |
| Jobs agendados | **@nestjs/schedule**; filas com **bullmq** quando houver volume ou retentativa |
| E-mail | **@nestjs-modules/mailer** + templates handlebars |
| Documentação da API | **@nestjs/swagger** |
| Cabeçalhos de segurança | **helmet** |
| SQL cru em banco externo | **pg** (Postgres) / **mysql2** (MySQL), com pool próprio |
| Testes | **Jest** + **supertest** |

## 5. Ferramentas e versões

- **Node LTS** atual; versão fixada no projeto (`.nvmrc` ou `engines`), para ninguém rodar com
  outra sem perceber.
- **TypeScript em modo estrito** nas duas pontas.
- **npm** como gerenciador (lockfile versionado). Um gerenciador por repositório.
- **ESLint + Prettier** configurados no repositório; formatação não se discute em PR.
- Versões **maiores** da stack (Next, Nest, Prisma, React) sobem de propósito, numa PR só para
  isso — nunca de carona em uma feature.

## 6. Antes de adicionar uma dependência

Pergunte, e responda na PR:

1. O kit padrão ou a plataforma já resolve? (`Intl` formata data e número; `crypto` gera id.)
2. Quem mantém, com que frequência, e quantas dependências ela puxa?
3. Quanto custa remover daqui a um ano?

Dependência para uma função de dez linhas é dívida, não economia.
