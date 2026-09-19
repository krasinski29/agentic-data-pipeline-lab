# 0008 — Modelo de `payments`

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: [KRA-31](https://linear.app/krasinski-projects/issue/KRA-31/modelar-payments)
- **Restringe**: [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1), [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

O [0007](0007-modelo-de-orders.md) fixou o que é um pedido e deixou duas
perguntas explicitamente para cá. Este registro fecha as duas e fixa **o que é
um pagamento** — a oitava tabela do degrau 1.

## Contexto

Pagamento entra pela **porta 3** do [0004](0004-terceira-porta-do-criterio.md):
é estrutura, não atributo. A issue trouxe três argumentos para isso, e a análise
descobriu que **um deles não se sustentava sozinho** — o que mudou o modelo. Vale
registrar os três com o veredito de cada um, porque é o que explica a forma
final.

### O vocabulário que separa este degrau do degrau 3

O esboço da issue tratava o ciclo de pagamento como um campo por momento. São
coisas distintas, e a distinção acaba sendo o que mantém o degrau 3 intacto:

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

### Argumento 1 — ciclo de vida próprio: **válido, mas só depois de uma correção**

A primeira versão deste modelo derivava o estado do pagamento do estado do
pedido: `delivered` → `captured`, `cancelled` → `voided`. O resultado é que

```text
SUM(amount) WHERE status = 'captured'  ==  SUM(total) FROM orders WHERE status = 'delivered'
```

seriam **sempre o mesmo número**. O ciclo de vida não estaria descolado do
pedido: estaria colado nele, e `payments` não responderia nenhuma pergunta de
faturamento que `orders` já não respondesse sozinha.

A correção está na Decisão, e custa zero estrutura: **uma fração dos pedidos
cancelados fica `captured`**, pelo caso real de *cancelamento depois do preparo*
— o restaurante já fez a comida, e a cobrança não volta. Não é estorno, não é
pendência, e não invade o degrau 3.

### Argumento 2 — grão 1—N: **válido, na forma fraca**

O 1—N vem **só da retentativa**. A divisão de valor entre métodos fica de fora
(ver Alternativas), pelo mesmo argumento com que o 0007 recusou `order_items`:
o 1—N que ela traria já está agendado nos eventos do degrau 2.

O custo disso está declarado na Ressalva honesta e é real: as linhas extras são
todas `failed`, então quem filtra por estado terminal recebe exatamente uma
linha por pedido e nunca sente o fan-out.

### Argumento 3 — atraso natural por método: **válido, e é o mais forte**

Com `settlement_days` na tabela de referência, "quanto dinheiro entra em que
dia" passa a ser respondível, e a resposta **não é** a curva de pedidos: Pix do
dia 1 entra no dia 1, crédito do dia 1 entra no dia 31. É uma pergunta que
nenhuma tabela anterior permitia formular, e no degrau 6 é o que torna o atraso
inerente em vez de encenado.

### Tempo: a assimetria que o 0007 cobra

O esboço da issue propunha `authorized_at`, `captured_at` e `settled_at`. O 0007
decidiu *decompor o valor agora, não decompor o tempo*, e barrou `delivered_at`
com o teste: **no degrau 1 não existe duração nenhuma para invalidar**, logo o
retrofit é barato e a porta 2 não se aplica.

O teste vale igual aqui, e há um segundo argumento independente: `settled_at`
seria `order_ts + settlement_days` — **derivável**. Persistir a coluna é
pré-computar a junção, o mesmo que o 0005 recusou em `h3_cell` e o 0007 em
`order_date_local`. O argumento 3 não se perde: o atraso continua real e
dirigido pelo método, calculado em vez de lido.

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

**Grão**: 1 linha por **tentativa de pagamento**.
**Chave primária**: `payment_id`.

| Coluna | Tipo | Nulo? | Papel |
| -- | -- | -- | -- |
| `payment_id` | uuid | não | chave natural da fonte |
| `order_id` | uuid FK → `orders` | não | o pedido pago |
| `payment_method_id` | int FK → `payment_methods` | não | método desta tentativa |
| `attempt_no` | int (1–2) | não | ordem das tentativas do mesmo pedido |
| `amount` | decimal(10,2) | não | valor tentado |
| `status` | text | não | `captured` / `voided` / `failed` |

`decimal`, nunca `float`, pelo mesmo argumento do 0007 — e a restrição sobre o
formato de pouso (KRA-28) se aplica a esta tabela igual.

**Não existe coluna de tempo.** `attempt_no` é o ordinal que carrega a ordem sem
carregar duração — mesmo movimento que manteve `price_tier` como int no 0005,
onde escala ordinal fechada resolve sem estrutura extra.

### Um método por pedido

Um pedido é pago com **um** método. O valor não se divide entre métodos, e
`amount` é sempre `orders.total`.

O 1—N existe, e vem inteiramente da **retentativa**: uma tentativa recusada e a
seguinte formam duas linhas para o mesmo pedido.

### Estados — três, e só um não é terminal

| Estado | Significado | Entra na soma de dinheiro? |
| -- | -- | -- |
| `captured` | cobrança efetivada | sim |
| `voided` | autorizado e nunca capturado | sim |
| `failed` | tentativa recusada, seguida de outra tentativa | **não** |

### Os 400 cancelados — a resposta que o 0007 deixou para cá

Todo pedido cancelado **tem** pagamento. A política:

| | Linhas | Estado terminal | História |
| -- | -- | -- | -- |
| Cancelado antes do preparo | 300 (75%) | `voided` | autorizado e nunca capturado |
| Cancelado depois do preparo | 100 (25%) | `captured` | o restaurante já fez a comida; a cobrança não volta |

Os 100 são sorteados **uniformemente** entre os 400, independentes de método,
faixa de preço, merchant e horário — mesma disciplina com que o 0007 sorteou os
próprios cancelamentos.

**É esta linha da tabela que sustenta o argumento 1.** Com ela, o estado do
pagamento deixa de ser função do estado do pedido, e "faturamento" passa a ter
três respostas legítimas e diferentes:

```text
SUM(orders.total)                          ~ R$ 771.000   tudo que foi cobrado
SUM(orders.total) WHERE delivered          ~ R$ 739.000   só o que foi entregue
SUM(payments.amount) WHERE captured        ~ R$ 747.000   o dinheiro que ficou
```

Nenhuma das três é errada, e **a terceira só é respondível com `payments`**. Sem
os 100, a terceira seria idêntica à segunda e a tabela não acrescentaria pergunta
nenhuma de faturamento.

Nada disso é estorno: os R$ 8.000 de diferença entre a segunda e a terceira são
dinheiro que **fica**, por política de cancelamento tardio. O estorno — dinheiro
que volta depois de contabilizado — continua inteiro no degrau 3.

### Recusa e retentativa

- **5% dos pedidos pagos em `credito`, `debito` ou `vale_refeicao`** têm uma
  primeira tentativa recusada
- a retentativa **troca de método em 60%** dos casos
- no máximo **uma** recusa por pedido: `attempt_no` vai a 2 e para
- a linha `failed` carrega `amount = orders.total` — é o valor que se tentou
  cobrar

`pix` e `dinheiro` **nunca aparecem como `failed`** — não há bandeira para
recusar. É restrição do modelo, não estatística.

O sorteio de recusa é **independente do status do pedido**. Um pedido cancelado
pode ter `failed` → `voided`, e a sequência é coerente: a primeira tentativa foi
recusada, a segunda foi autorizada, e o pedido foi cancelado antes da captura.

### Distribuição de métodos

Participação na **primeira** tentativa do pedido:

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

A distribuição medida sobre as linhas **terminais** não bate exatamente com esta
tabela, porque 60% das retentativas trocam de método. É esperado, não erro.

Os números são *fitting*, não princípio: foram escolhidos para dar um mix
plausível de delivery no Brasil em 2026, não extraídos de dado público.

### Volume em S=1

Não é alvo — é consequência das regras acima, e está aqui para ser conferido:

```text
10.000 pedidos
+   300 tentativas recusadas   (5% dos ~6.000 pedidos em metodo elegivel)
= 10.300 linhas
```

`COUNT(*) FROM payments` ≠ número de pedidos. A diferença é pequena de
propósito, e é toda de linhas `failed`.

### As invariantes

Deixaram de ser uma. São duas, e a segunda é a que tem dente:

```text
Para todo order_id:
  (1) COUNT(*) WHERE status IN ('captured','voided') = 1
  (2) essa linha tem amount = orders.total
```

A (2) é checagem de **linha**, da mesma família das do 0007. A (1) é de
**cardinalidade** e é o que impede o erro de verdade: um pedido com duas linhas
terminais é dinheiro contado duas vezes, e nenhuma soma isolada revela isso.

Mais a coerência de estado, que impede a invariante de fechar pelo motivo errado:

- pedido `delivered` → a linha terminal é `captured`, **nunca** `voided`
- pedido `cancelled` → a linha terminal é `voided` (300) ou `captured` (100)
- `failed` nunca é linha terminal

### `amount` é deliberadamente redundante

Com um método por pedido, `payments.amount` é **sempre** `orders.total` — dado
derivável, que este mesmo registro usou como motivo para barrar `settled_at`. A
inconsistência é aparente, e a distinção é a que o 0007 já fez ao guardar
`total`:

| | `settled_at` | `amount` |
| -- | -- | -- |
| Derivável de | **regra de referência** (`settlement_days`) | **o fato pai** (`orders.total`) |
| Derivar é | o trabalho da curated — a lição | reler a mesma coluna |
| No degrau 3 | continua derivável | **diverge** — estorno parcial, chargeback |

Um pagamento sem valor não é um pagamento. E é a redundância que torna
`payments.amount = orders.total` uma **checagem** — uma fonte que não oferece
nada para verificar não ensina o que verificar.

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
7. sortear o método do pedido pela distribuição acima
8. se o método for credito/debito/vale_refeicao, sortear a recusa (5%):
   recusada -> attempt_no = 1, status failed, amount = total
   terminal  -> attempt_no = 2, trocando de método em 60% dos casos
   no máximo UMA recusa por pedido
9. estado terminal, pelo status do pedido:
   delivered -> captured
   cancelled -> captured em 25% (cancelamento após o preparo), voided em 75%
```

Pagamento é gerado **depois** do pedido e a partir dele. Não há sorteio que
volte e mexa em `orders`.

### Regras duras, que viram asserção de teste

- as duas invariantes acima, para **todos** os 10.000 pedidos
- todo pedido tem **ao menos uma** linha em `payments`
- nenhum pedido `delivered` tem linha `voided`
- **exatamente 100** dos 400 cancelados têm linha terminal `captured`
- `failed` **nunca é a última tentativa** — toda recusa é seguida de uma
  tentativa terminal
- `pix` e `dinheiro` nunca aparecem com `failed`
- nenhuma FK órfã: todo `order_id` existe em `orders`, todo
  `payment_method_id` existe em `payment_methods`

## Alternativas consideradas e rejeitadas

**`payment_method` como coluna de `orders`.** O 0007 já a listou e apontou para
cá. Mata os três argumentos de uma vez: sem grão próprio não há ciclo de vida,
sem 1—N não há retentativa, e o atraso por método viraria atributo de um fato que
não tem tempo de liquidação.

**Pagamento dividido entre métodos** — parte no vale-refeição, parte no cartão.
É comum em delivery no Brasil, foi a proposta original da issue, e **foi a
decisão preliminar deste registro antes de ser revertida**. Merece o espaço
porque é a alternativa que alguém vai propor de novo.

O que ela entregava, e só ela: o fan-out que **sobrevive ao filtro**. Com split,
`JOIN payments WHERE status = 'captured'` devolve duas linhas para o mesmo
pedido, e `SUM(orders.total)` conta o pedido duas vezes sem que nada estoure. A
invariante também seria de **grupo** (`SUM(amount)` por `order_id`), não de
linha.

Rejeitada pelo mesmo argumento com que o 0007 recusou `order_items`: *o 1—N que
ela traria já está agendado* — os eventos de status do degrau 2 dão a lição de
fan-out, sobre a mesma tabela `orders`, sem precisar repartir valor no degrau 1.
Comprar a lição duas vezes custa a regra de repartição, a aritmética de centavos
que ela obriga, e um grão que o degrau 2 vai reorganizar de qualquer jeito.

**O custo está assumido e não é zero** — ver Ressalva honesta.

**Estado do pagamento derivado do estado do pedido** (`delivered` → `captured`,
`cancelled` → `voided`, sem exceção). Foi a primeira versão deste modelo, e
falha por um motivo que só aparece quando se escreve a consulta:
`SUM(captured)` seria idêntico a `SUM(total) WHERE delivered`, e a tabela não
acrescentaria pergunta de faturamento nenhuma. Os 100 cancelados capturados
existem exatamente para quebrar essa identidade.

**Os três timestamps do esboço** (`authorized_at`, `captured_at`, `settled_at`).
Máximo realismo, e contradiz frontalmente o 0007: captura − autorização é
duração, e o dataset passaria a ter duração de pagamento sem ter duração de
pedido.

**Só `settled_at` persistido, com calendário bancário.** A alternativa séria, e
a que quase passa: liquidação cai em dia útil, então D+30 em crédito pula fins
de semana e feriados — e aí **não é derivável**, o que a qualificaria pela porta
2 por mérito próprio. Rejeitada por trazer um calendário de feriados como
dependência nova do gerador e criar o primeiro tempo do dataset que não é
`order_ts`, para comprar uma precisão que nada no degrau 1 lê. **É por aqui que
a decisão de tempo deve ser atacada.**

**Estorno e chargeback no degrau 1.** É a correção retroativa que define o degrau
3, e sem `cancelled_at` não haveria nem âncora temporal para datá-la.

**Permitir divergência entre `amount` e `orders.total`.** Daria à conciliação
algo para *encontrar*, não só para *verificar*. Rejeitada pelo veredito que o
0002 e o 0007 já deram repetidas vezes: realismo não paga embaçar gabarito, e
divergência sem causa modelada é defeito injetado. A divergência honesta nasce no
degrau 3.

**Pedido cancelado sem linha de pagamento.** Criaria 400 casos de `LEFT JOIN`
sem correspondência — mas o 0005 já comprou esse caso com os 2 merchants sem
pedido e o 0006 com os 500 clientes sem pedido. Repetir ensina menos, e
descartaria a distinção void × estorno junto.

**Snapshot de `settlement_days` dentro de `payments`.** Protegeria o pagamento
antigo de uma mudança futura na tabela de referência. Rejeitada pela mesma
assimetria deliberada do 0005 com `price_tier`: não snapshotar é o que cria a dor
que justifica SCD2 no degrau 3.

**Correlacionar método com `price_tier`, bairro ou horário.** Nada no degrau 1 lê
a correlação, e ela deslocaria distribuições que já são asserção de teste.

**Split marketplace / merchant / entregador (repasse).** Conciliação de repasse é
assunto grande o bastante para merecer sua própria pressão, e o degrau 1 não tem
entregador para receber parte nenhuma.

**Adquirente / PSP como tabela.** Realista, e tecnicamente mais correto — o prazo
de liquidação é do contrato com o adquirente, não do método. Nenhuma pergunta do
degrau 1 lê, e pela porta 3 seria normalização sem anomalia que ela remova.

**Cartões salvos do cliente (`customer_payment_methods`).** É 1—N legítimo com
ciclo de vida próprio, então quase passa pela porta 3. Fica de fora porque nada
no degrau 1 referenciaria a entidade: `payments` já carrega o método, e o
instrumento específico não muda nenhuma agregação.

**Parcelamento.** Fora por domínio, não por convenção: ninguém parcela R$ 77.

## Simplificações assumidas

Registradas porque são onde este modelo é atacável.

- **`dinheiro` empresta um vocabulário que não é dele.** Não há autorização nem
  captura em pagamento na entrega: `captured` significa "recebido pelo
  entregador" e `voided` significa "nunca recebido". Pior, `settlement_days = 0`
  é ficção — em dinheiro o marketplace **não recebe nada**, quem recebe é quem
  entrega, e acertar isso é repasse, explicitamente fora de escopo.
- **`vale_refeicao` paga o pedido inteiro**, taxa de entrega incluída. No mundo
  real o VR paga comida, não taxa — e é justamente isso que produz o pagamento
  dividido. Com o split fora, a simplificação fica visível nesta linha.
- **Recusa uniforme a 5%** entre os métodos elegíveis. No mundo real correlaciona
  com valor, horário e histórico do cliente.
- **No máximo uma recusa por pedido.** Duas recusas seguidas existem.
- **A fração de 25% de cancelamento tardio é *fitting*.** Escolhida para que a
  diferença entre "entregue" e "capturado" seja visível sem dominar o número, não
  extraída de política real de marketplace.
- **A distribuição de métodos é *fitting*.** Mix plausível escolhido à mão, como
  as lognormais de ticket do 0007 — e mora na config pelo mesmo motivo.
- **`settlement_days` é fixo por método.** Adquirência real negocia prazo por
  merchant e antecipa recebível mediante taxa.

## Ressalva honesta

**O fan-out deste modelo é filtrável, e isso é a fraqueza central.** Com um
método por pedido, as únicas linhas extras são `failed`. Juntar `orders` com
`payments` sem filtrar infla o faturamento em ~3% (300 linhas de ~R$ 77 sobre
~R$ 771.000), mas `WHERE status = 'captured'` devolve **exatamente uma linha por
pedido** e desfaz o problema inteiro. Quem escrever a consulta defensiva óbvia
nunca vai sentir a armadilha. A decisão aposta que o degrau 2 traz a versão que
não se desfaz com um filtro — e se essa aposta falhar, é aqui que se ataca.

**A conciliação sempre fecha.** Com `amount = orders.total` por construção, o
exercício é de *método* — escrever e rodar a checagem — e nunca de *achado*.
Nenhum pedido vai divergir, porque nenhum foi gerado para divergir. A sensação de
conciliar dado real, em que algo sempre não fecha, só chega no degrau 3.

**`payments` tem cadência diária e nenhuma coluna de data própria.** É
consequência direta da decisão de tempo, e cai inteira no colo do KRA-28:
particionar `payments` exige a data do pai. Pode ser lição — tabela filha que
herda a partição do fato é situação comum de verdade — ou pode ser só atrito. Não
dá para saber antes daquela issue.

**O modelo não responde quando o dinheiro foi capturado nem quando liquidou de
fato** — só quando *deveria* liquidar, por derivação.

## Consequências

- **A fonte passa de sete para oito tabelas.** `payment_methods` junta-se a
  `cities`, `neighborhoods` e `cuisine_types` como tabela de referência.
- **KRA-28 herda três coisas**: a oitava tabela; `payments` com cadência diária
  e **sem coluna de data própria**, então particionar exige a data do pai ou não
  particionar no degrau 1; e `decimal` de novo.
- **KRA-29 herda**: gerar as duas tabelas novas; `payment_id` por UUIDv5 sobre
  `seed + índice`; a distribuição de métodos, a taxa de recusa e a fração de
  cancelamento tardio como config injetável; e as sete regras duras acima como
  teste, não comentário.
- **KRA-30 herda duas perguntas que nenhuma tabela anterior permitia**:
  *faturamento é o entregue ou o capturado?* — e agora são números **diferentes**,
  R$ 739.000 contra R$ 747.000 — e *quanto dinheiro entra em que dia?*, que exige
  o `JOIN` com `payment_methods` e não segue a curva de pedidos. Herda também a
  cardinalidade como checagem de qualidade (uma linha terminal por pedido) e o
  fan-out de ~3% por linhas `failed`.
- **O degrau 3 herda três coisas com nome**: estorno e chargeback, que a
  distinção void × refund manteve intactos; `settlement_days` mudando no tempo,
  que é SCD2 em tabela de **referência** — caso que `merchants` não oferece; e a
  primeira divergência honesta entre o que foi cobrado e o que foi pago.
- **O degrau 6 herda o atraso natural**: replay pela data de liquidação derivada
  faz a informação financeira do mesmo dia de pedido chegar espalhada por 30
  dias, dirigida pelo método. Não é encenação.
- **O modelo não responde** quando o pagamento foi capturado, quando liquidou de
  fato, com que instrumento específico foi pago, nem para onde o dinheiro foi
  depois de capturado. As quatro ausências são deliberadas.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede.
