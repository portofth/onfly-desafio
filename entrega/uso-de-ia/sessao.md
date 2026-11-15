# Sessão com a IA — prompts e respostas

**Ferramenta:** Claude (Sonnet 5.5), chat com sandbox Linux.

## Como ler este arquivo

- **Prompts do responsável:** transcritos literalmente.
- **Respostas do assistente:** a resposta final de cada turno, transcrita literalmente (dentro de blocos
  de código, para preservar o texto exato; as tabelas aparecem em markdown cru).
- **Ações executadas:** uma linha por turno com o que o assistente rodou ou editou. É um **resumo**.
  As saídas brutas dos comandos e as mensagens curtas de progresso entre as chamadas de ferramenta
  **não** estão aqui.
- Este arquivo foi montado pelo assistente a partir da conversa; **não é a exportação nativa** da
  ferramenta. A conversa em que ele foi pedido (turno 11) não está incluída.

---

## Turno 1 — leitura e resumo, sem alterar nada

### Prompt

```text
Vou te passar um desafio técnico de debugging em Python. Leia o README.md
inteiro, depois o código em src/onfly_report/ e os testes em tests/.

Regras que valem durante toda a sessão:
- NÃO alterar nada em tests/ nem nos CSVs de data/ (exceto exemplo-validacao, que é só leitura)
- NÃO mudar o contrato do CLI nem o schema do JSON
- Dinheiro sempre em Decimal, arredondando só na serialização (half-up)
- Só mude código quando eu pedir

Neste momento NÃO corrija nada. Me devolva um resumo do que o projeto faz,
do fluxo dos dados (CSV → parser → builder → JSON) e das regras de negócio
do README em formato de checklist.
```

(Anexo: `onfly-desafio-20261115T193546Z-1-001.zip`)

### Ações executadas

Extraiu o zip numa cópia de trabalho (`/home/claude/onfly-desafio`), leu README, `src/`, `tests/` e a
tabela de câmbio, e inspecionou os CSVs somente em leitura (contagens por status, moeda, hotéis).

### Resposta

````markdown
Terminei a leitura do README, de `src/onfly_report/` e de `tests/`, além de uma olhada só de leitura nos CSVs. O resumo vem abaixo e nada foi alterado.

## O que o projeto faz

É um CLI em Python 3.11, só com biblioteca padrão. Ele lê a exportação de reservas de hotel de um cliente e uma tabela de câmbio para BRL. Com isso gera um `report.json` com totais, agregação por mês e ranking de hotéis. Segundo o README, hoje ele entrega números errados.

O dataset real tem 300 reservas (235 `CONFIRMED`, 31 `CANCELLED`, 34 `NO_SHOW`), 12 hotéis e as moedas BRL, USD e EUR.

## Fluxo dos dados

```
CSV de reservas ─► CsvReservationRepository.read() ─► list[RawRow]
CSV de câmbio   ─► CsvRateRepository.read()        ─► dict[moeda, Decimal]
                                  │
RowParser.parse(RawRow) ─► Reservation (date, int, Decimal)
                                  │
ReportBuilder.build():
  1. conta status (total, CONFIRMED, CANCELLED, NO_SHOW)
  2. filtra faturáveis via status_policy.is_billable
  3. para cada faturável:
       calculate_room_nights(checkin, checkout, rooms)
       convert_to_brl(amount, currency, rates)
       acumula em total, por mês (mês do check-in) e por hotel
  4. média = total / noites, ordena by_month e top_hotels, corta em top_n
  5. format_money(): quantize 0.01 com ROUND_HALF_UP só aqui
                                  │
cli.py: json.dumps ─► output/report.json + resumo no stdout
```

- **Códigos de saída:** `0` sucesso. `2` para `InputFileError` e para `--top < 1` (via `parser.error`). `1` para erro inesperado ou falha de escrita.
- **Logs:** vão para stderr, e o stdout fica só com o resumo.
- **Testes:** cada teste monta um CSV temporário, roda `python -m onfly_report` como subprocesso com taxas BRL=1, USD=5.42 e EUR=5.88, e lê o JSON gerado. Todos os testes passam pelo CLI inteiro.

