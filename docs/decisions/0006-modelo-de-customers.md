# 0006 — Modelo de `customers`

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: [KRA-26](https://linear.app/krasinski-projects/issue/KRA-26/modelar-customers)
- **Restringe**: [KRA-27](https://linear.app/krasinski-projects/issue/KRA-27/modelar-orders), [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1), [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

O [ADR 0001](0001-parametros-dataset-degrau-1.md) fixou *quantos* clientes
existem. O [0004](0004-terceira-porta-do-criterio.md) fixou por que regra
algo entra num modelo, e o [0005](0005-fonte-normalizada.md) fixou que a fonte
é transacional normalizada. Este fixa **o que cada cliente é**.

## Contexto

`customers` é a população que **cresce**. O ADR 0002 separou os papéis:
merchants são estáveis entre janelas, e o caso de "membro de dimensão
aparecendo no meio do período" foi deliberadamente evitado lá — com 30
entidades ele distorceria a verificação de skew sem ensinar nada que customers
não ensine melhor. A dívida foi contraída naquele registro e é paga aqui.

### Uma leitura do ADR 0001 que este registro fixa

O 0001 diz "3.000 cadastrados, ~2.500 com ≥1 pedido" **e**
`customers = 2.500·S^0,85`. Os dois números não podem descrever a mesma coisa.
A leitura que fecha:

- média de 4,0 pedidos por cliente = 10.000 ÷ 2.500 → a fórmula governa os
  **ativos**;
- `2.500 · 167^0,85 ≈ 194K`, exatamente o valor que a tabela do 0001 dá para
  customers no degrau 4.

Logo: **`cadastrados = 3.000·S^0,85` e `ativos = 2.500·S^0,85`.** Isso
generaliza um número que no 0001 só existia em `S=1`; não contradiz nada lá, e
por isso não o supersede.

## Decisão

**Grão**: 1 linha por **cadastro**, não por pessoa. A distinção é
consequência das duplicatas deliberadas, abaixo, e é o ponto mais fácil de
errar neste modelo.
**Chave primária**: `customer_id`.

### Esquema

| Coluna | Tipo | Nulo? | Papel |
| -- | -- | -- | -- |
| `customer_id` | uuid | não | chave natural da fonte |
| `name` | text | não | **não é único** — homônimo real e duplicata |
| `email` | text | não | **não é único** — a quase-chave que engana |
| `neighborhood_id` | int FK → `neighborhoods` | não | endereço principal declarado no cadastro |
| `latitude` | double | não | ponto do endereço principal |
| `longitude` | double | não | idem |
| `created_at` | timestamp (UTC) | não | cadastro; 25% caem **dentro** da janela |

Sem `city` — chega por `neighborhoods.city_id`. Sem `status`: em `merchants`
ele existe para criar a dor do SCD2, que já está criada; repetir não ensina
nada novo, e cliente que para de pedir é churn, já visível pela ausência de
fato.

### Relacionamento

`customers 1 —— N orders`, via `orders.customer_id`.

A relação cliente↔merchant é **N—N e se materializa no fato**: sem FK direta
entre as duas entidades e sem tabela-ponte, porque `orders` já é a ponte. Uma
coluna `favorite_merchant_id` seria resultado de agregação guardado na fonte —
desatualizado no primeiro pedido novo, e é o que a camada curated calcula. Uma
tabela `customer_merchants` seria cópia do que `orders` já diz, com duas
fontes de verdade para o mesmo fato.

Como modelar isso no warehouse (star, snowflake, Inmon) é decisão do KRA-30,
não da fonte.

### Derivação da chave

```text
customer_id = uuid5(NAMESPACE_PROJETO, f"{seed}:customer:{i}")
```

Mesmo padrão do 0005, mesma propriedade: o índice `i` devolve o mesmo cliente
para qualquer `S`, execução ou máquina.

### Localização

`neighborhood_id` referencia `neighborhoods`, o mesmo vocabulário de 15
bairros do 0005 — **cliente e merchant precisam viver no mesmo espaço**, ou
distância de entrega não existe e o degrau 4 fica sem a estrutura de
clustering que torna o particionamento espacial uma lição real.

Ponto sorteado num raio de cerca de **1.500 m** do centróide, contra os 800 m
do merchant: comércio se concentra em corredor, moradia espalha. Distribuição
uniforme entre os 15 bairros — 200 por bairro em S=1. Ambos são *fitting*, não
princípio (ver Simplificações).

### PII deliberadamente suja

`name` e `email` entram, e entram **pela porta 2** do
[0004](0004-terceira-porta-do-criterio.md), não pela 3 — não são estrutura
transacional. O argumento é específico e vale registrar com precisão:

**PII limpa seria retrofit barato; PII suja não é.** Adicionar `name`/`email`
corretos num degrau futuro não contradiz dado nenhum — seriam
`f(seed, índice)`, estáveis. Mas duplicata de cadastro só existe se nascer com
o dataset: introduzi-la depois exigiria **reescrever o histórico**, e é a
ausência que corrompe.

**60 cadastros (~2% de 3.000) são segundo cadastro de alguém que já existe**
→ 2.940 pessoas distintas. Sorteados entre os 2.500 com ≥1 pedido, e **os dois
cadastros pedem** — senão a deduplicação não muda número nenhum e o exercício
é decorativo.

| Campo | Variação entre os dois cadastros da mesma pessoa |
| -- | -- |
| `customer_id` | **diferente** — são cadastros distintos de verdade, não erro de geração |
| `name` | acento removido, sobrenome abreviado, caixa trocada |
| `email` | caixa, ponto no gmail, provedor diferente |
| `created_at` | datas diferentes |
| localização | pode diferir |

**O gabarito fica fora do dataset**: um arquivo sob `ground_truth/` com os 60
pares, que o pipeline nunca lê e o teste lê. Uma coluna `same_person_as`
entregaria a resposta dentro da prova.

`name` e `email` são compostos a partir de duas listas curtas indexadas por
`seed + índice` — sem dependência nova. Isso produz de graça a propriedade que
torna deduplicação difícil de verdade: com ~200 nomes × ~100 sobrenomes para
3.000 clientes, **dois clientes diferentes vão se chamar igual por acaso**.
Duas causas de colisão, uma é duplicata e a outra não — que é exatamente o
erro que algoritmo de dedup comete. Não é efeito colateral tolerado; é o que
dá valor ao exercício.

### `created_at` — a dimensão que nasce no meio da janela

**25% (750 clientes) com `created_at` dentro de 2026-04-01..2026-06-30**,
uniforme. Os 2.250 restantes, uniformes entre 2023-01-01 e 2026-03-31.

**Regra dura, que vira asserção de teste**: nenhum pedido antes do
`created_at` do seu cliente.

### O mecanismo que evita duplicar o crescimento

O ADR 0001 já impõe tendência de +5%/mês. Se a chegada de clientes gerasse
pedidos por si, o crescimento sairia somado duas vezes. A ordem de geração
resolve:

```text
1. sortear a data do pedido pela curva do ADR 0001 (hora, dia da semana, tendência)
2. escolher o cliente entre os elegíveis naquela data (created_at <= data)
3. renormalizar as taxas-base para o total fechar em 10.000·S
4. reverificar os alvos de pedidos-por-cliente DEPOIS da renormalização
```

**A tendência é o alvo; a chegada de clientes é o mecanismo, não um segundo
somando.** A distribuição temporal continua sendo exatamente a do 0001 — o que
muda é *quem* pode receber cada pedido.

### Os 500 sem pedido

Distribuídos **75/25 entre pré-janela e intra-janela**, na mesma proporção da
população. Concentrá-los nos recém-chegados faria um
`WHERE created_at < '2026-04-01'` apagar o caso de `LEFT JOIN` sem
correspondência que o ADR 0001 pagou para ter — mesma disciplina das duas
regras de borda do 0005.

### Endereço: principal aqui, destino no pedido

`customers` guarda o endereço do cadastro. `orders` guarda
`delivery_latitude`, `delivery_longitude` e `delivery_neighborhood_id` — o
destino efetivo daquele pedido.

## Ressalva honesta

**O argumento fácil para essa divisão é o mais fraco dos três.** "A dimensão
vai versionar no degrau 3, então o fato precisa do snapshot" não se sustenta
aqui: a definição de domínio lista *endereço de merchant* entre as dimensões
que mudam, **não** o do cliente. E `merchants` apostou no sentido oposto de
propósito — `price_tier` não é snapshotado no pedido, justamente para criar a
dor do SCD2.

O que sustenta a decisão são outros dois, e é por eles que ela deve ser
atacada ou defendida:

1. **O cliente pede de mais de um lugar hoje**, no degrau 1 — casa, trabalho.
   É fato de domínio, não antecipação de degrau.
2. **O ponto de entrega é entrada de geração da rota de GPS do degrau 4.**
   Ground truth que gerou outro dataset tem que ficar preso onde a rota fica,
   ou os dois passam a discordar. `price_tier` é apenas *saída* — daí a
   assimetria com merchants não ser incoerência.

## Alternativas consideradas e rejeitadas

**Endereço só no cliente.** Um endereço por cliente, `orders` referenciando
apenas `customer_id`. O modelo mais simples dos três, e o que faz o cliente
pedir sempre do mesmo lugar. Rejeitada pelos dois argumentos acima.

**Tabela `customer_addresses` (1—N).** Endereços rotulados (casa, trabalho),
com `orders` apontando para `address_id`. Mais fiel ao domínio, e mantém o
furo: corrigir a linha do endereço reescreve o destino de pedidos antigos.
Cobra uma junção que nada no degrau 1 usa e não resolve o que a opção adotada
resolve.

**`acquisition_channel`.** Realismo puro — `neighborhood` e `cuisine_type` já
dão a agregação categórica que o degrau 1 precisa. Reprovada também pela porta
3: não exercita propriedade transacional nenhuma.

**CPF / documento.** Seria a chave de deduplicação perfeita, e dedup viraria
`GROUP BY documento`. Mata a lição que as 60 duplicatas existem para criar.

**`birth_date`, telefone.** A porta 3 autoriza estrutura, não atributo.
Retrofit é barato (`f(seed, índice)` numa tabela de 3.000 linhas), então a
decisão pode ser revista sem custo quando houver motivo.

**IP, device, user agent.** São atributos **do evento**, não da pessoa — o
mesmo cliente pede do wifi de casa e do 4G na rua. Guardá-los em `customers`
seria o mesmo erro de modelagem que guardar o endereço de entrega ali. Se
entrarem, entram em `orders` ou nos eventos de status do degrau 2.

**`status` / `is_active`.** Ver acima.

**`loyalty_tier`, `customer_sk`, CEP, endereço textual, IDs sequenciais.**
Pelos mesmos motivos registrados no [0005](0005-fonte-normalizada.md).

## Simplificações assumidas

- **Atribuição cliente↔merchant independente da distância.** A mais atacável
  deste registro. O realista seria pedido local — ninguém em Pinheiros pede do
  Tatuapé —, mas ponderar a escolha por distância perturbaria o skew lognormal
  do ADR 0001, que é gabarito verificável e virou asserção de teste. Custo
  assumido: no degrau 4 as rotas de GPS saem mais longas que a realidade,
  espalhadas sobre a matriz bairro×bairro em vez de concentradas na diagonal.
  Corrigir depois é *rebuild*, permitido pelo ADR 0002, mas encarece conforme
  houver camada derivada em cima.
- **`created_at` pré-janela uniforme.** A série de cadastros dá um degrau
  visível em 01/04: de ~58/mês para ~250/mês. Nenhuma pergunta do degrau 1 lê
  a série pré-janela. *Alternativa considerada*: comprimir os 2.250 em 9 meses
  (2025-07-01 em diante) para casar a taxa na emenda — rejeitada porque compra
  suavidade com uma mentira pior, a de um marketplace com merchants desde 2023
  e nenhum cliente até meados de 2025.
- **Distribuição uniforme entre os 15 bairros.** Desequilíbrio de oferta e
  demanda por região seria realista e deslocaria o skew sem ensinar nada novo
  no degrau 1.
- **Raio de 1.500 m.** Fitting, não princípio — escolhido por ser maior que o
  do merchant, pelo motivo dado acima.
- **Um endereço principal por cliente.** O segundo endereço, quando existir, é
  atributo do pedido (KRA-27).
- **A média de 4,0 pedidos por cliente do ADR 0001 é por cadastro.** Por
  *pessoa* ela é ligeiramente maior, porque os dois cadastros das 60
  duplicatas pedem. Os alvos do 0001 seguem medidos no grão da tabela, que é o
  cadastro.

## Consequências

- **KRA-27 herda**: `customer_id` como FK; `delivery_latitude`,
  `delivery_longitude` e `delivery_neighborhood_id` em `orders`; e a regra
  dura `order_ts >= customer.created_at`. Herda também duas perguntas em
  aberto — a fração de pedidos entregues fora do endereço principal, e se a
  escolha de merchant deve ser ponderada por distância ou preservar o skew do
  ADR 0001.
- **KRA-28 herda uma restrição concreta**: o pouso de `customers` precisa
  permitir recorte por `created_at`. Com o replay por cursor do ADR 0002, um
  snapshot full dos 3.000 faz o replay de 15/04 enxergar cliente que só se
  cadastra em junho — o dataset passa a mostrar o futuro e o gabarito deixa de
  valer.
- **KRA-29 herda**: derivar `customer_id` por UUIDv5 sobre `seed + índice`; a
  ordem de geração em quatro passos acima; as 60 duplicatas com suas variações
  de grafia; o arquivo de gabarito sob `ground_truth/`, fora do dataset; os
  500 sem pedido distribuídos 75/25; e a asserção de que nenhum pedido antecede
  o `created_at` do seu cliente.
- **KRA-30 herda**: `neighborhood` do cliente como dimensão de agregação (via
  `JOIN` com `neighborhoods`, que é o trabalho real da camada curated), e os
  500 clientes sem fato como caso de `LEFT JOIN`.
- **A escada ganha um tópico que não tinha.** Dedup / identity resolution,
  mascaramento e exclusão-com-reprocessamento entram no **degrau 3**, onde
  reprocessamento e contratos já moram. O documento de domínio é atualizado
  junto com este registro.
- **`email` não é único, e é pior que `name` nesse aspecto** — porque *quase*
  é. Junção ou deduplicação por e-mail exato vai parecer funcionar e vai
  errar nos 60 casos que importam.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede.
