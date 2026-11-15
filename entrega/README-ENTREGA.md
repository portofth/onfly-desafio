# Entrega — Consolidador de reservas de hotel

O CLI entregava números errados por **5 defeitos independentes**, um por arquivo.
Todos foram corrigidos com mudanças mínimas (uma linha cada, duas no último), sem alterar
`tests/`, os CSVs de `data/`, o contrato do CLI nem o schema do JSON.

**Resultado:** `pytest -q` → `5 passed`; a amostra de validação bate em todos os campos;
no dataset real o relatório fecha com as 300 reservas e coincide com um cálculo independente.

## Defeitos corrigidos

| # | Arquivo | Causa raiz (uma frase) | Correção |
|---|---------|------------------------|----------|
| 1 | `services/status_policy.py` | `is_billable` só excluía `CANCELLED`, então `NO_SHOW` entrava em noites, valores, `by_month` e `top_hotels`. | `status != "CANCELLED"` → `status == "CONFIRMED"` |
| 2 | `services/stay_calculator.py` | As noites eram calculadas com a diferença do *dia do mês* (`checkout.day - checkin.day`), o que dá resultado errado ou negativo quando a estadia vira o mês. | `(checkout.day - checkin.day) * rooms` → `(checkout - checkin).days * rooms` |
| 3 | `services/currency_converter.py` | A conversão **dividia** o valor pela taxa (em vez de multiplicar) e ainda passava por `float`, perdendo exatidão no dinheiro. | `Decimal(str(float(amount) / float(rate)))` → `amount * rate` (`Decimal` puro) |
| 4 | `services/report_builder.py` | O ranking de hotéis ordenava por valor **crescente**, mostrando os de menor faturamento primeiro. | chave `(amount, hotel_id)` → `(-amount, hotel_id)` |
| 5 | `repositories/csv_reservation_repository.py` | A leitura parava na 200ª linha (`if len(rows) == 200: break`), descartando 100 das 300 reservas do dataset real. | removido o corte; o arquivo é lido por inteiro |

Diffs completos:

```diff
--- services/status_policy.py
-    return status != "CANCELLED"
+    return status == "CONFIRMED"

--- services/stay_calculator.py
-    return (checkout.day - checkin.day) * rooms
+    return (checkout - checkin).days * rooms

--- services/currency_converter.py
-    return Decimal(str(float(amount) / float(rate)))
+    return amount * rate

--- services/report_builder.py
-        ranked_hotels = sorted(hotels.items(), key=lambda item: (item[1]["amount"], item[0]))
+        ranked_hotels = sorted(hotels.items(), key=lambda item: (-item[1]["amount"], item[0]))

--- repositories/csv_reservation_repository.py
                     rows.append(RawRow(line_number=reader.line_num, values=values))
-                    if len(rows) == 200:
-                        break
```

## Como investiguei

1. **Leitura antes de mexer.** Li o README inteiro, o código de `src/onfly_report/` e os testes, e
   montei o fluxo (CSV → parser → builder → JSON) e um checklist das regras de negócio. Ao ler,
   listei os pontos do código que divergiam do README (os 5 acima).
2. **Linha de base.** Rodei `pytest -q`: 5 falhas. Cada teste apontava para uma regra
   (câmbio, noites, status, ranking, regressão ponta a ponta).
3. **Amostra de validação.** Rodei o CLI em `data/exemplo-validacao.csv` e comparei com a tabela do
   README: contagens por status corretas, mas noites (-33), total (5962.41), média (0.00) e ranking
   errados — o que isolou os defeitos de noites, câmbio, status e ordenação.
4. **Um defeito por vez**, validando com `pytest` **e** com a amostra depois de cada correção:

   | Após corrigir | `pytest -q` | Amostra |
   |---|---|---|
   | (linha de base) | 5 failed | divergente |
   | 1 — status | 4 failed, 1 passed | `by_month` de jan. passa de 9 para 8 reservas |
   | 2 — noites | 3 failed, 2 passed | noites = 28 (bate) |
   | 3 — câmbio | 1 failed, 4 passed | total 7566.00 e média 270.21 (batem) |
   | 4 — ranking | **5 passed** | `top_hotels` idêntico ao README |
   | 5 — limite de 200 | 5 passed | (amostra tem só 12 linhas) |