## Regras de negócio (checklist)

**Status**
- [ ] Só `CONFIRMED` é faturável.
- [ ] `CANCELLED` e `NO_SHOW` entram em `total_reservations` e nas contagens por status.
- [ ] `CANCELLED` e `NO_SHOW` ficam fora de noites, valores, `by_month` e `top_hotels`.

**Noites**
- [ ] `room_nights = (checkout - checkin).days * rooms`.
- [ ] A diferença não é inclusiva: 10/01 a 13/01 dá 3 noites.
- [ ] Vale também na virada de mês e de ano.

**Câmbio**
- [ ] `amount_brl = amount * rate_to_brl`, usando `data/taxas_cambio.csv`.
- [ ] Todas as moedas do dataset têm taxa.

**Totais**
- [ ] `total_reservations` é o número de reservas no arquivo.
- [ ] `confirmed`, `cancelled` e `no_show` são contagens por status.
- [ ] `total_room_nights`, `total_amount_brl` e `average_nightly_rate_brl` usam só `CONFIRMED`.
- [ ] Ordem de processamento: ler → converter tipos → contar status → filtrar `CONFIRMED` → agregar.

**Exatidão**
- [ ] Dinheiro em `Decimal`, nunca `float`.
- [ ] Soma exata e arredondamento só na serialização, com 2 casas e half-up.
- [ ] `average = total_amount_brl / total_room_nights`, e `"0.00"` quando não há noites.
- [ ] Datas são civis (`YYYY-MM-DD`).
- [ ] `generated_at` usa offset fixo `-03:00` e é ignorado na comparação.

**Ordenação e recortes**
- [ ] `by_month` em ordem ascendente de `month`, usando o mês do check-in.
- [ ] `top_hotels` por `total_brl` decrescente, com empate por `hotel_id` ascendente.
- [ ] `top_hotels` tem `--top` itens, ou menos se houver menos hotéis.

**Contrato do CLI e do JSON**
- [ ] Argumentos: `--input` e `--rates` obrigatórios, `--output` com padrão `output/report.json`, `--top` com padrão 5 (inteiro ≥ 1).
- [ ] Dinheiro é string com 2 casas, sem separador de milhar.
- [ ] `report_version` é sempre `"1.0"`.
- [ ] O schema do JSON e os nomes dos campos ficam exatamente como no README.

**Fora do escopo de mudança**
- [ ] Não alterar `tests/`, `data/reservas_hoteis.csv` e `data/taxas_cambio.csv`.
- [ ] `data/exemplo-validacao.csv` é só para conferir a amostra: 12 reservas, 28 noites, total 7566.00, média 270.21.

**Entrega**
- [ ] Preencher `entrega/README-ENTREGA.md`.
- [ ] Incluir `entrega/uso-de-ia/` (sessão + `declaracao.md`) se houver uso de IA.
- [ ] Publicar o repositório no GitHub.

## Pontos que chamaram atenção na leitura (sem mexer em nada)

Estes pontos divergem do README. Ainda não rodei os testes nem confirmei nada.

1. **`status_policy.is_billable`:** só exclui `CANCELLED`, então `NO_SHOW` entra como faturável.
2. **`stay_calculator`:** usa `checkout.day - checkin.day` em vez de `(checkout - checkin).days`, o que quebra quando o mês muda.
3. **`currency_converter`:** divide pela taxa em vez de multiplicar e passa por `float`.
4. **`report_builder` (ranking):** ordena `top_hotels` de forma ascendente por valor, e o tie-break usa `hotel_id` ascendente sem inverter o valor.
5. **`csv_reservation_repository`:** corta a leitura em 200 linhas (`if len(rows) == 200: break`), mas o dataset tem 300.

Quando quiser, eu confirmo cada um rodando o `pytest` e a amostra de validação, e então corrigimos um a um.
````

