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

## Estrutura de arquivos a gerar (raiz da pasta de saída)

| Arquivo | Responsabilidade |
|---|---|
| `app/__init__.py` | Pacote |
| `app/config.py` | `TARIFA_HORA_CENTAVOS=550`, `FRACAO_MINUTOS=15`, `TETO_DIARIO_CENTAVOS=8000`, `TOLERANCIA_MINUTOS=0`, `PORTA_SERVICO=8004` |
| `app/clock.py` | `agora()` com fuso `-03:00` |
| `app/errors.py` | Exceções de domínio e mapa exceção → (status, código) |
| `app/models.py` | `dataclass Bilhete` com `status` e campos opcionais de saída |
| `app/store.py` | Repositório em memória, contador de id, `Lock`, `reset()` |
| `app/pricing.py` | Função pura `calcular_valor(minutos) -> int` (regra de cobrança do `spec.md`) |
| `app/service.py` | Casos de uso UC1–UC8 (validação, conflitos, relatório) |
| `app/main.py` | App FastAPI, rotas, handlers de erro, serialização |
| `tests/test_api.py` | Testes de API (`TestClient`) — um por cenário de `tests.md` |
| `tests/test_pricing.py` | Testes unitários de `calcular_valor` e do arredondamento da média |
| `requirements.txt` | `fastapi`, `uvicorn`, `pytest`, `httpx`, `ruff` com versões fixadas |
| `Containerfile` e `Dockerfile` | Mesmo conteúdo (ver abaixo) |
| `README.md` | Português: descrição, execução com podman/docker/uvicorn, testes, tabela de rotas com status de sucesso e erro |
| `pyproject.toml` | Seção `[tool.ruff]` (`line-length = 100`) |
| `.gitignore`, `.dockerignore` | `__pycache__/`, `.pytest_cache/`, `.venv/`, `.env`, `.ruff_cache/` |

## Containerfile esperado

```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
EXPOSE 8004
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8004"]
```

> [!NOTE]
> O serviço deve subir com `uvicorn app.main:app --host 0.0.0.0 --port 8004` a partir da raiz, sem variáveis de ambiente obrigatórias e sem segredos.

## Ordem de validação por rota (resumo)

| Rota | Ordem |
|---|---|
| `POST /bilhetes` | placa (422) → entrada (422) → placa já aberta (409) |
| `POST /bilhetes/{id}/encerramento` | id existe (404) → status `aberto` (409 `bilhete_ja_encerrado`) |
| `POST /bilhetes/{id}/cancelamento` | id existe (404) → status `aberto` (409 `bilhete_nao_aberto`) |
| `GET /bilhetes?placa=` | placa válida (422) → filtro |
| `GET /relatorios/diario?data=` | data válida (422) → agregação |

## Riscos conhecidos

> [!WARNING]
> (1) Não usar `float` em nenhuma etapa de valor. (2) Não descontar tolerância. (3) Não esquecer o teto de 8000. (4) Não deixar o FastAPI responder `detail` ou 422 com corpo próprio.