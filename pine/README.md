# Indicador Fluxo na Volta: mapa de parâmetros

`fluxo-na-volta.pine` é o indicador de gatilho que consolida as duas
estratégias em um único painel de parâmetros. Roda em TradingView (Pine v5) e
serve para marcar a entrada no gráfico, não para operar sozinho.

Este README é o mapa: cada parâmetro do indicador, de onde ele veio e o que ele
custa. Os documentos de referência são `docs/volta-na-media-m1.md` (método 1) e
`docs/fluxo-na-volta.md` (síntese).

---

## 1. Instalar

1. TradingView, gráfico de **1 minuto**, ativo real (não OTC, não sintético).
2. Pine Editor, cole o conteúdo de `fluxo-na-volta.pine`, salvar, "Adicionar ao
   gráfico".
3. Abra as configurações do indicador. Todos os parâmetros estão agrupados por
   camada, na mesma ordem em que a regra é avaliada.

O indicador usa `barstate.isconfirmed`: o sinal aparece quando o candle fecha.
No gráfico, isso significa que a seta nasce **no fechamento do candle de
gatilho**, que é exatamente o instante de entrada na abertura do seguinte.

---

## 2. Como ler o gráfico

| elemento | significado |
| --- | --- |
| linha laranja | EMA 20 (referência de agora) |
| linha branca | EMA 50 (direção de fundo) |
| área preenchida entre as médias | laranja quando há separação suficiente, cinza quando as médias estão juntas |
| triângulo verde com "CALL" | gatilho de compra no fechamento do candle |
| triângulo vermelho com "PUT" | gatilho de venda no fechamento do candle |
| linha pontilhada verde/vermelha | zona do corpo do impulso, válida por `iZoneWin` candles |
| fundo cinza | fora da janela de sessão |
| fundo amarelo | candle bloqueado por anomalia |

O painel no canto mostra, camada por camada, o motivo do bloqueio. "SEM ENTRADA"
sem motivo aparente quase sempre é uma camada em `NAO`: é ali que se olha
primeiro, em vez de mexer nos parâmetros.

---

## 3. Parâmetros: origem e efeito

### 3.1 Núcleo

| parâmetro | padrão | veio de | efeito prático |
| --- | --- | --- | --- |
| Fonte | close | ambos | fechamento é o que forma a média |
| EMA rápida | 20 | vídeo 1 | referência de "agora", onde o preço volta |
| EMA lenta | 50 | vídeo 1 | direção de fundo |
| ATR | 14 | ambos (implícito) | régua de volatilidade de todas as tolerâncias |
| Aquecimento | 250 | spec | abaixo disso o indicador não emite sinal |

### 3.2 Tendência

| parâmetro | padrão | veio de | efeito prático |
| --- | --- | --- | --- |
| Separação mínima (ATR) | 0,10 | vídeo 1 | médias coladas não formam tendência; subir para 0,20 reduz sinais e ruído |
| Janela de inclinação | 5 | vídeo 1 ("apontando para cima") | quantos candles medem a inclinação |
| Inclinação EMA rápida | 0,02 ATR/candle | vídeo 1 | subir exige mais força |
| Inclinação EMA lenta | 0,01 ATR/candle | vídeo 1 | evita alta com fundo plano |
| Sem cruzamento nos últimos | 10 | vídeo 1 ("bagunça") | janela de veto por cruzamento |
| Máximo de cruzamentos em 30 | 1 | spec | zero é mais rígido, 2 é mais permissivo |
| Preço do lado certo | ligado | vídeo 1 | no sinal o preço está na média, então a exigência é sobre a EMA lenta |
| Persistência | 3 | spec | quantos candles a tendência precisa durar para armar |

### 3.3 Impulso

| parâmetro | padrão | veio de | efeito prático |
| --- | --- | --- | --- |
| Janela da extensão | 10 | vídeo 1 | por quantos candles o afastamento conta |
| Afastamento mínimo (ATR) | 0,30 | vídeo 1 | sem afastamento não existe volta, existe arrasto |
| Candle de força: corpo (ATR) | 0,80 | vídeo 2 | corpo grande é a definição de força |
| Candle de força: corpo/range | 0,60 | vídeo 2 | evita candle grande que é só pavio |
| Range mínimo (ATR) | 0,15 | spec | descarta candle minúsculo, que engana padrão |
| Candle de força exige mesma cor | ligado | vídeo 2 | sequência de fluxo |
| Não perseguir (ATR) | 1,00 | spec | acima disso é continuação, não volta |
| Validade da zona | 10 | vídeo 2 | por quantos candles a zona do corpo vale |

### 3.4 Retorno

| parâmetro | padrão | veio de | efeito prático |
| --- | --- | --- | --- |
| Referência | EMA + Zona | ambos | a volta pode ser à média ou à zona do impulso |
| Tolerância do toque (ATR) | 0,02 | spec | "encostou" com número |
| Não afundar sob a referência (ATR) | 0,25 | spec | pavio que atravessa tudo não é volta |
| Cancelar se furar (ATR) | 0,10 | spec | volta que virou reversão é descartada |
| Janela da confluência | 3 | spec | quantos candles para as duas referências coincidirem |

### 3.5 Gatilho

| parâmetro | padrão | veio de | efeito prático |
| --- | --- | --- | --- |
| Padrão aceito | Engolfo + Força | vídeos 1 e 2 | ver a seção 4 |
| Janela do gatilho | 3 | spec | candles após o toque em que a rejeição vale |
| Martelo: pavio/corpo | 2,0 | vídeo 1 | "pavio grande, corpo pequeno" |
| Martelo: pavio mínimo (ATR) | 0,25 | spec | tamanho absoluto do pavio |
| Martelo: corpo/range | 0,35 | spec | corpo pequeno em relação ao range |
| Martelo: fechamento no range | 0,60 | spec | fechou no terço de cima |

