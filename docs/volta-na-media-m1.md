# Volta na média: especificação para automação

**Versão 1.0 · alvo: candles de 1 minuto (M1) · call e put · opções binárias**

Este documento descreve, em regras executáveis, a estratégia "volta na média"
apresentada por Pedro. Ele serve para uma pessoa ou um agente implementar a
automação sem precisar rever o vídeo: cada frase da fala virou uma condição
com número, tolerância e ponto de decisão declarados.

> **Leia a seção 21 antes de operar.** Opções binárias têm esperança matemática
> negativa por construção (a corretora é a contraparte e fica com a margem do
> payout), não têm câmara de compensação e no Brasil a CVM não autoriza essas
> plataformas. O que está aqui é a descrição de uma regra técnica e o método
> para medir se ela tem vantagem. Não é recomendação de investimento, não é
> previsão e não promete resultado nenhum.

---

## Sumário

| # | Seção |
| --- | --- |
| 1 | Resumo em uma página |
| 2 | Notação |
| 3 | Contrato de dados de entrada |
| 4 | Indicadores: EMA 20, EMA 50, ATR 14 |
| 5 | Regra 1: tendência |
| 6 | Regra 2: volta na média |
| 7 | Regra 3: vela de rejeição |
| 8 | Gatilho, entrada e expiração |
| 9 | Filtros (o "filtro" da estratégia) |
| 10 | Máquina de estados |
| 11 | Pseudocódigo completo |
| 12 | Parâmetros e arquivo de configuração |
| 13 | Formato do sinal (JSON) |
| 14 | Casos-limite e tratamento |
| 15 | O que muda no M1 em relação ao M5 |
| 16 | Os modos de falha |
| 17 | Backtest e validação estatística |
| 18 | Gestão de banca |
| 19 | Ambiguidades da fala e a decisão tomada |
| 20 | Checklist de aceitação |
| 21 | Riscos, limites e conformidade |

---

## 1. Resumo em uma página

Duas médias móveis exponenciais (20 e 50) sobre o fechamento dos candles de 1
minuto. A EMA 20 é a referência de "agora"; a EMA 50 é a direção de fundo. O
preço é tratado como um cachorro na coleira: ele se afasta da média e volta,
e a operação só existe nessa volta.

Para uma **compra (call)**:

1. **Tendência de alta.** EMA 20 acima da EMA 50, as duas subindo, sem
   entrelaçamento recente.
2. **O preço voltou.** O preço recuou da extensão e encostou na EMA 20 (ou
   entrou na faixa entre as duas médias) sem fechar abaixo da EMA 50.
3. **Vela de rejeição.** No candle que tocou a média, ou em até 3 candles
   depois, aparece um martelo ou um engolfo de alta.

Com as três: entra na **abertura do candle seguinte**, expiração de **1
candle (60 segundos)**. Sem uma das três: não entra. A venda (put) é o
espelho exato.

Fluxo de decisão:

```
candle fechou
   |
   +-- filtro de contexto ok? ---------- nao --> OFF (nao avalia)
   |
   +-- tendencia definida? ------------- nao --> OFF (lateral: nao entra)
   |
   +-- preco esticou da EMA20? --------- nao --> aguarda
   |
   +-- encostou na EMA20 sem furar EMA50? - nao --> aguarda / cancela
   |
   +-- vela de rejeicao? --------------- nao --> aguarda (janela 3 candles)
   |
   +-- SIM --> SINAL --> entrada na abertura do proximo candle
```

O que a automação **não** faz: nunca avalia o candle em formação, nunca entra
atrasada, nunca dobra a mão depois de loss, nunca opera contra a tendência
declarada e nunca abre uma segunda posição enquanto uma está aberta.

---

## 2. Notação

| símbolo | significado |
| --- | --- |
| `t` | índice do candle, 0 = primeiro candle da série carregada |
| `O, H, L, C` | abertura, máxima, mínima, fechamento do candle |
| `t_open`, `t_close` | instantes de abertura e fechamento (`t_close = t_open + 60s`) |
| `E20, E50` | EMA 20 e EMA 50 no fechamento do candle `t` |
| `A` | ATR 14 no candle `t` |
| `body` | `abs(C - O)` |
| `range` | `H - L` |
| `upper_wick` | `H - max(O, C)` |
| `lower_wick` | `min(O, C) - L` |
| `close_pos` | `(C - L) / range` (0 = fechou na mínima, 1 = na máxima) |
| `band` | faixa entre as médias: `[E50, E20]` em alta, `[E20, E50]` em baixa |
| `now` | instante da avaliação, sempre >= `t_close` do candle avaliado |

Todas as comparações de preço usam ponto flutuante com tolerância relativa de
`1e-9`. Arredondamento só na saída, nunca no meio do cálculo.

---

## 3. Contrato de dados de entrada

A automação consome uma série de candles de 1 minuto do **mercado real**
(nunca OTC, nunca sintético).

Requisitos obrigatórios:

1. **OHLC completo** e volume (o volume não entra na regra, mas é usado para
   detectar buraco de liquidez, seção 14).
2. **Grade de 60 segundos.** `t_open` múltiplo de 60 no fuso da fonte. Candle
   fora da grade é descartado, não interpolado.
3. **Fuso declarado.** Todo timestamp chega com fuso (`UTC`, `-03:00`,
   `America/New_York`). O motor converte para UTC internamente e mantém o
   horário de sessão em uma variável separada. Nunca deduzir o fuso do
   servidor: o gráfico da corretora pode estar em UTC, em Nova York ou em
   Brasília, e um erro de 1 a 3 horas desloca a janela de negociação inteira.
4. **Fechamento confirmado.** Um candle só entra no cálculo quando o relógio
   passou de `t_close`. Candle em formação é mantido em buffer separado e
   usado apenas para exibição, nunca para sinal.
5. **Sem buraco.** Se faltar candle na sequência (feriado, falha de feed,
   leilão), o motor reinicia os indicadores a partir dali e não trata o salto
   como movimento. Ver seção 14.
6. **Mesma fonte para histórico e tempo real.** Backtest com candles de um
   provedor e execução com candles da corretora produz resultados diferentes.
   O backtest precisa usar a série da corretora sempre que ela existir.

Volume mínimo de histórico: **250 candles** antes do primeiro sinal (ver 4.4).

---

## 4. Indicadores

### 4.1 EMA

Média móvel exponencial sobre o fechamento:

```
k       = 2 / (n + 1)
E[t]    = C[t] * k + E[t-1] * (1 - k)
E[n-1]  = SMA(C[0..n-1])          # semente
```

- `n = 20` para a EMA rápida, `n = 50` para a lenta.
- Semente pela média simples dos `n` primeiros fechamentos. Semente pelo
  primeiro fechamento também é aceitável desde que o aquecimento seja maior.
- **Aquecimento:** manter o indicador "não confiável" nos primeiros `5 * n`
  candles; a influência da semente cai para menos de 0,1% depois de umas 5
  constantes de tempo. Na prática: `>= 100` candles para a EMA 20 e `>= 250`
  para a EMA 50. Enquanto não houver 250 candles, o motor **não emite sinal**.
- Recalcular do zero a cada reconexão ou mudança de sessão, não continuar de
  um estado salvo em memória que possa estar dessincronizado.

### 4.2 ATR

Usado como régua universal de volatilidade, para que todas as tolerâncias se
adaptem ao ativo e ao horário.

```
TR[t]   = max(H[t] - L[t], abs(H[t] - C[t-1]), abs(L[t] - C[t-1]))
A[13]   = SMA(TR[0..13])                       # semente
A[t]    = (A[t-1] * 13 + TR[t]) / 14           # suavização de Wilder
```

Se `A[t]` for zero ou `nan`, o candle é inválido e o motor fica em `OFF`.

### 4.3 Por que ATR e não porcentagem

Uma tolerância fixa em reais ou em porcentagem quebra quando a volatilidade
muda: 0,05% de 100,00 é muito em um minuto calmo e nada em um minuto de CPI.
Com ATR, "encostou na média" quer dizer "chegou a menos de 2% da volatilidade
típica do último minuto", em qualquer ativo e em qualquer horário.

### 4.4 Estado dos indicadores

```
ready = (candles_processados >= 250) and (A != 0) and (E20 != nan) and (E50 != nan)
```

`ready == false` implica `OFF`. Nenhuma exceção.

---

## 5. Regra 1: tendência

A fala original: "a laranja acima da branca, as duas apontando para cima e o
preço por cima delas". Três condições, todas obrigatórias.

### 5.1 Alta

```
T1  E20 > E50 + sep_min * A
T2  E20 > E20[k_slope] + slope_min * A          # EMA rapida subindo
    E50 > E50[k_slope] + slope_slow_min * A     # EMA lenta subindo
T3  sem cruzamento nos ultimos no_cross candles
T4  numero de cruzamentos nos ultimos 30 candles <= max_crosses
T5  fechamento de referencia acima da E50
```

Em que "cruzamento" é a troca de sinal de `sign(E20 - E50)` entre dois candles
consecutivos.

Interpretação de cada condição:

- **T1 separação (`sep_min`).** Duas linhas que se tocam não formam tendência,
  formam indecisão. `sep_min = 0.10` ATR por padrão, faixa útil de 0,05 a
  0,30. Exigir separação é o que mata a maior parte dos falsos positivos no
  M1, onde 20 e 50 candles são apenas 20 e 50 minutos.
- **T2 inclinação.** "Apontando para cima" só existe se for medido. A
  inclinação é a diferença da EMA em `k_slope = 5` candles, dividida por
  `k_slope * A` para virar "ATR por candle". Padrões: `slope_min = 0.02` e
  `slope_slow_min = 0.01`. Uma EMA 50 que sobe menos de 0,01 ATR por minuto
  está praticamente plana: isso é lateral disfarçada de alta.
- **T3 e T4 entrelaçamento.** É a tradução numérica de "no lateral as linhas
  viram uma bagunça". `no_cross = 10` e `max_crosses = 1` em 30 candles.
- **T5 preço.** "Preço por cima delas" precisa ser lido com cuidado: no
  momento do sinal o preço está justamente encostado na EMA 20, então exigir
  `C > E20` no candle do sinal eliminaria a própria estratégia. A condição
  fica: **no candle do sinal, `C > E50 - 0.10 * A`** (estrutura preservada) e,
  no candle em que o preço se esticou, `H > E20 + ext_min * A` (seção 6.1).
  Para a EMA 20, exige-se `C > E20 - 0.25 * A`, ou seja, pode fechar um pouco
  abaixo, nunca muito.

### 5.2 Baixa

Espelho exato: `E20 < E50 - sep_min * A`, as duas caindo, sem cruzamento
recente, e `C < E50 + 0.10 * A`.

### 5.3 Lateral

`trend = NONE` quando nenhuma das duas condições fecha. É o estado padrão do
mercado e deve ser tratado como "não operar", não como erro.

### 5.4 Persistência

A tendência só passa a valer depois de `trend_persist = 3` candles
consecutivos com o mesmo veredito. Isso evita armar o motor no exato candle
em que as médias se abrem.

---

## 6. Regra 2: volta na média

A fala: "depois de subir, o preço recua e encosta na EMA 20 ou fica no espaço
entre as duas médias. Você nunca compra lá em cima, longe da média."

São duas partes: o preço **precisa ter se afastado** e depois **precisa
voltar**.

### 6.1 Extensão (o cachorro correu)

Em algum candle desde que a tendência foi armada, dentro dos últimos
`ext_window = 10` candles:

```
extended_up = (H[j] - E20[j]) >= ext_min * A[j]     para algum j na janela
```

`ext_min = 0.30` ATR. Isso codifica "nunca comprar lá em cima": se o preço
nunca se afastou, não existe volta, existe arrasto.

Também existe o limite oposto, o **não perseguir**:

```
chase = (C[t] - E20[t]) > chase_max * A[t]           # chase_max = 1.0
```

Se o sinal aparecesse com o preço já a um ATR inteiro acima da média, a
entrada estaria comprando o topo do impulso, não a volta. Veta o sinal.

### 6.2 Toque

```
touch_up = (L[t] <= E20[t] + touch_tol) and (L[t] >= E50[t] - depth_pad * A[t])
touch_tol  = max(0.02 * A[t], 0.0002 * price[t])
depth_pad  = 0.25
```

O primeiro termo diz "chegou na EMA 20 ou na faixa". O segundo diz "não
afundou abaixo da EMA 50": um pavio que atravessa as duas médias não é volta,
é rompimento. Toque na faixa (o "espaço entre as duas médias") está
automaticamente coberto, porque a faixa vai de `E50` a `E20`.

No M1 o toque costuma ser um pavio de 1 a 3 candles. Não exigir que o
fechamento encoste: o fechamento pode ficar acima, e é justamente isso que
caracteriza a rejeição.

