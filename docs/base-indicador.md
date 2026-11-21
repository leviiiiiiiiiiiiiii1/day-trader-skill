# Base de indicador: Fluxo na Volta

**Registro mestre de regras · versão 1.1 · espelha `pine/fluxo-na-volta.pine`**

Este é o documento-fonte do indicador. Toda regra do código tem um registro aqui
com identificador, condição em forma executável, parâmetro, origem e efeito de
desligar. Toda regra daqui tem implementação no código. Se um dos dois lados
divergir, o documento está errado e precisa ser corrigido antes de qualquer
backtest.

Documentos de apoio: `docs/volta-na-media-m1.md` (método 1 em profundidade),
`docs/fluxo-na-volta.md` (síntese e plano de testes), `pine/README.md` (uso).

---

## 0. Como usar esta base

Três leitores possíveis, três entradas:

| se você quer | vá para |
| --- | --- |
| saber quando entrar, em uma olhada | seção 2 (pipeline) e seção 5 (matriz de bloqueio) |
| implementar ou auditar o código | seção 4 (registros completos) |
| escolher um perfil e sair operando | seção 6 (perfis) e seção 7 (checklist) |

A regra de ouro da base: **nenhuma regra existe sem três coisas**, condição
mensurável, parâmetro com unidade, e efeito declarado de desligar. Regra sem
essas três é opinião, e opinião não vira input.

---

## 1. Inventário: o que passou pelo filtro

A fala dos dois vídeos tem 100% de conteúdo e 60% de regra. O filtro aplicado
foi: *pode ser escrito como comparação entre grandezas medidas no candle?* O que
passa vira regra; o que não passa vira exclusão documentada.

### 1.1 O que entrou (31 regras, 9 ferramentas)

| grupo | registros | quantos |
| --- | --- | --- |
| Contexto (M5) | C01 a C05 | 5 |
| Tendência | T01 a T07 | 7 |
| Impulso | I01 a I06 | 6 |
| Retorno | R01 a R06 | 6 |
| Gatilho | G01 a G07 | 7 |
| Execução | E01 a E05 | 5 |
| Bloqueios | B01 a B13 | 13 |
| Risco | K01 a K08 | 8 |

### 1.2 O que ficou de fora, e por quê

Estas não são "regras que faltaram". São coisas que apareceram nos vídeos e
foram **deliberadamente rejeitadas**. Cada uma tem motivo e consequência.

| # | o que foi dito | por que ficou fora | o que acontece se entrar |
| --- | --- | --- | --- |
| X01 | "com 50% de acerto você lucra a longo prazo" (vídeo 2, 11:53) | é falso: com payout 0,86, acertar 50% dá -7% por operação | a premissa vira regra e a conta assume expectativa positiva que não existe |
| X02 | "não olhe o histórico" (vídeo 2) | útil para o humano durante a sessão, inútil para sistema | sem log não há correção; o motor repete o erro indefinidamente |
| X03 | "simetria de pavios para marcar zona" (vídeo 2) | não tem critério numérico, é julgamento visual | não é reproduzível; dois operadores marcam zonas diferentes |
| X04 | "malícia do gráfico", "tempo de tela" (vídeo 2) | conhecimento tácito, resiste a virar `if` | automatizar o que não está escrito gera regra inventada atribuída ao autor |
| X05 | marcação manual de zona (vídeo 2) | depende do olho | a zona do motor nunca coincide com a do autor; melhor assumir e documentar |
| X06 | all-in / capital inteiro (vídeo 2) | expectativa igual ao stake fixo, risco de ruína total | um loss zera a banca; ver cálculo na seção 9 |
| X07 | "não fique olhando o histórico para não se descontrolar" | conselho comportamental, não regra de mercado | irrelevante para automação |
| X08 | prints de banca (1800, 2426, 3268, 493) | não auditável: sem número de operações nem taxa de acerto | vira prova social no lugar de evidência estatística |
| X09 | "payout é contra a gente, o mercado foi feito para tirar dinheiro" (vídeo 2) | diagnóstico correto, mas não é regra operacional | entra como contexto na seção 10, não como parâmetro |

