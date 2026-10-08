# Spec - especificações 

API REST para abrir e encerrar bilhetes de estacionamento rotativo por placa. Base URL: `http://localhost:8004`. Somente a API existe (sem front-end).

> [!IMPORTANT]
> Convenções globais: JSON; dinheiro em **centavos inteiros**; datas ISO-8601 com `-03:00`; ID inteiro sequencial global a partir de 1; todo erro tem corpo `{"erro": "<codigo>"}` (nunca `detail`). Parâmetros: tarifa **550**, fração **15 min**, teto **8000**, tolerância **0**.

## Modelo de bilhete

| Campo | Tipo | Quando aparece |
|---|---|---|
| `id` | int | sempre |
| `placa` | string | sempre |
| `entrada` | string ISO-8601 `-03:00` | sempre |
| `status` | `"aberto"` \| `"encerrado"` \| `"cancelado"` | listagens, UC1 e UC5 |
| `saida` | string ISO-8601 `-03:00` | só `encerrado` |
| `minutos` | int | só `encerrado` |
| `valor_centavos` | int | só `encerrado` |

Bilhete `cancelado` **não** tem `saida`, `minutos` nem `valor_centavos` (as chaves são omitidas, não `null`).

## Regra de cobrança (usada no UC2)

```
minutos = floor((saida - entrada) em segundos / 60)       # minutos inteiros completos, mínimo 0
fracoes = 0 se minutos <= 0 (tolerância), senão ceil(minutos / 15)
bruto   = (fracoes * 550) // 4                             # multiplica ANTES de dividir
valor_centavos = min(bruto, 8000)
```

- **Multiplicar antes de dividir** (`fracoes × 550 ÷ 4`), com divisão inteira (piso). Assim 4 frações = 550 exatos. O valor de uma fração (137,5) **nunca** é arredondado isoladamente.
- Fração exata cobra 1 fração (15 min → 1); 1 minuto a mais cobra a seguinte (16 min → 2).
- **Tolerância (UC7):** com `TOLERANCIA_MINUTOS = 0`, só `minutos == 0` custa 0. Qualquer duração acima da tolerância cobra **desde o primeiro minuto**; a tolerância nunca é descontada.
- Se `saida < entrada` (entrada no futuro), `minutos = 0` e `valor_centavos = 0`.

> [!WARNING]
> **O TETO DE 8000 CENTAVOS É OBRIGATÓRIO.** `valor_centavos` nunca pode superar 8000, por maior que seja o tempo. O teto é aplicado **depois** do cálculo da fração. 870 min = 7975; 871 min = 8000 (bruto 8112).

---

## UC1 — Abrir bilhete
`POST /bilhetes`

| Item | Definição |
|---|---|
| Body | `{"placa": "ABC1D23"}` e opcional `"entrada": "<ISO-8601 com fuso>"` |
| Sucesso | **201** `{"id": 1, "placa": "ABC1D23", "entrada": "2026-10-05T14:30:00-03:00", "status": "aberto"}` |
| `entrada` ausente | usa `agora()` |
| `entrada` presente | o bilhete abre nesse instante; offset diferente de `-03:00` (inclusive `Z`) é convertido para `-03:00` na resposta |

**Placa válida:** string de exatamente 7 caracteres, só `A-Z` e `0-9` (regex `^[A-Z0-9]{7}$`). Minúscula, tamanho diferente, símbolo, ausente ou não string → inválida.
**Entrada válida:** string ISO-8601 **com offset** (`-03:00`, `+00:00` ou `Z`). Sem offset, data sem hora, texto livre ou não string → inválida.

Ordem de validação: (1) placa → (2) entrada → (3) bilhete aberto da placa.

