# Tasks - Zona Azul Digital

## 1. Objetivo

Decompor a implementação da Zona Azul Digital em tarefas independentes, ordenadas por dependência e alinhadas à `spec.md`, `plan.md` e `tests.md`.

Cada tarefa deve produzir uma parte verificável da solução sem alterar as regras estabelecidas na especificação.

---

## 2. Fundação do projeto

### T01 - Configurar a aplicação

**Objetivo:** criar a estrutura inicial da API.

**Entregas:**

* configurar Python 3.12;
* configurar FastAPI;
* configurar pytest;
* definir ponto de entrada da aplicação;
* configurar execução na porta definida pelo ambiente;
* preparar estrutura de diretórios.

**Critério de conclusão:**
A aplicação deve iniciar sem erros e estar preparada para receber requisições HTTP.

---

### T02 - Configurar execução em container

**Objetivo:** permitir que a API seja executada no ambiente de correção.

**Entregas:**

* criar configuração do container;
* configurar dependências;
* configurar porta interna do serviço;
* garantir acesso externo pela porta `8004`;
* documentar o comando de execução.

**Critério de conclusão:**
A API deve poder ser iniciada em container e acessada através da porta de serviço da variante.

---

## 3. Modelo e armazenamento

### T03 - Criar modelo de bilhete

**Objetivo:** representar o bilhete e seu ciclo de vida.

**Entregas:**

* representar `id`;
* representar `placa`;
* representar `entrada`;
* representar `status`;
* representar `saida` quando encerrado;
* representar `minutos` quando encerrado;
* representar `valor_centavos` quando encerrado;
* representar os estados `aberto`, `encerrado` e `cancelado`.

**Critério de conclusão:**
O modelo deve permitir representar corretamente os três estados previstos pela especificação.

---

### T04 - Implementar repositório de bilhetes

**Objetivo:** centralizar o armazenamento e consulta dos bilhetes.

**Entregas:**

* armazenamento em memória;
* criação de bilhetes;
* busca por identificador;
* busca por placa;
* busca de bilhetes abertos por placa;
* listagem de bilhetes;
* suporte à ordenação necessária pelas consultas.

**Critério de conclusão:**
As regras de negócio não devem depender diretamente da estrutura interna utilizada para armazenar os bilhetes.

---

### T05 - Implementar geração de identificadores

**Objetivo:** garantir identificadores únicos para os bilhetes.

**Entregas:**

* gerar identificador para novos bilhetes;
* garantir unicidade durante a execução da aplicação;
* manter a responsabilidade de geração isolada do restante do domínio.

**Critério de conclusão:**
Cada novo bilhete deve possuir um identificador único.

---

## 4. Validação

### T06 - Implementar validação de placa

**Objetivo:** validar placas de acordo com o contrato.

**Regras:**

* exatamente 7 caracteres;
* somente caracteres alfanuméricos;
* letras em maiúsculas;
* ausência da placa também deve ser rejeitada.

**Critério de conclusão:**
Entradas inválidas devem produzir `422` com `placa_invalida`.

---

### T07 - Implementar validação de datas

**Objetivo:** validar datas e horários utilizados pela API.

**Entregas:**

* validar `entrada` em ISO-8601 com fuso;
* rejeitar entradas inválidas;
* validar `data` no formato `AAAA-MM-DD`;
* rejeitar datas inexistentes;
* utilizar o fuso operacional `-03:00`.

**Critério de conclusão:**
Entradas inválidas devem produzir os erros definidos no contrato.

---

### T08 - Implementar prioridade das validações

**Objetivo:** garantir que erros de formato tenham prioridade sobre conflitos de negócio.

**Regra:**

Validação de formato deve ocorrer antes da verificação de estados e conflitos.

**Critério de conclusão:**
Uma requisição com formato inválido não deve retornar um erro `409` de conflito.

---

## 5. Regras de negócio

### T09 - Implementar abertura de bilhete

**Objetivo:** implementar o UC1.

**Entregas:**

* validar os dados;
* utilizar horário atual quando `entrada` não for informada;
* criar bilhete com status `aberto`;
* impedir segunda abertura para uma placa que já possui bilhete aberto;
* retornar os campos definidos no contrato.

**Critério de conclusão:**
Os cenários T01–T13 de `tests.md` devem ser atendidos.

---

### T10 - Implementar cálculo da duração

**Objetivo:** determinar a duração do estacionamento no encerramento.

**Entregas:**

* utilizar `entrada` e horário de saída;
* calcular duração em minutos;
* manter tratamento de horário com fuso;
* permitir controle do relógio nos testes.

**Critério de conclusão:**
A duração registrada deve ser consistente com os instantes de entrada e saída.

---

### T11 - Implementar cálculo da cobrança

**Objetivo:** implementar as regras de cobrança da variante.

**Parâmetros:**

* tarifa: `550` centavos/hora;
* fração: `30` minutos;
* valor por fração: `275` centavos;
* tolerância: `15` minutos;
* teto: `5000` centavos.

**Entregas:**

* aplicar tolerância;
* cobrar desde o primeiro minuto quando a tolerância for ultrapassada;
* arredondar para cima em frações de 30 minutos;
* aplicar teto de 5000 centavos;
* manter valores monetários como inteiros.

**Critério de conclusão:**
Os cenários T21–T41 devem ser atendidos.

---

### T12 - Implementar encerramento de bilhete

**Objetivo:** implementar o UC2.

**Entregas:**