O item X01 é o mais importante da lista. Um único parâmetro fundado nele
(`payout_min`) é o que separa este indicador de um gerador de sinais aleatório.

### 1.3 Ambiguidades dos vídeos resolvidas em regra

| o que foi dito | vago em | registro que resolve |
| --- | --- | --- |
| "laranja acima da branca" | quanto acima | T01 (0,10 ATR) |
| "as duas apontando para cima" | o que é subir | T02, T03 (em ATR/candle) |
| "no lateral as linhas viram bagunça" | o que é bagunça | T04, T05 |
| "o preço encosta na média" | quanto é encostar | R01 (0,02 ATR) |
| "fica no espaço entre as médias" | basta entrar na faixa | R01 + R02 |
| "corpo pequeno e pavio grande" | quão pequeno, quão grande | G04 |
| "vela verde forte que engole a vermelha" | engole corpo ou range | G02, G03 |
| "candle de força" | que tamanho é força | I01 |
| "rompe a região" | que região | I02 + F09 (zona) |
| "não opera a primeira nem a última vela de M5" | como medir | B02 (relógio, não gráfico) |
| "cada candle inicia um ciclo" | ciclo de que | B03 (5 minutos) |
| "anomalia anula a operação" | anula só uma ou o dia | B04 (só a próxima) |
| "gráfico horrível" | o que é horrível | B05 + B11 |

---

## 2. Pipeline de decisão

A ordem importa: um filtro barato aplicado cedo economiza cálculo e, mais
importante, faz o motivo do bloqueio ser o mais informativo possível.

```
todo candle de 1 minuto fechado
  |
  +- FERRAMENTAS .......... EMA20, EMA50, ATR14, m5pos, corpo, pavios
  |
  +- C05  contexto M5 atualizado? -------- so no fechamento de M5
  +- B11  aquecimento 250? --------------- nao -> OFF
  +- B12  OTC / feed errado? ------------- sim -> OFF
  +- C03  contexto M5 definido? ---------- 0 (lateral) -> OFF
  |
  +- T01..T07  tendencia (3 candles)? ---- nao -> OFF
  |
  +- I06  ja esticou demais? ------------- sim -> OFF (nao perseguir)
  +- I01..I05 impulso existe? ------------ nao -> aguarda
  |
  +- R05  furou a referencia lateral? ---- sim -> descarta ciclo
  +- R01..R06 preco voltou? -------------- nao -> aguarda
  |
  +- G01  dentro da janela de 3? --------- nao -> descarta ciclo
  +- G02..G07 rejeicao? ------------------ nao -> aguarda
  |
  +- B01..B13 bloqueios de contexto ------ algum falhou -> OFF (motivo)
  +- K01..K08 risco permite? ------------- nao -> OFF (parada)
  |
  +- E01 sinal -> entrada na abertura do proximo candle
  |
  +- E02 expiracao 60s -> liquidacao -> log
```

Duas propriedades que o pipeline garante:

1. **Nenhum caminho chega a sinal com candle aberto.** A avaliação acontece
   uma vez por candle, depois do fechamento.
2. **Todo `OFF` tem motivo nomeado.** O painel e o log mostram qual registro
   barrou. "Sem entrada" não existe.

---

## 3. Ferramentas (grandezas medidas)

| id | nome | fórmula | unidade |
| --- | --- | --- | --- |
| F01 | EMA rápida | `ema(close, 20)` | preço |
| F02 | EMA lenta | `ema(close, 50)` | preço |
| F03 | ATR | Wilder 14 no M1 | preço |
| F04 | `m5pos` | `floor(t_epoch/60) mod 5` | 0 a 4 |
| F05 | corpo, range, pavios | `body=|C-O|`, `range=H-L`, `lower=min(O,C)-L`, `upper=H-max(O,C)` | preço |
| F06 | cruzamentos | troca de sinal de `E20-E50` em 30 candles | contagem |
| F07 | inclinação | `(E20 - E20[5]) / 5` e idem E50 | preço/candle |
| F08 | mediana do ATR | `median(ATR, 20)` | preço |
| F09 | zona | `max(O,C)` em alta, `min(O,C)` em baixa, corpo do candle de força | preço |