### 6.3 Cancelamento

A volta vira reversão quando:

```
C[t] < E50[t] - cancel_pad * A[t]         # cancel_pad = 0.10, em alta
```

Nesse caso o candle e a janela de espera são descartados, e o motor espera uma
nova tendência. Nunca "esperar mais um pouco" para ver se volta.

---

## 7. Regra 3: vela de rejeição

A fala: "pode ser um martelo, corpo pequeno e pavio grande embaixo, ou uma
vela verde forte que engole a vermelha de antes". Um padrão ou o outro, no
candle do toque ou em até `rej_window = 3` candles depois.

### 7.1 Martelo (pin bar) de alta

```
pin_up = (range >= min_range * A)                     # min_range = 0.15
     and (lower_wick >= pin_wick * body)              # pin_wick = 2.0
     and (lower_wick >= pin_wick_atr * A)             # pin_wick_atr = 0.25
     and (body <= pin_body * range)                   # pin_body = 0.35
     and (close_pos >= pin_close_pos)                 # pin_close_pos = 0.60
```

Leitura de cada linha: o candle precisa ter corpo útil (`range` mínimo); o
pavio de baixo tem que ser pelo menos o dobro do corpo; o pavio precisa ter
tamanho absoluto relevante em ATR (senão qualquer candle minúsculo "parece"
martelo); o corpo tem que ser pequeno em relação ao range; e o fechamento
tem que ficar no terço superior, porque "tentou cair e foi empurrado de
volta" é o fechamento voltando, não só o pavio.

**Cor do corpo:** o martelo pode ser vermelho. O vídeo não exige vela verde e
a definição clássica também não. Fica configurável em `pin_require_green`,
padrão `false`.

### 7.2 Engolfo de alta

```
engulf_up = (C[t-1] < O[t-1])                         # anterior vermelha
        and (C[t] > O[t])                             # atual verde
        and (C[t] >= O[t-1]) and (O[t] <= C[t-1])     # corpo engole o corpo
        and (body[t] >= body[t-1])
```

**Corpo que engole corpo**, não range que engole range: pavio alheio não
conta. "Verde forte" vira `body[t] >= body[t-1]`, opcionalmente com
`C[t] > H[t-1]` para exigir fechamento acima da máxima anterior
(`engulf_break_high`, padrão `false`).

### 7.3 Variante de confirmação

Para ativos com muito ruído no M1, existe uma terceira aceitação, mais
permissiva e mais fácil de medir:

```
confirm_up = (C[t] > H[t-1]) and (C[t] > O[t]) and (close_pos >= 0.6)
```

Fechou acima da máxima do candle anterior e no terço de cima. Ligada por
`accept_break_prev_high`, padrão `false`. Se ligada, ela **substitui** as
duas anteriores, não soma.

### 7.4 Espelho de baixa

`pin_down`, `engulf_down` e `confirm_down` com as desigualdades invertidas,
`upper_wick`, e `close_pos <= 0.40`.

### 7.5 Regra de ouro

A vela de rejeição é sempre um **candle fechado**. Um martelo que está se
formando às 10:31:40 é um candle em formação e não pode gerar entrada. Se a
automação aceitar candle em formação, ela vai comprar em cima de um desenho
que desaparece no fechamento, e o backtest vai ficar maravilhoso enquanto a
conta afunda.

---

## 8. Gatilho, entrada e expiração

### 8.1 Instante de decisão

```
t_decision = t_close(candle_de_rejeicao)
```

O motor avalia exatamente nesse instante, com o candle já fechado.

### 8.2 Entrada

```
t_entry   = t_open(candle+1) = t_decision
strike    = preco_no_instante_da_ordem
atraso    = t_fill - t_entry
```

- Entrada na **abertura do candle seguinte**. No relógio, é o mesmo instante
  do fechamento do candle de sinal.
- `max_entry_delay_s = 3`. Se a ordem não sair em até 3 segundos (API lenta,
  reconexão, confirmação manual), **o sinal é descartado**. Entrada atrasada
  não é a mesma operação: o strike muda e o candle já andou.
- `strike` é o preço do ativo no momento do preenchimento. Verificar que ele
  está dentro de `0.25 * A` da abertura do candle; fora disso, abortar.
- **Uma posição aberta por vez.** Nenhuma ordem nova enquanto existir
  contrato em aberto.

### 8.3 Expiração

```
t_expiry = t_close(candle+1) = t_entry + 60s
```

Expiração de exatamente um candle. Se a corretora só aceitar expiração
contada do preenchimento (`t_fill + 60s`), registrar o desvio
(`atraso` segundos) e considerá-lo parte do custo; acima de 3 segundos, o
sinal é inválido.

### 8.4 Liquidação

| condição | resultado |
| --- | --- |
| `S(t_expiry) > strike` (call) | ganho = `stake * payout` |
| `S(t_expiry) < strike` (call) | perda = `stake` |
| `S(t_expiry) == strike` | **empate**: consultar a regra da corretora (devolução do stake ou perda). Padrão do motor: contar como empate e registrar separado, nunca como acerto |

Para o put, invertem-se os sinais. O `payout` usado no registro é o que a
plataforma exibiu no momento da ordem, não o do dia anterior: payout muda com
volatilidade e com o ativo.

### 8.5 Custo real da operação no M1

Um detalhe que decide a conta: o strike é comparado com um preço exatamente
60 segundos depois, e a diferença entre compra e venda (spread) entra duas
vezes nessa comparação. Se o spread for grande em relação ao ATR de 1 minuto,
a estratégia começa o jogo perdendo.

```
spread_ratio = (ask - bid) / A[t]
```

Padrão: `spread_ratio > 0.10` implica não operar naquele ativo. Em ações
líquidas e índices o spread costuma ser pequeno; em ativo exótico de corretora
de binárias, não.

---

## 9. Filtros (o "filtro" da estratégia)

O vídeo anuncia "três regras e um filtro", mas a transcrição só entrega as
três regras. Para a automação não existe estratégia com filtro indefinido:
abaixo estão os filtros que a fala sustenta e que precisam existir em
qualquer implementação. Todos são **bloqueantes**: se algum falhar, `OFF`.

