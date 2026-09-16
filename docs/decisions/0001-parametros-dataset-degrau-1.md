# 0001 — Parâmetros do dataset do degrau 1

- **Status**: aceita
- **Data**: 2026-09-16
- **Issue**: [KRA-24](https://linear.app/krasinski-projects/issue/KRA-24/definir-parametros-do-dataset-inicial)
- **Raciocínio completo**: [Parâmetros do Dataset — Degrau 1](https://linear.app/krasinski-projects/document/parametros-do-dataset-degrau-1-c5e2974a425d)
- **Domínio**: [Domain Definition — Marketplace de Delivery](https://linear.app/krasinski-projects/document/domain-definition-marketplace-de-delivery-14d902344e91)

Este arquivo é a referência normativa dos números. O documento no Linear
guarda o porquê de cada um; aqui ficam os valores que o gerador
(KRA-29) tem que respeitar.

## Contexto

O degrau 1 tem alvo de ~10K pedidos — pequeno de propósito, volume em que
qualquer abordagem funciona. O que restringe a escolha não é esse número
isolado, e sim a escada: os degraus seguintes crescem girando um botão, e
os parâmetros do degrau 1 precisam ser plausíveis **para a mesma empresa**
que, no degrau 4, gera ~500M linhas de GPS. Um valor que só fecha numa das
pontas está errado nas duas.

## Decisão

### Cardinalidades e janela

| Parâmetro  | Valor                                          |
| ---------- | ---------------------------------------------- |
| Janela     | 2026-04-01 a 2026-06-30 (91 dias, 3 meses cheios) |
| Merchants  | 30 (28 com pedidos, 2 com zero)                |
| Customers  | 3.000 cadastrados, ~2.500 com ≥1 pedido        |
| Pedidos    | 10.000 (~110/dia)                              |
| Timezone   | gerado em `America/Sao_Paulo`, persistido em UTC |
| Seed       | fixa — mesma seed + mesmos parâmetros = dataset idêntico |

Três meses completos dão partições mensais fechadas desde o início; a
lição de partição parcial entra depois, de propósito. As entidades com
zero pedidos são intencionais: produzem o caso de borda de `LEFT JOIN` e
de dimensão sem fato sem custo de geração.

### Distribuição por hora local

| Faixa            | Share |
| ---------------- | ----- |
| 00–06h           | 2%    |
| 06–10h           | 3%    |
| 10–11h           | 4%    |
| 11–14h (almoço)  | 30%   |
| 14–17h           | 8%    |
| 17–18h           | 5%    |
| 18–22h (jantar)  | 40%   |
| 22–24h           | 8%    |

### Distribuição por dia da semana

Pesos, soma 7,0:

| seg  | ter  | qua  | qui  | sex  | sáb  | dom  |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0,75 | 0,75 | 0,85 | 0,95 | 1,30 | 1,40 | 1,00 |

### Tendência

+5% ao mês, composto, normalizado para o total fechar em 10.000. Sem
tendência as três partições mensais saem do mesmo tamanho e a série fica
estacionária — o oposto do que um pipeline real enfrenta.

### Skew de merchant

Taxa-base por merchant como lognormal (σ ≈ 1,0). O que vale é o alvo
verificável, não a família da distribuição:

- top 20% dos merchants concentram 55–60% dos pedidos
- maior merchant fica em ≤ 15% do total

### Pedidos por cliente

Cauda longa: mediana ~2 pedidos no trimestre, p95 ~15, média 4,0.

### Regra de escala

Um multiplicador `S` governa a escada inteira. A janela **não** muda:

```text
pedidos   = 10.000 × S
merchants =     30 × √S
customers =  2.500 × S^0,85
```

|                       | S=1 (degrau 1) | S≈17–83 (degraus 2–3) | S≈167 (degrau 4) |
| --------------------- | -------------- | --------------------- | ---------------- |
| pedidos               | 10.000         | 170K – 830K           | 1,67M            |
| merchants             | 30             | 124 – 273             | 388              |
| customers             | 2.500          | 28K – 107K            | 194K             |
| pedidos/merchant/dia  | 3,7            | 15 – 33               | 47               |

A regra passa em dois testes: 1,67M pedidos × ~300 pings (1 a cada 5s por
~25 min de deslocamento) ≈ 500M linhas de GPS, o alvo do degrau 4; e 388
merchants cai em "merchants (centenas)" da definição de domínio. Os
degraus 2–3 têm alvo em eventos (~6 por pedido), daí a faixa de `S`.

### Calibração com dado público

As distribuições acima são escolhidas à mão, não extraídas de dado real —
por ora. O degrau 1 não tem courier, GPS nem duração de entrega, que é o
que dado público calibraria bem.

A decisão estrutural que sustenta o adiamento: **as distribuições entram
no gerador como parâmetro injetável**. Trocar "escolhido à mão" por
"calibrado" tem que ser mudança de config, nunca reescrita do gerador.

Fontes candidatas são brasileiras — o táxi de NY sai de cena, por ser
curva de commute e não de refeição:

| Fonte | O que calibra | Quando |
| ----- | ------------- | ------ |
| [Delivery Center](https://www.kaggle.com/datasets/nosbielcs/brazilian-delivery-center) — ~370K pedidos de delivery BR, jan–abr 2021 | sazonalidade hora/dia com a curva certa, skew de loja, tempo de entrega | serve já no degrau 1, se quisermos |
| [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — 100K pedidos de marketplace BR + geolocalização por CEP | skew de seller, distribuição geográfica | degraus 1–3 |
| [Pesquisa OD 2017 — Metrô SP](https://transparencia.metrosp.com.br/dataset/pesquisa-origem-e-destino) | duração e distância de deslocamento por zona e hora, em SP | degrau 4 |
| [API Olho Vivo — SPTrans](https://www.sptrans.com.br/desenvolvedores/api-do-olho-vivo-guia-de-referencia/) | perfil de velocidade urbana real por hora e região | degrau 4; API instável de verdade no degrau 5 |

## Fora de escopo

Atributos por entidade — ticket médio, mix de itens, taxa de cancelamento,
endereço, categoria de merchant — ficam em KRA-25 (merchants), KRA-26
(customers) e KRA-27 (orders). Aqui só cardinalidade, janela, distribuição
temporal e skew.

## Consequências

- KRA-29 recebe esses números como entrada e cria o arquivo de configuração
  executável que materializa o "parâmetro injetável" decidido aqui.
- O gerador aceita `S` como parâmetro desde o primeiro commit, mesmo valendo
  1 no degrau 1 — retrofitar o multiplicador depois sai mais caro.
- Os alvos de skew viram asserção de teste no gerador, não comentário: são
  ground truth verificável.
- Mudar `S` num degrau novo é editar este arquivo e o documento no Linear,
  num PR — não uma edição silenciosa.