**Normalização.** Toda comparação de preço é feita em múltiplos de `F03` (ATR),
nunca em reais ou porcentagem. Isso faz a mesma regra funcionar em qualquer ativo
e em qualquer horário do dia. `F04` é a única ferramenta que não usa ATR: ela lê
o relógio.

---

## 4. Registros de regra

Formato de cada registro:

```
ID  nome
    QUANDO   condição executável
    PARAM    input, padrão, unidade, faixa
    ORIGEM   de onde veio
    DESLIGAR efeito de desligar
```

### 4.1 Contexto, em M5

**C01 · contexto por médias no M5**

    QUANDO   E20_5 > E50_5 + 0.10*ATR5  AND  ambos subindo
    PARAM    m5_ema_fast=20, m5_ema_slow=50, m5_sep=0.10 ATR
    ORIGEM   video 1, transposto para o tempo grafico maior
    DESLIGAR sem ele, o contexto fica apenas por cor de candle, o que aceita
             lateral com duas velas verdes

**C02 · contexto por cor no M5**

    QUANDO   >= 2 dos ultimos 3 candles de M5 fechados sao da mesma cor
    PARAM    janela=3, minimo=2
    ORIGEM   video 2 (fluxo de velas, sequencia de cor)
    DESLIGAR sem ele, uma unica vela de M5 pode definir o lado do dia

**C03 · contexto estrito**

    QUANDO   C01 e C02 concordam (ambos +1 ou ambos -1)
    PARAM    contexto_mode = strict | loose
    ORIGEM   sintese (interseccao dos dois metodos)
    DESLIGAR em loose, aceita C01 ou C02 isolado: mais sinais, mais ruido

**C04 · apenas M5 fechado**

    QUANDO   o candle de M5 usado ja fechou
    PARAM    fixo
    ORIGEM   consequencia do metodo
    DESLIGAR impossivel: candle em formacao nao entra em nenhum calculo

**C05 · atualização só no fechamento do M5**

    QUANDO   contexto recalculado somente quando M5 fecha
    PARAM    fixo
    ORIGEM   video 2 (o ciclo e uma unidade)
    DESLIGAR recalculando a cada M1, o contexto oscila e o motor liga e desliga

---

### 4.2 Tendência, no M1

**T01 · separação das médias**

    QUANDO   alta:  E20 - E50 > 0.10 * ATR
             baixa: E50 - E20 > 0.10 * ATR
    PARAM    sep_min = 0.10 ATR (faixa 0.05 a 0.30)
    ORIGEM   video 1, "laranja acima da branca" quantificado
    DESLIGAR médias coladas passam a valer como tendência; no M1 isso ocorre
             boa parte do dia

**T02 · inclinação da média rápida**

    QUANDO   (E20 - E20[5]) / 5 > 0.02 * ATR
    PARAM    slope_k=5, slope_min=0.02 ATR/candle
    ORIGEM   video 1, "as duas apontando para cima"
    DESLIGAR aceita média subindo 0,005 ATR por minuto, que é plano com ruído

**T03 · inclinação da média lenta**

    QUANDO   (E50 - E50[5]) / 5 > 0.01 * ATR
    PARAM    slope_slow_min = 0.01 ATR/candle
    ORIGEM   video 1
    DESLIGAR aceita alta com fundo caindo: o pior cenário para comprar volta

**T04 · sem cruzamento recente**

    QUANDO   nenhum cruzamento nos últimos 10 candles
    PARAM    no_cross_bars = 10
    ORIGEM   video 1, "no lateral as linhas viram uma bagunça"
    DESLIGAR o motor arma tendência logo após um cruzamento, no pior momento

**T05 · teto de cruzamentos**

    QUANDO   <= 1 cruzamento nos últimos 30 candles
    PARAM    max_crosses_30 = 1 (0 a 2)
    ORIGEM   video 1
    DESLIGAR entrelaçamento espaçado (cruz sem cruzar em 10) passa como tendência

