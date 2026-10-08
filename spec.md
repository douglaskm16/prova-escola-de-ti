# Specification - Zona Azul Digital

## 1. Objetivo

Desenvolver uma API REST para gerenciamento de bilhetes de estacionamento da Zona Azul Digital.

A API deve permitir:

* abrir um bilhete;
* encerrar um bilhete e calcular seu valor;
* listar bilhetes ativos;
* consultar o relatório diário;
* cancelar um bilhete;
* consultar o histórico de uma placa.

Não faz parte do escopo uma interface administrativa ou aplicação de back-office.

## 2. Parâmetros da variante

A implementação deve utilizar os seguintes valores da variante desta prova:

| Parâmetro              | Valor |
| ---------------------- | ----: |
| `TARIFA_HORA_CENTAVOS` |   550 |
| `FRACAO_MINUTOS`       |    30 |
| `TETO_DIARIO_CENTAVOS` |  5000 |
| `TOLERANCIA_MINUTOS`   |    15 |
| `PORTA_SERVICO`        |  8004 |

Uma hora custa `550` centavos e cada fração de 30 minutos corresponde a `275` centavos.

## 3. Modelo de domínio

Cada bilhete deve possuir:

| Campo            | Regra                                               |
| ---------------- | --------------------------------------------------- |
| `id`             | Identificador único do bilhete                      |
| `placa`          | Exatamente 7 caracteres alfanuméricos em maiúsculas |
| `entrada`        | Data e hora de entrada em ISO-8601                  |
| `status`         | `aberto`, `encerrado` ou `cancelado`                |
| `saida`          | Data e hora de saída quando encerrado               |
| `minutos`        | Duração em minutos quando encerrado                 |
| `valor_centavos` | Valor cobrado em centavos quando encerrado          |

Um bilhete aberto não deve possuir `saida`, `minutos` ou `valor_centavos`.

Um bilhete cancelado não deve possuir `saida` ou `valor_centavos`.

## 4. Regras gerais de validação

### 4.1 Placa

A placa deve:

* ser obrigatória quando exigida pelo endpoint;
* possuir exatamente 7 caracteres;
* conter somente caracteres alfanuméricos;
* utilizar letras maiúsculas.

Uma placa ausente ou inválida deve retornar:

`HTTP 422`

```json
{"erro":"placa_invalida"}
```

### 4.2 Data de entrada

Quando `entrada` for informada, deve ser uma data e hora ISO-8601 válida com informação de fuso horário.

Quando `entrada` não for informada, deve ser utilizado o horário atual no fuso operacional `-03:00`.

Uma entrada inválida deve retornar:

`HTTP 422`

```json
{"erro":"entrada_invalida"}
```

### 4.3 Data do relatório

O parâmetro `data` deve utilizar exatamente o formato `AAAA-MM-DD` e representar uma data válida do calendário.

Uma data inválida deve retornar:

`HTTP 422`

```json
{"erro":"data_invalida"}
```

### 4.4 Prioridade da validação

Validações de formato devem ocorrer antes das regras de conflito de estado.

Portanto, uma requisição com dados inválidos deve retornar `422`, mesmo que exista também uma possível situação de conflito.

## 5. UC1 - Abrir bilhete

### Endpoint

`POST /bilhetes`

### Entrada

Corpo JSON:

```json
{
  "placa": "ABC1D23",
  "entrada": "2026-10-07T10:00:00-03:00"
}
```

O campo `entrada` é opcional.

### Comportamento

Ao abrir um bilhete:

1. validar a placa;
2. validar `entrada`, caso fornecida;
3. verificar se a placa já possui um bilhete aberto;
4. rejeitar a abertura caso exista outro bilhete aberto para a mesma placa;
5. criar o novo bilhete com status `aberto`.

Quando `entrada` não for fornecida, utilizar o horário atual.

A entrada deve ser representada na resposta em ISO-8601 com o fuso `-03:00`.

### Sucesso

Retornar:

`HTTP 201`

```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-07T10:00:00-03:00",
  "status": "aberto"
}
```

### Erros

Placa inválida:

`422 {"erro":"placa_invalida"}`

Entrada inválida:

`422 {"erro":"entrada_invalida"}`

Já existe bilhete aberto para a placa:

`409 {"erro":"bilhete_em_aberto"}`

### Critérios de aceitação

* **CA1.1:** uma placa válida sem bilhete aberto deve gerar um novo bilhete com `HTTP 201` e status `aberto`.
* **CA1.2:** quando `entrada` não for informada, o sistema deve registrar o horário atual no fuso `-03:00`.
* **CA1.3:** uma placa inválida deve resultar em `422` com `erro = placa_invalida`.
* **CA1.4:** uma entrada inválida deve resultar em `422` com `erro = entrada_invalida`.
* **CA1.5:** uma segunda abertura para uma placa que já possui bilhete aberto deve resultar em `409` com `erro = bilhete_em_aberto`.

## 6. UC2 - Encerrar bilhete

### Endpoint

`POST /bilhetes/{id}/encerramento`

### Comportamento

O encerramento deve:

1. localizar o bilhete pelo identificador;
2. verificar se o bilhete ainda pode ser encerrado;
3. registrar a saída utilizando o horário atual no fuso `-03:00`;
4. calcular a duração em minutos;
5. calcular o valor conforme as regras de tolerância, fração e teto;
6. alterar o status para `encerrado`;
7. retornar os dados do encerramento.

### Regra de cobrança

Para esta variante:

* tolerância: 15 minutos;
* fração: 30 minutos;
* valor por fração: 275 centavos;
* teto: 5000 centavos.

Se a duração for menor ou igual a 15 minutos:

`valor_centavos = 0`

Se a duração for superior a 15 minutos, a tolerância não deve ser descontada.

A quantidade de frações cobradas é determinada arredondando a duração para cima em blocos de 30 minutos.

Exemplos:

| Duração | Frações | Valor |
| ------: | ------: | ----: |
|  15 min |       0 |     0 |
|  16 min |       1 |   275 |
|  30 min |       1 |   275 |
|  31 min |       2 |   550 |
|  60 min |       2 |   550 |
|  61 min |       3 |   825 |

O valor final deve ser limitado a `5000` centavos.

Exemplos próximos ao teto:

| Frações |         Cálculo | Valor final |
| ------: | --------------: | ----------: |
|      18 | 18 × 275 = 4950 |        4950 |
|      19 | 19 × 275 = 5225 |        5000 |

Todos os valores devem permanecer como inteiros em centavos.

### Sucesso

Retornar:

`HTTP 200`

```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-07T08:00:00-03:00",
  "saida": "2026-10-07T09:31:00-03:00",
  "minutos": 91,
  "valor_centavos": 825
}
```

### Erros

Bilhete inexistente:

`404 {"erro":"bilhete_nao_encontrado"}`

Bilhete que não pode mais ser encerrado:

`409 {"erro":"bilhete_ja_encerrado"}`

### Critérios de aceitação

* **CA2.1:** encerrar um bilhete existente e aberto deve retornar `HTTP 200`.
* **CA2.2:** o encerramento deve registrar `saida`, `minutos` e `valor_centavos`.
* **CA2.3:** duração de até 15 minutos deve gerar valor `0`.
* **CA2.4:** duração de 16 minutos deve cobrar uma fração completa de 30 minutos, totalizando `275` centavos.
* **CA2.5:** duração de 31 minutos deve cobrar duas frações, totalizando `550` centavos.
* **CA2.6:** uma cobrança calculada acima de `5000` centavos deve retornar exatamente `5000`.
* **CA2.7:** nenhum cálculo monetário deve utilizar valores fracionários de moeda.
* **CA2.8:** tentar encerrar um bilhete inexistente deve retornar `404`.
* **CA2.9:** tentar encerrar novamente um bilhete já encerrado deve retornar `409`.

## 7. UC3 - Listar bilhetes ativos

### Endpoint

`GET /bilhetes/ativos`

### Comportamento

Retornar somente os bilhetes cujo status seja `aberto`.

Os bilhetes devem ser apresentados em ordem decrescente de criação, com os mais recentemente criados primeiro.

O identificador deve ser utilizado somente como critério de desempate para garantir uma ordenação determinística.


