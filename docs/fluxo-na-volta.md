# Fluxo na Volta: síntese das duas estratégias para automação em M1

**Versão 1.0 · call e put · candles de 1 minuto · complementa `volta-na-media-m1.md`**

---

## 0. Escopo, e uma correção de contagem

Você pediu a análise de "3 vídeos" e uma estratégia baseada neles. O que chegou
foram **dois conteúdos distintos**:

1. **Volta na Média** (Pedro, com EMA 20 e EMA 50), enviado **duas vezes** na
   mesma mensagem, com o mesmo texto e apenas o pedido diferente no topo
   ("descrever para automação", "descrever para automação em M1").
2. **Mestre do Fluxo** (aula de fluxo de velas, com o filtro M5, zona por corpo,
   candle de força e engolfo).

Dois vídeos duplicados não viram três. A análise abaixo é honesta sobre o que
existe: uma interseção real entre dois métodos, e uma estratégia sintetizada a
partir dela. Se o terceiro vídeo existir e não veio, mande que eu encaixo as
regras dele no mesmo esquema de camadas.

Documento complementar: `volta-na-media-m1.md`, que já define indicadores,
execução, formato de log, risco e protocolo de validação. Aqui eu cito em vez de
repetir.

---

## 1. O que cada método é, mecanicamente

| elemento | Volta na Média | Mestre do Fluxo |
| --- | --- | --- |
| instrumento de leitura | duas médias móveis exponenciais (20 e 50) | nenhum indicador: só o candle |
| direção | `E20 > E50`, ambas subindo | sequência de candles da mesma cor, "fluxo" |
| força | EMA separada e inclinada | "candle de força": corpo grande, rompe região, mesma cor do anterior |
| gatilho | martelo ou engolfo na média | engolfo e candle de força |
| zona de entrada | a própria EMA 20 / faixa entre as médias | "zona SMR", marcada **no corpo** do candle |
| filtro temporal | nenhum | **M5 para M1**: não operar a primeira e nem a última vela de M1 do ciclo de M5 |
| tendência contra | lateral, definida pelo entrelaçamento das médias | "brigando", "retraindo demais", leitura visual |
| anomalia de preço | não trata | GAP e "candle de anomalia gigante" anulam só a operação seguinte |
| unidade de ciclo | nenhuma | "cada candle novo inicia um ciclo" |
| psicologia | não trata | não olhar histórico, loss é inevitável |
| gestão | não trata | alavancagem all-in, "arrisco o capital todo" |

---

## 2. O que as duas têm em comum

Quatro coisas, e são elas que sustentam a síntese.

### 2.1 Preço se afasta de uma referência e volta a ela

O vídeo 1 diz isso com a metáfora explícita do cachorro na coleira. O vídeo 2 diz
isso o tempo todo sem nomear: "buscou, rompeu, deu pullback", "retraindo",
"estica muito e retrai". O vocabulário muda; o fenômeno é o mesmo. **A operação
não existe no impulso, existe no retorno ao nível que o impulso rompeu.**

### 2.2 A entrada é uma rejeição na zona, não um preço qualquer

Os dois exigem que o preço vá até a referência e **volte de lá**. No vídeo 1, com
pavio e corpo do martelo; no vídeo 2, com engolfo. Nenhum dos dois compra no meio
do caminho, e nenhum compra no topo do impulso.

### 2.3 Força é corpo, não pavio

Vídeo 1: "corpo pequeno e pavio grande". Vídeo 2: "você só vai marcar no corpo,
não precisa se preocupar com pavio". Vídeo 1 usa o corpo como ausência de força
(martelo); vídeo 2 usa o corpo como presença de força (candle de força). Nos dois,
**o corpo é a informação e o pavio é o contexto**.

### 2.4 Quando não operar é parte da regra

Vídeo 1: lateral, com as médias entrelaçadas, é explicitamente "não entra".
Vídeo 2: "tá brigando", "está retraindo demais", "gráfico horrível", "não achei
entrada". Os dois passam a maior parte do tempo não operando, e os dois tratam
isso como acerto de método, não como falta de oportunidade.

---

## 3. O que só um deles tem

### 3.1 Exclusivo do vídeo 1

- **Definição objetiva de tendência.** Separação e inclinação das médias é
  mensurável; "tendência de alta" visual, não. Sem isso, a automação não tem
  como decidir lateral.
