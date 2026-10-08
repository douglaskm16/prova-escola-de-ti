# Tests - Zona Azul Digital

## 1. Objetivo

Definir cenários de teste para validar os fluxos principais, regras de negócio, limites e situações de erro da API Zona Azul Digital.

Os testes devem verificar tanto o código HTTP quanto o conteúdo relevante da resposta.

## 2. Convenções

Os valores desta variante são:

| Parâmetro        |         Valor |
| ---------------- | ------------: |
| Tarifa por hora  |  550 centavos |
| Fração           |    30 minutos |
| Valor por fração |  275 centavos |
| Teto             | 5000 centavos |
| Tolerância       |    15 minutos |

As respostas de erro devem possuir o campo `erro` exatamente conforme definido na especificação.

---

## 3. Testes de abertura de bilhete

| ID  | Cenário                             | Entrada                                  | Resultado esperado                         |
| --- | ----------------------------------- | ---------------------------------------- | ------------------------------------------ |
| T01 | Abrir bilhete válido                | Placa `ABC1D23`                          | `201`, status `aberto`                     |
| T02 | Abrir sem informar entrada          | Placa válida, sem `entrada`              | `201`, entrada definida pelo horário atual |
| T03 | Placa com menos de 7 caracteres     | `ABC1D2`                                 | `422`, `placa_invalida`                    |
| T04 | Placa com mais de 7 caracteres      | `ABC1D234`                               | `422`, `placa_invalida`                    |
| T05 | Placa com letras minúsculas         | `abc1d23`                                | `422`, `placa_invalida`                    |
| T06 | Placa com caractere inválido        | `ABC-123`                                | `422`, `placa_invalida`                    |
| T07 | Placa ausente                       | Sem `placa`                              | `422`, `placa_invalida`                    |
| T08 | Entrada em formato inválido         | `entrada` inválida                       | `422`, `entrada_invalida`                  |
| T09 | Entrada sem informação de fuso      | Data sem timezone                        | `422`, `entrada_invalida`                  |
| T10 | Entrada com data impossível         | Data ISO-8601 inválida                   | `422`, `entrada_invalida`                  |
| T11 | Abrir segunda vaga para mesma placa | Placa já possui bilhete aberto           | `409`, `bilhete_em_aberto`                 |
| T12 | Abrir novamente após encerramento   | Placa possui somente bilhetes encerrados | `201`                                      |
| T13 | Abrir novamente após cancelamento   | Placa possui somente bilhetes cancelados | `201`                                      |

---

## 4. Testes de encerramento

| ID  | Cenário                              | Resultado esperado                              |
| --- | ------------------------------------ | ----------------------------------------------- |
| T14 | Encerrar bilhete aberto existente    | `200` com `saida`, `minutos` e `valor_centavos` |
| T15 | Encerrar ID inexistente              | `404`, `bilhete_nao_encontrado`                 |
| T16 | Encerrar bilhete já encerrado        | `409`, `bilhete_ja_encerrado`                   |
| T17 | Verificar alteração para `encerrado` | Status passa de `aberto` para `encerrado`       |
| T18 | Encerrar duas vezes                  | Segunda tentativa retorna `409`                 |
| T19 | Valor monetário no encerramento      | `valor_centavos` é inteiro                      |
| T20 | Saída registrada no fuso operacional | `saida` utiliza `-03:00`                        |

---

## 5. Testes de tolerância

A tolerância de 15 minutos possui comportamento de tudo ou nada.

| ID  | Duração | Resultado esperado |
| --- | ------: | -----------------: |
| T21 |   0 min |         0 centavos |
| T22 |   1 min |         0 centavos |
| T23 |  14 min |         0 centavos |
| T24 |  15 min |         0 centavos |
| T25 |  16 min |       275 centavos |
| T26 |  17 min |       275 centavos |
| T27 |  29 min |       275 centavos |
| T28 |  30 min |       275 centavos |
| T29 |  31 min |       550 centavos |
| T30 |  59 min |       550 centavos |
| T31 |  60 min |       550 centavos |
| T32 |  61 min |       825 centavos |

