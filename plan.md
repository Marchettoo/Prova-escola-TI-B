# Plan — Decisões técnicas e arquitetura

Resumo crítico (repetido de propósito): Python 3.11 + FastAPI + uvicorn, porta **8004** em `0.0.0.0`, dados em memória, dinheiro em **centavos inteiros**, erros `{"erro": "<codigo>"}`, fuso `-03:00`, parâmetros: tarifa **550**, fração **15**, teto **8000**, tolerância **0**.

## Decisões e justificativas

| # | Decisão | Justificativa |
|---|---|---|
| D1 | Python 3.11 + FastAPI + uvicorn | Contrato REST simples; geração rápida e verificável; imagem pequena (`python:3.11-slim`). |
| D2 | Persistência em memória com `threading.Lock` | A prova não exige banco; o lock garante a regra "uma vaga por placa" mesmo com requisições concorrentes. |
| D3 | Dinheiro em `int` com aritmética inteira (`(fracoes * 550) // 4`) | Elimina erro de ponto flutuante; multiplicar antes de dividir mantém 4 frações = 550 exatos, apesar de 550/4 = 137,5. |
| D4 | Relógio único `agora()` em `app/clock.py`, fuso fixo `timezone(timedelta(hours=-3))` | Testabilidade (monkeypatch) e sem dependência de `tzdata`; o Brasil não tem horário de verão vigente. |
| D5 | `minutos` = piso de `(saida - entrada)` em minutos | Testes abrem bilhetes com `entrada = agora - N min`; milissegundos de latência não podem virar 1 minuto extra de cobrança. |
| D6 | Corpo do POST lido como JSON bruto e validado manualmente | O FastAPI/pydantic devolve 422 com `detail`; o contrato exige `{"erro": ...}` com códigos específicos. |
| D7 | Handlers globais de exceção: `RequestValidationError`, `HTTPException`, `Exception` | Garante R2 (todo erro em `{"erro": ...}`) para rota inexistente, método errado e `id` não numérico (→ 404 `bilhete_nao_encontrado`). |
| D8 | Exceções de domínio (`PlacaInvalida`, `EntradaInvalida`, `DataInvalida`, `BilheteNaoEncontrado`, `BilheteJaEncerrado`, `BilheteNaoAberto`, `BilheteEmAberto`) mapeadas para status/código em um único lugar | Regras ficam em `service.py` sem conhecer HTTP; mapeamento testável. |
| D9 | Datas via `datetime.fromisoformat` (aceitar sufixo `Z`), exigir `tzinfo`, converter para `-03:00`, serializar com `isoformat(timespec="seconds")` | Formato idêntico em todas as respostas; entrada sem fuso é rejeitada. |
| D10 | Relatório diário calculado sobre bilhetes `encerrado` com `saida.date() == data` (em `-03:00`); média com `(2*soma + n) // (2*n)` | Arredondamento 0,5 para cima sem float. |
| D11 | Rota `/bilhetes/ativos` registrada **antes** de qualquer rota `/bilhetes/{id}...` | Evita captura de "ativos" como `id`. |
| D12 | Constantes da variante só em `app/config.py` | Uma única fonte da verdade (R10). |

