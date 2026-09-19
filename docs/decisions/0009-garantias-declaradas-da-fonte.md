# 0009 — Garantias declaradas da fonte: unicidade e nulidade

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: nenhuma — a questão apareceu na revisão do modelo v0, feita antes
  de abrir o [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto).
  Mesmo caminho do [0002](0002-modelo-de-replay-e-geracao.md)
- **Restringe**: [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1), [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

## Contexto

A revisão do modelo v0 comparou o ERD das oito tabelas com os registros 0005 a
0008 e encontrou o mesmo vazio nos dois eixos que um diagrama não desenha: o
modelo declara chave primária e chave estrangeira em toda tabela, e **não
declara mais nada**.

Para a **nulidade** a ausência é só de enunciado: as tabelas dos 0005–0008 já
trazem a coluna "Nulo?" preenchida com "não" em todas as linhas. O que nunca
foi dito é a consequência que sai disso quando se olham as oito juntas — *a
fonte do degrau 1 não tem um único valor nulo*, e isso é deliberado.

Para a **unicidade** a ausência é real, e ela morde exatamente no argumento que
abriu a porta 3. O [0005](0005-fonte-normalizada.md) normalizou por causa da
anomalia de atualização: "corrigir a grafia de um bairro passa a ser corrigir N
linhas". Só que uma tabela de lookup que aceita `Pinheiros` em duas linhas
**reintroduz a anomalia que ela existe para remover** — passam a existir dois
ids para a mesma coisa do mundo, e a FK deixa de significar o que promete.
Unicidade do rótulo é o que faz de um lookup um lookup.

## Decisão

A fonte declara, além de PK e FK, **unicidade** e **não-nulidade**.

### Unicidade

| Tabela | Única, além da PK |
| -- | -- |
| `cities` | `name` |
| `neighborhoods` | `(city_id, name)` |
| `cuisine_types` | `name` |
| `payment_methods` | `name` |
| `payments` | `(order_id, attempt_no)` |
| `merchants` | **nenhuma** — `name` repete de propósito |
| `customers` | **nenhuma** — `name` e `email` repetem de propósito |
| `orders` | **nenhuma** |

**`neighborhoods` é composta, e não `name` sozinho.** Nome de bairro é único
dentro da cidade, não no mundo: "Centro" existe em toda cidade brasileira. Com
uma cidade só a diferença não aparece hoje — e é justamente por isso que
escrevê-la certa agora custa zero. A forma composta também não antecipa
povoamento nenhum: `cities` continua com uma linha, como o 0005 decidiu.

**`(order_id, attempt_no)` fecha um furo que a invariante do 0008 não cobre.**
Aquela é de cardinalidade — exatamente uma linha terminal por pedido — e não
diz nada sobre duas linhas com o mesmo `attempt_no`. Duas tentativas número 1
no mesmo pedido não contam dinheiro duas vezes, mas descrevem uma ordem que não
existe, e nenhuma soma revela isso.

**O que não é único continua não sendo, e agora por declaração.**
`merchants.name`, `customers.name` e `customers.email` são as armadilhas de
junção que o 0005 e o 0006 compraram de propósito — `email` sendo a pior,
porque *quase* é chave. Estão nesta tabela com "nenhuma" escrito para que a
ausência seja legível como escolha, não como esquecimento.

### Nulidade

**Nenhuma coluna das oito tabelas é anulável.** Três razões independentes, e a
segunda é a que impede mudar isso por conveniência:

1. **Nada no degrau 1 tem valor ausente legítimo.** Toda coluna é derivada de
   `seed + índice` ou de uma regra de distribuição. Não existe campo opcional.
2. **É pré-requisito do degrau 3.** O schema drift que a definição de domínio
   promete é *"campo antes preenchido começando a vir nulo"*. Um campo que já
   vem nulo às vezes não tem como *começar* a vir nulo — a lição precisa que o
   estado inicial seja não-nulo.
3. **"Nulo de verdade" já tem degrau marcado.** A definição de domínio o atribui
   ao degrau 5, na fonte externa real, junto com API instável e rate limit.

A consequência prática é que **o pipeline do degrau 1 nunca encontra `NULL` na
raw**. `COALESCE`, lógica de três valores e junção NULL-safe não aparecem — e
isso é cronograma da escada, não esquecimento.

### As três garantias, e o que o pouso faz com elas

A fonte passa a declarar três coisas: integridade referencial (0005),
unicidade e não-nulidade (aqui). **Nenhuma das três sobrevive a um arquivo.**
Arquivo não tem `FOREIGN KEY`, não tem `UNIQUE` e não tem `NOT NULL` — o
gerador garante por construção, e o formato não garante nada.

Isso não é defeito; é a lição que o 0005 já tinha identificado, agora com três
garantias em vez de uma. E vira critério concreto do KRA-28:

| Opção de pouso | FK | Unicidade | Não-nulidade |
| -- | -- | -- | -- |
| CSV / JSON em disco | não | não | não |
| Parquet | não | não | **sim** (campo `required`) |
| SQLite / DuckDB / Postgres | **sim** | **sim** | **sim** |

A tabela é entrada da decisão, não a decisão: carregar mais garantias não é
automaticamente melhor — um pouso que impõe tudo tira do pipeline a chance de
descobrir o que a fonte assegurava e o formato não. É esse trade-off que o
KRA-28 resolve.

## Por que unicidade e nulidade moram no mesmo registro

A regra do [0001](0001-parametros-dataset-degrau-1.md) é um ADR por decisão, e
vale explicar por que isto é uma só: as duas respondem a mesma pergunta — *o
que a fonte declara além de PK e FK?* —, nascem do mesmo argumento (a porta 3
do [0004](0004-terceira-porta-do-criterio.md) autoriza estrutura), e produzem o
mesmo conjunto de consequências nas três issues abaixo.

Quando o degrau 3 introduzir a primeira coluna anulável, este arquivo é
supersedido inteiro e a parte de unicidade é **repetida** no registro novo —
exatamente como o 0005 fez com o 0003. Supersessão é do arquivo, não do
parágrafo.

## Alternativas consideradas e rejeitadas

**Deixar tudo implícito, já que arquivo não impõe nada mesmo.** É o argumento
mais forte contra este registro, e merece ser escrito com força: se nenhuma das
três garantias é enforçável no pouso escolhido, declará-las é decoração.
Rejeitada por três motivos operacionais — o gerador precisa saber o que
garantir, o teste precisa saber o que verificar, e o KRA-28 precisa de um
critério para comparar formatos. **Garantia não declarada não vira asserção.**

**Unicidade em `merchants.name` ou `customers.email`.** Mataria no nascimento as
duas armadilhas de junção que o 0005 e o 0006 compraram de propósito, e tornaria
decorativo o exercício de dedup do degrau 3.

**`name` único global em `neighborhoods`.** Funciona hoje e está errado pelo
motivo certo: a unicidade é dentro da cidade. Ver acima.

**Chave natural composta em `orders`, como `(customer_id, order_ts)`.** Tentador
porque parece identificar o pedido sem o UUID. Rejeitada porque dois pedidos do
mesmo cliente no mesmo segundo são improváveis mas legítimos — declarar
unicidade ali transformaria uma coincidência em erro de geração.

**Declarar a nulidade só quando ela aparecer, no degrau 3.** Inverte o
argumento: o ponto do registro é que a não-nulidade de hoje é pré-requisito
daquele degrau. Declarar depois seria declarar o que já teria mudado.

## Simplificações assumidas

- **A unicidade dos rótulos de referência é trivialmente verdadeira hoje.** As
  quatro listas são fixas e digitadas uma vez; nenhuma é gerada. A declaração só
  ganha dente quando alguma referência passar a ser produzida programaticamente.
- **Unicidade é de valor, não de grafia.** `Pinheiros` e `pinheiros` seriam dois
  valores distintos e ambos passariam na checagem. Normalização de grafia é
  assunto de deduplicação, que a definição de domínio coloca no degrau 3.
- **`orders` fica sem nenhuma chave única além da PK.** É o certo para o modelo
  e significa que um pedido duplicado por bug de geração não seria detectado por
  unicidade — só pelos alvos de volume do ADR 0001.

## Consequências

- **KRA-28 herda um critério de comparação novo**: das três garantias da fonte,
  quantas cada formato de pouso consegue carregar — com a tabela acima como
  ponto de partida, e com a constatação de que nenhuma opção de arquivo puro
  carrega unicidade ou FK.
- **KRA-29 herda três asserções de teste**: rótulo único em cada uma das quatro
  tabelas de referência; `(order_id, attempt_no)` único em `payments`; e
  nenhuma coluna nula em nenhuma das oito tabelas.
- **KRA-30 herda duas checagens de qualidade**: a unicidade dos rótulos tem que
  sobreviver à denormalização — resolver um lookup não pode multiplicar linha —
  e **qualquer `NULL` encontrado na raw do degrau 1 é defeito de pouso, não da
  fonte**, porque a fonte não produz nenhum. É uma asserção barata que testa o
  formato escolhido pelo KRA-28.
- **O degrau 3 supersede este registro** quando o schema drift introduzir a
  primeira coluna anulável. A parte de unicidade segue valendo e será repetida
  no registro que o substituir.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede.
