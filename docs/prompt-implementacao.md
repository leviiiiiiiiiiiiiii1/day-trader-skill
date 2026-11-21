# Prompt de implementação: volta na média em M1

Como usar:

- **Dentro deste repositório** (Claude Code, Arena, Cursor com o repo aberto): cole
  o bloco abaixo como a primeira mensagem. O agente tem acesso a
  `docs/volta-na-media-m1.md` e às convenções do pacote, e o prompt manda ler os dois.
- **Fora do repositório**: cole o bloco e, em seguida, o conteúdo de
  `docs/volta-na-media-m1.md`. O prompt funciona sozinho, mas a spec tem as
  tolerâncias em ATR, os casos-limite e o protocolo de validação que valem mais
  que qualquer resumo.

O prompt foi escrito para um agente que vai **escrever código e testes**, não para
um agente que vai operar. Ele proíbe rede, proíbe ordem e exige prova por teste.

---

```
Você é engenheiro de software sênior. Vai implementar o motor de sinal de uma
estratégia de day trade chamada "volta na média", em candles de 1 minuto, com
call e put. O trabalho é código mais testes. Nada além disso.

## Leia antes de escrever qualquer linha

1. docs/volta-na-media-m1.md, a especificação completa. Ela é a fonte da verdade.
   Em qualquer conflito entre este prompt e a spec, a spec vence.
2. tests/test_repo.py, para saber quais convenções o repositório verifica.
3. trader-entry/entry.py, apenas como referência de estilo: Python 3.8, biblioteca
   padrão, funções puras, docstring de módulo explicando as regras.

## Objetivo

Um motor determinístico que, dada uma série de candles de 1 minuto, responde:
houve sinal? de que lado? com que contexto? E um backtest que mede se a regra tem
vantagem estatística sobre o ponto de equilíbrio do payout.

## Resumo das regras (a spec detalha cada tolerância)

Indicadores: EMA 20 e EMA 50 do fechamento, ATR 14 de Wilder. Aquecimento de 250
candles, antes disso nenhum sinal é emitido.

Tendência, lado compra: E20 > E50 + 0.10 ATR, as duas subindo (inclinação medida
em 5 candles, 0.02 ATR por candle para a EMA 20 e 0.01 para a EMA 50), nenhum
cruzamento nos últimos 10 candles e no máximo 1 cruzamento em 30. Venda é o
espelho. Caso contrário o estado é lateral, e lateral significa não operar.

Volta: o preço precisa ter se afastado antes (H chegou a 0.30 ATR acima da E20 nos
últimos 10 candles) e depois encostar na E20 (tolerância de 2% do ATR) sem afundar
mais de 0.25 ATR abaixo da E50. Se fechar abaixo de E50 menos 0.10 ATR, a volta
virou reversão e é cancelada.

Rejeição: martelo (pavio inferior >= 2x o corpo, >= 0.25 ATR, corpo <= 35% do
range, fechamento no terço superior), engolfo de alta (corpo engole corpo,
anterior vermelha) ou confirmação por fechamento acima da máxima anterior. Vale no
candle do toque ou em até 3 candles depois.

Gatilho: o candle de rejeição precisa estar FECHADO. Entrada na abertura do candle
seguinte, expiração em 1 candle (60 segundos), strike no preenchimento. Atraso
maior que 3 segundos descarta o sinal. Uma posição por vez, no máximo 3 sinais por
perna de tendência, cooldown de 2 candles após cada sinal.

Filtros bloqueantes: janela de sessão, payout mínimo 0.85, spread/ATR <= 0.10,
spike de ATR (3x a mediana), buraco na série, feed atrasado. Nenhum deles é aviso:
todos colocam o motor em OFF.

## Entregáveis

    trader-reversion/reversion.py          motor e CLI
    trader-reversion/config.example.json   os padrões da seção 12 da spec
    trader-reversion/README.md             uso, formato de entrada, decisões tomadas
    tests/test_reversion.py                os testes da lista abaixo
    tests/fixtures/trader-reversion/       séries sintéticas, claramente marcadas

Além disso, uma linha em tests/test_repo.py: a lista da raiz passa a incluir
"trader-reversion". Não adicione a pasta à lista SKILLS, porque ela não é um skill
com SKILL.md e o teste de frontmatter falharia.

## CLI

Sem rede. Toda entrada vem de arquivo, para o resultado ser reproduzível.

    python3 trader-reversion/reversion.py check --series FILE [--config FILE] [--now "2026-11-20 14:31"] [--json]
    python3 trader-reversion/reversion.py backtest --series FILE [--payout 0.85] [--config FILE] [--json]

--series aceita CSV com cabeçalho time,open,high,low,close,volume (time em ISO 8601
com fuso). Opcionalmente aceite também o JSON v8 do Yahoo, no mesmo formato dos
fixtures de trader-entry, mas o CSV é obrigatório.

check imprime, em texto:

    WIN$N  2026-11-20 14:31 ET  1m
    CALL  strike 128545.0  expira 14:32  (pin, sep 0.67 ATR, ext 0.52 ATR)
    Nada foi enviado: este motor não envia ordens.

ou, sem sinal, a razão objetiva:

    SEM SINAL  tendencia lateral (2 cruzamentos em 30 candles)

O texto do motivo precisa nomear a regra que barrou, não dizer "condições não
atendidas".

backtest imprime operações, ganhos, perdas, empates, taxa de acerto, intervalo de
confiança de 95% (Wilson), o ponto de equilíbrio 1/(1+payout), o resultado médio
por operação em stakes e o drawdown máximo em stakes.

## Testes obrigatórios

Cada um precisa existir e falhar quando a regra correspondente for removida. Use
séries sintéticas construídas no próprio teste, com valores conferíveis à mão.

1. EMA e ATR batem com valores calculados manualmente em uma série curta conhecida.
2. Menos de 250 candles: nenhum sinal, independentemente do formato do gráfico.
3. Série lateral com médias entrelaçadas: nenhum sinal, e o motivo cita lateral.
4. Série de alta com afastamento, toque e martelo: exatamente um CALL, com os
   campos de contexto preenchidos.
5. O espelho da anterior (série invertida em preço): exatamente um PUT.
6. Preço que nunca se afasta da média: nenhum sinal, por ausência de extensão.
7. Fechamento abaixo de E50 menos 0.10 ATR durante a espera: sinal cancelado.
8. Rejeição no quarto candle após o toque: nenhum sinal (janela é 3).
9. Sem look-ahead: avaliar a série truncada no candle i produz a mesma decisão que
   avaliar a série completa até i. Teste com pelo menos 5 pontos de truncamento.
10. Candle em formação (série terminando no meio de um minuto) nunca gera sinal.
11. Buraco na série reinicia os indicadores e bloqueia sinal até novo aquecimento.
12. Cooldown e limite de 3 sinais por perna são respeitados em série longa de alta.
13. Backtest: em série sintética com resultado conhecido, as contagens de
    ganho/perda/empate batem, o intervalo de Wilson bate com valores precalculados
    e o ponto de equilíbrio é 54.05% para payout 0.85.
14. Uma posição por vez: dois sinais no mesmo minuto geram uma única operação.

## Restrições do repositório

- Python 3.8, só biblioteca padrão. Sem zoneinfo, sem match, sem anotações list[]
  ou dict[].
- Nenhum travessão em nenhum arquivo: tests/test_repo.py verifica isso.
- Não toque em nenhuma pasta que já existe, exceto a linha da raiz em
  tests/test_repo.py.
- Determinismo total: nenhuma dependência de relógio do sistema nos caminhos
  testados, nenhum valor aleatório sem semente declarada.

## Não faça

- Não envie ordem, não conecte em corretora, não leia API, não peça chave.
- Não avalie candle em formação, nem use o fechamento do candle antes de ele
  fechar.
- Não ajuste tolerância para o teste passar. Se a regra não gera sinal nos
  fixtures, o motor está errado; se gera sinal demais, o problema é a série
  sintética, que precisa ser corrigida com o motivo escrito no README.
- Não implemente martingale, hedge, dobra de stake nem recuperação de perda.
- Não invente parâmetro que não esteja na spec sem registrar a decisão no README.

## Definição de pronto

1. cd tests && python3 -m unittest discover -q termina com OK e com os testes novos
   contando na soma.
2. O comando check roda sobre um fixture e imprime um sinal CALL completo.
3. O comando backtest roda sobre uma série longa sintética e imprime as métricas com
   intervalo de confiança.
4. git status mostra apenas os arquivos dos entregáveis e a linha em tests/test_repo.py.

## Como reportar no final

Liste, em no máximo 15 linhas: os arquivos criados, os comandos que rodou e o
resultado, as decisões que tomou onde a spec era ambígua, e o que deixou de fora e
por quê. Se algum teste obrigatório não passou, diga qual e por quê, sem contornar.
```
