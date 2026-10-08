# Heurísticas de trabalho

Como conduzir a tarefa, não como escrever a linha. São as regras que mais evitam retrabalho e
incidente — valem para qualquer stack.

## 1. Pedido de análise é análise

"Analise", "verifique", "avalie", "por que aconteceu" pedem **diagnóstico**: causa, evidência e
opções de correção com esforço e impacto — terminando com a pergunta de implementar ou não.

Codifique quando o pedido disser "corrija", "ajuste", "implemente", ou quando a pessoa confirmar.
Exceção óbvia: "avalie X e garanta que não quebra" já contém o pedido de correção.

Implementar junto de um pedido de análise gera trabalho que precisa ser revertido — e revertido é
o melhor caso; o pior é entrar sem revisão.

## 2. Confirme decisões de produto antes de codificar

Quando há mais de um caminho defensável, pergunte de forma objetiva, com opções e a consequência
de cada uma:

- O que acontece com os registros irmãos ou já existentes?
- Quem pode executar isso — papel, escopo, concessão explícita?
- O dado atual será corrigido, e por qual critério?
- Qual o recorte da regra: por dia, por dia e tipo, global?

Uma pergunta custa menos que uma implementação descartada. Mas não pergunte o que o código
responde: investigue primeiro e traga a pergunta com o contexto pronto ("existem as abas X, Y e Z
e não existe 'W'; para onde deve ir?").

## 3. Valide contra dados reais

Antes de afirmar a causa, confirme no banco (em leitura). Depois de implementar, confirme o
efeito. Diagnóstico com número convence e evita corrigir o problema errado:

> "141 registros duplicados, 51 órfãos, 14 com status invertido."

Em ambiente de desenvolvimento, exercite os endpoints de verdade, não só o teste unitário: ordem
de rotas, guard e serialização só aparecem numa chamada real.

Ao montar credencial de teste em shell, cuidado com `source .env` — cifrões dentro de segredos
são expandidos pelo shell e o token sai inválido. Carregue o `.env` dentro do runtime.

## 4. Mudou regra de classificação ou cálculo? Faça dry-run do delta

Rode a **regra antiga** e a **nova** sobre os dados reais e mostre apenas o que muda entre elas.

Comparar a regra nova com o estado atual do banco mistura a sua mudança com divergências
históricas e produz uma lista assustadora e inútil. O número que importa é o delta:

> "8 de 4.568 registros mudam de categoria por causa desta alteração — todos do tipo esperado."

## 5. Produção é leitura por padrão

Qualquer escrita em produção (UPDATE, DELETE, script de correção, sincronização manual) é
**apresentada e aguarda confirmação explícita**. Mostre o comando, o recorte, quantas linhas
atinge e o que não será tocado.

Ao propor correção de dados, separe o que é mecânico do que precisa de decisão humana ("12 casos
têm uma alternativa só; 2 têm duas e alguém precisa dizer qual vale").

## 6. Limpe o que criou para testar

Registros, arquivos temporários, scripts de verificação. Limpe e diga que limpou. Resíduo de
teste em base compartilhada confunde quem olhar depois — inclusive você, semanas adiante.

Dado de teste que **deve permanecer** (um exemplo para a pessoa navegar) é criado de propósito,
com aviso explícito de que ficou lá.

## 7. Nunca afirme verificação que não fez

Diga o que foi verificado — typecheck, lint, testes, compilação, chamada real à API — e o que
ficou por conta do usuário ("não consegui percorrer o fluxo no navegador; o resto está limpo").

Afirmar "está tudo funcionando" sem ter testado destrói a confiança em tudo que você disse antes,
inclusive no que estava certo.

## 8. Corrija o próprio diagnóstico em voz alta

Se uma conclusão anterior estava errada, diga com todas as letras e explique o que mudou. Um
diagnóstico errado que fica de pé vira decisão errada mais adiante — e a correção tardia custa
mais que o constrangimento.

## 9. O código pode ter mudado desde a última vez que você olhou

Em sessões longas, outra pessoa (ou outra sessão) mexe nos mesmos arquivos. Antes de editar algo
que você leu há muitas mensagens, releia. Memória de sessão é um ponto no tempo, não o estado
atual do repositório.

## 10. Falha de teste alheia não é sua

Ao rodar a suíte, separe o que a sua mudança quebrou do que já estava quebrado, e aponte a falha
pré-existente com a causa provável. Não "conserte" silenciosamente um teste que reflete decisão
de outra pessoa — pergunte.

## 11. Comentário registra o incidente

Quando uma linha existe por causa de um bug específico, o comentário conta qual foi. Sem isso, a
próxima pessoa "limpa" o código e o bug volta.

```ts
// Um único carimbo para gravação e poda: comparar com now() do banco arredondava a fração
// na coluna de precisão menor e apagava a rodada inteira.
```

## 12. Confira credenciais e serviços antes de culpar o código

Erro de integração (e-mail, API externa, banco de terceiro) muitas vezes é credencial vencida,
chave trocada ou serviço fora. Teste a credencial isoladamente — sem efeito colateral (ex.:
`verify()` do SMTP, `GET` de leitura com `limit=1`) — antes de mexer no código. E nunca exiba
segredo no terminal para "conferir": cheque formato e presença, não o valor.

## 13. Teste no caminho real, inclusive o de quem está logado

Teste de serviço sem usuário logado pula auditoria, guards e limites de requisição — e foi
exatamente aí que já falhou (uma transação estourou o prazo só quando a auditoria rodava).
Antes de dizer que está pronto, exercite o fluxo como o usuário real: HTTP, sessão, lote do
tamanho real.

## 14. Avise o que a entrega exige

Toda entrega diz o que precisa acontecer para valer: migration a aplicar, variável de ambiente
nova, ordem de deploy entre repositórios, cache a limpar. E o que foi alterado em ambiente
compartilhado durante o teste (senha de usuário de teste, registro criado).