**T06 · preço do lado certo**

    QUANDO   alta: C > E50;  baixa: C < E50
    PARAM    price_side = true
    ORIGEM   video 1, "o preço por cima delas"
    DESLIGAR permite comprar com preço abaixo da média lenta: reversão disfarçada

    NOTA a exigência é sobre a EMA lenta, não a rápida. No candle do sinal o
    preço está encostado na rápida por definição; exigir C > E20 eliminaria a
    própria estratégia.

**T07 · persistência**

    QUANDO   os 3 últimos candles satisfazem os critérios crus
    PARAM    trend_persist = 3 (2 a 6)
    ORIGEM   sintese
    DESLIGAR o instante exato em que as médias se abrem vira sinal

---

### 4.3 Impulso: o preço se afastou

**I01 · candle de força**

    QUANDO   body >= 0.80*ATR  AND  body >= 0.60*range  AND  range >= 0.15*ATR
             alta: C > O;  baixa: C < O
    PARAM    forca_body_atr=0.80, forca_body_range=0.60, min_range=0.15
    ORIGEM   video 2 ("candle de força, corpo grande")
    DESLIGAR sem ele, o impulso depende só de extensão de média, o que aceita
             tendência morna subindo sem nenhum candle de convicção

**I02 · rompimento local**

    QUANDO   alta: C > H[1];  baixa: C < L[1]
    PARAM    fixo na definição de força
    ORIGEM   video 2, "romper a região"
    DESLIGAR candle grande dentro do range anterior não rompe nada, é ruído

**I03 · sequência de cor**

    QUANDO   candle de força tem a mesma cor do anterior
    PARAM    require_sequence_color = true
    ORIGEM   video 2, "golpe na mesma cor"
    DESLIGAR aceita força isolada contra o fluxo

**I04 · extensão da média**

    QUANDO   max(H - E20, últimos 10 candles) >= 0.30 * ATR
    PARAM    ext_window=10, ext_min=0.30 ATR
    ORIGEM   video 1, "você nunca compra lá em cima"
    DESLIGAR permite sinal sem afastamento prévio: não há volta, há arrasto

**I05 · zona do corpo**

    QUANDO   zona = max(O,C) do último candle de força, válida 10 candles
    PARAM    zone_from=body, zone_window=10
    ORIGEM   video 2, "marque no corpo, não no pavio"
    DESLIGAR sem zona, só a EMA serve de referência e o retorno perde a segunda
             leitura

**I06 · não perseguir**

    QUANDO   alta: (C - E20) <= 1.00 * ATR
    PARAM    chase_max = 1.00 ATR (0.6 a 1.5)
    ORIGEM   sintese (proteção contra entrada no topo do impulso)
    DESLIGAR o motor compra a um ATR e meio da média, exatamente o que o vídeo 1
             proíbe

---

### 4.4 Retorno: o preço voltou

**R01 · toque na média**

    QUANDO   alta: L <= E20 + tol  AND  L >= E50 - 0.25*ATR
             tol = max(0.02*ATR, 0.0002*preço)
    PARAM    touch_tol=0.02 ATR, depth_pad=0.25 ATR
    ORIGEM   video 1, "encosta no EMA 20 ou fica entre as médias"
    DESLIGAR sem toque, o gatilho pode ocorrer a qualquer distância da média

**R02 · não afundar sob a referência lateral**

    QUANDO   alta: L >= E50 - 0.25*ATR
    PARAM    depth_pad = 0.25 ATR
    ORIGEM   sintese
    DESLIGAR pavio que atravessa as duas médias passa como "toque": é rompimento

**R03 · toque na zona**

    QUANDO   alta: L <= zona + tol  AND  L >= zona - 0.25*ATR
    PARAM    herda R01
    ORIGEM   video 2
    DESLIGAR perde a referência do vídeo 2 e a possibilidade de confluência

**R04 · escolha da referência**

    QUANDO   ref_mode = ema | zona | ambas
    PARAM    padrão "ambas"
    ORIGEM   síntese
    DESLIGAR é a chave do teste A/B 4 da seção 8

