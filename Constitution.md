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