Critérios de aceite:
- AC1.1 `{"placa":"ABC1D23"}` → 201, `id` int, `status` `"aberto"`, `entrada` termina em `-03:00`.
- AC1.2 Dois bilhetes seguidos (placas diferentes) recebem ids `1` e `2`.
- AC1.3 `entrada` informada é devolvida no mesmo instante, em `-03:00` (ex.: `2026-10-05T17:00:00Z` → `2026-10-05T14:00:00-03:00`).
- AC1.4 Placa ausente/inválida → **422** `{"erro":"placa_invalida"}`.
- AC1.5 `entrada` inválida → **422** `{"erro":"entrada_invalida"}`.
- AC1.6 Body que não é JSON objeto → **422** `{"erro":"placa_invalida"}`.

## UC2 — Encerrar bilhete
`POST /bilhetes/{id}/encerramento` (sem body)

| Item | Definição |
|---|---|
| Sucesso | **200** `{"id", "placa", "entrada", "saida", "minutos", "valor_centavos"}` — exatamente estas 6 chaves |
| `saida` | `agora()` no formato `-03:00` |
| Valor | regra de cobrança acima |
| Efeito | `status` passa a `encerrado`; a placa volta a poder abrir bilhete |

Erros: `id` inexistente (ou não numérico) → **404** `{"erro":"bilhete_nao_encontrado"}`; bilhete já `encerrado` **ou** `cancelado` → **409** `{"erro":"bilhete_ja_encerrado"}`.

Critérios de aceite:
- AC2.1 Bilhete aberto há 60 min → `minutos: 60`, `valor_centavos: 550`.
- AC2.2 Aberto há 15 min → 550×1÷4 → `137`; aberto há 16 min → `275`.
- AC2.3 Aberto há 871 min → `valor_centavos: 8000`; há 870 min → `7975`.
- AC2.4 `valor_centavos` e `minutos` são `int` no JSON (sem `.0`).
- AC2.5 Segundo encerramento do mesmo id → 409 `bilhete_ja_encerrado`.
- AC2.6 `id` 999 → 404 `bilhete_nao_encontrado`.

## UC3 — Listar ativos
`GET /bilhetes/ativos` → **200** array dos bilhetes com `status = "aberto"`, ordenado por `entrada` decrescente (empate: `id` decrescente). Sem ativos → `[]`. Cada item: `id`, `placa`, `entrada`, `status`.

> [!NOTE]
> Registrar a rota `/bilhetes/ativos` de modo que **não** seja capturada por rota com parâmetro de caminho.

Critérios de aceite:
- AC3.1 Bilhetes encerrados e cancelados **não** aparecem.
- AC3.2 O bilhete aberto mais recentemente é o primeiro do array.
- AC3.3 Sem bilhetes abertos → `200` e `[]`.

## UC4 — Relatório diário
`GET /relatorios/diario?data=AAAA-MM-DD` → **200**

```
{"data": "2026-10-05", "total_bilhetes": 12,
 "faturamento_centavos": 8400, "tempo_medio_minutos": 47}
```

- Conjunto do relatório: bilhetes **encerrados** cuja `saida` cai no dia `data` (calendário em `-03:00`).
- `total_bilhetes` = quantidade desse conjunto. Abertos e cancelados **não** entram.
- `faturamento_centavos` = soma dos `valor_centavos` desse conjunto (inteiro).
- `tempo_medio_minutos` = média dos `minutos` desse conjunto, **0,5 arredonda para cima**, em inteiros: `(2*soma + n) // (2*n)`. Conjunto vazio → `0`.
- Dia sem bilhetes → `{"data": ..., "total_bilhetes": 0, "faturamento_centavos": 0, "tempo_medio_minutos": 0}`.
- `data` ausente, fora de `AAAA-MM-DD` ou inexistente no calendário (ex.: `2026-02-30`) → **422** `{"erro":"data_invalida"}`.

