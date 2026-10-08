## Parâmetros da variante (valores obrigatórios)

| Parâmetro | Valor | Significado |
|---|---|---|
| `TARIFA_HORA_CENTAVOS` | **550** | Hora cheia, em centavos |
| `FRACAO_MINUTOS` | **15** | Cobrança em frações de 15 min, sempre arredondando para cima |
| `TETO_DIARIO_CENTAVOS` | **8000** | Máximo cobrado por bilhete |
| `TOLERANCIA_MINUTOS` | **0** | Sem minutos grátis (apenas duração 0 min custa 0) |
| `PORTA_SERVICO` | **8004** | Porta do serviço, host `0.0.0.0` |

Frações por hora = 60 ÷ 15 = **4**.

## Regras operacionais:
| # | Regra |
|---|---|
| R1 | **Dinheiro é sempre `int` em centavos.** Nunca `float`, nem em cálculo intermediário, nem em JSON.[^1] |
| R2 | **Todo erro** responde `{"erro": "<codigo_snake_case>"}` — inclusive erros gerados pelo framework (JSON inválido, validação, rota inexistente, `id` não numérico). Nunca usar o corpo padrão do framework (`detail`). |
| R3 | **Todo endpoint documenta** seus status de sucesso e de erro (no `summary`/docstring da rota e na tabela de rotas do `README.md`). |
| R4 | Datas em ISO-8601 com offset **`-03:00`**, precisão de segundos (ex.: `2026-10-05T14:30:00-03:00`). |
| R5 | O instante "agora" vem de **uma única função** `agora()` em `app/clock.py`. Nenhum outro módulo chama `datetime.now()`. |
| R6 | IDs de bilhete: inteiros sequenciais **globais**, começando em 1. |
| R7 | Persistência **em memória**, protegida por `threading.Lock`. Sem banco. |
| R8 | Nomes de rotas, campos JSON e códigos de erro **exatamente** como em `spec.md` (português, sem acento). |
| R9 | **Proibido** criar endpoints, campos de resposta ou códigos de erro fora de `spec.md`. |
| R10 | Constantes da variante vivem **somente** em `app/config.py`; nenhum outro módulo repete esses números. |
| R11 | Higiene de produção: sem segredos no repositório, `.gitignore` e `.dockerignore` presentes, `ruff check` sem erros, type hints nas funções públicas. |
| R12 | Dependências permitidas: `fastapi`, `uvicorn`, `pytest`, `httpx`, `ruff`. Nenhuma outra. |
| R13 | Testes com `pytest` em `tests/`; toda regra de negócio tem teste feliz **e** de borda; mínimo de 40 funções `test_`. |

> [!WARNING]
> **Contrato manda, exemplo não.** O exemplo `{"id": 7, "valor": 12.50}` do enunciado é propositalmente **inválido** (usa `valor` e float). A resposta correta usa `valor_centavos` inteiro.

## Entregáveis obrigatórios do código gerado
Na raiz da pasta de saída: pacote `app/`, `tests/`, `requirements.txt`, `Containerfile`, `Dockerfile` (mesmo conteúdo), `README.md`, `pyproject.toml` (config do ruff), `.gitignore`, `.dockerignore`. Detalhes em `plan.md`.

[^1]: Ponto flutuante acumula erro de arredondamento (`0.1 + 0.2 != 0.3`). Centavos inteiros eliminam essa classe de bug, e a suíte de correção testa exatamente isso.