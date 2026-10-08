# Constitution - Zona Azul Digital

## 1. Fidelidade ao contrato

O contrato da prova é a fonte de verdade do sistema. A implementação deve atender aos endpoints, campos, estados, códigos HTTP e mensagens de erro especificados.

Não devem ser criados comportamentos que contradigam o contrato ou substituam suas regras por exemplos antigos ou inconsistentes.

## 2. API REST e respostas

* A API deve utilizar os endpoints definidos na especificação.
* As respostas devem utilizar JSON quando definido pelo contrato.
* Os nomes dos campos devem ser mantidos exatamente como especificados.
* Erros devem retornar o código HTTP e o objeto `erro` definidos no contrato.
* Uma resposta de erro não deve misturar informações de sucesso com informações de erro.

## 3. Validação antes das regras de negócio

Validações de formato devem ocorrer antes das validações de estado ou regras de negócio.

Quando uma requisição possuir dados inválidos e também uma possível situação de conflito, deve prevalecer o erro de validação HTTP 422 definido para aquele campo.

Exemplos:

* placa inválida deve resultar em `422 placa_invalida`;
* entrada inválida deve resultar em `422 entrada_invalida`;
* data inválida deve resultar em `422 data_invalida`.

## 4. Identificação das placas

Uma placa válida deve possuir exatamente 7 caracteres alfanuméricos e estar em letras maiúsculas.

A API não deve transformar ou corrigir automaticamente uma placa inválida.

Uma mesma placa não pode possuir mais de um bilhete com status `aberto` simultaneamente.

## 5. Estados do bilhete

Os únicos estados permitidos para um bilhete são:

* `aberto`;
* `encerrado`;
* `cancelado`.

As transições devem respeitar o ciclo de vida definido na especificação:

* `aberto` pode ser encerrado;
* `aberto` pode ser cancelado;
* `encerrado` não pode ser encerrado ou cancelado novamente;
* `cancelado` não pode ser encerrado ou cancelado novamente.

Um bilhete cancelado não gera cobrança.

## 6. Regra monetária

Todos os valores monetários devem ser representados como números inteiros em centavos.

Para esta variante da prova:

* tarifa por hora: `550` centavos;
* fração de cobrança: `30` minutos;
* valor de cada fração: `275` centavos;
* teto por bilhete: `5000` centavos;
* tolerância gratuita: `15` minutos.

Não devem ser utilizados valores monetários em ponto flutuante para representar ou calcular cobranças.

## 7. Regra de cobrança

A cobrança deve utilizar frações de 30 minutos e sempre arredondar a duração para a próxima fração quando necessário.

A tolerância é integral e não deve ser descontada da duração cobrada.

Portanto:

* duração de até 15 minutos resulta em `0` centavos;
* duração superior a 15 minutos é cobrada desde o primeiro minuto;
* a cobrança de duração superior à tolerância é arredondada para cima em frações de 30 minutos;
* o valor final nunca pode ultrapassar `5000` centavos.

## 8. Datas e horários

Os horários devem ser tratados como instantes com informação de fuso horário.

O fuso horário operacional da aplicação é `-03:00`.

A data de entrada, quando não informada, deve utilizar o horário atual da aplicação.

Datas e horários inválidos devem ser rejeitados antes de qualquer alteração no estado dos bilhetes.

## 9. Relatórios

O relatório diário deve considerar somente bilhetes encerrados na data solicitada.

Bilhetes ainda abertos ou cancelados não devem ser contabilizados no faturamento, na quantidade de bilhetes encerrados ou no tempo médio.

O tempo médio deve ser calculado somente com os tempos dos bilhetes encerrados no dia e, quando o resultado possuir `0,5`, deve ser arredondado para cima.

Quando não houver bilhetes encerrados no dia, os valores agregados devem ser `0`.

## 10. Ordenação

Quando o contrato solicitar os bilhetes mais recentes primeiro, a resposta deve apresentar primeiro os bilhetes criados mais recentemente.

A ordenação deve ser determinística, utilizando o identificador como critério de desempate quando necessário.

## 11. Isolamento das responsabilidades

As regras de validação, ciclo de vida, cobrança e relatório devem possuir responsabilidades separadas.

Alterações em uma regra de negócio não devem exigir alterações desnecessárias nas demais regras.

## 12. Testabilidade e qualidade

As regras de negócio devem ser verificáveis por testes automatizados.

Os testes devem contemplar tanto os fluxos normais quanto os limites e conflitos definidos na especificação.

O sistema deve permanecer restrito ao escopo da Zona Azul Digital e não deve introduzir funcionalidades administrativas ou de interface que não sejam necessárias para a API.