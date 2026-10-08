# UI/UX

Leia ao desenhar uma tela, escrever textos da interface, definir estados de carregamento e erro,
ou revisar a usabilidade de algo pronto. Vale para aplicação interna e para site público — o
que muda é a ênfase (operação × conversão), não os princípios.

## Índice

1. Prioridades
2. Tema, cores e tipografia
3. Layout e responsividade
4. Os quatro estados
5. Formulários
6. Ações, confirmação e feedback
7. Tabelas, filtros e listas
8. Textos da interface
9. Acessibilidade
10. Telas de operação e painéis
11. Sites públicos
12. Checklist antes de entregar

## 1. Prioridades

Na ordem: **dado correto e legível → ação óbvia → consistência → beleza.** Uma tela bonita com
número errado ou botão escondido falha; uma tela simples e correta funciona.

Consistência vale mais que criatividade local: o mesmo tipo de ação no mesmo lugar, com o mesmo
nome e o mesmo ícone em todo o sistema. Quem usa aprende uma vez.

## 2. Tema, cores e tipografia

- **Tokens de tema** (`--background`, `--primary`, `--destructive`...) do shadcn/Tailwind, nunca
  cor solta por componente. Mudar a marca ou o modo escuro vira uma mudança, não uma varredura.
- **Modo escuro funcionando** em toda tela nova: toda cor fixa tem a variante `dark:`.
- **Cores com significado fixo**: vermelho = erro/atrasado/destrutivo; âmbar = atenção;
  verde = sucesso/concluído; azul = informação/neutro-ativo. Não use vermelho como enfeite.
- **Nunca comunique só por cor**: acompanhe com texto, ícone ou forma (daltonismo, impressão,
  tela ruim).
- **Tipografia**: uma família, poucas variações de tamanho. Números que se comparam (contadores,
  horários, valores) com `tabular-nums` para alinhar.

## 3. Layout e responsividade

- Funcionar **do celular ao monitor largo**. Sem rolagem horizontal da página; tabela larga rola
  dentro do próprio contêiner.
- Hierarquia clara: título da tela, descrição de uma linha, ações principais no topo à direita,
  filtros acima do conteúdo, sempre visíveis — filtro escondido no cabeçalho de coluna é
  descoberto por poucos.
- Espaçamento pela escala do Tailwind (`gap-2/3/4/6`), não por pixel avulso.
- Cards de altura variável não esticam para igualar os vizinhos (ver `frontend.md` §10).

## 4. Os quatro estados

Todo bloco que carrega dados tem os quatro, sempre:

| Estado | O que mostrar |
|---|---|
| **Carregando** | Skeleton no formato do conteúdo (ou spinner curto); nunca a tela em branco |
| **Vazio** | O que significa estar vazio e o próximo passo ("Nenhum técnico vinculado ainda. Use 'Vincular técnicos'…") |
| **Erro** | A mensagem do backend + o que fazer; quando houver dado anterior, mantenha-o com um aviso |
| **Com dados** | O conteúdo |

Vazio por filtro ≠ vazio de verdade: "Nenhum resultado com esse filtro" é diferente de "Ainda
não há registros".

## 5. Formulários