- **A regra de lateral como veto**, formulada como condição, não como sensação.
- **A expiração igual à unidade de tempo do gráfico**, que fecha a operação no
  mesmo candle da decisão.
- **Recusa explícita de OTC.**

### 3.2 Exclusivo do vídeo 2

- **O filtro M5 para M1.** É a contribuição mais original e a mais automatizável
  dos dois vídeos. Detalhado na seção 5.
- **A definição de candle de força**, que o vídeo 1 não tem: corpo grande,
  rompendo região, mesma cor do anterior.
- **O tratamento de anomalia como local.** Um candle gigante ou um GAP não
  invalidam o dia; invalidam a operação imediatamente seguinte. O ciclo recomeça
  no candle saudável seguinte. Isso é uma regra de recuperação de estado, e é
  raro em material de day trade.
- **A ideia de ciclo por candle**, que dá uma unidade natural de contagem para
  "quantas operações por ciclo" e "quando rearmar".
- **Zona marcada no corpo**, que é mais estável que zona marcada em pavio, porque
  o corpo é onde houve negociação e o pavio é onde houve rejeição.

---

## 4. O que nenhum dos dois entrega

Isso importa porque a automação precisa disso e não pode inventar sem registrar.

| lacuna | por que a automação precisa | onde está resolvido |
| --- | --- | --- |
| tolerância do toque na zona | "encostou" é visual | seção 7.3, em ATR |
| janela de validade do gatilho | "espera a rejeição" não diz por quanto tempo | seção 7.4, 3 candles |
| distância mínima de afastamento | "estica" não tem número | seção 7.2, em ATR |
| definição de "gráfico horrível" | o vídeo 2 decide por sensação | seção 8, filtros F5 a F7 |
| spread e payout | nenhum dos dois menciona custo | seção 9 |
| tamanho de posição | o vídeo 2 faz all-in, o vídeo 1 não fala | seção 11 |
| o que fazer quando a ordem atrasa | nenhum dos dois | doc 1, seção 8.2 |

---

## 5. O filtro M5, que é o achado dos dois vídeos

### 5.1 O que ele diz

O vídeo 2 afirma que não se deve operar:

- a **primeira** vela de M1 de um ciclo de M5;
- a **última** vela de M1 de um ciclo de M5.

E dá o motivo: o candle de M5 "estica muito" e "na última dela de M1 para M5 ela
retrai sempre, finaliza com um pavio". Ou seja: o fechamento do M5 é um evento de
retração, e a última vela de M1 é justamente esse pavio.

### 5.2 Por que faz sentido

O fechamento de um candle de M5 é o instante em que o preço realiza a média do
período. Quem está posicionado tem incentivo a realizar resultado ali; quem opera
por tempo ajusta posição. O resultado é concentração de fluxo contrário na virada.
A última vela de M1 do ciclo é o último minuto antes desse evento, e a primeira é
o primeiro minuto depois, quando o preço ainda está digerindo o fechamento.

Para uma estratégia de **volta à média**, isso é duplamente relevante: se o
gatilho é a rejeição na média, e a hora tem retração forçada, a rejeição que
aparece ali é a mais provável de falhar. O filtro ataca exatamente o ponto onde a
estratégia é mais frágil.

### 5.3 Formalização

Com o ciclo de M5 ancorado no relógio absoluto (`t` em segundos UTC, múltiplo de
60):

```
m = ((t // 60) % 5)                  # posicao do candle de M1 dentro do ciclo
entrada_permitida = m in {1, 2, 3}
```

- `m = 0`: primeira vela do ciclo. Bloqueada.
- `m = 1, 2, 3`: miolo do ciclo. Permitidas.
- `m = 4`: última vela. Bloqueada.

Consequência de execução: com entrada em `m = 3` e expiração de 60 segundos, a
liquidação acontece no fim de `m = 3`, **um minuto antes** do fechamento do M5.
Toda a operação, da entrada à liquidação, fica dentro do miolo. Não existe
operação que encoste na virada.

O filtro reduz a janela de negociação para **60% do tempo** (3 de cada 5
minutos). Isso precisa entrar na conta de expectativa de número de sinais: se a
versão sem filtro gera 8 sinais por dia, a com filtro gera no máximo 5, e na
prática menos, porque o filtro remove justamente o minuto em que o preço costuma
estar mais próximo de uma média.