O caso T25 é obrigatório para garantir que a tolerância não seja simplesmente subtraída da duração.

---

## 6. Testes de fração e teto

A cobrança deve utilizar frações de 30 minutos e sempre arredondar a duração para cima.

| ID  |             Duração |   Frações cobradas | Valor esperado |
| --- | ------------------: | -----------------: | -------------: |
| T33 |              30 min |                  1 |            275 |
| T34 |              31 min |                  2 |            550 |
| T35 |              60 min |                  2 |            550 |
| T36 |              61 min |                  3 |            825 |
| T37 |              90 min |                  3 |            825 |
| T38 |              91 min |                  4 |           1100 |
| T39 |          18 frações |                 18 |           4950 |
| T40 |          19 frações |                 19 |           5000 |
| T41 | Acima de 19 frações | limitado pelo teto |           5000 |

### Regra de teto

O valor calculado nunca deve ultrapassar `5000` centavos.

O caso T40 verifica especificamente a transição entre o valor calculado abaixo do teto e o valor limitado pelo teto.

---

## 7. Testes de cancelamento

| ID  | Cenário                              | Resultado esperado                             |
| --- | ------------------------------------ | ---------------------------------------------- |
| T42 | Cancelar bilhete aberto              | `200`, status `cancelado`                      |
| T43 | Cancelar bilhete inexistente         | `404`, `bilhete_nao_encontrado`                |
| T44 | Cancelar bilhete encerrado           | `409`, `bilhete_nao_aberto`                    |
| T45 | Cancelar bilhete já cancelado        | `409`, `bilhete_nao_aberto`                    |
| T46 | Verificar cobrança após cancelamento | Bilhete não possui `valor_centavos`            |
| T47 | Verificar saída após cancelamento    | Bilhete não possui `saida`                     |
| T48 | Abrir novo bilhete após cancelamento | Nova abertura para a mesma placa retorna `201` |

---

## 8. Testes de bilhetes ativos

| ID  | Cenário                                 | Resultado esperado              |
| --- | --------------------------------------- | ------------------------------- |
| T49 | Listar com um bilhete aberto            | Bilhete aparece                 |
| T50 | Listar sem bilhetes abertos             | `200` com `[]`                  |
| T51 | Misturar aberto e encerrado             | Somente aberto aparece          |
| T52 | Misturar aberto e cancelado             | Somente aberto aparece          |
| T53 | Vários bilhetes abertos                 | Mais recentes aparecem primeiro |
| T54 | Bilhete encerrado não volta para ativos | Não aparece                     |

---

## 9. Testes de histórico por placa

| ID  | Cenário                                     | Resultado esperado                     |
| --- | ------------------------------------------- | -------------------------------------- |
| T55 | Consultar placa existente                   | Todos os bilhetes da placa aparecem    |
| T56 | Consultar placa sem histórico               | `200` com `[]`                         |
| T57 | Histórico com aberto, encerrado e cancelado | Os três estados podem aparecer         |
| T58 | Histórico com outras placas existentes      | Bilhetes de outras placas não aparecem |
| T59 | Histórico de placa inválida                 | `422`, `placa_invalida`                |
| T60 | Consulta sem parâmetro `placa`              | `422`, `placa_invalida`                |
| T61 | Histórico ordenado                          | Mais recentes primeiro                 |

---

## 10. Testes do relatório diário

| ID  | Cenário                        | Resultado esperado                                                          |
| --- | ------------------------------ | --------------------------------------------------------------------------- |
| T62 | Um bilhete encerrado no dia    | Contabilizado no relatório                                                  |
| T63 | Bilhete aberto no dia          | Não contabilizado                                                           |
| T64 | Bilhete cancelado no dia       | Não contabilizado                                                           |
| T65 | Bilhete encerrado em outro dia | Não contabilizado                                                           |
| T66 | Dois bilhetes encerrados       | Quantidade e faturamento somados                                            |
| T67 | Dia sem encerramentos          | `total_bilhetes = 0`, `faturamento_centavos = 0`, `tempo_medio_minutos = 0` |
| T68 | Data válida                    | `200`                                                                       |
| T69 | Data com formato incorreto     | `422`, `data_invalida`                                                      |
| T70 | Data impossível do calendário  | `422`, `data_invalida`                                                      |
| T71 | Média inteira                  | Valor médio retornado normalmente                                           |
| T72 | Média com `.5`                 | Arredondar para cima                                                        |