---

## Turno 2 — rodar o pytest

### Prompt

```text
rode o pytest
```

### Ações executadas

Instalou o pytest 8.3.4 fora do projeto (não estava instalado) e rodou `pytest -q`, sem bytecode nem
cache, para não alterar a árvore.

### Resposta

````markdown
Rodei o `pytest -q` e os 5 testes falharam, como o README já avisava. O pytest não estava instalado no sandbox, então instalei a versão 8.3.4 (a de `requirements-dev.txt`) fora do projeto. Sem bytecode nem cache, nada no projeto foi alterado.

| Teste | Resultado | Causa provável |
|---|---|---|
| `test_nights_across_month_boundary` | obteve **-28**, esperava 3 | `checkout.day - checkin.day` (30/01 → 02/02 dá 2 - 30) |
| `test_status_rules_exclude_cancelled_and_no_show` | obteve **120.00**, esperava 100.00 | o `NO_SHOW` de 20.00 entra como faturável |
| `test_top_hotels_are_ranked_descending` | obteve H04, H03, H02, esperava H01, H02, H03 | ordenação ascendente por valor |
| `test_currency_conversion_to_brl` | falhou na asserção | divide em vez de multiplicar (usa `float`) |
| `test_regression_report_on_sample` | falhou na asserção | efeito combinado de câmbio e noites |

Os 4 primeiros bugs que listei antes se confirmam. No `test_regression_report_on_sample` não vi o valor exato obtido, porque o trecho do log que mostrei estava cortado. Isso não muda o diagnóstico, pois todos os campos dele dependem de câmbio e noites.

O bug do limite de 200 linhas em `csv_reservation_repository` **não é coberto por nenhum teste fornecido**, porque os testes usam poucas linhas. Ele só aparece no dataset real (300 reservas), então precisa de verificação própria, por exemplo pela amostra de validação ou por um teste novo.

Quer que eu rode o CLI na amostra `exemplo-validacao.csv` e compare com a tabela do README antes de começar a corrigir?
````

---

## Turno 3 — rodar a amostra de validação

### Prompt

```text
ok
```

### Ações executadas

Rodou o CLI em `data/exemplo-validacao.csv` com saída em `/tmp/exemplo.json` e comparou, campo a
campo, com a tabela esperada do README.

### Resposta

````markdown
Rodei o CLI na amostra, com a saída em `/tmp/exemplo.json` para não tocar no projeto. Ele terminou com exit 0, mas os números divergem do README em quase tudo.

**Resumo:**

| Campo | Obtido | Esperado |
|---|---|---|
| `total_reservations`, `confirmed`, `cancelled`, `no_show` | 12, 9, 2, 1 | 12, 9, 2, 1 (**OK**) |
| `total_room_nights` | **-33** | 28 |
| `total_amount_brl` | **5962.41** | 7566.00 |
| `average_nightly_rate_brl` | **0.00** | 270.21 |

As contagens por status estão certas porque são feitas antes do filtro. O que quebra são noites, valores e tudo que vem depois.

**`by_month`:**

| Mês | Obtido | Esperado |
|---|---|---|
| 2025-01 | 9 reservas, -36 noites, 5936.90 | 8 reservas, 25 noites, 6684.00 |
| 2025-02 | 1 reserva, 3 noites, 25.51 | 1 reserva, 3 noites, 882.00 |

O 9 contra 8 em janeiro é o `NO_SHOW` vazando para dentro dos faturáveis.

**`top_hotels`:** vieram H09, H02, H05, H08, H07, quando o esperado era H01, H02, H12, H10, H09. A ordem está invertida (ascendente), e os valores estão errados por causa do câmbio dividido e do `NO_SHOW`.

Os defeitos 1 a 4 explicam essas divergências, e o `by_month` de fevereiro, com noites certas e valor errado (25.51 em vez de 882.00), isola o câmbio dividido. A amostra tem só 12 linhas, então o limite de 200 linhas **continua sem confirmação**. Ele só aparece com o dataset real de 300.