| # | filtro | regra | origem |
| --- | --- | --- | --- |
| F1 | **Mercado real, nunca OTC** | feed marcado como OTC ou sintético = bloqueado, sempre | explícito no vídeo |
| F2 | **Janela de sessão** | operar de `open + 5min` até `close - 10min`; fora disso, `OFF` | padrão de day trade |
| F3 | **Primeiro candle** | nunca no primeiro candle da sessão (leilão, abertura, spread largo) | padrão de day trade |
| F4 | **Payout mínimo** | `payout >= payout_min` (padrão 0,85). Abaixo, o empate estatístico fica caro demais (seção 17.1) | consequência matemática |
| F5 | **Notícia de alto impacto** | bloquear `[-2, +5]` minutos em torno de evento agendado (CPI, payroll, decisão de juros, resultado) | o vídeo cita notícia como falha |
| F6 | **Liquidez** | `spread_ratio <= 0.10` e `volume >= 20% da mediana dos últimos 20 candles` | ver 8.5 e 14.3 |
| F7 | **Volatilidade extrema** | `A[t] >= 3 * mediana(A[t-20..t])` implica `OFF` (clímax, spike) | o vídeo cita exaustão como falha |
| F8 | **Dia de fechamento antecipado** | em dia de fechamento antecipado, antecipar o fim da janela junto com o fechamento | padrão de day trade |

Feriado, fim de semana, sessão fechada: `OFF` sem avaliar nada.

---

## 10. Máquina de estados

```
        +-----------------------------------------------------+
        |                                                     |
        v                                                     |
  [ OFF ] --filtro ok + trend != NONE (3 candles)--> [ TREND ]
        ^                                                     |
        |                                     preco esticou    |
        |                                            v          |
        +--- close furou E50 / janela expirou --- [ PULLBACK ]
                                                              |
                                                    tocou a media
                                                              v
                                                        [ ARMED ]
                                                              |
                                              rejeicao + regra 1 ok
                                                              v
                                                          [ SIGNAL ]
                                                              |
                                              cooldown        v
                                                        [ COOLDOWN ] ---> OFF
```

| estado | significado | transições |
| --- | --- | --- |
| `OFF` | sem contexto válido | para `TREND` quando filtros ok e tendência persiste 3 candles |
| `TREND` | tendência confirmada, preço ainda não esticou | para `PULLBACK` quando `extended` fica verdadeiro; para `OFF` se tendência cair |
| `PULLBACK` | preço se afastou, esperando voltar | para `ARMED` no primeiro toque; para `OFF` se `extensão` envelhecer (`ext_window`) ou tendência cair |
| `ARMED` | tocou a média, esperando rejeição | para `SIGNAL` na rejeição; para `OFF` em `rej_window` candles sem rejeição, ou se `C < E50 - cancel_pad*A` |
| `SIGNAL` | sinal emitido, ordem enviada | para `COOLDOWN` após o envio (ou após aborto) |
| `COOLDOWN` | espera obrigatória | para `TREND`/`OFF` após `cooldown_bars` |

Detalhes que a máquina precisa respeitar:

- **Um sinal por perna de tendência:** depois de um sinal, é preciso um novo
  afastamento e um novo toque. Não basta o preço encostar de novo 2 candles
  depois.
- **`max_signals_per_leg = 3`.** Guarda-corpo contra dia de tendência forte em
  que o motor entra 15 vezes seguidas e concentra todo o risco em uma
  direção.
- **Reavaliação da regra 1 no sinal**, não só na armação. Uma tendência que
  morreu durante o toque não vale.
- **Histerese:** entrar em `OFF` exige uma condição clara (furo da EMA 50,
  cruzamento, filtro), não a simples oscilação da separação em torno do
  limite. Sem histerese o motor liga e desliga a cada candle.

---

## 11. Pseudocódigo completo

Executado uma única vez por candle fechado, em ordem cronológica.

```python
state = OFF
leg   = None          # {"dir": +1, "signals": 0}
pend  = None          # {"dir": +1, "age": 0, "extended_at": idx}

def on_candle_closed(i):
    global state, leg, pend

    if not ready(i):                                 # < 250 candles, ATR nan
        state = OFF; leg = None; pend = None; return None

    if not filters_ok(i):                            # secao 9
        state = OFF; leg = None; pend = None; return None

    if now < t_close(i):                             # nunca eval. candle aberto
        return None

    tr = trend(i)                                    # +1, -1 ou 0   (secao 5)
    if tr == 0:
        state = OFF; leg = None; pend = None; return None

    if leg is None or leg["dir"] != tr:
        leg = {"dir": tr, "signals": 0}

    # ---- 1. extensao (cachorro correu) -------------------------------
    if state in (OFF, TREND) or pend is None:
        if extended(i, tr):                          # secao 6.1
            state = PULLBACK
            pend  = {"dir": tr, "age": 0, "extended_at": i}

    # ---- 2. toque e rejeicao -----------------------------------------
    if pend is not None and pend["dir"] == tr:
        pend["age"] += 1

        if breaks_band(i, tr):                       # C furou a E50
            cancel(); return None

        if pend["age"] > ext_window + rej_window:
            cancel(); return None

        if touch(i, tr):
            state = ARMED

        if state == ARMED:
            if rejection(i, tr) and not chase(i, tr) and trend(i) == tr:
                if leg["signals"] < max_signals_per_leg:
                    order = {
                        "direction": "call" if tr > 0 else "put",
                        "t_signal":  t_close(i),
                        "t_entry":   t_close(i),
                        "t_expiry":  t_close(i) + 60,
                        "ref_open":  O[i+1] if exists(i+1) else None,
                        "ref_price": C[i],
                        "ema20": E20[i], "ema50": E50[i], "atr": A[i],
                        "pattern": pattern_name(i, tr),
                        "strike_source": "fill",
                    }
                    leg["signals"] += 1
                    state = COOLDOWN
                    pend  = None
                    return order
    return None


def cancel():
    global state, pend
    state = OFF
    pend  = None
```

Notas de implementação:

- `O[i+1]` ainda não existe no instante do sinal. O campo `ref_open` é
  preenchido quando o candle abre, e serve para conferir o strike (8.2).
- `C[i]` é o último preço conhecido e o que se usa para validar o strike
  quando a corretora preenche rápido.
- `pattern_name` devolve `pin`, `engulf` ou `confirm`, para o log permitir
  medir o desempenho de cada um separadamente. Isso importa: são padrões
  diferentes com taxas de acerto diferentes, e medir os três juntos esconde
  qual funciona.
- O pseudocódigo assume que `i+1` chega em seguida. Se o feed pular, ver 14.2.

---