### Sucesso

Retornar:

`HTTP 200`

```json
[
  {
    "id": 2,
    "placa": "XYZ9A99",
    "entrada": "2026-10-07T10:00:00-03:00",
    "status": "aberto"
  }
]
```

Caso não existam bilhetes ativos, retornar um array vazio.

### Critérios de aceitação

* **CA3.1:** bilhetes abertos devem aparecer na resposta.
* **CA3.2:** bilhetes encerrados não devem aparecer.
* **CA3.3:** bilhetes cancelados não devem aparecer.
* **CA3.4:** os bilhetes devem ser retornados em ordem decrescente de criação, do mais recentemente criado para o mais antigo.
* **CA3.5:** quando não houver bilhetes ativos, a resposta deve ser `HTTP 200` com `[]`.

## 8. UC4 - Relatório diário

### Endpoint

`GET /relatorios/diario?data=AAAA-MM-DD`

### Comportamento

O relatório deve considerar somente os bilhetes encerrados na data solicitada.

A data do encerramento deve ser determinada pela `saida` no fuso operacional `-03:00`.

O relatório deve apresentar:

* quantidade de bilhetes encerrados;
* faturamento total em centavos;
* tempo médio dos bilhetes encerrados.

Bilhetes abertos e cancelados não devem ser contabilizados.

### Faturamento

`faturamento_centavos` é a soma dos `valor_centavos` dos bilhetes encerrados na data.

### Tempo médio

`tempo_medio_minutos` é a média aritmética dos minutos dos bilhetes encerrados na data.

Quando o resultado possuir `0,5`, deve ser arredondado para cima.

Exemplo:

* 30 minutos e 31 minutos → média de 30,5 → `31`;
* 30 minutos e 32 minutos → média de 31 → `31`.

Quando não houver bilhetes encerrados, o tempo médio deve ser `0`.

### Sucesso

Retornar:

`HTTP 200`

```json
{
  "data": "2026-10-07",
  "total_bilhetes": 2,
  "faturamento_centavos": 825,
  "tempo_medio_minutos": 31
}
```

### Critérios de aceitação

* **CA4.1:** somente bilhetes encerrados na data solicitada devem ser contabilizados.
* **CA4.2:** o faturamento deve ser retornado em centavos inteiros.
* **CA4.3:** bilhetes cancelados não devem aumentar o faturamento.
* **CA4.4:** bilhetes ainda abertos não devem ser contabilizados.
* **CA4.5:** uma média de `30,5` minutos deve resultar em `31`.
* **CA4.6:** um dia sem bilhetes encerrados deve retornar totais e tempo médio iguais a `0`.
* **CA4.7:** uma data inválida deve retornar `422` com `erro = data_invalida`.

## 9. UC5 - Cancelar bilhete

### Endpoint

`POST /bilhetes/{id}/cancelamento`

### Comportamento

Somente bilhetes com status `aberto` podem ser cancelados.

Ao cancelar:

* alterar o status para `cancelado`;
* não gerar cobrança;
* não preencher `saida`;
* não preencher `valor_centavos`.

Após o cancelamento, a placa poderá possuir um novo bilhete aberto.

### Sucesso

Retornar:

`HTTP 200`

```json
{
  "id": 1,
  "placa": "ABC1D23",
  "entrada": "2026-10-07T10:00:00-03:00",
  "status": "cancelado"
}
```

### Erros

Bilhete inexistente:

`404 {"erro":"bilhete_nao_encontrado"}`

Bilhete encerrado ou cancelado:

`409 {"erro":"bilhete_nao_aberto"}`

### Critérios de aceitação

* **CA5.1:** um bilhete aberto deve poder ser cancelado com `HTTP 200`.
* **CA5.2:** um bilhete cancelado não deve possuir cobrança.
* **CA5.3:** um bilhete cancelado não deve possuir `saida`.
* **CA5.4:** após o cancelamento, a mesma placa deve poder abrir outro bilhete.
* **CA5.5:** cancelar um bilhete encerrado deve retornar `409`.
* **CA5.6:** cancelar um bilhete inexistente deve retornar `404`.

## 10. UC6 - Consultar histórico por placa