**R05 · cancelamento por furo**

    QUANDO   alta: C < E50 - 0.10*ATR -> descarta o ciclo
    PARAM    cancel_pad = 0.10 ATR
    ORIGEM   sintese
    DESLIGAR o motor continua esperando rejeição em tendência que já morreu

**R06 · confluência**

    QUANDO   as duas referências foram tocadas com diferença <= 3 candles
    PARAM    confluence_bars = 3
    ORIGEM   sintese (definição da grade A)
    DESLIGAR a grade A deixa de existir e todo sinal vira B ou C

---

### 4.5 Gatilho: a rejeição

**G01 · janela do gatilho**

    QUANDO   0 <= (bar_index - bar_do_toque) <= 3
    PARAM    trigger_window = 3 (1 a 5)
    ORIGEM   sintese
    DESLIGAR com janela infinita, uma rejeição 20 candles depois do toque vira
             entrada a um preço que não tem mais relação com a referência

**G02 · rejeição por reversão (engolfo de reversão)**

    QUANDO   C[1] < O[1]  AND  C > O  AND  C >= O[1]  AND  O <= C[1]
             AND  body >= body[1]
    PARAM    modo "Engolfo"
    ORIGEM   video 1, "vela verde que engole a vermelha de antes"
    DESLIGAR perde o gatilho principal do primeiro método

**G03 · força de continuação**

    QUANDO   alta: C > O  AND  C > H[1]  AND  body >= 0.80*ATR
    PARAM    modo "Forca"
    ORIGEM   video 2
    DESLIGAR perde o gatilho do segundo método

    ATENÇÃO G02 e G03 são coisas diferentes que os dois vídeos chamam de
    "engolfo". G02 é reversão (o preço caiu, foi rejeitado, volta). G03 é
    continuação (a força se confirma e segue). Misturar as duas produz um
    número médio que não descreve nenhuma.

**G04 · martelo**

    QUANDO   range >= 0.15*ATR  AND  lower_wick >= 2.0*body
             AND  lower_wick >= 0.25*ATR  AND  body <= 0.35*range
             AND  close_pos >= 0.60
    PARAM    wick_body=2.0, wick_atr=0.25, body_range=0.35, close_pos=0.60
    ORIGEM   video 1, "corpo pequeno e pavio grande"
    DESLIGAR perde o gatilho mais permissivo, o preferido de quem está começando

**G05 · range mínimo universal**

    QUANDO   range >= 0.15 * ATR, aplicado a todo padrão
    PARAM    min_range = 0.15 ATR
    ORIGEM   sintese
    DESLIGAR candle minúsculo passa a "parecer" martelo por ter corpo pequeno

**G06 · apenas candle fechado**

    QUANDO   o padrão é avaliado no fechamento
    PARAM    fixo
    ORIGEM   consequência do método
    DESLIGAR aceitar formação significa comprar um desenho que pode desaparecer;
             é o erro que faz backtest bonito e conta vazia

**G07 · modo de gatilho**

    QUANDO   Gatilho ∈ {Engolfo, Forca, Martelo, Engolfo + Forca, Todos}
    PARAM    padrão "Engolfo + Forca"
    ORIGEM   sintese
    DESLIGAR é a chave dos testes A/B 6 e 7

---

### 4.6 Execução

**E01 · entrada**: abertura do candle seguinte; no relógio é o mesmo instante do
fechamento do candle de gatilho. `PARAM` fixo.

**E02 · expiração**: exatamente 1 candle (60 s). `PARAM` expiry_candles=1.

**E03 · atraso máximo**: 3 segundos. Acima disso o sinal é descartado, porque o
strike já mudou. `PARAM` max_entry_delay_s=3.

**E04 · conferência de strike**: o preço de preenchimento precisa estar dentro de
0,25 ATR da abertura do candle. `PARAM` strike_tol=0.25 ATR.

**E05 · uma posição por vez**: nenhuma ordem nova com contrato em aberto.
`PARAM` uma_posicao=true.

---

### 4.7 Bloqueios: quando NÃO entrar