## 12. Parâmetros e arquivo de configuração

Valores padrão são **ponto de partida**, não valores validados. Todo backtest
deve varrer a faixa indicada e escolher no período de treino, nunca no de
teste (seção 17.5).

| parâmetro | padrão | faixa | afeta |
| --- | --- | --- | --- |
| `ema_fast` | 20 | fixo | regra 1 e 2 |
| `ema_slow` | 50 | fixo | regra 1 |
| `atr_period` | 14 | 14 | todas as tolerâncias |
| `sep_min` | 0.10 | 0.05 a 0.30 | regra 1 (separação mínima em ATR) |
| `slope_k` | 5 | 3 a 8 | regra 1 (janela da inclinação) |
| `slope_min` | 0.02 | 0.01 a 0.04 | regra 1 (EMA 20 em ATR/candle) |
| `slope_slow_min` | 0.01 | 0.005 a 0.02 | regra 1 (EMA 50) |
| `no_cross_bars` | 10 | 5 a 20 | regra 1 (sem cruzamento recente) |
| `max_crosses_30` | 1 | 0 a 2 | regra 1 (entrelaçamento) |
| `trend_persist` | 3 | 2 a 6 | armação da tendência |
| `ext_window` | 10 | 5 a 20 | regra 2 (janela da extensão) |
| `ext_min` | 0.30 | 0.20 a 0.60 | regra 2 (afastamento mínimo em ATR) |
| `touch_tol` | 0.02 ATR | 0.01 a 0.05 | regra 2 (tolerância do toque) |
| `depth_pad` | 0.25 | 0.10 a 0.40 | regra 2 (não afundar sob a EMA 50) |
| `cancel_pad` | 0.10 | 0.05 a 0.20 | cancelamento por furo |
| `chase_max` | 1.00 | 0.60 a 1.50 | não perseguir |
| `rej_window` | 3 | 1 a 5 | janela da rejeição |
| `pin_wick` | 2.0 | 1.5 a 3.0 | martelo (pavio / corpo) |
| `pin_wick_atr` | 0.25 | 0.15 a 0.40 | martelo (pavio mínimo) |
| `pin_body` | 0.35 | 0.25 a 0.50 | martelo (corpo / range) |
| `pin_close_pos` | 0.60 | 0.55 a 0.75 | martelo (fechamento no topo) |
| `min_range` | 0.15 ATR | 0.10 a 0.30 | qualquer padrão |
| `engulf_break_high` | false | bool | engolfo mais estrito |
| `accept_break_prev_high` | false | bool | terceira variante |
| `max_entry_delay_s` | 3 | 1 a 10 | execução |
| `spread_ratio_max` | 0.10 | 0.05 a 0.20 | filtro F6 |
| `payout_min` | 0.85 | 0.80 a 0.90 | filtro F4 |
| `cooldown_bars` | 2 | 1 a 5 | após sinal |
| `max_signals_per_leg` | 3 | 1 a 5 | guarda-corpo |
| `session_start_pad_min` | 5 | 5 a 15 | filtro F2 |
| `session_end_pad_min` | 10 | 5 a 20 | filtro F2 |
| `stake_pct` | 0.02 | 0.005 a 0.02 | risco por operação |
| `max_daily_loss_pct` | 0.05 | 0.02 a 0.06 | parada diária |
| `max_trades_day` | 10 | 5 a 20 | parada diária |

```json
{
  "version": 1,
  "symbol": "WIN$N",
  "timeframe": "1m",
  "market": "real",
  "allow_otc": false,
  "sides": { "call": true, "put": true },
  "indicators": { "ema_fast": 20, "ema_slow": 50, "atr_period": 14, "warmup_candles": 250 },
  "trend": {
    "sep_min": 0.10, "slope_k": 5, "slope_min": 0.02, "slope_slow_min": 0.01,
    "no_cross_bars": 10, "max_crosses_30": 1, "persist": 3
  },
  "pullback": {
    "ext_window": 10, "ext_min": 0.30, "touch_tol_atr": 0.02,
    "depth_pad": 0.25, "cancel_pad": 0.10, "chase_max": 1.00
  },
  "rejection": {
    "window": 3,
    "pin": { "wick_body": 2.0, "wick_atr": 0.25, "body_range": 0.35, "close_pos": 0.60, "require_green": false },
    "engulf": { "enabled": true, "break_prev_high": false },
    "break_prev_high": { "enabled": false, "close_pos": 0.60 },
    "min_range_atr": 0.15
  },
  "execution": {
    "entry": "next_candle_open", "expiry_candles": 1,
    "max_entry_delay_s": 3, "one_position_at_a_time": true
  },
  "filters": {
    "session_start_pad_min": 5, "session_end_pad_min": 10,
    "payout_min": 0.85, "spread_atr_max": 0.10,
    "news_blackout_min_before": 2, "news_blackout_min_after": 5,
    "atr_spike_x": 3.0, "volume_floor_of_median": 0.20
  },
  "risk": {
    "stake_pct": 0.02, "max_daily_loss_pct": 0.05, "max_trades_day": 10,
    "max_consecutive_losses": 3, "cooldown_bars": 2, "max_signals_per_leg": 3,
    "martingale": false
  },
  "paper": true
}
```

---

## 13. Formato do sinal (JSON)

Todo sinal emitido é registrado antes de qualquer ordem, com o estado completo
que o produziu. Sem esse registro não existe auditoria nem backtest honesto.

```json
{
  "id": "2026-11-20T14:31:00Z-WIN-call-01",
  "created_at": "2026-11-20T14:31:00Z",
  "symbol": "WIN$N",
  "timeframe": "1m",
  "direction": "call",
  "status": "emitted",
  "signal": {
    "candle_index": 10432,
    "candle_close_at": "2026-11-20T14:31:00Z",
    "pattern": "pin",
    "pattern_detail": { "lower_wick": 42.0, "body": 8.0, "range": 55.0, "close_pos": 0.78 }
  },
  "context": {
    "trend": "up",
    "ema20": 128540.5, "ema50": 128498.2, "atr": 63.4,
    "sep_atr": 0.67, "slope_atr": 0.041,
    "extended_at": "2026-11-20T14:24:00Z",
    "ext_max_atr": 0.52,
    "touch_at": "2026-11-20T14:30:00Z",
    "crosses_30": 0
  },
  "order": {
    "entry_at": "2026-11-20T14:31:00Z",
    "expiry_at": "2026-11-20T14:32:00Z",
    "strike": 128545.0,
    "payout": 0.87,
    "stake": 20.0,
    "delay_s": 0.84,
    "broker_ref": null
  },
  "result": {
    "settle_price": 128552.0,
    "outcome": "win",
    "pnl": 17.4
  }
}
```

