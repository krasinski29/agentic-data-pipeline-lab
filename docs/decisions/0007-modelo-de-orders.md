# 0007 — Modelo de `orders`

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: [KRA-27](https://linear.app/krasinski-projects/issue/KRA-27/modelar-orders)
- **Restringe**: [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1), [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated), [KRA-31](https://linear.app/krasinski-projects/issue/KRA-31/modelar-payments)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

O [0005](0005-fonte-normalizada.md) fixou a forma da fonte e o [0006](0006-modelo-de-customers.md)
o que é um cliente. Este fixa **o que é um pedido** — o fato que liga as duas
populações e a única tabela do degrau 1 com cadência diária.

## Contexto

Duas restrições governam este modelo, e elas puxam em direções opostas.

**O pedido do degrau 1 é o estado final.** No degrau 2 ele passa a ser
reconstruído a partir de eventos de transição de status, e sentir essa migração
é parte da lição. Modelar sabendo disso significa não construir hoje a máquina
de estados que aquele degrau existe para trazer.

**O que não nascer decomposto aqui não pode ser decomposto depois sem inventar
histórico.** Uma tabela nova é barata em qualquer degrau — o [0004](0004-terceira-porta-do-criterio.md)
já registrou isso. Quebrar um `total` já gerado em itens, taxa e desconto não é:
qualquer decomposição retroativa é ficção que passa a discordar do que a camada
derivada já leu.

A tensão entre as duas é o que esta decisão resolve: **decompor o valor agora,
não decompor o tempo**.

## Decisão

**Grão**: 1 linha por pedido, **estado final**, sem versionamento.
**Chave primária**: `order_id`.

### Esquema

| Coluna | Tipo | Nulo? | Papel |
| -- | -- | -- | -- |
| `order_id` | uuid | não | chave natural da fonte |
| `customer_id` | uuid FK → `customers` | não | quem pediu |
| `merchant_id` | uuid FK → `merchants` | não | de onde |
| `delivery_neighborhood_id` | int FK → `neighborhoods` | não | bairro do destino **deste** pedido |
| `delivery_latitude` | double | não | ponto de entrega — entrada da rota do degrau 4 |
| `delivery_longitude` | double | não | idem |
| `order_ts` | timestamp (UTC) | não | momento do pedido |
| `status` | text (`delivered`/`cancelled`) | não | estado final |
| `items_amount` | decimal(10,2) | não | valor dos itens, condicionado por `price_tier` |
| `delivery_fee` | decimal(10,2) | não | taxa de entrega |
| `service_fee` | decimal(10,2) | não | taxa de serviço |
| `discount_amount` | decimal(10,2) | não | desconto, sempre ≥ 0 |
| `total` | decimal(10,2) | não | o que foi cobrado |

### Dinheiro é `decimal`, nunca `float`

Ponto flutuante binário não representa `0,10` exatamente, e somar centavos em
`float` acumula erro — o bug clássico de sistema transacional. A escolha só vale
alguma coisa se sobreviver ao pouso: CSV entrega texto e deixa o tipo para quem
lê, Parquet tem `decimal` de verdade. É restrição concreta para o
[KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto),
não preferência estética.

### `total` é guardado, e a invariante é o que ele compra

Guardar um valor derivável parece violação da 3NF. Não é: o total é o **valor
congelado do contrato**, e as taxas mudam com o tempo — reconstruí-lo amanhã com
a tabela de taxas de amanhã daria outro número. O mesmo argumento vale para
`service_fee`, que hoje é 6% de `items_amount` e no degrau 3 deixa de ser.

O que a redundância compra é uma **invariante verificável**:

```text
total = items_amount + delivery_fee + service_fee - discount_amount
```

Vira asserção no gerador (KRA-29) e checagem de qualidade na camada curated
(KRA-30). Uma fonte que não oferece nenhuma invariante não ensina o que
verificar.

### `order_ts`, não `created_at`

As outras tabelas usam `created_at` e esta não, de propósito: no degrau 6 vão
existir dois tempos para o mesmo pedido — quando ele aconteceu e quando chegou
ao pipeline (*event time* × *ingestion time*). O nome já precisa dizer qual é
qual, ou a coluna que sobreviver vai ser lida como as duas coisas.

Gerado na curva do [0001](0001-parametros-dataset-degrau-1.md) em
`America/Sao_Paulo`, persistido em UTC. São Paulo é UTC−3 e não tem horário de
verão desde 2019, então a conversão é um deslocamento fixo — e ainda assim
**cerca de 18% dos pedidos caem no dia UTC seguinte** ao dia local em que
aconteceram (os 8% de 22–24h mais a última hora da faixa de jantar). Agregação
por dia sem converter erra um sexto do dataset, e a série diária fica com uma
quebra que nenhum erro estoura.

### Grão: `order_items` **não** existe no degrau 1

A itemização falha na porta 1 — nenhuma pergunta do degrau 1 precisa dela — e
passa mal pela porta 3, porque o 1—N que ela traria já está agendado: os eventos
de status do degrau 2 dão a mesma lição de *fan-out* (a explosão de linhas numa
junção, quando um pedido vira N linhas e `SUM(total)` passa a contar repetido)
sem arrastar cardápio junto.

E item honesto exige cardápio. O [0005](0005-fonte-normalizada.md) já mapeou que
esse dilema não tem saída boa aqui: menu estático é mentira, menu versionado é o
degrau 3. A premissa que o 0005 registrou — *sem `order_items` um menu seria
dimensão que ninguém referencia* — fica confirmada, não invertida.

`items_amount` é o valor do que foi pedido, sem dizer o que foi pedido. É o que
o degrau 1 precisa para todas as suas agregações.

### `status` e os 4% cancelados

Valores: `delivered` e `cancelled`. **4% cancelados** — 400 linhas em S=1 —
sorteados independentemente de faixa, horário, merchant e cliente.

Sem cancelamento a coluna seria constante, e o degrau 2 herdaria uma máquina de
estados sem ramo. Com ele, o degrau 1 ganha a primeira pergunta de faturamento
que não tem resposta única: `SUM(total)` inclui cancelado? A resposta é uma
decisão de negócio que o modelo não toma pelo analista — e é a mesma família de
armadilha do `WHERE status = 'active'` de merchants.

**Isso não antecipa o degrau 3.** Lá o cancelamento é *retroativo*: o pedido de
abril cancelado em junho, depois de contabilizado, que obriga a reprocessar. Aqui
é só o estado final de um pedido que nasceu e morreu dentro da janela.

**Não existe `cancelled_at`.** Pela mesma regra que barra `delivered_at`
(abaixo) — o estado final diz *o quê*, não *quando*. Saber quando o pedido foi
cancelado é exatamente o que os eventos do degrau 2 trazem.

**As 10.000 linhas do [0001](0001-parametros-dataset-degrau-1.md) são linhas da
tabela**, 9.600 entregues + 400 canceladas. Os alvos de skew e de pedidos por
cliente seguem medidos no grão da tabela, mesma leitura que o
[0006](0006-modelo-de-customers.md) fixou para a média de 4,0 por cadastro.

### Tempo: só `order_ts`

`delivered_at` fica de fora pelo mesmo teste que barrou `avg_prep_time_minutes`
no [0005](0005-fonte-normalizada.md): **no degrau 1 não existe duração nenhuma
para invalidar.** Acrescentá-lo no degrau 2, quando os eventos existirem, não
contradiz dado anterior — retrofit barato, porta 2 não se aplica.

A assimetria com o valor é o ponto central deste registro e vale explicitar:
decompor o dinheiro depois seria inventar histórico; acrescentar o tempo depois é
só acrescentar dado que ainda não existia.

### Composição do valor

`items_amount` sai de uma lognormal por faixa de preço do merchant, truncada em
R$ 15:

| `price_tier` | Mediana | σ (log) | Média resultante |
| -- | -- | -- | -- |
| 1 | R$ 28 | 0,45 | ~R$ 31 |
| 2 | R$ 45 | 0,45 | ~R$ 50 |
| 3 | R$ 70 | 0,50 | ~R$ 79 |
| 4 | R$ 115 | 0,55 | ~R$ 134 |

Como a taxa-base do merchant é independente da faixa (0005), a participação
esperada de cada faixa nos pedidos é a mesma da população de merchants —
20/40/30/10% — e o ticket médio de itens fica em **~R$ 63**.

| Componente | Regra |
| -- | -- |
| `delivery_fee` | R$ 4,90 + R$ 0,95/km sobre a distância *haversine* merchant → ponto de entrega, ruído multiplicativo U(0,92; 1,08), arredondado a R$ 0,10, piso R$ 5,90 |
| `service_fee` | 6% de `items_amount`, arredondado ao centavo |
| `discount_amount` | 15% dos pedidos, U(10%; 25%) de `items_amount`; os outros 85% com `0,00` |
| `total` | a invariante acima |

O ruído na taxa de entrega não é enfeite: sem ele a taxa seria um mapa
invertível de volta para a distância, e calcular distância é lição do degrau 4 —
o mesmo motivo que manteve `h3_cell` fora no 0005.

`discount_amount` entra como valor, não como percentual nem como cupom: não
existe entidade de promoção no degrau 1, e criá-la seria estrutura sem pergunta
que a referencie.

### Destino da entrega

**80% dos pedidos vão para o endereço principal** — `delivery_neighborhood_id`,
`delivery_latitude` e `delivery_longitude` copiados da linha do cliente no
momento do pedido. **20% vão para um endereço alternativo**: bairro sorteado
uniformemente entre os outros 14, ponto num raio de 1.500 m do centróide, o mesmo
raio de moradia do 0006.

É a fração que o 0006 deixou em aberto aqui. A cópia nos 80% é o que fecha o furo
que aquele registro apontou na alternativa `customer_addresses`: corrigir o
endereço do cliente **não** reescreve o destino de pedidos antigos, porque o
destino não é referência, é valor.

### Derivação da chave — e a garantia que `orders` não tem

```text
order_id = uuid5(NAMESPACE_PROJETO, f"{seed}:order:{i}")
```

Mesmo padrão de 0005 e 0006, e a propriedade 2 do
[0002](0002-modelo-de-replay-e-geracao.md) vale igual: mesma seed e mesmo `S`
devolvem o mesmo dataset em qualquer execução ou máquina.

**A garantia extra não se estende.** Para merchants e customers, o índice `i`
devolve *a mesma entidade* em qualquer `S` — os índices 0–29 continuam sendo os
mesmos 30 merchants quando a população vai a 388. Para `orders` isso é
impossível: a data de cada pedido sai de uma curva normalizada para `10.000·S`,
então o pedido de índice 7 em S=1 e em S=17 não são o mesmo pedido. A população
de pedidos não é aninhada, e nenhuma escolha de chave conserta isso.

Vale dizer o que a consequência **não** é: o dataset de S=1 continua
reproduzível byte a byte. O que não existe é continuidade entre degraus — subir
`S` é gerar outra história, não a mesma história com mais linhas.

### Escolha de merchant: independência, com o botão instalado desligado

A escolha do merchant **não pondera por distância**, como o 0006 assumiu. O skew
lognormal do [0001](0001-parametros-dataset-degrau-1.md) é gabarito verificável e
já virou asserção de teste; ponderar por proximidade o deslocaria.

O que muda em relação ao 0006 é que a ponderação passa a existir como
**parâmetro do gerador**, `locality_weight`, com valor **0** no degrau 1 — mesma
disciplina de "distribuição é parâmetro injetável" que o 0001 usou para adiar a
calibração. Ligar depois é mudança de config mais *rebuild*, nunca reescrita do
gerador.

### Ordem de geração

Estende a do [0006](0006-modelo-de-customers.md), que continua valendo:

```text
1. sortear data e hora pela curva do 0001 (local) e converter para UTC
2. escolher o merchant pela taxa-base renormalizada, respeitando a desativação de maio
3. escolher o cliente entre os elegíveis naquela data (created_at <= order_ts)
4. sortear o destino (80% principal / 20% alternativo)
5. sortear os valores a partir de price_tier e da distância resultante
6. renormalizar e reverificar os alvos do 0001 DEPOIS do truncamento
```

Merchant e cliente são sorteados **independentemente** — é onde a simplificação
acima se materializa.

### Regras duras, que viram asserção de teste

- `order_ts >= customer.created_at` para toda linha (herdada do 0006)
- `total = items_amount + delivery_fee + service_fee - discount_amount`
- nenhum pedido de merchant desativado com `order_ts` posterior à data de
  desativação de maio — data que **não** é persistida em lugar nenhum (0005)
- os 2 merchants sem pedido continuam sem pedido, e os 500 clientes sem pedido
  também
- nenhuma FK órfã: todo `customer_id`, `merchant_id` e
  `delivery_neighborhood_id` existe na tabela referenciada

## Alternativas consideradas e rejeitadas

**`order_items` + `menu_items`.** O modelo completo, e o que um marketplace real
tem. Rejeitada porque obriga a escolher entre menu estático — mentira, já que
preço de cardápio muda — e menu versionado, que é o SCD2 do degrau 3 comprado
antes da pressão que o justifica. A lição de cabeçalho/linha chega no degrau 2
pelos eventos, de graça.

**`order_items` sem cardápio**, com nome do produto em texto livre. Entrega o
1—N barato e recria exatamente a anomalia de atualização que o 0005 existe para
eliminar: o rótulo do produto repetido em N linhas, corrigível em N lugares.

**Só `total`.** A alternativa que este registro existe para recusar. Ver
Contexto.

**`delivered_at` no cabeçalho.** Tentadora por uma razão boa: o degrau 2 herdaria
a pergunta de reconciliação *"o `delivered_at` do cabeçalho bate com o
`MAX(event_ts)` dos eventos?"*, que é problema real de quem mantém fato e evento
lado a lado. Rejeitada porque é lição do degrau 2 sendo comprada no degrau 1, e
porque o retrofit é barato — não corrompe nada que já exista.

**Sem `status`, todo pedido entregue.** Mais enxuto, e adiaria a ambiguidade de
faturamento até `payments` criá-la por outro caminho (KRA-31). Rejeitada: a
coluna constante não ensina nada, e o degrau 2 precisaria introduzir a
terminação alternativa junto com os eventos, misturando duas mudanças numa.

**`order_date_local` como coluna.** Resolveria a armadilha de fuso de graça, e é
por isso que fica fora: converter UTC → local é trabalho da camada curated, e
entregá-lo pronto apaga a única lição de tempo disponível no degrau 1.

**`distance_km` ou `h3_cell` pré-computados.** Mesmo argumento do 0005 —
calcular a chave espacial *é* a lição do degrau 4.

**Ponderar a escolha de merchant por distância e renormalizar com IPF**
(*iterative proportional fitting*, o ajuste iterativo de tabela de contingência
usado em censo). Resolveria a simplificação mais atacável do modelo mantendo os
alvos de skew. Rejeitada pelo custo no KRA-29 e por uma razão mais séria: o alvo
deixaria de ser verificável por construção e passaria a depender da convergência
do ajuste — gabarito que depende de um algoritmo iterativo é gabarito mais fraco.

**Ponderar sem renormalizar.** Mais realista e mais simples que o IPF, ao preço
de abrir mão de um ground truth que já é asserção de teste. Realismo não paga
embaçar gabarito — o mesmo veredito que o 0005 deu para correlacionar
`price_tier` com volume.

**Taxa de entrega fixa, independente da distância.** Mais simples e teria uma
vantagem: esconderia a inflação de taxa que a independência de distância produz.
Rejeitada por isso mesmo — os dois pontos já estão no dataset, e uma taxa que os
contradiz é incoerência interna, pior que uma simplificação declarada.

**IP, device e user agent.** São atributos do evento, e a porta 3 do
[0004](0004-terceira-porta-do-criterio.md) autoriza estrutura, não atributo.
Retrofit barato. Entram nos eventos do degrau 2, se entrarem.

**`payment_method` como coluna de `orders`.** É a decisão do
[KRA-31](https://linear.app/krasinski-projects/issue/KRA-31/modelar-payments), e
a resposta de lá já é conhecida: pagamento tem grão próprio.

**`order_number` sequencial ou `PED-0001`.** Mesmos motivos do 0005.

## Simplificações assumidas

Registradas porque são onde este modelo é atacável.

- **Atribuição cliente↔merchant independente da distância.** Herdada do 0006 e
  confirmada aqui. O custo agora é mensurável: com distância média de ~7,5 km
  entre os 15 bairros, a taxa média sai em torno de **R$ 12**, contra R$ 5–10 da
  realidade. A simplificação deixou de ser abstrata e passou a ter preço em
  reais.
- **Cancelamento uniforme a 4%.** No mundo real correlaciona com merchant, com
  horário de pico e com ticket. Nada no degrau 1 lê essa correlação, e introduzi-la
  deslocaria os alvos por merchant.
- **`service_fee` a 6% fixo.** Marketplace real varia a taxa por merchant e por
  período. A variação é justamente o que o degrau 3 traz.
- **Ticket independente de horário e dia da semana.** Jantar de sábado tem ticket
  maior que almoço de terça. Introduzir isso só mudaria números que ninguém
  compara no degrau 1.
- **Os parâmetros da lognormal de ticket são *fitting*, não princípio.** Foram
  escolhidos para dar ticket médio plausível para delivery em São Paulo, não
  extraídos de dado público. É exatamente o que a calibração do 0001 substitui
  quando chegar, e por isso moram na config.
- **Desconto sem promoção por trás.** Um valor sorteado, sem cupom, campanha ou
  regra. Quem concedeu o desconto e por quê não existe no modelo.
- **Endereço alternativo uniforme entre os outros 14 bairros.** O realista seria
  "trabalho", concentrado em poucos bairros comerciais. Uniforme não distorce
  nada que o degrau 1 leia.

## Ressalva honesta

**O argumento do 0006 para o endereço morar no pedido fica valendo para 1 pedido
em 5.** Aquele registro se apoiou em dois pilares, e o primeiro era *"o cliente
pede de mais de um lugar hoje"*. Com 80% dos pedidos copiando o ponto do
cadastro, esse pilar sustenta só a minoria das linhas — e se a fração de endereço
alternativo caísse a zero, `delivery_*` viraria cópia pura de `customers`,
defensável apenas pelo segundo pilar (o ponto de entrega é entrada de geração da
rota do degrau 4). É por aqui que a decisão deve ser atacada: **os 20% não são
enfeite, são o que mantém o primeiro argumento de pé.**

**A taxa de entrega vaza a distância, ainda que embaralhada.** O ruído de ±8%
impede a inversão exata, mas a ordem de grandeza continua legível na coluna. É
vazamento pequeno e deliberado, aceito porque a lição do degrau 4 é *chave
espacial*, não distância — mas é vazamento.

## Consequências

- **KRA-28 herda três restrições concretas**: `orders` é a única tabela com
  cadência diária, então é a única candidata a partição por data — e escolher
  entre data UTC e data local é a primeira armadilha, com ~18% dos pedidos
  caindo no dia UTC seguinte ao dia local; o formato de pouso precisa preservar
  `decimal` ou a invariante de total vira aproximação; e `orders` é a tabela que
  cresce de 10.000 linhas (S=1) para 1,67M (degrau 4), enquanto as de referência
  não se mexem.
- **KRA-29 herda**: a ordem de geração em seis passos; `order_id` por UUIDv5
  sobre `seed + índice`; o parâmetro `locality_weight = 0`; as distribuições de
  ticket, taxa e desconto como config injetável, não código; os 4% de
  cancelamento; os 20% de endereço alternativo; e as cinco regras duras acima
  como teste, não comentário.
- **KRA-30 herda o primeiro trabalho real da camada curated**: resolver os
  lookups, converter `order_ts` para o fuso local antes de agregar por dia,
  verificar a invariante do total, decidir explicitamente se faturamento inclui
  cancelado, e preservar no `LEFT JOIN` os 2 merchants e os 500 clientes sem
  fato.
- **KRA-31 herda a contraparte da conciliação**: `orders.total` existe e está
  decomposto, então `payments.amount` tem com o que ser comparado. Herda também
  uma pergunta que esta decisão cria: **os 400 pedidos cancelados têm pagamento?**
  Capturado e estornado, ou nunca capturado? E, sem `cancelled_at`, não há âncora
  temporal no degrau 1 para o estorno.
- **O degrau 2 herda o teste de si mesmo.** Quando o pedido passar a ser
  reconstruído a partir de eventos, esta tabela é o gabarito que a reconstrução
  precisa reproduzir — e a primeira pergunta daquele degrau é se os dois batem.
- **O modelo não responde** qual era o `price_tier` do merchant no dia do pedido,
  nem quando o pedido foi entregue ou cancelado, nem o que foi pedido. As três
  ausências são deliberadas e têm degrau marcado: 3, 2 e — se algum dia — 2.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede.