Estes são os registros que respondem "quando não entrar". Todos bloqueiam.

**B01 · lateral**

    QUANDO   contexto = 0 (C03) -> OFF
    ORIGEM   video 1
    EFEITO   não opera em lateral; é a maior fonte de perda em M1

**B02 · janela do ciclo M5**

    QUANDO   m5pos in {1,2,3};  bloqueado em 0 e 4
    PARAM    allowed_minutes = [1,2,3]
    ORIGEM   video 2
    EFEITO   remove 40% do tempo e justamente os minutos de retração forçada

**B03 · um sinal por ciclo M5**

    QUANDO   já houve sinal no ciclo de 5 minutos corrente
    PARAM    max_signals_per_m5 = 1
    ORIGEM   video 2, "cada candle inicia um ciclo"
    EFEITO   impede que o motor entre três vezes no mesmo movimento

**B04 · anomalia local**

    QUANDO   range > 4.0*ATR  OR  |O - C[1]| > 0.5*ATR
             -> bloqueia exatamente 1 candle seguinte
    PARAM    anomaly_atr=4.0, gap_pad=0.5, block_next=1
    ORIGEM   video 2, "o GAP anula a operação depois dele, não o dia"
    EFEITO   sem isso, o motor interpreta um candle de notícia como tendência

**B05 · spike de volatilidade**

    QUANDO   ATR > 3.0 * mediana(ATR,20) -> OFF
    PARAM    atr_spike_x = 3.0
    ORIGEM   videos 1 e 2 ("exaustão", "candle gigante é ruim")
    EFEITO   evita operar no clímax, onde a expiração de 60 s é imprevisível

**B06 · sessão**

    QUANDO   fora de 09:35 a 15:50 em Nova York -> OFF
    PARAM    session = "0935-1550"
    ORIGEM   padrão de day trade
    EFEITO   evita leilão de abertura, spread largo e o fim do dia

**B07 · cooldown**

    QUANDO   menos de 2 candles desde o último sinal -> OFF
    PARAM    cooldown = 2
    ORIGEM   sintese
    EFEITO   evita sequência de entradas em zigue-zague

**B08 · notícia**

    QUANDO   evento agendado em [-2, +5] minutos -> OFF
    PARAM    news_before=2, news_after=5
    ORIGEM   videos 1 e 2
    EFEITO   não mecanizável no indicador: é bloqueio manual declarado

**B09 · spread**

    QUANDO   spread / ATR > 0.10 -> OFF
    PARAM    spread_atr_max = 0.10
    ORIGEM   economia (nenhum dos vídeos cita)
    EFEITO   sem isso, o custo implícito come a vantagem em M1

**B10 · payout**

    QUANDO   payout < 0.85 -> OFF
    PARAM    payout_min = 0.85
    ORIGEM   economia
    EFEITO   abaixo disso o equilíbrio passa de 54% e a exigência cresce

**B11 · mercado real**

    QUANDO   feed OTC, sintético ou fora do horário real -> OFF
    PARAM    allow_otc = false
    ORIGEM   video 1, explícito
    EFEITO   operar OTC é operar contra um gráfico que a contraparte desenha

**B12 · aquecimento**

    QUANDO   menos de 250 candles de histórico -> OFF
    PARAM    warmup = 250
    ORIGEM   consequência numérica
    EFEITO   sinais com EMA contaminada pela semente

**B13 · grade mínima**

    QUANDO   grade do sinal < min_grade (padrão B) -> OFF
    PARAM    min_grade = A | B | C
    ORIGEM   hipótese (a ser testada)
    EFEITO   em C aceita contexto frouxo ou referência única

---

### 4.8 Risco

**K01 · stake**: 2% da banca por operação. `PARAM` stake_pct=0.02.

**K02 · martingale**: proibido. O parâmetro existe como `false` e o motor
**recusa subir** com `true`. `PARAM` martingale=false.

**K03 · all-in**: proibido, mesma regra de recusa. `PARAM` all_in=false.

**K04 · perda diária**: 5% da banca encerra o dia. `PARAM` max_daily_loss=0.05.