### 5.4 Como provar que ele serve para algo

Este é o teste A/B mais importante do documento, porque é a única afirmação
original de um dos vídeos e é barata de medir:

1. Rodar o motor **sem** o filtro, registrando `m` de cada sinal.
2. Agrupar o resultado por `m`.
3. Comparar a taxa de acerto e o resultado médio de `m ∈ {1,2,3}` com `m ∈ {0,4}`.

Se a diferença não existir na sua amostra, o filtro é folclore e deve sair da
configuração. Se existir, você acabou de ganhar uma vantagem que a maioria dos
operadores de binárias não tem, e sabe exatamente de onde ela vem. Em ambos os
casos, você deixa de acreditar e passa a saber.

O mesmo desenho vale para cada elemento dos dois vídeos. Ver seção 13.

---

## 6. Uma divergência real entre os vídeos, que precisa ser resolvida

Os dois usam a palavra "engolfo", e **significam coisas opostas**.

| | Volta na Média | Mestre do Fluxo |
| --- | --- | --- |
| definição | vela verde cujo corpo engole o corpo da **vermelha anterior** | candle **da mesma cor** do anterior, que rompe a região |
| função | **reversão**: o preço caiu, foi rejeitado, volta | **continuação**: a força se confirma e segue |
| momento | depois do pullback | no impulso |

Chamar as duas de engolfo é fonte garantida de bug. Na síntese, elas ocupam
camadas diferentes e recebem nomes diferentes:

- **`rejeicao_reversao`** (vídeo 1): anterior contra a tendência, corpo atual
  engole o corpo anterior, a favor da tendência. É o **gatilho**, na camada 4.
- **`forca_continuacao`** (vídeo 2): mesma cor do anterior, corpo maior que o
  anterior e fechamento além da máxima anterior. É o **impulso**, na camada 2.

Isso não é preciosismo: uma compra no topo de uma força de continuação e uma
compra na volta à média depois de uma rejeição são entradas com risco, alvo e
probabilidade completamente diferentes. Misturar as duas faz o backtest produzir
um número médio que não descreve nenhuma das duas.

---

## 7. A estratégia sintetizada: Fluxo na Volta

Nome escolhido para descrever o que ela faz, sem prometer resultado: lê o fluxo
nas camadas maiores e entra na volta à referência na camada de execução.

### 7.1 Arquitetura em quatro camadas

```
  M5   ---------------------------------------------------------------
       CAMADA 1: CONTEXTO      qual lado tem fluxo?  (cor + EMA no M5)
  M1   ---------------------------------------------------------------
       CAMADA 2: IMPULSO       houve candle de forca? (o cao correu)
       CAMADA 3: RETORNO       o preco voltou a referencia? (a coleira)
       CAMADA 4: GATILHO       houve rejeicao? (o dono puxou)
  --------------------------------------------------------------------
       FILTROS: M5 janela | lateral | anomalia | sessao | custo
       EXECUCAO: abertura do proximo candle, expiracao 60s
```

Cada camada responde uma pergunta e pode ser ligada ou desligada na configuração,
para que o backtest meça a contribuição de cada uma separadamente. Isso é o ponto
central do desenho: **a estratégia é uma pilha de hipóteses testáveis, não um
bloco indivisível**.

### 7.2 Camada 1: contexto (M5)

Reaproveita a definição objetiva do vídeo 1, calculada no M5 em vez do M1, que
resolve o problema de escala apontado no documento 1:

```
E20_5, E50_5 = EMA(close_M5, 20), EMA(close_M5, 50)

contexto_ma  = +1  se  E20_5 > E50_5 + sep_min * A5  e ambas subindo
               -1  o espelho
                0  caso contrario

cor5         = numero de candles M5 verdes entre os ultimos 3 fechados
contexto_cor = +1  se cor5 >= 2
               -1  se cor5 <= 1
                0  caso contrario

contexto     = +1  se contexto_ma > 0 e contexto_cor > 0     (estrito)
               -1  se contexto_ma < 0 e contexto_cor < 0
                0  caso contrario                            -> NAO OPERA
```

`contexto_mode = "strict"` (padrão) exige as duas leituras de acordo, o que é a
interseção real dos dois métodos. `"loose"` aceita qualquer uma. A diferença entre
os dois modos é uma das coisas a medir na seção 13.