### Teste específico de média

Criar dois bilhetes encerrados no mesmo dia com:

* primeiro bilhete: 30 minutos;
* segundo bilhete: 31 minutos.

A média é `30,5`.

Resultado esperado:

`tempo_medio_minutos = 31`.

O arredondamento deve ser para cima quando houver exatamente `0,5`, e não arredondamento bancário.

---

## 11. Testes de prioridade das validações

As validações de formato devem ocorrer antes das regras de negócio.

| ID  | Cenário                                                    | Resultado esperado         |
| --- | ---------------------------------------------------------- | -------------------------- |
| T73 | Placa inválida que também possui bilhete aberto            | `422`, `placa_invalida`    |
| T74 | Entrada inválida durante tentativa de abertura conflitante | `422`, `entrada_invalida`  |
| T75 | Data inválida no relatório                                 | `422`, `data_invalida`     |
| T76 | Placa válida já aberta                                     | `409`, `bilhete_em_aberto` |

Esses testes garantem que um conflito de estado não substitua uma falha de validação.

---

## 12. Testes de ciclo de vida

| ID  | Transição                                       | Resultado esperado  |
| --- | ----------------------------------------------- | ------------------- |
| T77 | `aberto → encerrado`                            | Permitida           |
| T78 | `aberto → cancelado`                            | Permitida           |
| T79 | `encerrado → encerrado`                         | Rejeitada com `409` |
| T80 | `encerrado → cancelado`                         | Rejeitada com `409` |
| T81 | `cancelado → cancelado`                         | Rejeitada com `409` |
| T82 | `cancelado → encerrado`                         | Rejeitada           |
| T83 | Após `encerrado`, abrir novo bilhete para placa | Permitido           |
| T84 | Após `cancelado`, abrir novo bilhete para placa | Permitido           |

---

## 13. Testes de consistência dos dados

| ID  | Cenário                | Resultado esperado                                                  |
| --- | ---------------------- | ------------------------------------------------------------------- |
| T85 | Bilhete aberto         | Não possui `saida`, `minutos` ou `valor_centavos`                   |
| T86 | Bilhete encerrado      | Possui `saida`, `minutos` e `valor_centavos`                        |
| T87 | Bilhete cancelado      | Não possui `saida` ou `valor_centavos`                              |
| T88 | Cobrança acima do teto | `valor_centavos` permanece em `5000`                                |
| T89 | Valor monetário        | Sempre inteiro em centavos                                          |
| T90 | Relatório              | Faturamento corresponde à soma dos valores dos encerramentos do dia |

---

## 14. Testes de isolamento entre placas

| ID  | Cenário                         | Resultado esperado                        |
| --- | ------------------------------- | ----------------------------------------- |
| T91 | Placa A aberta e placa B aberta | Ambas podem possuir um bilhete aberto     |
| T92 | Placa A aberta duas vezes       | Segunda abertura rejeitada                |
| T93 | Encerrar placa A                | Placa B permanece inalterada              |
| T94 | Cancelar placa A                | Histórico da placa B permanece inalterado |
| T95 | Consultar histórico da placa A  | Não retorna dados da placa B              |

---

## 15. Critérios mínimos para aprovação

A implementação deve passar, no mínimo, pelos seguintes grupos de comportamento:

1. abertura e validação de bilhetes;
2. encerramento e cálculo da cobrança;
3. tolerância de 15 minutos;
4. fração de 30 minutos;
5. teto de 5000 centavos;
6. cancelamento;
7. regra de uma abertura por placa;
8. listagem de ativos;
9. histórico por placa;
10. relatório diário;
11. arredondamento de média em `0,5`;
12. códigos HTTP e mensagens de erro;
13. prioridade das validações sobre conflitos de estado;
14. consistência dos estados e dos campos retornados.
