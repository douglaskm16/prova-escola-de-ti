# Plan - Zona Azul Digital

## 1. Objetivo técnico

Implementar uma API REST pequena, determinística e testável para o gerenciamento dos bilhetes da Zona Azul Digital.

A solução deve priorizar:

* aderência ao contrato da API;
* isolamento das regras de negócio;
* cálculos monetários exatos;
* tratamento consistente de datas e fusos;
* facilidade de testes automatizados;
* execução em container.

## 2. Stack tecnológica

### Backend

Utilizar **Python 3.12 com FastAPI**.

### Justificativa

FastAPI é adequado para uma API REST pequena porque fornece uma estrutura direta para definição de endpoints, validação de dados e respostas HTTP.

Python também possui suporte adequado para:

* manipulação de datas e horários;
* testes automatizados;
* estruturas de dados em memória;
* execução simples em container.

A solução deve evitar dependências desnecessárias que não contribuam para os requisitos da prova.

### Testes

Utilizar **pytest** para os testes automatizados.

Os testes devem cobrir as regras de negócio descritas em `tests.md`, especialmente limites de cobrança, estados dos bilhetes, validações e relatórios.

## 3. Arquitetura

A aplicação será organizada em responsabilidades separadas:

### API

Responsável por:

* receber requisições HTTP;
* validar o formato das entradas;
* chamar as regras de negócio;
* converter resultados em respostas HTTP;
* retornar os códigos e mensagens definidos no contrato.

### Domínio

Responsável pelas regras de negócio dos bilhetes:

* abertura;
* encerramento;
* cancelamento;
* ciclo de vida;
* regra de uma abertura por placa;
* cálculo da cobrança;
* geração dos dados do relatório.

### Repositório

Responsável pelo armazenamento e consulta dos bilhetes.

Para o escopo da prova, será utilizado armazenamento em memória.

### Relatórios

A geração do relatório diário ficará separada das operações de criação e alteração dos bilhetes.

Isso permite calcular os dados agregados sem duplicar as regras de consulta.

## 4. Armazenamento

Será utilizado um repositório em memória durante a execução da aplicação.

### Justificativa

O contrato não exige banco de dados ou persistência após o encerramento da aplicação.

O armazenamento em memória reduz a complexidade da solução e evita introduzir uma dependência externa que não é necessária para os requisitos funcionais.

O repositório deve, entretanto, possuir uma responsabilidade isolada para que as regras de negócio não dependam diretamente da estrutura utilizada para armazenar os dados.

## 5. Identificadores

Cada bilhete deverá possuir um identificador único.

O mecanismo de geração do identificador ficará concentrado na responsabilidade de armazenamento/criação do bilhete.

As demais partes da aplicação não devem depender da forma como o identificador é gerado.

## 6. Datas e horários

A aplicação deverá trabalhar com objetos de data e hora que mantenham a informação de fuso.

O fuso operacional será `-03:00`.

### Justificativa

O contrato utiliza datas ISO-8601 com fuso e exige que os horários retornados utilizem `-03:00`.

Manter os horários como instantes com fuso evita comparar diretamente strings e reduz erros relacionados à conversão de horário.

A entrada fornecida pelo cliente deve ser validada antes de ser utilizada pelas regras de negócio.

## 7. Relógio da aplicação

A obtenção do horário atual deverá ficar isolada de forma que as regras de negócio possam ser testadas com horários controlados.

No uso normal da API, o relógio utilizará o horário atual do ambiente.

Nos testes automatizados, o horário poderá ser controlado para reproduzir durações específicas.

### Justificativa

O encerramento utiliza o horário atual para determinar a duração do estacionamento.

Sem separar a obtenção do horário atual, testes de casos como 15, 16, 30 e 31 minutos dependeriam do relógio real e seriam menos determinísticos.

## 8. Cálculo monetário

Os valores monetários serão tratados exclusivamente como inteiros em centavos.

Para esta variante:

* tarifa horária: `550`;
* fração: `30` minutos;
* valor por fração: `275`;
* teto: `5000`;
* tolerância: `15` minutos.

O cálculo seguirá a seguinte sequência conceitual:

1. determinar a duração;
2. verificar a tolerância;
3. caso esteja dentro da tolerância, retornar zero;
4. caso ultrapasse a tolerância, calcular as frações de 30 minutos;
5. multiplicar as frações por `275`;
6. limitar o resultado ao teto de `5000`.