### 3.6 Filtros

| parâmetro | padrão | veio de | efeito prático |
| --- | --- | --- | --- |
| Filtro M5 | ligado | **vídeo 2** | bloqueia a primeira e a última vela de M1 do ciclo |
| Ciclo M5: minuto inicial | 1 | vídeo 2 | 0 bloquearia a primeira vela |
| Ciclo M5: minuto final | 3 | vídeo 2 | 4 bloquearia a última |
| Um sinal por ciclo M5 | ligado | vídeo 2 ("cada candle inicia um ciclo") | mede-se no relógio, não no gráfico |
| Anomalia | ligada | vídeo 2 | candle gigante ou gap |
| Anomalia: range (ATR) | 4,0 | spec | o que conta como gigante |
| Anomalia: gap (ATR) | 0,5 | spec | abertura longe do fechamento anterior |
| Anomalia bloqueia N seguintes | 1 | vídeo 2 | "anula a operação seguinte", não o dia |
| Spike de volatilidade | ligado | vídeos 1 e 2 | ATR 3x a mediana é clímax |
| Sessão | ligada | padrão | janela em horário de Nova York |
| Cooldown | 2 | spec | candles após um sinal |
| Grade mínima | B | hipótese | A é confluência, C é o mais frouxo |

---

## 4. A escolha que importa: qual "engolfo"

Os dois vídeos usam a palavra engolfo para coisas opostas. O indicador expõe as
três aceitações separadas em `Padrão aceito`, e o padrão é `Engolfo + Força`:

| opção | regra | de onde vem | caráter |
| --- | --- | --- | --- |
| Engolfo | corpo atual engole o corpo da vela anterior, que era contra a tendência | vídeo 1 | **reversão**, no retorno |
| Força | candle rompe a máxima (ou mínima) anterior, corpo grande, mesma direção | vídeo 2 | **continuação** |
| Martelo | pavio longo, corpo pequeno, fechamento no topo do range | vídeo 1 | **reversão**, mais permissivo |
| Engolfo + Força | os dois primeiros | padrão | cobre as duas teses |
| Todos | os três | máximo de sinais | mais ruído |

Se você for medir, comece separando os dois. São hipóteses diferentes com
probabilidades diferentes, e juntar produz um número médio que não descreve
nenhuma das duas.

---

## 5. Filtro M5: como conferir no gráfico

O filtro usa o relógio absoluto: `minuto % 5`. Em um candle de M1, a posição
dentro do ciclo de M5 é:

- `m = 0`: primeiro minuto. Bloqueado.
- `m = 1, 2, 3`: miolo. Permitido.
- `m = 4`: último minuto. Bloqueado.

Com expiração de 1 candle, uma entrada em `m = 3` liquida no fim de `m = 3`,
um minuto antes do fechamento do M5. Nenhuma operação encosta na virada.

Para ver os ciclos no gráfico sem trocar de tempo gráfico, coloque o candle de
M5 ao lado: no TradingView, abra o mesmo ativo em 5 minutos e use "Sincronizar
tempo gráfico". Os minutos 0 e 4 de cada bloco de M5 ficam evidentes.

---

## 6. Testes que valem mais que ajustar parâmetro

O indicador é uma pilha de hipóteses. Cada uma pode ser testada desligando um
input e comparando o resultado no log de operações:

| # | desligar | pergunta |
| --- | --- | --- |
| 1 | Filtro M5 | os sinais em `m = 0` e `m = 4` têm expectativa pior? |
| 2 | Separação + inclinação | tendência fraca produz resultado pior? |
| 3 | "Não perseguir" | perseguir impulso é pior que esperar? |
| 4 | Zona no corpo (usar só EMA) | a zona do vídeo 2 acrescenta alguma coisa? |
| 5 | Grade mínima (C) | a grade é informação ou ruído? |
| 6 | Força (usar só Engolfo) | continuação e reversão dão o mesmo resultado? |

A amostra mínima para qualquer conclusão está em `docs/fluxo-na-volta.md`,
seção 13. Em resumo: menos de 200 operações não serve para triagem, e provar
uma vantagem de 2 pontos percentuais exige mais de 2.000.

---

## 7. O que este indicador não faz, e por quê

- **Não envia ordem.** Nenhum indicador envia ordem; quem envia é a mão de quem
  está na frente da tela.
- **Não calcula payout.** O ponto de equilíbrio depende do payout da corretora e
  do horário: `1 / (1 + payout)`. Com payout de 86%, acertar 50% perde 7% por
  operação. O indicador marca regra; a conta é do operador.
- **Não mostra histórico de acerto.** O painel mostra o estado atual, não o
  passado. Histórico de acerto no próprio gráfico não é validação: a amostra é
  curta e o viés de seleção é enorme. A validação é o log, fora da tela.
- **Não mede spread.** Em M1, o spread entra duas vezes na comparação com o
  expiratório. Em ativo com spread largo em relação ao ATR, nenhuma regra
  resolve.
- **Não reconhece notícia.** O bloco de notícia é manual, por escolha: o
  indicador não tem agenda de eventos.
- **Não usa OTC.** Se a corretora oferece OTC, o indicador não sabe distinguir:
  é o operador que escolhe o gráfico certo.

---

## 8. Limites

Opções binárias têm expectativa negativa por construção quando o acerto fica
abaixo do equilíbrio do payout, a corretora é a contraparte e não há câmara de
compensação. No Brasil, a CVM não autoriza essas plataformas e já suspendeu
dezenas delas. Este indicador é uma ferramenta de leitura de gráfico: ele não
promete resultado, não é recomendação de investimento e não deve ser a única
coisa entre você e o botão de compra.