Critérios de aceite:
- AC4.1 Dois encerrados hoje com 10 e 15 min → `total_bilhetes 2`, `faturamento_centavos 274`, `tempo_medio_minutos 13` (12,5 → 13).
- AC4.2 Bilhete aberto ou cancelado não altera nenhum dos 3 números.
- AC4.3 `data=2026-13-01`, `05/10/2026` ou ausente → 422 `data_invalida`.

## UC5 — Cancelar bilhete
`POST /bilhetes/{id}/cancelamento` (sem body)

- Sucesso: **200** `{"id", "placa", "entrada", "status": "cancelado"}` (sem `saida`, `minutos`, `valor_centavos`).
- Só bilhete `aberto` pode ser cancelado. Efeito: a placa volta a poder abrir bilhete.
- `id` inexistente → **404** `bilhete_nao_encontrado`. Bilhete `encerrado` ou `cancelado` → **409** `{"erro":"bilhete_nao_aberto"}`.

Critérios de aceite:
- AC5.1 Cancelar aberto → 200 com `status: "cancelado"` e sem as chaves de cobrança.
- AC5.2 Cancelar o mesmo id de novo → 409 `bilhete_nao_aberto`.
- AC5.3 Encerrar um bilhete cancelado → 409 `bilhete_ja_encerrado`.
- AC5.4 Cancelado não entra no relatório nem em ativos.

## UC6 — Histórico por placa
`GET /bilhetes?placa=ABC1D23` → **200** array com **todos** os bilhetes da placa (qualquer status), ordenado por `entrada` decrescente (empate: `id` decrescente). Itens seguem o modelo de bilhete (encerrado inclui `saida`, `minutos`, `valor_centavos`).
- Placa válida que nunca estacionou → `[]`.
- `placa` ausente ou inválida → **422** `{"erro":"placa_invalida"}`.

Critérios de aceite:
- AC6.1 Placa com 1 cancelado, 1 encerrado e 1 aberto → array com 3 itens, com seus status.
- AC6.2 Placa nunca vista → 200 `[]`.
- AC6.3 `?placa=abc` → 422 `placa_invalida`.

## UC7 — Tolerância gratuita
Regra descrita em "Regra de cobrança". Com `TOLERANCIA_MINUTOS = 0`: duração de 0 min → `valor_centavos: 0`; 1 min → `137`.

Critérios de aceite:
- AC7.1 Abrir e encerrar imediatamente (0 min) → `minutos: 0`, `valor_centavos: 0`.
- AC7.2 1 min → `valor_centavos: 137` (cobrança desde o primeiro minuto).

## UC8 — Uma vaga por placa
`POST /bilhetes` com placa que já tem bilhete `aberto` → **409** `{"erro":"bilhete_em_aberto"}`. Após encerrar **ou** cancelar, a mesma placa abre novo bilhete (201, novo `id`). Placas diferentes não interferem.

Critérios de aceite:
- AC8.1 Abrir duas vezes a mesma placa → 201 e depois 409.
- AC8.2 Encerrar e reabrir → 201. Cancelar e reabrir → 201.

## Tabela de erros (consolidada)

| Situação | Status | Body |
|---|---|---|
| Placa ausente/inválida (POST ou GET histórico) | 422 | `{"erro":"placa_invalida"}` |
| `entrada` fora de ISO-8601 com fuso | 422 | `{"erro":"entrada_invalida"}` |
| `data` fora de AAAA-MM-DD | 422 | `{"erro":"data_invalida"}` |
| Bilhete inexistente | 404 | `{"erro":"bilhete_nao_encontrado"}` |
| Encerrar bilhete não aberto | 409 | `{"erro":"bilhete_ja_encerrado"}` |
| Cancelar bilhete não aberto | 409 | `{"erro":"bilhete_nao_aberto"}` |
| Placa já com bilhete aberto | 409 | `{"erro":"bilhete_em_aberto"}` |

> [!CAUTION]
> Nenhuma resposta pode conter `detail`, `valor` (use `valor_centavos`) ou números com ponto decimal.