Campos obrigatórios: `id`, `created_at`, `symbol`, `direction`, `signal`,
`context`, `order`. `result` é preenchido na liquidação. `outcome` aceita
`win`, `loss`, `tie`, `void` (ordem não preenchida ou cancelada).

---

## 14. Casos-limite e tratamento

### 14.1 Candle em formação

Nunca avaliado. Buffer separado, sem tocar em `E20`, `E50` nem em `A`. Se a
plataforma só entregar candle fechado, melhor ainda.

### 14.2 Buraco na série

Se `t_open[i] - t_open[i-1] != 60`:

- marcar `gap = true` no candle `i`;
- não calcular indicadores atravessando o buraco: reiniciar EMA e ATR a partir
  de `i` e voltar ao estado de aquecimento (250 candles);
- nunca tratar o salto de preço como movimento válido. Um buraco de 3 horas
  produz um "candle" gigante que engole todas as médias e gera sinais falsos.

### 14.3 Candle com volume zerado ou fora do comum

`volume == 0` em ativo que normalmente tem volume: candle suspeito, marcar e
não usar como vela de rejeição. `volume > 10 x mediana(20)` pode ser notícia:
aplicar F7 e F5.

### 14.4 Feed atrasado

`now - t_close(ultimo_candle) > 90s` durante a sessão: `OFF`, parar de emitir
sinal e alarmar. Um motor que continua operando com preço de 5 minutos atrás
está operando às cegas, e o strike vai sair errado.

### 14.5 Relógio

Toda decisão usa relógio sincronizado (NTP) e o fuso da fonte de dados, não o
fuso da máquina. Desvio maior que 2 segundos entre o relógio local e o do
servidor da corretora invalida a entrada.

### 14.6 Reconexão

Ao reconectar: recarregar os últimos 300 candles, recalcular tudo do zero e
descartar qualquer ordem pendente. Nunca retomar de um estado parcial salvo em
memória.

### 14.7 Ordem rejeitada, não preenchida ou preenchida com atraso

`slippage` de preço maior que `0.25 * A` em relação à abertura do candle:
cancelar se ainda der tempo; se já preencheu, registrar como `void` e **não**
considerar no resultado da estratégia. Contar ordens ruins como perdas
esconde o problema de execução; contá-las como ganhos é pior ainda.

### 14.8 Duplicidade

Chave de idempotência `symbol + t_entry + direction`. Reprocessar o mesmo
candle (retry, reconexão, reinício) não pode abrir uma segunda ordem.

### 14.9 Fim de sessão com posição aberta

A expiração de 1 candle sempre fecha dentro da sessão se a entrada respeitou
F2. Ainda assim: nenhuma entrada dentro dos últimos `expiry + 1` minutos.

### 14.10 Mudança de dia, de horário de verão e de feriado

O calendário da sessão é dado de entrada, não inferido. Horário de verão
americano, feriado brasileiro, fechamento antecipado: cada um muda a janela.
Um motor que assume "9:30 às 16:00 sempre" erra em pelo menos 4 dias por ano e
nesses dias costuma operar no pior momento possível.

---

## 15. O que muda no M1 em relação ao M5

O vídeo ensina a estratégia no M5 e recomenda M5 para iniciantes. Rodar a
mesma receita no M1 não é "a mesma estratégia em velocidade maior": muda a
escala de tempo das médias.

| | M5 (como ensinado) | M1 com 20/50 | M1 com 100/250 |
| --- | --- | --- | --- |
| memória da EMA 20 | 100 minutos | 20 minutos | 100 minutos |
| memória da EMA 50 | 250 minutos | 50 minutos | 250 minutos |
| expiração | 5 minutos | 1 minuto | 1 minuto |
| horizontes equivalentes | sim | **não** | sim |

Consequências práticas de usar 20/50 no M1:

1. **As médias abraçam o preço.** 20 e 50 minutos é pouco: em boa parte do dia
   a faixa entre as duas é menor que o ATR de 1 candle, e "tocou a média" e
   "furou a média" viram a mesma coisa. Por isso `sep_min` em ATR deixa de ser
   refinamento e passa a ser obrigatório.
2. **Muito mais sinais, muito mais ruído.** Estime de 3 a 8 sinais por dia por
   ativo, contra algo como 1 a 3 no M5. O custo por operação cai, o número de
   decisões sobe.
3. **O spread pesa mais.** Pela raiz do tempo, um ATR de 1 minuto fica perto
   de 0,45 do ATR de 5 minutos, enquanto o spread não encolhe na mesma
   proporção. A razão spread/ATR piora e pode comer a vantagem inteira
   (seção 8.5).
4. **O alvo é o tick.** Com 60 segundos de expiração, o resultado depende de
   poucos ticks. É aí que a diferença entre o preço da corretora e o preço do
   mercado real deixa de ser detalhe e vira a variável dominante.

**Recomendação técnica:** implementar os parâmetros `ema_fast` e `ema_slow`
como configuráveis e rodar o backtest com **as duas configurações**, 20/50
(fiel ao autor) e 100/250 (equivalente ao horizonte do M5). Comparar as duas
com o mesmo critério da seção 17. A escolha é resultado de medição, não de
preferência.

---

## 16. Os modos de falha

O vídeo promete "quatro momentos onde essa estratégia vai falhar", mas a
transcrição termina antes de listá-los. Os quatro abaixo são os modos de falha
que a própria fala descreve ao longo do texto (lateral, extensão, notícia,
mercado manipulado), mais um técnico que aparece na prática. A automação
precisa tratá-los como condições de bloqueio, não como aviso.

| # | modo de falha | como aparece | tratamento |
| --- | --- | --- | --- |
| 1 | **Lateral** | médias entrelaçadas, cruzamentos frequentes, separação perto de zero | regra 1 (T1, T3, T4) + `trend = NONE` |
| 2 | **Tendência exausta / clímax** | preço muito longe da média, candles cada vez maiores, ATR 3x a mediana | `chase_max`, F7, `ext_min` como piso de afastamento |
| 3 | **Notícia** | candle anormal, buraco de preço, spread abre | F5, F6, 14.3 |
| 4 | **Mercado manipulado / OTC** | feed sintético, preço que não corresponde ao real | F1, sempre, sem exceção |
| 5 | **Volta que virou reversão** | preço fura a EMA 50 e continua | `cancel_pad`, cancelamento imediato, sem "esperar voltar" |