- **Rótulo visível** em todo campo (placeholder não é rótulo — some ao digitar).
- **Validação no próprio campo**, com a mensagem dizendo o que corrigir ("A senha precisa ter
  pelo menos um número"), não "campo inválido". A mesma regra roda no backend.
- **Valores padrão inteligentes** e sugestões pré-preenchidas quando o sistema sabe a resposta
  provável — a pessoa confere em vez de digitar.
- **Botão de envio desabilitado** durante a submissão, com texto de progresso ("Salvando...").
- Em erro, **mantenha o que foi digitado**; o modal não fecha.
- Campos relacionados agrupados; ordem igual à do processo real da operação.
- Senha: mostrar/ocultar, e gerador com cópia quando o sistema define a senha.

## 6. Ações, confirmação e feedback

- **Toda ação dá feedback**: toast de sucesso dizendo o que aconteceu; toast de erro repassando
  a mensagem do backend (ela é específica; texto genérico descarta a informação).
- **Ação destrutiva ou irreversível pede confirmação**, dizendo exatamente o que será afetado
  ("Excluir 3 vagas de 05/10?"). O botão de confirmar repete o verbo ("Excluir"), não "OK".
- **Ação que só aparece uma vez** (senha gerada, token criado) avisa que não será exibida de
  novo e oferece copiar.
- Ação proibida para o usuário: esconda, ou mostre desabilitada com o motivo no tooltip quando
  saber que ela existe ajuda. A autorização real é do backend.
- Janela que **não pode** ser fechada (troca de senha obrigatória) não tem X, nem fecha com Esc
  ou clique fora — e oferece uma saída explícita ("Sair").

## 7. Tabelas, filtros e listas

- Cabeçalho fixo em listas longas; colunas numéricas alinhadas à direita.
- Busca **sem diferenciar acento e caixa** ("joao" acha "JOÃO").
- Filtros do usuário lembrados por tela (ver `frontend.md` §10) e um jeito óbvio de limpar.
- Paginação ou rolagem com total visível ("124 protocolos").
- Linha clicável quando abre detalhe; ações da linha num menu de três pontos ou ícones com
  tooltip, sempre na mesma coluna.
- Datas: hora quando é hoje, dia e hora quando não é — exceto onde comparar datas lado a lado
  importa (abertura × prazo), aí sempre completo.

## 8. Textos da interface

- **No idioma do usuário**, no vocabulário da operação ("protocolo", "vaga", "roteamento"), não
  no do código ("entity", "record", "null").
- Botão diz o verbo e o objeto: "Salvar vínculos", "Gerar senha" — não "Enviar", "OK".
- Mensagem de erro diz **o que houve e o que fazer**: "Este horário acabou de ser ocupado —
  escolha outro."
- Sem jargão técnico, código de erro ou nome de tabela na tela.
- Consistência de termos: escolha "excluir" ou "remover" e use sempre o mesmo.
- Changelog e avisos escritos do ponto de vista de quem usa (ver `documentacao.md`).

## 9. Acessibilidade

O mínimo, que o shadcn já facilita:

- **Contraste AA** (4,5:1 para texto normal) nos dois temas.
- **Foco visível** e toda ação alcançável por teclado; ordem de tabulação na ordem visual.
- Botão só com ícone tem `aria-label`; ícone decorativo tem `aria-hidden`.
- `label` associado a cada campo; erros anunciados junto do campo.
- Área de toque de pelo menos 40 px no celular.
- Movimento e animação discretos; nada que pisque.

## 10. Telas de operação e painéis

Telas usadas o dia inteiro (agenda, roteamento, monitoramento) têm necessidades próprias:

- **Densidade** maior que a de um site: mais informação por tela, menos espaço decorativo.
- **Indicadores clicáveis** que filtram a lista abaixo; o filtro ativo fica evidente.
- **Situação por cor + texto** (borda lateral colorida + badge com o nome da situação).
- **Atualização automática** com a idade do dado visível (ver `frontend.md` §10).
- **Abas** para separar equipes/setores, com contador no título da aba.
- Nada que mude de lugar sozinho enquanto a pessoa está prestes a clicar.

## 11. Sites públicos

- **Desempenho é UX**: imagens otimizadas (`next/image`), fontes locais, pouca JS no
  carregamento.
- **SEO básico**: título e descrição por página, URLs legíveis, `sitemap` e `robots`, metatags
  de compartilhamento.
- Uma chamada para ação principal por página; contato fácil de achar.
- Formulário público com proteção contra robô e limite de envio no backend.

## 12. Checklist antes de entregar

- [ ] Quatro estados (carregando, vazio, erro, dados) em todo bloco de dados
- [ ] Modo escuro e celular conferidos
- [ ] Textos no idioma e no vocabulário do usuário; mensagens de erro acionáveis
- [ ] Ações com feedback; destrutivas com confirmação
- [ ] Teclado e foco funcionando; ícones com `aria-label`
- [ ] Nada comunicado só por cor