Só candles de M5 **fechados** entram. O M5 em formação não vale, pela mesma razão
que um M1 em formação não vale.

### 7.3 Camada 2: impulso (M1)

Duas formas de "o cachorro correu", uma por vídeo, e pelo menos uma precisa
acontecer:

**a) Candle de força** (vídeo 2)

```
forca = (body >= forca_body_atr * A)              # 0.8 ATR
    and (body >= forca_body_range * range)        # 60% do range
    and (range >= min_range * A)                  # 0.15 ATR
    and (close > max(O, C_anterior) * 1.0 se alta)  # rompeu a referencia local
    and (cor == cor do candle anterior se forca_seq)  # sequencia de fluxo
```

**b) Extensão da média** (vídeo 1)

```
extensao = (H[j] - E20[j]) >= ext_min * A[j]     # ext_min = 0.30 ATR
           para algum j nos ultimos ext_window = 10 candles
```

O impulso é o **candle mais recente** que satisfez (a) ou (b). Guardo o corpo
dele, porque é dele que sai a zona:

```
zona_compra = max(O_impulso, C_impulso)          # topo do corpo do impulso
zona_venda  = min(O_impulso, C_impulso)
```

Marcar no corpo e não no pavio é a instrução do vídeo 2, e faz sentido: o pavio é
onde o preço foi rejeitado, o corpo é onde houve negociação aceita.

**Limite de perseguição**, herdado do documento 1: se o preço já está a mais de
`chase_max = 1.0` ATR da referência, não é volta, é continuação. Veta.

### 7.4 Camada 3: retorno

A volta pode ser até **qualquer uma** de duas referências. Cada uma tem o seu
mérito, e a confluência das duas é o que a síntese oferece de melhor:

```
ref_ema  = E20
ref_zona = zona_compra                        # em alta

tocou_ema  = (L[t] <= E20[t] + tol(t))  and (L[t] >= E50[t] - depth_pad * A[t])
tocou_zona = (L[t] <= zona + tol(t))    and (L[t] >= zona - depth_pad * A[t])

tol(t) = max(touch_tol * A[t], 0.0002 * preco[t])     # touch_tol = 0.02
```

O retorno está válido se `tocou_ema or tocou_zona`, conforme
`ref_mode = ema | zona | ambas`. A **confluência** (as duas ao mesmo tempo, dentro
de `confluence_bars = 3` candles) não é obrigatória; ela classifica o sinal (seção
12).

Regra de cancelamento, esta sim obrigatória: se o fechamento furar a referência
lateral no sentido contrário, a volta virou reversão e o ciclo é descartado.

```
cancel = (C[t] < E50[t] - cancel_pad * A[t])      # em alta, cancel_pad = 0.10
```

### 7.5 Camada 4: gatilho

Três aceitações, escolhidas por configuração:

**T1: rejeição por reversão** (vídeo 1, a principal)

```
martelo   = (range >= min_range * A) and (lower_wick >= pin_wick * body)
            and (lower_wick >= pin_wick_atr * A) and (body <= pin_body * range)
            and (close_pos >= pin_close_pos)

rejeicao  = (C[t-1] < O[t-1]) and (C[t] > O[t])          # anterior vermelha
            and (C[t] >= O[t-1]) and (O[t] <= C[t-1])     # corpo engole corpo
            and (body[t] >= body[t-1])
```

**T2: força de continuação** (vídeo 2)

```
forca_gatilho = (C[t] > O[t]) and (C[t] > H[t-1]) and (body[t] >= forca_body_atr * A)
```

**T3: martelo puro** (vídeo 1, o mais permissivo)

Já definido acima.

`gatilho_mode = "T1|T2|T1+T2|T3|todos"`, padrão `T1+T2`. A janela é de
`rej_window = 3` candles contados do toque. Gatilho depois da janela: ciclo
encerrado, espera o próximo.

### 7.6 Filtros (todos bloqueantes)

