# 0008 — Modelo de `payments`

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: [KRA-31](https://linear.app/krasinski-projects/issue/KRA-31/modelar-payments)
- **Restringe**: [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1), [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

O [0007](0007-modelo-de-orders.md) fixou o que é um pedido e deixou duas
perguntas explicitamente para cá. Este registro fecha as duas e fixa **o que é
um pagamento** — a oitava tabela do degrau 1, e a primeira com grão 1—N sobre
um fato que já existe.

## Contexto

Pagamento entra pela **porta 3** do [0004](0004-terceira-porta-do-criterio.md):
é estrutura, não atributo. Os três argumentos que sustentam isso estão na issue
e continuam valendo — ciclo de vida próprio, grão 1—N, e atraso de liquidação
dirigido pelo método.

O que a análise acrescentou a eles é a distinção de vocabulário que o esboço da
issue não fazia, e que acaba sendo o que separa este degrau do degrau 3:

| Termo | O que é |
| -- | -- |
| **autorização** (*authorization*) | a bandeira reserva o limite; nenhum dinheiro saiu |
| **captura** (*capture*) | a cobrança é efetivada — em delivery, tipicamente na entrega |
| **void** | a autorização expira ou é cancelada **sem nunca virar captura** |
| **estorno** (*refund*) | desfaz uma captura que **já aconteceu** |
| **liquidação** (*settlement*) | quando o dinheiro cai na conta de quem recebe |

*Void* é estado terminal alcançável dentro da janela. *Estorno* é correção
retroativa — a definição do degrau 3. É por essa fronteira, e não por uma regra
de escopo arbitrária, que estorno e chargeback ficam de fora daqui.

### A invariante muda de natureza

A do [0007](0007-modelo-de-orders.md) é de **linha**:
`total = items_amount + delivery_fee + service_fee - discount_amount`,
verificável olhando um registro. A de `payments` é de **grupo**:

```text
SUM(amount) por order_id = orders.total
```

Isso é categoricamente mais difícil de verificar, e é a armadilha de *fan-out*
— a explosão de linhas numa junção, em que `SUM` passa a contar repetido —
aplicada a **dinheiro**, não a contagem de linhas. É o argumento mais forte da
issue inteira, e ele **só existe se o grão for de fato 1—N em pedidos normais**.
É essa constatação que decide o grão abaixo.

### Tempo: a assimetria que o 0007 cobra

O esboço da issue propunha `authorized_at`, `captured_at` e `settled_at`. O
0007 decidiu *decompor o valor agora, não decompor o tempo*, e barrou
`delivered_at` com o teste: **no degrau 1 não existe duração nenhuma para
invalidar**, logo o retrofit é barato e a porta 2 não se aplica.

O teste vale igual aqui, e há um segundo argumento independente: com
`settlement_days` na tabela de referência, `settled_at` é
`order_ts + settlement_days` — **derivável**. Persistir a coluna é pré-computar
a junção, o mesmo que o 0005 recusou em `h3_cell` e o 0007 em
`order_date_local`.

E o argumento 3 da issue não se perde: o atraso continua real e dirigido pelo
método, calculado em vez de lido.

## Decisão

**A fonte passa a ter oito tabelas.** `payments` e `payment_methods` entram sob
a convenção do [0005](0005-fonte-normalizada.md).

### `payment_methods` — tabela de referência, 5 linhas

| `payment_method_id` | `name` | `settlement_days` |
| -- | -- | -- |
| 1 | `pix` | 0 |
| 2 | `credito` | 30 |
| 3 | `debito` | 1 |
| 4 | `vale_refeicao` | 15 |
| 5 | `dinheiro` | 0 |

Int pequeno como chave, pela regra do 0005: dado de referência é lista fixa, não
população derivada de seed.

`settlement_days` **não é rótulo cosmético** — é o dado que dirige quando o
dinheiro chega, e é o que dá conteúdo real ao `JOIN` que a camada curated do
KRA-30 precisa fazer para responder qualquer pergunta de liquidação. É a
diferença exata em relação a `price_tier`, que o 0005 manteve como int
justamente por ser rótulo.

### `payments`

**Grão**: 1 linha por **tentativa de pagamento, por perna**.
**Chave primária**: `payment_id`.

| Coluna | Tipo | Nulo? | Papel |
| -- | -- | -- | -- |
| `payment_id` | uuid | não | chave natural da fonte |
| `order_id` | uuid FK → `orders` | não | o pedido pago |
| `payment_method_id` | int FK → `payment_methods` | não | método desta tentativa |
| `attempt_no` | int (1–2) | não | ordem das tentativas da mesma perna |
| `amount` | decimal(10,2) | não | valor desta perna |
| `status` | text | não | `captured` / `voided` / `failed` |

`decimal`, nunca `float`, pelo mesmo argumento do 0007 — e a restrição sobre o
formato de pouso (KRA-28) se aplica a esta tabela igual.

**Não existe coluna de tempo.** `attempt_no` é o ordinal que carrega a ordem sem
carregar duração — mesmo movimento que manteve `price_tier` como int no 0005,
onde escala ordinal fechada resolve sem estrutura extra.

### Estados — os três são terminais, menos um

| Estado | Significado | Entra na soma? |
| -- | -- | -- |
| `captured` | cobrança efetivada | sim |
| `voided` | autorizado e nunca capturado | sim |
| `failed` | tentativa recusada, seguida de outra tentativa | **não** |

`voided` é o que responde a pergunta que o 0007 deixou aberta: **os 400 pedidos
cancelados têm pagamento sim, autorizado e nunca capturado.** É a história
natural de um cancelamento anterior à entrega, e não exige âncora temporal
nenhuma — que é o que o 0007 não tinha para oferecer.

`refunded` e `chargeback` **não existem no degrau 1**. Ver Contexto.

### Split: com causa, não por moeda ao ar

O split é sempre **vale-refeição + outro método**, que é o caso real no Brasil:
o saldo do VR cobre parte da conta e o resto vai em cartão ou Pix.

- **12% dos pedidos** têm `vale_refeicao` como método primário
- **65% desses dividem** → ≈ 780 pedidos com duas pernas em S=1
- a perna VR sai de **U(40%; 85%) do total**, arredondada ao centavo
- **a outra perna é a subtração**, nunca um segundo arredondamento

O último ponto é regra dura, não detalhe de implementação: arredondar as duas
pernas separadamente quebra a invariante por um centavo em uma fração das
linhas, e é precisamente o bug que um modelo de dinheiro existe para não ter.

Split por coin flip entre dois métodos quaisquer foi recusado — ver
Alternativas.

### Recusa e retentativa

- **5% das pernas pagas em `credito`, `debito` ou `vale_refeicao`** têm uma
  primeira tentativa recusada
- a retentativa **troca de método em 60%** dos casos
- no máximo **uma** recusa por perna: `attempt_no` vai a 2 e para

`pix` e `dinheiro` **nunca aparecem como `failed`** — não há bandeira para
recusar. É restrição do modelo, não estatística.

O sorteio de recusa é **independente do status do pedido**. Um pedido cancelado
pode ter `failed` → `voided`, e a sequência é coerente: a primeira tentativa foi
recusada, a segunda foi autorizada, e o pedido foi cancelado antes da captura.

### Volume em S=1

Não é alvo — é consequência das regras acima, e está aqui para ser conferido:

```text
10.000 pedidos
+   780 pernas de split
+  ~320 tentativas recusadas
= ~11.100 linhas
```

**`COUNT(*) FROM payments` ≠ número de pedidos já é a lição.** Um modelo em que
os dois números coincidem não tem 1—N nenhum, só uma tabela satélite.

### Distribuição de métodos

Participação como método primário do pedido:

| Método | Share |
| -- | -- |
| `credito` | 38% |
| `pix` | 35% |
| `vale_refeicao` | 12% |
| `debito` | 10% |
| `dinheiro` | 5% |

**Independente** de `price_tier`, de bairro e de horário — mesmo veredito que o
0005 deu para `price_tier` × volume e o 0007 para merchant × distância. E, como
todas as distribuições desde o [0001](0001-parametros-dataset-degrau-1.md),
entra no gerador como **parâmetro injetável**, não como código.

Os números são *fitting*, não princípio: foram escolhidos para dar um mix
plausível de delivery no Brasil em 2026, não extraídos de dado público.

### A invariante, e a regra de coerência que anda com ela

```text
Para todo order_id:
  SUM(amount) WHERE status IN ('captured','voided') = orders.total
```

Mais a coerência de estado, que é o que impede a invariante de fechar pelo
motivo errado:

- pedido `delivered` só tem linhas `captured` (além das `failed`)
- pedido `cancelled` só tem linhas `voided` (além das `failed`)
- nunca os dois no mesmo pedido

O par vira asserção no gerador (KRA-29) e checagem de qualidade na curated
(KRA-30) — e é a primeira checagem de **grupo** do projeto, contra as de linha
que o 0007 entregou.

### Derivação da chave

```text
payment_id = uuid5(NAMESPACE_PROJETO, f"{seed}:payment:{i}")
```

Mesmo padrão de 0005, 0006 e 0007. **E a mesma ressalva do 0007 se estende**: a
população de pagamentos não é aninhada entre degraus, porque a de pedidos não é.
O pagamento de índice 7 em S=1 e em S=17 não são o mesmo pagamento. Subir `S` é
gerar outra história.

### Ordem de geração

Estende a do [0007](0007-modelo-de-orders.md), que continua valendo inteira:

```text
7.  sortear o método primário do pedido pela distribuição acima
8.  se for vale_refeicao, decidir o split (65%); perna VR = U(40%;85%) do total
    arredondada ao centavo, outra perna = total - perna VR
9.  para cada perna elegível, sortear a recusa (5%): a recusada recebe
    attempt_no = 1 e status failed, a terminal recebe attempt_no = 2,
    trocando de método em 60% dos casos
10. atribuir o estado terminal pelo status do pedido:
    delivered -> captured, cancelled -> voided
```

Pagamento é gerado **depois** do pedido e a partir dele. Não há sorteio que
volte e mexa em `orders`.

### Regras duras, que viram asserção de teste

- a invariante de grupo acima, para **todos** os 10.000 pedidos
- todo pedido tem **ao menos uma** linha em `payments`
- nenhum pedido tem `captured` e `voided` ao mesmo tempo
- `failed` **nunca é a última tentativa de uma perna** — toda recusa é seguida
  de uma tentativa terminal
- `pix` e `dinheiro` nunca aparecem com `failed`
- nenhuma FK órfã: todo `order_id` existe em `orders`, todo
  `payment_method_id` existe em `payment_methods`

## Alternativas consideradas e rejeitadas

**`payment_method` como coluna de `orders`.** O 0007 já a listou e apontou para
cá. Mata os três argumentos da issue de uma vez: sem grão próprio não há ciclo
de vida, sem 1—N não há fan-out de dinheiro, e o atraso por método viraria
atributo de um fato que não tem tempo de liquidação.

**`payments` 1—1 com `orders`**, como tabela satélite que registra método e
estado. Mais barata, e ainda entregaria a ambiguidade de faturamento do
argumento 1. Rejeitada porque a invariante continuaria de linha, e o exercício
que justifica a tabela — verificar uma soma com fan-out no meio — não existiria.

**1—N só por retentativa, sem split.** A alternativa mais próxima da escolhida,
e a mais fácil de defender: mantém a história causal (recusou no crédito, refez
no Pix) e evita repartir valores. Rejeitada por um motivo específico: as linhas
extras são todas `failed`, e quem escreve `JOIN payments WHERE status =
'captured'` recebe **exatamente uma linha por pedido**. A armadilha de fan-out
existiria no schema e nunca dispararia no uso. Compra o 1—N no papel e não na
prática.

**Os três timestamps do esboço** (`authorized_at`, `captured_at`, `settled_at`).
Máximo realismo, e contradiz frontalmente o 0007: captura − autorização é
duração, e o dataset passaria a ter duração de pagamento sem ter duração de
pedido. A própria issue já antecipava esse risco.

**Só `settled_at` persistido, com calendário bancário.** A alternativa séria, e
a que quase passa: liquidação cai em dia útil, então D+30 em crédito pula fins
de semana e feriados — e aí **não é derivável**, o que a qualificaria pela porta
2 por mérito próprio. Rejeitada por duas razões e nenhuma é definitiva: traz um
calendário de feriados como dependência nova do gerador, e cria o primeiro tempo
do dataset que não é `order_ts` para comprar uma precisão que nada no degrau 1
lê. **É por aqui que esta decisão deve ser atacada** — o dia da semana da
liquidação é a diferença entre um exercício de fluxo de caixa plausível e um
aproximado.

**Permitir divergência entre `SUM(amount)` e `orders.total`.** Daria à
conciliação algo para *encontrar*, não só para *verificar*. Rejeitada pelo
veredito que o 0002 e o 0007 já deram repetidas vezes: realismo não paga embaçar
gabarito. E divergência sem causa modelada é defeito injetado — o oposto do
problema inerente que a definição de domínio exige. A divergência honesta nasce
no degrau 3, com estorno e chargeback.

**Pedido cancelado sem linha de pagamento.** Criaria 400 casos de `LEFT JOIN`
sem correspondência — mas o 0005 já comprou esse caso com os 2 merchants sem
pedido e o 0006 com os 500 clientes sem pedido. Repetir ensina menos, e
descartaria a distinção void × estorno, que é justamente o que mantém o degrau 3
intacto.

**Pedido cancelado com pagamento capturado e estornado.** É o degrau 3 comprado
no degrau 1, e sem `cancelled_at` não haveria nem âncora temporal para o
estorno.

**Perna VR limitada a `items_amount`.** Mais fiel — no mundo real o
vale-refeição paga comida, não taxa de entrega — e teria a elegância de tornar a
decomposição do 0007 estruturalmente necessária aqui. Rejeitada porque a perna
viraria função determinística de `orders`: o split deixaria de carregar
informação própria e o valor de cada perna seria reconstruível sem ler
`payments`. Mesma família de objeção que barrou pré-computar `h3_cell`.

**Split entre dois métodos quaisquer, sorteados.** Entrega o mesmo 1—N com menos
regra. Rejeitada porque split sem causa é exatamente o "realismo sozinho" que a
porta 3 não autoriza — ancorar no saldo do VR é o que torna a estrutura um fato
de domínio em vez de uma moeda ao ar.

**`leg_no` para amarrar a retentativa à perna que ela substitui.** Resolveria o
caso em que split e recusa coincidem. Rejeitada por uma coluna a mais para
~25 pedidos, e porque linhagem de tentativa é o que os eventos do degrau 2
trazem de graça.

**Snapshot de `settlement_days` dentro de `payments`.** Protegeria o pagamento
antigo de uma mudança futura na tabela de referência. Rejeitada pela mesma
assimetria deliberada do 0005 com `price_tier`: não snapshotar é o que cria a
dor que justifica SCD2 no degrau 3. Snapshotar aqui resolveria de graça uma
migração cuja dificuldade é a lição.

**Correlacionar método com `price_tier`, bairro ou horário.** Nada no degrau 1
lê a correlação, e ela deslocaria distribuições que já são asserção de teste.

**Split marketplace / merchant / entregador (repasse).** Conciliação de repasse
é assunto grande o bastante para merecer sua própria pressão, e o degrau 1 não
tem entregador para receber parte nenhuma.

## Simplificações assumidas

Registradas porque são onde este modelo é atacável.

- **`dinheiro` empresta um vocabulário que não é dele.** Não há autorização nem
  captura em pagamento na entrega: `captured` significa "recebido pelo
  entregador" e `voided` significa "nunca recebido". Pior, `settlement_days = 0`
  é ficção — em dinheiro o marketplace **não recebe nada**, quem recebe é quem
  entrega, e acertar isso é repasse, explicitamente fora de escopo.
- **Recusa uniforme a 5% entre os métodos elegíveis.** No mundo real a recusa
  correlaciona com valor, com horário e com o histórico do cliente.
- **No máximo uma recusa por perna.** Duas recusas seguidas existem.
- **Split só VR + outro.** Cartão + cartão existe e fica de fora.
- **A distribuição de métodos é *fitting*.** Mix plausível escolhido à mão, como
  as lognormais de ticket do 0007 — e mora na config pelo mesmo motivo: é o que
  a calibração substitui quando chegar.
- **`settlement_days` é fixo por método.** Adquirência real negocia prazo por
  merchant e antecipa recebível mediante taxa. Nada no degrau 1 lê isso.

## Ressalva honesta

**A conciliação do degrau 1 sempre fecha.** Com igualdade estrita, o exercício é
de *método* — verificar uma invariante de grupo com fan-out no caminho — e nunca
de *achado*. Ninguém vai encontrar um pedido que não bate, porque não existe.
Isso é deliberado e tem preço: a sensação de conciliar dado real, em que algo
sempre não fecha, só chega no degrau 3.

**Quando split e recusa coincidem, a linhagem se perde.** São ~25 pedidos em
S=1, com três linhas cada, e o modelo não diz qual tentativa substituiu qual
perna. É pouco, e é uma ausência real.

**`payments` tem cadência diária e nenhuma coluna de data própria.** É
consequência direta da decisão de tempo, e cai inteira no colo do KRA-28:
particionar `payments` exige a data do pai. Pode ser lição — tabela filha que
herda a partição do fato é situação comum de verdade — ou pode ser só atrito.
Não dá para saber antes daquela issue.

**O modelo não responde quando o dinheiro foi capturado nem quando liquidou de
fato** — só quando *deveria* liquidar, por derivação. O retrofit continua
barato, e é a aposta que o parágrafo do calendário bancário acima deixa exposta.

## Consequências

- **A fonte passa de sete para oito tabelas.** `payment_methods` junta-se a
  `cities`, `neighborhoods` e `cuisine_types` como tabela de referência.
- **KRA-28 herda três coisas**: a oitava tabela; `payments` com cadência diária
  e **sem coluna de data própria**, então particionar exige a data do pai ou não
  particionar no degrau 1; e `decimal` de novo — a invariante de grupo é ainda
  mais sensível a arredondamento que a de linha, porque o erro se acumula sobre
  várias linhas antes de ser comparado.
- **KRA-29 herda**: gerar as duas tabelas novas; `payment_id` por UUIDv5 sobre
  `seed + índice`; a distribuição de métodos, a taxa de split e a de recusa como
  config injetável; **calcular uma perna do split e derivar a outra por
  subtração**; e as seis regras duras acima como teste, não comentário.
- **KRA-30 herda o primeiro trabalho de verificação de grupo**: a invariante por
  `order_id`, a junção com `payment_methods` para derivar a data de liquidação,
  e a armadilha de fan-out ao juntar `orders` com `payments` — `SUM(total)` com
  o `JOIN` no meio conta os 780 pedidos de split duas vezes. Herda também que
  **faturamento passa a ter dois eixos**: status do pedido *e* status do
  pagamento. A pergunta que o 0007 criou (`SUM(total)` inclui cancelado?) agora
  tem uma segunda metade.
- **O degrau 3 herda três coisas com nome**: estorno e chargeback, que a
  distinção void × refund manteve intactos; `settlement_days` mudando no tempo,
  que é SCD2 em tabela de **referência** — caso que `merchants` não oferece; e a
  primeira divergência honesta entre o que foi cobrado e o que foi pago.
- **O degrau 6 herda o atraso natural** do argumento 3 da issue: replay pela
  data de liquidação derivada faz a informação financeira do mesmo dia de pedido
  chegar espalhada por 30 dias, dirigida pelo método. Não é encenação.
- **O modelo não responde** quando o pagamento foi capturado, quando liquidou de
  fato, qual perna uma retentativa substituiu, nem para onde o dinheiro foi
  depois de capturado. As quatro ausências são deliberadas e três têm degrau
  marcado: 2, 3 e 3.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede.
