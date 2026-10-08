# Tests — Casos de teste e bordas (TDD)

Cada linha vira **uma função `test_`** em `tests/test_api.py` (ou `tests/test_pricing.py` para as tabelas de cobrança). Mínimo exigido: 40 funções.

## Convenções dos testes
- Variante: tarifa **550**, fração **15 min**, teto **8000**, tolerância **0**, porta **8004**. Frações por hora = 4. Valor = `min((fracoes*550)//4, 8000)`.
- Fixture `autouse` chama `store.reset()` (ids voltam a 1).
- Helper `abrir(placa, minutos_atras)`: envia `entrada = agora() - minutos_atras` em ISO-8601 com `-03:00`. Encerrar logo em seguida produz `minutos == minutos_atras`.
- Erros sempre conferem **status e body exato** (`{"erro": ...}`); valores monetários conferem `isinstance(x, int)`.

## 1. Cobrança — fração, adjacência, teto (UC2, UC7)

| # | Minutos | Frações | `valor_centavos` esperado | Regra testada |
|---|---|---|---|---|
| T01 | 0 | 0 | 0 | duração 0 ≤ tolerância 0 |
| T02 | 1 | 1 | 137 | cobra desde o 1º minuto (550/4 = 137,5 → piso) |
| T03 | 14 | 1 | 137 | dentro da 1ª fração |
| T04 | 15 | 1 | 137 | fração **exata** cobra 1 fração |
| T05 | 16 | 2 | 275 | **+1 min** cobra a fração seguinte |
| T06 | 30 | 2 | 275 | fração exata (2) |
| T07 | 31 | 3 | 412 | +1 min após fração exata |
| T08 | 45 | 3 | 412 | fração exata (3) |
| T09 | 60 | 4 | 550 | hora cheia = tarifa |
| T10 | 61 | 5 | 687 | +1 min após hora cheia |
| T11 | 95 | 7 | 962 | valor intermediário (3850 ÷ 4 = 962,5 → piso) |
| T12 | 120 | 8 | 1100 | 2 horas cheias |
| T13 | 870 | 58 | 7975 | **último valor abaixo do teto** |
| T14 | 871 | 59 | **8000** | bruto 8112 → **teto aplicado** |
| T15 | 1440 | 96 | **8000** | bruto 13200 → teto |
| T16 | — | — | — | `valor_centavos` e `minutos` são `int` (sem `.0`); resposta de encerramento tem exatamente as 6 chaves e não tem `valor` |

> [!WARNING]
> T13/T14 protegem a regra do **teto**. T02, T07, T10, T11 protegem a regra "multiplicar antes de dividir, piso".

## 2. Abrir bilhete (UC1)

| # | Entrada | Esperado |
|---|---|---|
| T17 | `{"placa":"ABC1D23"}` | 201, `id` 1, `status "aberto"`, `entrada` termina em `-03:00` |
| T18 | duas placas diferentes em sequência | ids 1 e 2 |
| T19 | `entrada` `2026-10-05T17:00:00Z` | 201, `entrada` `2026-10-05T14:00:00-03:00` |
| T20 | `entrada` com `+00:00` | convertida para `-03:00` |
| T21 | `entrada` `"ontem"` | 422 `entrada_invalida` |
| T22 | `entrada` `2026-10-05T14:00:00` (sem fuso) | 422 `entrada_invalida` |
| T23 | `entrada` `2026-13-45T10:00:00-03:00` | 422 `entrada_invalida` |
| T24 | placa ausente (`{}`) | 422 `placa_invalida` |
| T25 | `abc1d23` (minúscula) | 422 `placa_invalida` |
| T26 | `ABC1D2` (6 chars) e `ABC1D234` (8 chars) | 422 `placa_invalida` |
| T27 | `ABC-D23` (símbolo) | 422 `placa_invalida` |
| T28 | placa numérica `1234567` (número JSON, não string) | 422 `placa_invalida` |
| T29 | body não-JSON / lista `[]` | 422 `placa_invalida` |