| # | filtro | regra | fonte |
| --- | --- | --- | --- |
| F1 | mercado real | feed OTC ou sintético: bloqueado sempre | vídeo 1 |
| **F2** | **janela do ciclo M5** | `m in {1,2,3}`, seção 5.3 | **vídeo 2** |
| F3 | lateral | `contexto == 0` | vídeo 1 |
| F4 | anomalia | `range > anomalia_atr * A` (4) ou `abs(O[t]-C[t-1]) > gap_pad * A` (0.5): bloqueia **só o candle seguinte**, libera a partir do segundo se ele for saudável | vídeo 2 |
| F5 | notícia | `[-2, +5]` min de evento agendado | vídeo 1 |
| F6 | volatilidade extrema | `A[t] > 3 * mediana(A[t-20..t])` | vídeos 1 e 2 |
| F7 | sessão | `[open+5min, close-10min]`, fora disso OFF | padrão |
| F8 | um sinal por ciclo M5 | `max_signals_per_m5 = 1` | vídeo 2 ("cada candle novo inicia um ciclo") |
| F9 | custo | `spread/ATR <= 0.10` e `payout >= 0.85` | economia |

F4 merece destaque porque é contraintuitiva e está certa: um candle de 4 ATR não
invalida o dia, invalida a próxima decisão. É a diferença entre um sistema que se
trava por causa de um evento e um sistema que respira.

---

## 8. Máquina de estados

Delta em relação ao documento 1. A diferença é a camada de contexto, que roda em
M5 e alimenta a máquina de M1.

```
                    contexto M5 == 0
                    contexto M5 == +1
   [ OFF ] ------------------------------> [ CONTEXTO ]
      ^                                          |
      |  filtro F2..F9 falha                     | impulso (camada 2)
      |                                          v
      |                                     [ IMPULSO ]
      |                                          |
      |  cancelamento                           | retorno (camada 3)
      |  ou janela expirada                      v
      |                                     [ RETORNO ]
      |                                          |
      |                                          | gatilho (camada 4)
      |                                          v
      +------------------------------------- [ SINAL ] --> COOLDOWN
                                                   |
                                            max 1 por ciclo M5
```

Regras adicionais:

- O contexto M5 é reavaliado **apenas quando um candle de M5 fecha**. Entre
  fechamentos, ele é constante. Isso reduz ruído e é fiel ao vídeo.
- A janela F2 é reavaliada **a cada candle de M1**. Ela pode derrubar um sinal
  armado; nunca criar um.
- `max_signals_per_m5 = 1` conta por ciclo de 5 minutos, e não por hora. É a
  tradução de "cada candle novo inicia um ciclo".

---

## 9. Custo, que nenhum dos dois vídeos menciona

Antes de qualquer regra, o piso econômico. Com payout `p`, a taxa de acerto de
equilíbrio é `1/(1+p)`:

| payout | acerto de equilíbrio | resultado com 50% de acerto |
| --- | --- | --- |
| 0,80 | 55,56% | -10,0% por operação |
| **0,86** (o do vídeo 2) | **53,76%** | **-7,0% por operação** |
| 0,90 | 52,63% | -5,0% por operação |
| 1,00 | 50,00% | 0,0% |

Leitura direta: **com payout de 86%, acertar metade não é empate, é perder 7% por
operação.** O filtro M5, o candle de força e a volta na média valem zero se não
empurrarem a taxa acima de 53,76%. Esse é o número a bater, e é ele que a seção 13
manda medir.

---

## 10. Parâmetros e configuração

Só o que é novo ou diverge do documento 1. O resto, `volta-na-media-m1.md`,
seção 12.

```json
{
  "version": 1,
  "symbol": "WIN$N",
  "timeframe": "1m",
  "strategy": "fluxo-na-volta",

  "context": {
    "mode": "strict",
    "m5_ema_fast": 20, "m5_ema_slow": 50,
    "m5_sep_min_atr": 0.10,
    "m5_color_window": 3, "m5_color_min": 2
  },

  "impulse": {
    "forca_body_atr": 0.80, "forca_body_range": 0.60, "min_range_atr": 0.15,
    "require_sequence_color": true,
    "ext_window": 10, "ext_min_atr": 0.30,
    "zone_from": "body", "chase_max_atr": 1.00
  },

  "return": {
    "ref_mode": "ambas",
    "touch_tol_atr": 0.02, "depth_pad_atr": 0.25,
    "cancel_pad_atr": 0.10, "confluence_bars": 3
  },

  "trigger": {
    "mode": "T1+T2", "window_bars": 3,
    "pin": { "wick_body": 2.0, "wick_atr": 0.25, "body_range": 0.35, "close_pos": 0.60 },
    "engulf_break_prev_high": false,
    "forca_break_prev_high": true
  },

  "filters": {
    "m5_window": { "enabled": true, "allowed_minutes": [1, 2, 3] },
    "anomaly_atr": 4.0, "gap_pad_atr": 0.50, "block_next_candles": 1,
    "atr_spike_x": 3.0, "spread_atr_max": 0.10, "payout_min": 0.85,
    "news_blackout_min_before": 2, "news_blackout_min_after": 5,
    "max_signals_per_m5": 1
  },

  "execution": {
    "entry": "next_candle_open", "expiry_candles": 1,
    "max_entry_delay_s": 3, "one_position_at_a_time": true
  },

  "risk": {
    "stake_pct": 0.02, "max_daily_loss_pct": 0.05, "max_trades_day": 8,
    "max_consecutive_losses": 3, "martingale": false, "all_in": false
  },

  "paper": true
}
```