Por onde quer começar as correções? Sugiro um defeito por vez, rodando pytest e amostra depois de cada um, para ficar fácil registrar causa raiz e validação no `README-ENTREGA.md`.
````

---

## Turno 4 — defeito 1 (status)

### Prompt

```text
Vamos começar pelo primeiro defeito
```

### Ações executadas

Editou `services/status_policy.py`; rodou `pytest` e a amostra; comparou o arquivo com o original do zip.

### Resposta

````markdown
Corrigi o defeito 1 e o `test_status_rules_exclude_cancelled_and_no_show` passou. Agora são 4 falhas e 1 teste passando, contra 5 falhas antes.

**Mudança** (única linha alterada, confirmada por diff contra o original do zip), em `services/status_policy.py`:

```diff
-    return status != "CANCELLED"
+    return status == "CONFIRMED"
```

**Causa raiz:** o `is_billable` excluía só `CANCELLED`, então `NO_SHOW` entrava em noites, valores, `by_month` e `top_hotels`.

**Efeito na amostra:**
- `by_month` de janeiro passou de 9 para **8 reservas**, o valor esperado.
- O total caiu de 5962.41 para 5662.41. A diferença de 300.00 é o `NO_SHOW` que saiu.
- Noites, valores e ranking continuam errados, porque dependem dos defeitos 2 a 4.

Restam 4 falhas, uma por defeito pendente: noites, câmbio, ordenação do ranking e a regressão ponta a ponta, que depende dos outros. Os testes e os CSVs não foram tocados.

As edições estão na cópia de trabalho em `/home/claude/onfly-desafio`, que não está em `/mnt/user-data/outputs`. Eu a exporto no final, junto com o `README-ENTREGA.md`.

Seguimos para o defeito 2, as noites na virada de mês?
````

---

## Turno 5 — defeito 2 (noites)

### Prompt

```text
ok
```

### Ações executadas

Editou `services/stay_calculator.py`; rodou `pytest` e a amostra; comparou o arquivo com o original.

### Resposta

````markdown
Corrigi o defeito 2: o `test_nights_across_month_boundary` passou e agora são 3 falhas e 2 testes passando.

**Mudança** (uma linha, confirmada por diff contra o original), em `services/stay_calculator.py`:

```diff
-    return (checkout.day - checkin.day) * rooms
+    return (checkout - checkin).days * rooms
```

**Causa raiz:** o cálculo subtraía o dia do mês em vez da diferença entre as datas, então a virada de mês dava resultado negativo. Exemplo: 30/01 → 02/02 resultava em 2 - 30 = -28.

**Efeito na amostra:**
- `total_room_nights` passou de -34 para **28**, o valor esperado.
- `by_month` bate nas noites: janeiro com 8 reservas e 25 noites, fevereiro com 1 reserva e 3 noites.

As 3 falhas restantes são o câmbio (defeito 3), o ranking (defeito 4) e a regressão ponta a ponta, que depende do câmbio. A média ainda vai divergir até o câmbio ser corrigido. O limite de 200 linhas segue sem confirmação.

Seguimos para o defeito 3, o câmbio?
````

---

## Turno 6 — defeito 3 (câmbio)

### Prompt

```text
sim
```

### Ações executadas

Editou `services/currency_converter.py`; rodou `pytest` e a amostra; comparou o arquivo com o original.

### Resposta

````markdown
Corrigi o defeito 3: o `test_currency_conversion_to_brl` e o de regressão passaram, e agora são 4 testes passando e 1 falhando.

**Mudança** (uma linha, confirmada por diff contra o original), em `services/currency_converter.py`:

```diff
-    return Decimal(str(float(amount) / float(rate)))
+    return amount * rate
```

**Causa raiz:** a conversão dividia o valor pela taxa em vez de multiplicar, e ainda passava por `float`, o que perde exatidão no dinheiro. Agora o cálculo é `Decimal` puro, sem arredondamento no meio. O arredondamento half-up continua só no `format_money`, na serialização.