O modo 5 merece destaque porque é o que transforma uma sequência de acertos em
uma sequência de losses rápidos: é exatamente o cenário em que o operador
iniciante aumenta a mão para "recuperar". A automação não deve ter essa opção
(seção 18).

---

## 17. Backtest e validação estatística

Esta é a parte que decide se o resto vale alguma coisa. A regra é simples: a
estratégia só pode ser operada com dinheiro real depois de mostrar vantagem
estatística contra o ponto de equilíbrio do payout, **fora da amostra**.

### 17.1 Ponto de equilíbrio

Com payout `p` (ganho por unidade apostada), a taxa de acerto mínima para não
perder é:

```
win_rate_min = 1 / (1 + p)
```

| payout | acerto mínimo |
| --- | --- |
| 0,70 | 58,82% |
| 0,75 | 57,14% |
| 0,80 | 55,56% |
| 0,85 | **54,05%** |
| 0,90 | 52,63% |
| 0,95 | 51,28% |

Note que 50% de acerto não é empate: com payout de 85%, 50% de acerto perde
dinheiro em todas as operações. E a cada ponto de payout que a corretora corta,
o mínimo sobe.

### 17.2 Quantas operações são necessárias

Uma taxa de acerto medida em poucas operações é indistinguível de sorte. Com
intervalo de confiança de 95% (Wilson), para afirmar que a taxa real supera o
ponto de equilíbrio de 54,05%:

| taxa observada | operações necessárias |
| --- | --- |
| 56% | ~2.500 |
| 58% | ~590 |
| 60% | ~260 |
| 62% | ~140 |

Leitura prática: um backtest de 100 operações com 56% de acerto tem limite
inferior perto de 46%, ou seja, **não distingue a estratégia de uma moeda**.
Dizer "acertei 56 de 100" é dizer nada. Para provar uma vantagem pequena
(56%) são necessárias milhares de operações; para provar uma vantagem grande
(60% ou mais) bastam algumas centenas. Isso muda a expectativa de tempo: de 3
a 8 sinais por dia por ativo, cobrir 2.500 operações exige muitos meses e
vários ativos.

### 17.3 Métricas a reportar

Toda rodada de backtest entrega, no mínimo:

- número de sinais, número de operações executadas, e a diferença (ordens
  abortadas);
- taxa de acerto com intervalo de confiança de 95%;
- **resultado médio por operação em unidades de stake** (é isso que precisa
  ser positivo, não o total);
- empates contados separado;
- drawdown máximo em stakes e em sequência de losses;
- resultado por hora do dia, por padrão (`pin`, `engulf`, `confirm`), por lado
  (call/put) e por faixa de `sep_atr`;
- comparação com a taxa de equilíbrio do payout usado.

Sem o intervalo de confiança, a taxa de acerto é propaganda.

### 17.4 Armadilhas que invalidam o resultado

1. **Look-ahead.** Usar `H` ou `C` do candle do sinal para decidir antes do
   fechamento. Todo o cálculo de backtest usa apenas candles `<= i`.
2. **Sobrevivência de parâmetro.** Testar 40 combinações e publicar a melhor.
   A melhor de 40 em dados aleatórios parece boa.
3. **Amostra curta.** Menos de 200 operações não serve nem para triagem.
4. **Regime único.** Seis meses de tendência de alta não dizem nada sobre o
   comportamento em lateral. Separar os resultados por regime.
5. **Ignorar custo.** Sem spread, sem payout variável, sem atraso de
   preenchimento. No M1 isso não é detalhe, é o jogo.
6. **Feed diferente.** Backtest com um provedor, execução com outro.

### 17.5 Protocolo

1. Baixar no mínimo 12 meses de candles de 1 minuto do **mesmo feed** da
   execução.
2. Dividir em treino (6 meses), validação (3 meses) e teste (3 meses), em
   ordem cronológica, sem embaralhar.
3. Escolher parâmetros no treino, olhando uma única métrica (resultado médio
   por operação).
4. Conferir na validação. Se cair muito, voltar ao passo 3 ou abandonar.
5. Rodar uma única vez no teste. O resultado do teste é o resultado da
   estratégia. Rodar de novo depois de "ajustar um detalhe" transforma o teste
   em treino.
6. Reexecutar o passo a passo em **walk-forward**: treinar em janelas móveis e
   testar sempre no período imediatamente seguinte.
7. Só então ir para demo, por no mínimo 200 operações, comparando a taxa de
   acerto da demo com a do backtest no mesmo período.

Se o resultado médio por operação no teste não for positivo com o limite
inferior do intervalo acima do ponto de equilíbrio, a resposta é **não
operar**. Não existe ajuste de gestão de risco que conserte uma estratégia sem
vantagem.

---

## 18. Gestão de banca

Regras que a automação deve impor no código, porque no calor da hora ninguém
impõe em si mesmo:

1. **Stake fixo** em porcentagem da banca atual (`stake_pct`, padrão 2%). Não
   existe "aumentar porque está acertando".
2. **Martingale proibido.** Não é questão de estilo: dobrar a aposta depois de
   um loss não muda o valor esperado por operação (que continua negativo se a
   taxa for menor que o ponto de equilíbrio) e aumenta a probabilidade de
   ruína. O efeito visível é trocar muitas perdas pequenas por uma perda
   total, no exato momento em que a sequência de losses aparece.
3. **Parada diária por perda:** `max_daily_loss_pct` (padrão 5%). Atingiu,
   encerra o dia, sem reabrir "só para recuperar".
4. **Parada por sequência:** `max_consecutive_losses` (padrão 3) encerra o dia.
5. **Limite de operações por dia:** `max_trades_day` (padrão 10). Em M1, sem
   limite, o motor encontra 30 sinais em um dia lateral.
6. **Sem hedge:** nunca call e put abertos ao mesmo tempo, nem "travar"
   resultado.
7. **Uma posição por vez**, sem exceção.
8. **Encerramento no horário:** nenhuma posição atravessa o fim da janela.
9. **Log de tudo.** Toda decisão, inclusive as de não operar, com o motivo.
   Sem log não há como descobrir por que a estratégia parou de funcionar.
10. **Kill switch manual** acessível, e um automático que desliga após
    `3` erros consecutivos de API ou depois de `10` minutos sem feed.