## 3. Uma vaga por placa (UC8)

| # | Cenário | Esperado |
|---|---|---|
| T30 | abrir `ABC1D23` duas vezes | 201 e depois 409 `bilhete_em_aberto` |
| T31 | abrir, encerrar, abrir de novo | 201, 200, 201 (novo id) |
| T32 | abrir, cancelar, abrir de novo | 201, 200, 201 |
| T33 | placas diferentes abertas ao mesmo tempo | ambas 201 |

## 4. Encerrar e cancelar (UC2, UC5) — conflitos mínimos

| # | Cenário | Esperado |
|---|---|---|
| T34 | encerrar id inexistente (999) e id não numérico (`abc`) | 404 `bilhete_nao_encontrado` |
| T35 | encerrar duas vezes o mesmo id | 200 e depois 409 `bilhete_ja_encerrado` |
| T36 | cancelar bilhete aberto | 200, `status "cancelado"`, **sem** `saida`, `minutos`, `valor_centavos` |
| T37 | cancelar duas vezes | 200 e depois 409 `bilhete_nao_aberto` |
| T38 | cancelar bilhete já encerrado | 409 `bilhete_nao_aberto` |
| T39 | encerrar bilhete cancelado | 409 `bilhete_ja_encerrado` |
| T40 | cancelar id inexistente | 404 `bilhete_nao_encontrado` |

## 5. Listagens (UC3, UC6)

| # | Cenário | Esperado |
|---|---|---|
| T41 | sem bilhetes: `GET /bilhetes/ativos` | 200 `[]` |
| T42 | 3 abertos com entradas distintas | ordem por `entrada` decrescente (mais recente primeiro) |
| T43 | 1 aberto + 1 encerrado + 1 cancelado | ativos devolve só o aberto |
| T44 | histórico de placa com aberto, encerrado e cancelado | 200, 3 itens, status corretos, mais recente primeiro; encerrado traz `saida/minutos/valor_centavos` |
| T45 | histórico de placa válida nunca usada | 200 `[]` |
| T46 | `GET /bilhetes?placa=abc` e `GET /bilhetes` (sem placa) | 422 `placa_invalida` |
| T47 | histórico não mistura placas | só bilhetes da placa consultada |

## 6. Relatório diário (UC4)

| # | Cenário | Esperado |
|---|---|---|
| T48 | dia sem encerrados | `{"data":..., "total_bilhetes":0, "faturamento_centavos":0, "tempo_medio_minutos":0}` |
| T49 | encerrados hoje com 10 e 15 min | total 2, faturamento 274, média **13** (12,5 → 13) |
| T50 | encerrados com 10 e 11 min | média **11** (10,5 → 11) |
| T51 | encerrados com 10, 10 e 11 min | média **10** (10,33 → 10) |
| T52 | 1 encerrado (60 min) + 1 aberto + 1 cancelado | total 1, faturamento 550, média 60 |
| T53 | encerrado com 871 min | faturamento soma **8000** (teto), não 8112 |
| T54 | `data=2026-13-01`, `05/10/2026`, `2026-2-5`, `2026-02-30`, ausente | 422 `data_invalida` |
| T55 | faturamento e média são `int` | sem ponto decimal |

## 7. Formato geral

| # | Cenário | Esperado |
|---|---|---|
| T56 | rota inexistente (`GET /xyz`) | 404 com body `{"erro": ...}` (nunca `detail`) |
| T57 | todas as respostas de erro dos testes acima | possuem a chave `erro` e **não** possuem `detail` |

## Testes unitários de `calcular_valor` (`tests/test_pricing.py`)
Repetir as linhas T01–T15 como tabela parametrizada (`@pytest.mark.parametrize`), mais: tolerância 0 aceita `minutos=0` → 0; retorno é sempre `int`; nunca supera 8000.