**Efeito na amostra:**
- `total_amount_brl` é **7566.00** e `average_nightly_rate_brl` é **270.21**, os dois iguais ao esperado.
- `by_month` bate: janeiro com 6684.00 e fevereiro com 882.00.

A única falha restante é o `test_top_hotels_are_ranked_descending`, o ranking com ordem invertida. O `top_hotels` da amostra ainda não bate por isso. O limite de 200 linhas segue sem confirmação.

Seguimos para o defeito 4, a ordenação do `top_hotels`?
````

---

## Turno 7 — defeito 4 (ranking)

### Prompt

```text
sim
```

### Ações executadas

Editou `services/report_builder.py`; rodou `pytest`, comparou o `top_hotels` da amostra com o README,
checou empates de valor no dataset real e comparou o arquivo com o original.

### Resposta

````markdown
Corrigi o defeito 4: o `pytest` agora dá **5 passed**.

**Mudança** (uma linha, confirmada por diff contra o original), em `services/report_builder.py`:

```diff
-        ranked_hotels = sorted(hotels.items(), key=lambda item: (item[1]["amount"], item[0]))
+        ranked_hotels = sorted(hotels.items(), key=lambda item: (-item[1]["amount"], item[0]))
```

**Causa raiz:** o ranking ordenava por valor ascendente, então os hotéis de menor valor apareciam primeiro. O sinal negativo inverte só o valor e mantém `hotel_id` ascendente no desempate, como o README pede.

**Efeito na amostra:** o `top_hotels` ficou idêntico à tabela do README (H01, H02, H12, H10, H09, com os mesmos valores e noites). A amostra agora bate em todos os campos: summary, `by_month` e `top_hotels`.

No dataset real não há empates entre os 12 hotéis, nem pelo valor exato nem pelo arredondado, então o desempate por `hotel_id` não tem efeito nele. Ele continua valendo para outros dados.

Os números do dataset real que apareceram acima ainda **não são confiáveis**: o limite de 200 linhas no `csv_reservation_repository.py` continua ativo e descarta 100 das 300 reservas. Nenhum teste fornecido cobre isso, então o pytest verde não o detecta.

Seguimos para o defeito 5, o corte em 200 linhas?
````

---

## Turno 8 — defeito 5 (limite de 200 linhas)

### Prompt

```text
ok
```

### Ações executadas

Editou `repositories/csv_reservation_repository.py`; rodou `pytest`; gerou o relatório do dataset real
e o comparou com um cálculo independente (CSV + `Decimal`, sem usar o código do projeto); comparou o
arquivo com o original.

### Resposta

````markdown
Corrigi o defeito 5 e o `pytest` segue em **5 passed**. Com isso os 5 defeitos que listei estão corrigidos.

**Mudança** (duas linhas removidas, confirmadas por diff contra o original), em `repositories/csv_reservation_repository.py`:

```diff
                     rows.append(RawRow(line_number=reader.line_num, values=values))
-                    if len(rows) == 200:
-                        break
```

**Causa raiz:** a leitura parava na reserva 200, então 100 das 300 reservas do dataset real ficavam fora do relatório.

**Validação no dataset real** (`data/reservas_hoteis.csv`):
- O CLI sai com exit 0 e processa as **300 reservas**: 235 confirmadas, 31 canceladas e 34 `NO_SHOW`.
- Dá 1347 noites, 396170.00 de total e 294.11 de média.
- Como nenhum teste cobre o dataset completo, escrevi um cálculo independente em Python (CSV + `Decimal`, sem usar o código do projeto) e comparei com o JSON. `summary`, os 7 meses de `by_month` e o `top_hotels` ficaram **idênticos**.
- O top 5 é H06, H10, H12, H07 e H11.

Esse cálculo independente foi escrito por mim a partir das regras do README. Ele confirma que o código implementa as regras, mas não substitui a conferência do time da Onfly.