---

## 19. Ambiguidades da fala e a decisão tomada

A fala é didática e, por isso, deixa margem. Cada linha abaixo virou uma
decisão explícita no documento; se o autor ou o operador discordar, o
parâmetro existe para mudar.

| o que a fala diz | o que fica indefinido | decisão adotada |
| --- | --- | --- |
| "as duas apontando para cima" | o que é "para cima" | inclinação de 5 candles em ATR/candle, limites separados para EMA 20 e 50 (5.1) |
| "laranja acima da branca" | quanto acima | `sep_min` 0,10 ATR (5.1) |
| "no lateral as linhas viram uma bagunça" | o que é bagunça | `max_crosses_30 <= 1` e `no_cross_bars = 10` (5.1) |
| "o preço encosta na EMA 20" | quanto é encostar | toque com tolerância de 2% do ATR, sem furar a EMA 50 além de 0,25 ATR (6.2) |
| "ou fica no espaço entre as duas médias" | basta entrar na faixa? | sim: a faixa é `[E50, E20]` e o toque na EMA 20 já a cobre (6.2) |
| "corpo pequeno e pavio grande" | quão pequeno, quão grande | pavio >= 2x corpo, >= 0,25 ATR, corpo <= 35% do range (7.1) |
| "vela verde forte que engole a vermelha" | engole corpo ou range | corpo engole corpo, anterior vermelha, corpo atual >= anterior (7.2) |
| "martelo" | pode ser vermelho? | sim, `pin_require_green = false` (7.1) |
| "expiração de uma vela" | 60s do fill ou até o fechamento | até o fechamento do candle seguinte (8.3) |
| "entra na próxima abertura" | quanto atraso tolera | 3 segundos, depois descarta (8.2) |
| "três regras e um filtro" | qual é o filtro | a transcrição não entrega; adotados F1 a F8 (seção 9) |
| "quatro momentos onde vai falhar" | quais são | a transcrição não entrega; reconstruídos em 16 |
| nada sobre stake | quanto apostar | 2% da banca, nunca martingale (seção 18) |
| nada sobre empate | strike igual | registrar como empate, nunca como acerto (8.4) |
| estratégia ensinada no M5 | mesmo período no M1? | configurável: 20/50 ou 100/250 (seção 15) |

---

## 20. Checklist de aceitação

Antes de considerar a implementação pronta, cada item deve passar:

**Dados**

- [ ] Candle em formação nunca entra em cálculo nem em sinal (teste com candle parcial).
- [ ] Candle fora da grade de 60s é descartado.
- [ ] Buraco na série reinicia os indicadores e bloqueia sinal.
- [ ] Fuso correto em dia de horário de verão americano e no horário de verão brasileiro.
- [ ] Feriado e fechamento antecipado bloqueiam a janela.

**Regras**

- [ ] EMA 20 e 50 calculadas conferem com uma implementação de referência em 10 pontos aleatórios.
- [ ] `trend = NONE` em série lateral sintética (médias entrelaçadas), sem sinal.
- [ ] Sinal correto em série sintética de alta com um toque e um martelo.
- [ ] Nenhum sinal quando o preço nunca se afastou da média (`ext_min`).
- [ ] Nenhum sinal quando o preço furou a EMA 50 no candle do toque.
- [ ] Espelho de baixa gera exatamente os sinais do espelho de alta em série invertida.

**Execução**

- [ ] Sinal gera no máximo uma ordem (idempotência).
- [ ] Atraso > 3s descarta a ordem.
- [ ] Strike fora de `0.25 * A` da abertura aborta.
- [ ] Nenhuma segunda ordem com posição aberta.
- [ ] Expiração exatamente 60s após a entrada.

**Risco**

- [ ] Parada diária por perda funciona e não é contornada por reinício do processo.
- [ ] Parada por sequência funciona.
- [ ] Limite de operações por dia funciona.
- [ ] Martingale não existe no código (busca por multiplicador de stake no repositório).

**Auditoria**

- [ ] Todo sinal gera registro completo no formato da seção 13.
- [ ] Toda recusa (`OFF`) registra motivo.
- [ ] Resultado é conciliado com o extrato da corretora, operação por operação.

**Validação**

- [ ] Backtest com >= 12 meses e >= 200 operações fora da amostra.
- [ ] Intervalo de confiança da taxa de acerto reportado e comparado ao ponto de equilíbrio do payout.
- [ ] Walk-forward executado.
- [ ] Demo por >= 200 operações com taxa aderente ao backtest.

---

## 21. Riscos, limites e conformidade

**Sobre o produto.** Opções binárias são um contrato de tudo ou nada em que a
corretora é a contraparte: ela ganha quando o cliente perde e define o payout.
Não há bolsa nem câmara de compensação. Reguladores na Europa e no Reino Unido
barraram a oferta ao varejo, e no Brasil a CVM não autoriza essas plataformas
e já suspendeu dezenas delas por atuação irregular. Operar em plataforma não
autorizada significa, na prática, não ter a quem recorrer.

**Sobre a matemática.** Com payout de 85%, a estratégia precisa acertar mais
de 54,05% só para empatar, e provar estatisticamente uma vantagem de 56% exige
cerca de 2.500 operações. Esse é o tamanho real do problema. Qualquer material
que prometa resultado consistente com essa estratégia sem apresentar essa
conta está vendendo outra coisa.

**Sobre o vídeo.** O autor divulga a própria corretora no mesmo material em
que ensina a estratégia. Isso não invalida a regra técnica, mas é conflito de
interesse e deve ser considerado ao avaliar as promessas.

**Sobre esta especificação.** Ela descreve uma regra, não uma recomendação.
Um sinal correto não é uma previsão, e uma sequência de acertos não é
evidência de vantagem. A única coisa que autoriza operar com dinheiro real é o
resultado fora da amostra, com intervalo de confiança acima do ponto de
equilíbrio, e ainda assim o risco de ruína existe.

**Integração com este repositório.** O pacote atual automatiza outro setup (rompimento
da faixa de abertura) em `trader-entry` e planeja saída em `trader-exit`. Este
documento define um setup novo, de reversão à média, com gatilho de padrão de
candle. A implementação entraria como ferramenta separada, reaproveitando
`trader-entry/entry.py` para os utilitários já resolvidos de calendário de
sessão, fuso e leitura de candles de 1 minuto, e seguindo o mesmo princípio do
resto do pacote: nada coloca ordem sozinho.