### Justificativa

Representar dinheiro em centavos inteiros evita erros de precisão associados a ponto flutuante e está de acordo com o contrato.

## 9. Máquina de estados do bilhete

O domínio utilizará três estados:

* `aberto`;
* `encerrado`;
* `cancelado`.

As operações de encerramento e cancelamento deverão verificar o estado atual antes de realizar qualquer alteração.

### Justificativa

Centralizar as transições de estado evita que operações inválidas sejam realizadas em diferentes pontos da aplicação.

## 10. Validação

A validação será realizada em duas etapas conceituais:

### Etapa 1 - Formato

Validar:

* placa;
* entrada;
* data do relatório;
* parâmetros obrigatórios.

### Etapa 2 - Regra de negócio

Após os dados serem considerados válidos, verificar:

* existência do bilhete;
* existência de bilhete aberto para a placa;
* possibilidade de encerramento;
* possibilidade de cancelamento.

### Justificativa

Essa separação garante a prioridade definida pelo contrato: erros de formato devem retornar `422` antes de conflitos de estado `409`.

## 11. Relatório diário

O relatório será calculado a partir dos bilhetes armazenados.

Somente bilhetes encerrados cuja saída pertença à data consultada no fuso `-03:00` serão considerados.

O relatório deverá calcular:

* quantidade de bilhetes;
* soma do faturamento em centavos;
* média dos minutos.

A média deverá utilizar arredondamento com `0,5` para cima.

## 12. Ordenação

As consultas que exigem os bilhetes mais recentes primeiro deverão aplicar uma ordenação determinística.

A criação mais recente deve aparecer primeiro e o identificador poderá ser utilizado como critério de desempate.

Isso será aplicado tanto à listagem de ativos quanto ao histórico por placa.

## 13. Tratamento de erros

Os erros de negócio não devem ser representados por exceções genéricas expostas diretamente ao cliente.

Cada situação prevista pelo contrato deverá ser convertida para seu código HTTP e mensagem correspondente.

Os principais erros são:

| Situação                       | HTTP |
| ------------------------------ | ---: |
| Placa inválida                 |  422 |
| Entrada inválida               |  422 |
| Data inválida                  |  422 |
| Bilhete inexistente            |  404 |
| Bilhete já encerrado           |  409 |
| Bilhete não aberto             |  409 |
| Placa já possui bilhete aberto |  409 |

## 14. Containerização

A aplicação deverá ser executável em container.

O serviço deverá ficar disponível para o ambiente da prova através da porta de serviço da variante:

`8004`

O serviço poderá utilizar a porta interna definida pelo ambiente de execução, mantendo o mapeamento necessário para que `http://localhost:8004` seja acessível pela correção.

## 15. Documentação

O projeto gerado deverá possuir documentação mínima para execução e teste da API.

A documentação deve informar:

* como instalar as dependências;
* como executar a aplicação;
* como executar os testes;
* como executar o container;
* quais endpoints estão disponíveis.

A documentação deve permanecer coerente com a especificação.

## 16. Critérios técnicos

A implementação deve:

* manter as regras de negócio independentes da camada HTTP;
* evitar duplicação das regras de cobrança;
* evitar cálculos monetários em ponto flutuante;
* utilizar datas com fuso horário;
* permitir testes determinísticos das regras dependentes do horário;
* manter os erros do contrato centralizados e consistentes;
* não introduzir banco de dados ou serviços externos sem necessidade;
* não adicionar endpoints fora do escopo sem justificativa.

## 17. Ordem de implementação

A implementação deverá seguir esta ordem lógica:

1. definir o modelo de bilhete e seus estados;
2. implementar o armazenamento;
3. implementar validações;
4. implementar abertura de bilhetes;
5. implementar encerramento e cobrança;
6. implementar cancelamento;
7. implementar listagem de ativos;
8. implementar histórico por placa;
9. implementar relatório diário;
10. integrar as respostas HTTP;
11. adicionar testes automatizados;
12. configurar execução em container e documentação.

## 18. Resultado esperado

Ao final, a aplicação deverá fornecer uma API REST executável em container, com comportamento compatível com `spec.md` e cobertura dos cenários definidos em `tests.md`.

Nenhuma decisão técnica deve alterar as regras de negócio estabelecidas na especificação.