O defeito 5 não tem teste que o proteja. O README permite testes novos, mas sua regra proíbe mexer em `tests/`, então não criei nenhum. Se quiser um teste de regressão para o limite de 200 linhas, posso criá-lo, mas preciso da sua autorização por causa dessa regra.

Os próximos passos possíveis são:
- Rodar o `ruff` (está em `requirements-dev.txt`) para conferir o estilo.
- Escrever o `entrega/README-ENTREGA.md` com os 5 defeitos, causa raiz, investigação e validação.
- Exportar o projeto corrigido para `/mnt/user-data/outputs`.

Qual deles quer agora?
````

---

## Turno 9 — busca por outros defeitos

### Prompt

```text
Vamos verificar se não existem outros defeitos
```

### Ações executadas

Releu todo o código e o README; auditou o dataset real (BOM, CRLF, duplicatas, espaços, datas,
durações, valores, nomes de hotel); rodou `ruff check`; mediu a sensibilidade do arredondamento
(half-up × half-even); executou o CLI em casos de borda com arquivos temporários em `/tmp`; validou
o contrato do JSON; conferiu com `cmp` que `tests/` e `data/` não mudaram.

### Resposta

````markdown
Não encontrei nenhum outro defeito contra as regras do README. Reli o código inteiro e conferi cada regra, tanto no dataset real quanto em casos montados à mão. Não alterei nada nesta etapa.

## Regras do README conferidas

| Regra | Como conferi | Resultado |
|---|---|---|
| Só `CONFIRMED` é faturável | dataset real e casos com `CANCELLED` e `NO_SHOW` de 9999.00 | ✅ ficam de fora de tudo, menos das contagens |
| Noites com `.days * rooms` | 93 estadias viram mês e 16 viram ano no dataset | ✅ |
| `by_month` pelo mês do check-in | caso 30/12 → 02/01 cai em `2024-12` | ✅ |
| Câmbio `amount * rate` em `Decimal` | cálculo independente, sem `float` no `src/` | ✅ |
| Arredondamento só na serialização, half-up | 0.005 vira 0.01 (half-even daria 0.00) | ✅ |
| Desempate por `hotel_id` ascendente | H09 e H03 empatados, com H09 lido primeiro: sai H03, H09 | ✅ |
| Média com `"0.00"` sem noites | CSV só com cabeçalho | ✅ |
| `--top` maior que o número de hotéis | `--top 99` com 2 hotéis | ✅ devolve 2 |
| Contrato do CLI e códigos de saída | `--top` 0, -3 ou "abc", argumento faltando e arquivo inexistente dão **2**; sucesso dá **0** | ✅ |
| Schema do JSON | chaves, tipos, dinheiro em string com 2 casas, `generated_at` com `-03:00` | ✅ |
| Consistência interna | soma de `by_month` bate com os totais (1347 noites, 235 reservas) | ✅ |
| Resultado determinístico | duas execuções iguais, exceto `generated_at` | ✅ |
| stdout e stderr | 1 linha no stdout e stderr vazio no sucesso | ✅ |
| `tests/` e `data/` inalterados | `cmp` contra o zip original | ✅ todos idênticos |

O dataset real é limpo: sem BOM, duplicatas, espaços sobrando, datas fora do padrão, `checkout <= checkin`, valores negativos, hotéis com nomes divergentes ou moedas sem taxa. Nenhum dos bugs que corrigi mascarava outro por trás.

O total exato do dataset é `396169.99940000`. Arredondar por linha daria 396169.99, e o código dá 396170.00, o valor correto pela regra do README.

## Observações fora do contrato (não são defeitos)

O README diz que os dados são consistentes, então estas entradas não ocorrem. Mapeei o comportamento caso apareçam:

- **Moeda sem taxa ou em minúscula** (`GBP`, `usd`): dá `KeyError` e exit **1**, com traceback. Por ser entrada inválida, poderia ser exit 2 com mensagem clara.
- **Status em minúsculo ou desconhecido** (`confirmed`, `PENDING`): fica fora do faturável e das contagens por status, mas entra em `total_reservations`, sem erro.
- **CSV com BOM** (comum em exportações do Excel): dá exit 2 com "colunas obrigatórias ausentes". A mensagem é clara, mas ler com `utf-8-sig` aceitaria o arquivo.
- **`--output` apontando para um diretório**: exit 1 com log de erro. Está de acordo com o contrato.

## Limites desta verificação

- O sandbox usa Python **3.12.3**, e o README pede 3.11+. O código usa só recursos compatíveis, mas não rodei em 3.11.
- O `ruff check` passa. O `ruff format --check` acusaria `report_builder.py`, mas já acusava no original e o formatador não faz parte da configuração do projeto, então deixei como está.
- Os testes fornecidos não cobrem o limite de 200 linhas, o half-up, o desempate por `hotel_id` nem o `by_month` na virada de ano. Conferi tudo isso manualmente, então nenhuma dessas regras está protegida contra regressão.

Quer que eu faça alguma das duas coisas abaixo?
- **Endurecer a entrada**: moeda sem taxa vira exit 2, normalizar caixa de status e moeda, ler com `utf-8-sig`. São mudanças opcionais, fora do que o README exige.
- **Seguir para a entrega**: escrever o `README-ENTREGA.md` e exportar o projeto para `/mnt/user-data/outputs`.
````

---

## Turno 10 — entrega

### Prompt

```text
seguir para a entrega
```

### Ações executadas

Rodou `pytest -q` (`5 passed`) e listou os arquivos que diferem do zip original (exatamente 5, em
`src/`); criou `entrega/README-ENTREGA.md`, `entrega/uso-de-ia/declaracao.md` e a primeira versão do
`sessao.md`; removeu o `.ruff_cache` criado pelas próprias execuções; empacotou o projeto em
`onfly-desafio-corrigido.zip` e validou numa extração limpa (`5 passed`, CLI com exit 0 e 300
reservas, `tests/` e `data/` idênticos ao original).

### Resposta

````markdown
O projeto corrigido está empacotado em `onfly-desafio-corrigido.zip`, com `entrega/` preenchida. Extraí o zip numa pasta limpa e rodei de novo: `pytest -q` dá `5 passed` e o CLI processa as 300 reservas (exit 0). Os arquivos de `tests/` e `data/` são idênticos aos do zip original.

Em `entrega/` ficaram:
- **`README-ENTREGA.md`:** os 5 defeitos com causa raiz em uma frase e o diff de cada um, como investiguei, a evolução do `pytest` a cada correção, como validar e o resultado final.
- **`uso-de-ia/declaracao.md`:** ferramenta e modelo (Claude Sonnet 5.5) e o que a IA fez e o que você fez.
- **`uso-de-ia/sessao.md`:** registro consolidado da sessão.

Antes de subir para o GitHub:

1. **Exportação nativa:** o `sessao.md` é um registro consolidado, escrito por mim ao final da sessão, e não a exportação nativa da ferramenta. O README pede o formato nativo, então, se a interface do Claude permitir exportar a conversa, anexe essa exportação em `uso-de-ia/` (por exemplo, `sessao-02.md`).
2. **Sua revisão:** a `declaracao.md` tem um aviso "Complete antes de enviar". Preencha o que você revisou ou rodou por conta própria, pois não posso afirmar isso por você. Depois apague o aviso.
3. **Python 3.11:** a verificação foi feita em Python 3.12.3, e o README pede 3.11+. Se puder, rode o `pytest -q` na sua máquina com 3.11 antes de enviar.
4. **Repositório:** o repositório público e o envio do link pelo formulário ficam com você.

Não criei testes novos, por causa da sua regra de não mexer em `tests/`. O README permite testes adicionais e diz que eles são considerados na avaliação. Se quiser proteger o limite de 200 linhas, o half-up e o desempate do ranking, posso escrevê-los.
````