5. **O defeito 5 não é pego por nenhum teste fornecido nem pela amostra** (poucas linhas). Foi
   encontrado na leitura do código e confirmado no dataset real: antes o relatório saía com 200
   reservas; agora com 300.
6. **Verificação independente.** Escrevi um cálculo à parte (CSV + `Decimal`, sem usar o código do
   projeto) a partir das regras do README e comparei com o JSON do dataset real: `summary`,
   `by_month` (7 meses) e `top_hotels` ficaram idênticos.
7. **Busca por outros defeitos.** Releitura completa do código, auditoria do dataset (sem BOM,
   duplicatas, espaços, datas fora do padrão ou valores negativos) e testes de borda pelo CLI:
   arquivo só com cabeçalho, `--top` inválido ou maior que o nº de hotéis, desempate por
   `hotel_id`, half-up em `0.005`, virada de ano, diretório de saída aninhado e códigos de saída.
   Nenhum defeito adicional contra as regras do README.

## Como validar

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt

pytest -q

PYTHONPATH=src python -m onfly_report \
  --input data/exemplo-validacao.csv \
  --rates data/taxas_cambio.csv \
  --output output/exemplo.json

PYTHONPATH=src python -m onfly_report \
  --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv \
  --output output/report.json
```

## Resultado

**`pytest -q`:**

```
.....                                                                    [100%]
5 passed in 0.34s
```

**Amostra (`exemplo-validacao.csv`):** 12 reservas (9 / 2 / 1), 28 noites, `7566.00`, média
`270.21`; `by_month` 2025-01 = 8 / 25 / `6684.00` e 2025-02 = 1 / 3 / `882.00`; `top_hotels`
H01, H02, H12, H10, H09 com os valores do README.

**Dataset real (`reservas_hoteis.csv`):**

| Campo | Valor |
|---|---|
| `total_reservations` | 300 |
| `confirmed` / `cancelled` / `no_show` | 235 / 31 / 34 |
| `total_room_nights` | 1347 |
| `total_amount_brl` | 396170.00 |
| `average_nightly_rate_brl` | 294.11 |
| `by_month` | 7 meses, de 2024-12 a 2025-06 |
| `top_hotels` (top 5) | H06 (45165.46), H10 (45041.69), H12 (41647.69), H07 (39920.52), H11 (37455.69) |

O total exato é `396169.99940000`. Arredondar linha a linha daria `396169.99`; o código soma os valores
exatos e arredonda só na serialização (half-up), como o README exige.

## Notas e limites

- **Verificado em Python 3.12.3**; o projeto declara 3.11+ e o código só usa recursos compatíveis,
  mas a execução em 3.11 não foi testada aqui.
- **Não adicionei testes novos nesta entrega.** Os testes fornecidos não cobrem o limite de 200
  linhas, o half-up, o desempate por `hotel_id` nem o `by_month` na virada de ano; essas regras foram
  conferidas manualmente (item 7 acima). Recomendo testes de regressão para elas.
- **Fora do escopo, não alterado** (o README garante dados consistentes): moeda sem taxa ou em
  minúsculo gera `KeyError` (exit 1, poderia ser 2); status em minúsculo ou desconhecido fica fora do
  faturável sem erro; CSV com BOM é recusado com exit 2 (poderia ser lido com `utf-8-sig`).
- `ruff check` passa. `ruff format --check` acusaria `report_builder.py`, mas já acusava no original e
  o formatador não faz parte da configuração do projeto.

## Uso de IA

Usei um assistente de IA nesta tarefa. A declaração e o registro da sessão estão em
[`uso-de-ia/`](uso-de-ia/).
