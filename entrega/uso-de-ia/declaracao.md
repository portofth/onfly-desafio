# Declaração de uso de IA

## Ferramenta e modelo

- **Ferramenta:** Claude, da Anthropic, pela interface de chat (claude.ai / app), com acesso a um
  ambiente Linux (sandbox) para ler arquivos e executar comandos.
- **Modelo:** Claude Sonnet 5.5.
- **Sessão:** uma única conversa, registrada em [`sessao.md`](sessao.md).

## O que a IA fez

- Extraiu o zip e leu o README, o código de `src/onfly_report/` e os testes de `tests/`.
- Resumiu o projeto, o fluxo dos dados (CSV → parser → builder → JSON) e as regras de negócio em
  checklist, e apontou os trechos do código que divergiam do README.
- Rodou o `pytest`, o CLI na amostra de validação e no dataset real, e comparou os resultados com o
  README e com um cálculo independente em `Decimal`.
- Aplicou as 5 correções (uma por vez, cada uma só depois de pedida) e rodou `pytest` e a amostra após
  cada uma.
- Auditou o código e os dados em busca de outros defeitos e testou casos de borda do CLI.
- Redigiu o `entrega/README-ENTREGA.md`, esta declaração e o registro da sessão, e empacotou o projeto.

## O que o responsável fez

- Definiu as regras da sessão: não alterar `tests/` nem os CSVs de `data/`, não mudar o contrato do
  CLI nem o schema do JSON, usar `Decimal` com arredondamento half-up só na serialização, e **só
  mudar código quando pedisse**.
- Conduziu o trabalho em etapas (resumo → pytest → amostra → um defeito por vez → busca por outros
  defeitos → entrega) e autorizou cada correção antes de ela ser feita.
- Decidiu o escopo: seguir para a entrega sem aplicar as melhorias opcionais de robustez de entrada
  (moeda sem taxa, normalização de caixa, BOM) que a IA ofereceu.

> **Complete antes de enviar, se for o caso:** o que você revisou ou executou por conta própria
> (por exemplo, rodar `pytest` e o CLI no seu ambiente, conferir os diffs, testar em Python 3.11).

## Observações

- Nenhum arquivo de `tests/` nem de `data/` foi alterado (conferido com `cmp` contra o zip original).
- As verificações foram feitas em Python 3.12.3 no sandbox da IA; o projeto declara Python 3.11+.
- O arquivo `sessao.md` é um **log consolidado de prompts e respostas**, montado pelo assistente a
  partir da conversa: os prompts e as respostas finais de cada turno estão transcritos literalmente;
  as ações executadas (comandos e edições) aparecem resumidas, e as saídas brutas dos comandos e as
  mensagens curtas de progresso não estão incluídas. Não é a exportação nativa da ferramenta. Se a
  interface permitir exportar a conversa, anexe também essa exportação nesta pasta (por exemplo,
  como `sessao-02.md`).