---

## 11. Risco: por que all-in e martingale ficam fora, com número

O vídeo 2 demonstra alavancagem arriscando o capital inteiro. Vale examinar o que
isso faz, porque é a única parte dos dois vídeos que pode zerar a conta em uma
operação.

**O que o vídeo mostra:** banca de 86 chegando a 493. Multiplicador de 5,73. Com
payout de 86%, isso exige `ln(5,73)/ln(1,86) = 2,81`, ou seja, **três acertos
seguidos** com o capital inteiro.

**O que isso significa:**

| acerto real | probabilidade de 3 seguidas | expectativa da sequência |
| --- | --- | --- |
| 50% | 12,5% | -19,5% |
| 53,76% (equilíbrio) | 15,5% | 0,0% |
| 60% | 21,6% | +39% |
| 65% | 27,5% | +77% |

A linha do equilíbrio é a chave: **com 53,76% de acerto, apostar 2% fixo e apostar
tudo têm exatamente a mesma expectativa.** A diferença não está no retorno
esperado, está na distribuição:

- com 2% fixo, uma sequência de 20 losses custa 33% da banca e você continua vivo;
- com all-in, **um** loss custa 100% da banca e a série termina.

A probabilidade de 20 acertos seguidos com all-in a 60% de acerto é de
0,0037% (1 em 27 mil). Não é uma estratégia agressiva; é uma estratégia com data
de validade.

**Duas consequências para a automação:**

1. `martingale` e `all_in` ficam no config como `false` e o motor **recusa subir**
   se forem `true`, em vez de apenas ignorar. Um parâmetro que existe e não é
   aplicado é um convite para alguém ligar depois.
2. `max_consecutive_losses = 3` e `max_daily_loss_pct = 5%` encerram o dia. O
   vídeo 2 reconhece o problema ("entra com o dobro e quebra a banca") e ao mesmo
   tempo demonstra a prática. A automação fica com o diagnóstico, não com a
   prática.

**Expectativa honesta de crescimento.** Com 2% de stake, payout 0,86 e 58% de
acerto, o ganho esperado por operação é 0,158% da banca. Dobrar a banca leva
`ln(2)/0,00158 = 440` operações. A 3 operações por dia, isso é cerca de **5
meses**. Esse é o número real, e é ele que deve calibrar a expectativa, não o
print de 86 para 493.

---

## 12. Classificação do sinal

Automação precisa de um campo de qualidade explícito, mesmo que a qualidade ainda
não esteja provada. O grade é uma **hipótese**, e a seção 13 manda testá-la.

| grade | condição |
| --- | --- |
| **A** | contexto estrito + confluência das duas referências + gatilho T1 |
| **B** | contexto estrito + uma referência + gatilho T1 ou T2 |
| **C** | contexto frouxo, ou referência única, ou gatilho T3 |

Uso prático: `min_grade` na configuração (padrão `B`). Grades C vão para o log e
para o backtest, não para a conta. Se o backtest mostrar que C tem a mesma
expectativa que A, o grade é ruído e sai do documento.

---

## 13. Plano de testes: o que provar, e quanto custa provar

Esta é a parte que separa a síntese de uma colagem de regras. Cada elemento
herdado ganha um teste que pode derrubá-lo.

### 13.1 Bateria A/B

Rodar o motor completo e, em cada rodada, desligar **um** componente. Medir a
diferença no resultado médio por operação.