* localizar o bilhete;
* verificar se pode ser encerrado;
* registrar saída;
* calcular duração;
* calcular cobrança;
* alterar status para `encerrado`;
* retornar os campos de encerramento;
* tratar bilhete inexistente;
* impedir encerramento repetido.

**Critério de conclusão:**
Os cenários T14–T20 devem ser atendidos.

---

### T13 - Implementar cancelamento

**Objetivo:** implementar o UC5.

**Entregas:**

* permitir cancelamento somente de bilhete aberto;
* alterar status para `cancelado`;
* não gerar cobrança;
* não registrar saída;
* permitir nova abertura para a mesma placa após cancelamento.

**Critério de conclusão:**
Os cenários T42–T48 devem ser atendidos.

---

## 6. Consultas

### T14 - Implementar listagem de bilhetes ativos

**Objetivo:** implementar o UC3.

**Entregas:**

* retornar somente bilhetes `aberto`;
* excluir encerrados;
* excluir cancelados;
* ordenar pela ordem de criação, do mais recente para o mais antigo;
* retornar `[]` quando não houver ativos.

**Critério de conclusão:**
Os cenários T49–T54 devem ser atendidos.

---

### T15 - Implementar histórico por placa

**Objetivo:** implementar o UC6.

**Entregas:**

* validar placa;
* localizar todos os bilhetes da placa;
* incluir todos os estados;
* excluir bilhetes de outras placas;
* ordenar pela ordem de criação, do mais recente para o mais antigo;
* retornar `[]` quando não houver histórico.

**Critério de conclusão:**
Os cenários T55–T61 devem ser atendidos.

---

### T16 - Implementar relatório diário

**Objetivo:** implementar o UC4.

**Entregas:**

* validar a data;
* considerar somente bilhetes encerrados na data;
* calcular `total_bilhetes`;
* calcular `faturamento_centavos`;
* calcular `tempo_medio_minutos`;
* ignorar bilhetes abertos;
* ignorar bilhetes cancelados;
* arredondar média com `0,5` para cima;
* retornar zero quando não houver encerramentos.

**Critério de conclusão:**
Os cenários T62–T72 devem ser atendidos.

---

## 7. Camada HTTP

### T17 - Implementar endpoints REST

**Objetivo:** conectar as regras de negócio aos endpoints definidos no contrato.

**Endpoints:**

* `POST /bilhetes`
* `POST /bilhetes/{id}/encerramento`
* `GET /bilhetes/ativos`
* `GET /relatorios/diario`
* `POST /bilhetes/{id}/cancelamento`
* `GET /bilhetes?placa=...`

**Critério de conclusão:**
Cada endpoint deve utilizar o método HTTP, parâmetros, corpo, respostas e códigos definidos na `spec.md`.

---

### T18 - Implementar tratamento HTTP de erros

**Objetivo:** garantir respostas consistentes para falhas.

**Entregas:**

* `422` para validações;
* `404` para bilhete inexistente;
* `409` para conflitos de estado;
* mensagens de erro exatamente conforme o contrato;
* prioridade de validação sobre conflito.

**Critério de conclusão:**
Os cenários T73–T76 devem ser atendidos.

---

## 8. Testes automatizados

### T19 - Implementar testes de ciclo de vida

**Objetivo:** verificar as transições dos bilhetes.

**Cobrir:**

* `aberto → encerrado`;
* `aberto → cancelado`;
* tentativa de encerramento repetido;
* tentativa de cancelamento repetido;
* tentativa de cancelar encerrado;
* tentativa de encerrar cancelado;
* nova abertura após encerramento;
* nova abertura após cancelamento.

**Critério de conclusão:**
Os cenários T77–T84 devem ser atendidos.

---

### T20 - Implementar testes de consistência

**Objetivo:** garantir que os dados dos bilhetes permaneçam coerentes com seus estados.

**Cobrir:**

* campos de bilhete aberto;
* campos de bilhete encerrado;
* campos de bilhete cancelado;
* limite monetário;
* valores inteiros em centavos;
* consistência do faturamento.

**Critério de conclusão:**
Os cenários T85–T90 devem ser atendidos.

---

### T21 - Implementar testes de isolamento entre placas

**Objetivo:** garantir que operações de uma placa não afetem outras.

**Critério de conclusão:**
Os cenários T91–T95 devem ser atendidos.

---

## 9. Integração e verificação final

### T22 - Executar a suíte completa

**Objetivo:** verificar a integração de todos os componentes.

**Verificações:**

* endpoints;
* validações;
* estados;
* cobrança;
* tolerância;
* teto;
* consultas;
* relatório;
* erros HTTP.

**Critério de conclusão:**
Todos os testes definidos em `tests.md` devem passar.

---

### T23 - Validar execução em container

**Objetivo:** garantir compatibilidade com o ambiente da prova.

**Verificações:**

* construção do container;
* inicialização da aplicação;
* disponibilidade na porta `8004`;
* acesso aos endpoints;
* execução dos testes.

**Critério de conclusão:**
A API deve estar acessível pelo endereço de serviço esperado pelo ambiente de correção.

---

### T24 - Revisar contrato e documentação

**Objetivo:** verificar se a implementação permanece alinhada à especificação.

**Verificações finais:**

* endpoints correspondem à `spec.md`;
* campos possuem os nomes definidos;
* códigos HTTP estão corretos;
* mensagens de erro estão corretas;
* regras de cobrança estão corretas;
* relatório diário está correto;
* documentação descreve a execução real;
* não existem funcionalidades fora do escopo que alterem o comportamento esperado.

**Critério de conclusão:**
A solução final deve estar coerente com `constitution.md`, `spec.md`, `plan.md` e `tests.md`.
