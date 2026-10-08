# Tasks — Decomposição e instruções ao agente

## Instruções de entrada (leia primeiro)

1. Leia, nesta ordem: `constitution.md` → `spec.md` → `plan.md` → `tests.md` → este arquivo.
2. Gere **todo** o código na raiz da pasta de saída, seguindo a estrutura de arquivos do `plan.md`.
3. Siga TDD: para cada tarefa, escreva primeiro os testes indicados de `tests.md`, veja falhar, implemente, veja passar.
4. Não invente rotas, campos ou códigos de erro. Em conflito entre arquivos, `spec.md` vence no comportamento.
5. Ao final, rode `pytest` e `ruff check .`, e suba o serviço com `uvicorn app.main:app --host 0.0.0.0 --port 8004` para conferir que responde.
6. Registre no `README.md` qualquer ambiguidade encontrada e a decisão tomada.

> [!IMPORTANT]
> Parâmetros: tarifa **550**, fração **15**, teto **8000**, tolerância **0**, porta **8004**. Dinheiro sempre `int` em centavos. Erros sempre `{"erro": "<codigo>"}`.

## Tarefas

- [ ] **T1 — Scaffolding.** Criar `requirements.txt` (versões fixadas), `pyproject.toml` (ruff), `.gitignore`, `.dockerignore`, `app/__init__.py`, `app/config.py` (5 constantes), `app/clock.py` (`agora()` em `-03:00`), `tests/` e fixture `autouse` de reset. *Depende de: nada.*
- [ ] **T2 — Modelos, erros e repositório.** `app/models.py`, `app/errors.py` (exceções e mapa para status/código), `app/store.py` (dict, contador de id global, `Lock`, `reset()`). *Depende de: T1.*
- [ ] **T3 — Tratamento global de erros.** Handlers em `app/main.py` para `RequestValidationError`, `HTTPException` (404/405) e `Exception`, sempre `{"erro": ...}`. Testes T56, T57. *Depende de: T2.*
- [ ] **T4 — Cobrança.** `app/pricing.py::calcular_valor(minutos)`: frações com `ceil`, `(fracoes*550)//4`, teto 8000, tolerância 0. Testes T01–T16 e `tests/test_pricing.py`. *Depende de: T1.*
- [ ] **T5 — UC1 Abrir bilhete.** Validação de placa e `entrada` (ISO-8601 com fuso, conversão para `-03:00`). Testes T17–T29. *Depende de: T2, T3.*
- [ ] **T6 — UC8 Uma vaga por placa.** Bloqueio 409 `bilhete_em_aberto` sob `Lock`; liberação após encerrar/cancelar. Testes T30–T33. *Depende de: T5.*
- [ ] **T7 — UC2/UC7 Encerrar bilhete.** `minutos` por piso, `valor_centavos` via T4, resposta com exatamente 6 chaves. Testes T34, T35, T01–T16 via API. *Depende de: T4, T6.*
- [ ] **T8 — UC5 Cancelar bilhete.** Resposta sem chaves de cobrança; erros 404/409 `bilhete_nao_aberto`. Testes T36–T40. *Depende de: T6.*
- [ ] **T9 — UC3/UC6 Listagens.** `GET /bilhetes/ativos` (registrada antes das rotas com `{id}`) e `GET /bilhetes?placa=`; ordenação por `entrada` desc, depois `id` desc. Testes T41–T47. *Depende de: T7, T8.*
- [ ] **T10 — UC4 Relatório diário.** Agregação sobre encerrados com `saida` no dia; média `(2*soma + n)//(2*n)`; `data_invalida`. Testes T48–T55. *Depende de: T7.*
- [ ] **T11 — SDLC.** `Containerfile` e `Dockerfile` (idênticos, `EXPOSE 8004`, `CMD uvicorn`), `README.md` em português (podman, docker, uvicorn, pytest, tabela de rotas com status de sucesso e erro). *Depende de: T1.*
- [ ] **T12 — Verificação final.** Rodar `pytest` (≥ 40 testes passando), `ruff check .`, subir o serviço na 8004 e conferir manualmente UC1→UC2 com `entrada` antiga. *Depende de: T3–T11.*

## Definition of Done (checklist)

- [ ] Nenhuma resposta contém `detail`, `valor` ou número com ponto decimal.
- [ ] Teto 8000 aplicado (871 min → 8000; 870 min → 7975).
- [ ] 15 min → 137; 16 min → 275; 60 min → 550; 61 min → 687.
- [ ] Placa libera após encerrar e após cancelar.
- [ ] `/bilhetes/ativos` não é capturada por rota com `{id}`.
- [ ] Existem `Containerfile`, `Dockerfile`, `README.md`, `requirements.txt`, `tests/` com ≥ 40 testes.
- [ ] Sem segredos, sem implementação duplicada de constantes fora de `app/config.py`.