| # | componente | hipótese | como testar |
| --- | --- | --- | --- |
| 1 | **janela M5 (F2)** | sinais em `m ∈ {0,4}` têm expectativa pior | agrupar resultados por `m`, com filtro desligado |
| 2 | contexto M5 | contexto estrito supera o frouxo | `context.mode` = strict vs loose |
| 3 | candle de força | o impulso por força supera o impulso por extensão de EMA | `impulse` = forca vs ext |
| 4 | zona de corpo | a zona no corpo supera a zona no pavio | `zone_from` = body vs wick |
| 5 | confluência | grade A supera B e C | resultados por grade |
| 6 | gatilho | T1 (reversão) e T2 (continuação) têm expectativas diferentes | resultados por `pattern` |
| 7 | filtro de anomalia | bloquear o candle seguinte ao gigante melhora a expectativa | ligado vs desligado |
| 8 | um por ciclo M5 | 1 sinal por ciclo supera vários | `max_signals_per_m5` = 1 vs 99 |
| 9 | multiplicidade | um sinal por perna supera vários | `max_signals_per_leg` = 3 vs 99 |

Nove testes, um por afirmação. O que sobreviver entra na versão 2.0 da
estratégia; o que não sobreviver sai, mesmo que esteja no vídeo.

### 13.2 Tamanho de amostra

Da tabela do documento 1: para provar que a taxa supera os 53,76% de equilíbrio
com 95% de confiança, são necessárias cerca de **2.500 operações se a vantagem for
de 2 pontos** (56%), e cerca de **260 se for de 6 pontos** (60%).

Isso tem uma consequência prática para a bateria acima: **não divida a amostra
nove vezes.** Com 2.500 operações, cada teste individual fica com 1.250 de cada
lado, o que detecta apenas vantagens grandes. A ordem de trabalho correta é:

1. Testes 1, 3, 6 e 8 com a amostra inteira (são os de maior efeito esperado).
2. Só depois os de efeito menor (2, 4, 5, 7, 9), e de preferência com mais dados
   ou com um critério menos rigoroso, declarado como exploratório.

Declarar o que é confirmatório e o que é exploratório evita o clássico
"encontrei um padrão em 40 comparações".

### 13.3 Ordem de trabalho recomendada

1. Implementar os utilitários do documento 1 (indicadores, sessão, execução).
2. Implementar camadas 2, 3 e 4 **sem filtros**, e o log com todos os campos.
3. Rodar o backtest e agrupar por `m`, existindo ou não o filtro.
4. Ligar F2 apenas se o teste 1 mostrar efeito.
5. Repetir para os demais componentes, um por vez.

A tentação é implementar tudo e testar no fim. O problema: com tudo ligado, você
não sabe qual peça carrega o resultado, e qualquer ajuste vira superstição.

---

## 14. Onde os vídeos erram, e por que isso importa para o código

Três afirmações dos vídeos não se sustentam. Isso não invalida o método técnico,
mas se alguma delas entrar na configuração, a conta vai embora.

### 14.1 "Até com 50% de assertividade você consegue lucrar a longo prazo"

Falso com qualquer payout abaixo de 1,00, e binárias pagam abaixo de 1,00 por
construção. Com 86% de payout, 50% de acerto é -7% por operação. A afirmação está
na seção 11:53 do vídeo 2. Se ela virar premissa da automação, o motor vai operar
uma regra com expectativa negativa e o resultado será perda lenta e constante,
que é pior que a perda rápida porque demora a ser percebida.

**No código:** `min_grade` e `payout_min` existem para isso. E o backtest reporta
a taxa de acerto sempre comparada com `1/(1+payout)`, nunca com 50%.

### 14.2 "O loss é inevitável, não olhe o histórico"

A primeira metade está certa. A segunda é boa para o psicológico de um humano e
**terrível para uma automação**. Um sistema que não mede não corrige. O motor
registra tudo, inclusive as recusas, e o operador consulta o log na hora de
ajustar, não durante a operação.

**No código:** todo sinal e toda recusa em JSON, com o motivo nomeado. A regra
"não olhar o histórico" vira "não olhar o histórico *durante* o dia; analisar o
histórico *depois* do fechamento".

### 14.3 Prova social por print de banca

1800, 2426, 3268, 493. Prints de banca não são auditáveis: não mostram número de
operações, taxa de acerto, saques, depósitos nem loss no período. E o mesmo vídeo
que mostra a banca crescendo mostra a operação de capital inteiro, o que explica
o crescimento sem precisar de vantagem nenhuma.