**K05 · sequência de perdas**: 3 losses seguidos encerram o dia.
`PARAM` max_consecutive_losses=3.

**K06 · teto de operações**: 8 a 10 por dia. `PARAM` max_trades_day=8.

**K07 · teto por perna**: 3 sinais por perna de tendência.
`PARAM` max_signals_per_leg=3.

**K08 · kill switch**: desliga após 3 erros de API ou 10 minutos sem feed.
`PARAM` kill_switch_errors=3, kill_switch_stale_s=600.

---

## 5. Matriz: quando NÃO entrar

Uma linha por cenário, com o registro que barra e o que o painel mostra. Esta
tabela é a resposta operacional para "por que não entrou agora".

| cenário no gráfico | barrado por | painel mostra |
| --- | --- | --- |
| médias coladas, entrelaçadas | T01, T04, T05 | `Tendencia: NENHUMA` |
| EMA 20 sobe, EMA 50 cai | T03 | `Tendencia: NENHUMA` |
| preço abaixo da EMA 50 numa "alta" | T06 | `Tendencia: NENHUMA` |
| contexto M5 diz alta, M1 diz baixa | C03 | `Estado: sem entrada` |
| primeira vela do ciclo de M5 | B02 | `Ciclo M5: 0 BLOQ` |
| última vela do ciclo de M5 | B02 | `Ciclo M5: 4 BLOQ` |
| já entrou neste ciclo de 5 minutos | B03 | `Estado: sem entrada` |
| candle de 5 ATR de range | B04 | fundo amarelo |
| gap de 0,8 ATR na abertura | B04 | fundo amarelo |
| ATR 4x a mediana do dia | B05 | `Spike ATR: BLOQ` |
| 09:32 ou 15:52 em Nova York | B06 | fundo cinza |
| 1 minuto após um sinal | B07 | `Estado: sem entrada` |
| CPI em 3 minutos | B08 | bloqueio manual |
| spread de 0,15 ATR | B09 | nenhum (fora do gráfico) |
| payout caiu para 0,80 | B10 | nenhum (fora do gráfico) |
| gráfico OTC | B11 | nenhum (escolha do operador) |
| primeira hora após abrir o gráfico | B12 | `Estado: sem entrada` |
| sinal de grade C com mínimo B | B13 | `Grade: C/C` |
| preço 1,2 ATR acima da EMA 20 | I06 | `Estado: sem entrada` |
| preço nunca se afastou da média | I04 | `Impulso: NAO` |
| pavio atravessou as duas médias | R02 | `Retorno: aguarda` |
| rejeição no 5º candle após o toque | G01 | `Gatilho: aguarda` |
| candle ainda aberto | G06 | nada é avaliado |
| banca já perdeu 5% hoje | K04 | `Estado: parada diaria` |
| 3 losses seguidos | K05 | `Estado: parada diaria` |
| 8 operações feitas hoje | K06 | `Estado: parada diaria` |
| 3 sinais na mesma perna | K07 | `Estado: sem entrada` |

Vinte e oito cenários, vinte e oito motivos nomeados. Se alguma entrada
acontecer sem que nenhuma linha desta tabela tenha sido checada, o log está
incompleto.

---

## 6. Perfis

Três presets para começar. Só o conservador é sugestão de uso; os outros dois
existem para teste e para comparar.

| parâmetro | conservador | padrão | permissivo |
| --- | --- | --- | --- |
| contexto | strict | strict | loose |
| sep_min | 0.15 | 0.10 | 0.05 |
| trend_persist | 4 | 3 | 2 |
| ext_min | 0.40 | 0.30 | 0.20 |
| ref_mode | ambas | ambas | ema |
| gatilho | Engolfo | Engolfo + Força | Todos |
| janela do gatilho | 2 | 3 | 5 |
| grade mínima | A | B | C |
| filtro M5 | sim | sim | não |
| um por ciclo M5 | sim | sim | não |
| cooldown | 3 | 2 | 1 |
| stake | 1% | 2% | 2% |