Endpoint

GET /bilhetes?placa=ABC1D23

Comportamento

Retornar todos os bilhetes associados à placa, independentemente do status.

Os resultados devem ser apresentados em ordem decrescente de criação, do mais recentemente criado para o mais antigo.

Uma placa que nunca possuiu bilhete deve retornar um array vazio.

Erros

Placa ausente ou inválida:

HTTP 422

{"erro":"placa_invalida"}

Critérios de aceitação

CA6.1: uma placa válida deve retornar todos os seus bilhetes.
CA6.2: o histórico deve incluir bilhetes aberto, encerrado e cancelado.
CA6.3: bilhetes de outras placas não devem aparecer.
CA6.4: os resultados devem ser ordenados em ordem decrescente de criação, do mais recentemente criado para o mais antigo.
CA6.5: uma placa sem histórico deve retornar HTTP 200 com [].
CA6.6: uma placa inválida ou ausente deve retornar HTTP 422 com erro = placa_invalida.

## 11. UC7 - Regra de tolerância

A tolerância desta variante é de 15 minutos.

A regra é do tipo tudo ou nada:

* até 15 minutos: não cobrar;
* acima de 15 minutos: cobrar desde o primeiro minuto;
* nunca subtrair os 15 minutos da duração antes de calcular a cobrança.

### Critérios de aceitação

* **CA7.1:** 15 minutos devem resultar em `0` centavos.
* **CA7.2:** 16 minutos devem resultar em `275` centavos.
* **CA7.3:** 30 minutos devem resultar em `275` centavos.
* **CA7.4:** 31 minutos devem resultar em `550` centavos.
* **CA7.5:** a tolerância não deve ser descontada quando a duração ultrapassar 15 minutos.

## 12. UC8 - Um único bilhete aberto por placa

Uma placa não pode possuir dois bilhetes simultaneamente com status `aberto`.

Ao tentar abrir um novo bilhete para uma placa que já possui bilhete aberto, retornar:

`HTTP 409`

```json
{"erro":"bilhete_em_aberto"}
```

Depois que o bilhete anterior for encerrado ou cancelado, a placa poderá abrir novamente.

### Critérios de aceitação

* **CA8.1:** uma placa com bilhete aberto não pode abrir outro bilhete.
* **CA8.2:** a tentativa duplicada deve retornar `409`.
* **CA8.3:** após encerramento, a placa pode abrir um novo bilhete.
* **CA8.4:** após cancelamento, a placa pode abrir um novo bilhete.
* **CA8.5:** bilhetes encerrados ou cancelados não devem impedir uma nova abertura.

## 13. Catálogo de erros

| Situação                                  | HTTP | Resposta                            |
| ----------------------------------------- | ---: | ----------------------------------- |
| Placa ausente ou inválida                 |  422 | `{"erro":"placa_invalida"}`         |
| Entrada inválida                          |  422 | `{"erro":"entrada_invalida"}`       |
| Data inválida                             |  422 | `{"erro":"data_invalida"}`          |
| Bilhete inexistente                       |  404 | `{"erro":"bilhete_nao_encontrado"}` |
| Bilhete já encerrado                      |  409 | `{"erro":"bilhete_ja_encerrado"}`   |
| Bilhete não está aberto para cancelamento |  409 | `{"erro":"bilhete_nao_aberto"}`     |
| Placa já possui bilhete aberto            |  409 | `{"erro":"bilhete_em_aberto"}`      |

## 14. Critérios gerais de aceite

* A API deve responder aos endpoints especificados utilizando os métodos HTTP definidos.
* Os códigos HTTP devem corresponder exatamente aos casos definidos.
* Valores monetários devem ser inteiros em centavos.
* A cobrança deve respeitar tolerância, frações e teto da variante.
* O ciclo de vida dos bilhetes deve impedir transições inválidas.
* O relatório diário deve considerar somente encerramentos ocorridos na data consultada.
* As consultas devem respeitar a ordenação dos bilhetes mais recentes primeiro.
* Os erros devem utilizar exatamente os identificadores definidos no catálogo de erros.
* A aplicação deve estar disponível na porta de serviço `8004` no ambiente da prova.