**No código:** o que autoriza operar é a seção 13, não o print de ninguém. E o
que se compara é o resultado por operação, não o saldo final.

---

## 15. O que não é mecanizável, e o que fazer com isso

Duas coisas dos vídeos resistem a virar regra, e é melhor admitir:

**"Malícia do gráfico" e "tempo de tela".** O vídeo 2 diz explicitamente que não
se trata de cálculo e sim de reconhecer como o gráfico se comporta. Isso é
julgamento treinado, e julgamento treinado não vira `if`. A automação não
substitui esse julgamento; ela o congela na forma de uma regra fixa, que é
justamente o oposto. Um operador com malícia vai superar o motor nos casos que o
motor não previu, e vai errar mais nos casos que o motor cobre.

**"Simetria de pavios" para marcar zona.** Visual e sem critério numérico. Fica
de fora. Se a seção 13 mostrar que a zona de corpo já resolve, não se perde nada.

**Marcação manual de zonas.** O vídeo 2 marca as zonas olhando o gráfico. A
automação tem que gerar a zona por regra (impulso mais recente, seção 7.3). As
duas coisas não vão coincidir sempre, e a diferença é uma fonte de divergência
entre o que o autor faria e o que o motor faz. Não é bug; é o custo de automatizar
algo que o autor faz a olho.

---

## 16. Checklist de implementação

**Reaproveitado do documento 1** (não reimplementar): EMA, ATR, sessão,
calendário, horário de verão, execução, liquidação, log, risco, validação
estatística. Todos os itens do checklist de lá continuam valendo.

**Específico desta síntese:**

- [ ] Contexto M5 recalculado apenas no fechamento de candle de M5.
- [ ] `m` calculado no relógio absoluto, não no fuso local.
- [ ] Nenhum sinal em `m = 0` nem em `m = 4` com F2 ligado.
- [ ] Liquidação sempre dentro do ciclo de M5 (entrada em `m=3` liquida em `m=3`).
- [ ] Impulso guarda o corpo do candle, e a zona sai do corpo, nunca do pavio.
- [ ] Rejeição usa `rejeicao_reversao`; continuação usa `forca_continuacao`. Os
      dois nomes nunca trocados no código.
- [ ] Cancelamento por furo da referência lateral acontece no mesmo candle, sem
      espera.
- [ ] Anomalia bloqueia exatamente 1 candle seguinte, e libera no segundo se ele
      for saudável.
- [ ] `max_signals_per_m5 = 1` conta por ciclo de 5 minutos.
- [ ] `martingale` e `all_in` recusam subir quando `true`.
- [ ] Grade A, B e C calculada e registrada em todo sinal.
- [ ] Backtest agrupa resultado por `m`, por grade, por `pattern` e por referência.
- [ ] Relatório cita a taxa de equilíbrio do payout em toda tabela de acerto.

**Testes com série sintética:**

- [ ] Alta no M5 com um sinal em `m=2` e nenhum em `m=0` e `m=4`.
- [ ] Candle gigante seguido de candle saudável: o gigante bloqueia o seguinte e
      o saudável volta a permitir.
- [ ] Série invertida em preço produz exatamente o espelho dos sinais de venda.
- [ ] Contexto frouxo gera superconjunto estrito dos sinais do contexto estrito.

---

## 17. Limites, de novo, porque é a segunda vez que o assunto aparece

Os dois vídeos são materiais de venda: o primeiro divulga a corretora do autor, o
segundo vende um treinamento e mostra prints de alunos. A regra técnica pode ser
boa e a comunicação continuar sendo venda. Isso não desqualifica o método, mas
muda o peso que se dá às afirmações que não vêm com número.

Mais importante: opções binárias têm expectativa negativa por construção quando a
taxa de acerto fica abaixo do equilíbrio do payout, a corretora é a contraparte e
não há câmara de compensação. No Brasil, a CVM não autoriza essas plataformas e
já suspendeu dezenas delas. Nada neste documento é recomendação de investimento.
O que ele entrega é uma forma de descobrir, com dados seus, se essas regras têm
vantagem maior que o custo de operá-las.

Se a resposta for não, o resultado correto não é ajustar o motor: é não operar.