Uso correto: rodar o permissivo durante um mês para **coletar amostra**, e o
conservador para **operar depois** de o permissivo mostrar vantagem. Rodar o
permissivo com dinheiro real é usar o período de aprendizado como cobaias.

---

## 7. Checklist antes de cada entrada

Seis perguntas, na ordem. Qualquer "não" encerra. Serve para o humano que opera
com o indicador na frente, e é o que a automação faz em código.

1. O gráfico é **real**, não OTC, e estou em M1?
2. O painel mostra **tendência** (ALTA para call, BAIXA para put)?
3. O **ciclo M5** está em 1, 2 ou 3, e não houve sinal neste ciclo?
4. O **retorno** mostra TOCOU, e o preço não está longe da média?
5. O **gatilho** mostra OK, no candle fechado?
6. Meu **risco** está no tamanho certo (2%) e **dentro do limite do dia**?

Se as seis respostas forem sim e a seta apareceu no fechamento, a entrada é na
abertura do candle seguinte, com expiração de 60 segundos. Se aparecer seta com
qualquer resposta "não", a seta está errada e o log precisa ser conferido antes
de qualquer outra operação.

---

## 8. Como acrescentar uma regra nova

Toda regra nova entra como registro, nunca direto no código.

```
ID    proximo livre no grupo (ex.: T08)
      nome curto, sem adjetivo
      QUANDO   comparacao entre grandezas medidas (F01..F09 ou derivadas)
      PARAM    nome, padrao, unidade, faixa
      ORIGEM   video, documento ou hipotese propria
      DESLIGAR efeito observavel de desligar
```

Critérios de aceitação, todos obrigatórios:

1. **Mensurável.** A condição usa ferramenta existente ou define uma nova em F.
2. **Normalizada.** A comparação é em ATR, nunca em reais ou porcentagem.
3. **Testável isoladamente.** Existe um jeito de desligar só ela.
4. **Reversível.** Desligar não quebra outra regra.
5. **Rastreável.** A origem está declarada, e ser "achismo meu" é uma origem
   válida, desde que escrita.

Regra que não passa nos cinco é opinião. Opinião não entra em base.

---

## 9. Por que all-in e martingale ficam fora, com número

O vídeo 2 demonstra 86 chegando a 493 arriscando o capital inteiro. Multiplicador
de 5,73, o que exige `ln(5,73)/ln(1,86) = 2,81`, ou seja, **três acertos
seguidos**.

| acerto real | P(3 seguidas) | expectativa da sequência |
| --- | --- | --- |
| 50% | 12,5% | -19,5% |
| 53,76% (equilíbrio do payout 0,86) | 15,5% | 0,0% |
| 60% | 21,6% | +39% |
| 65% | 27,5% | +77% |

A linha do equilíbrio é a chave: **com 53,76% de acerto, apostar 2% fixo e
apostar tudo têm a mesma expectativa.** A diferença está na distribuição, não no
retorno esperado. Com 2% fixo, 20 losses custam 33% da banca e você continua
vivo. Com all-in, um loss custa 100% e a série acaba. A chance de 20 acertos
seguidos a 60% é de 1 em 27 mil.

Por isso K02 e K03 são recusas de subida, não parâmetros ignorados. Um parâmetro
que existe e não é aplicado é um convite para alguém ligar depois.

---

## 10. Limites

Opções binárias têm expectativa negativa por construção quando a taxa de acerto
fica abaixo do equilíbrio do payout, a corretora é a contraparte e não há câmara
de compensação. Com payout de 86%, acertar 50% perde 7% por operação: o
equilíbrio é 53,76%. No Brasil a CVM não autoriza essas plataformas e já
suspendeu dezenas delas.

Esta base é engenharia de regra: descreve o que os vídeos ensinam, em forma
testável, e o que neles não se sustenta. Não é recomendação de investimento, não
promete resultado e não substitui a medição com dados seus. Nenhuma regra aqui
autoriza operar com dinheiro real antes de o backtest fora da amostra mostrar
vantagem acima do equilíbrio, com intervalo de confiança, e nem depois disso o
risco de ruína deixa de existir.
