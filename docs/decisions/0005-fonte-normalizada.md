# 0005 — Fonte normalizada: tabelas de referência e `merchants` restabelecido

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: [KRA-25](https://linear.app/krasinski-projects/issue/KRA-25/modelar-merchants) (reaberta), questão levantada no [KRA-26](https://linear.app/krasinski-projects/issue/KRA-26/modelar-customers)
- **Supersede**: [0003](0003-modelo-de-merchants.md), junto com [0004](0004-terceira-porta-do-criterio.md)
- **Restringe**: [KRA-26](https://linear.app/krasinski-projects/issue/KRA-26/modelar-customers), [KRA-27](https://linear.app/krasinski-projects/issue/KRA-27/modelar-orders), [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1), [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

Este registro substitui o [0003](0003-modelo-de-merchants.md) quanto ao
esquema de `merchants` e acrescenta as tabelas de referência da fonte. O que o
0003 decidiu e continua valendo está **repetido aqui**, não incorporado por
referência — supersessão é do arquivo inteiro, e um registro que só aponta
para o anterior obriga a ler os dois para saber o que vale.

## Contexto

A fonte que o gerador produz representa o **sistema transacional** do
marketplace: o banco que um time de aplicação manteria. O 0003 modelou
`merchants` com `cuisine_type`, `neighborhood` e `city` como texto, e
justificou com *"como uma fonte real guardaria"*.

Essa justificativa está errada. Fonte transacional real **normaliza** valor de
referência em tabela própria, porque repetir o rótulo em N linhas cria
anomalia de atualização: corrigir a grafia de um bairro passa a ser corrigir N
linhas, e corrigir N−2 faz o dado discordar de si mesmo.

O erro tinha uma consequência que só apareceu depois, e é a que mais pesa:
**uma fonte já achatada não deixa nada para a camada curated denormalizar.** O
degrau 1 promete "ingestão, camadas raw/curated, modelagem básica"; com
lookups resolvidos na origem, a transformação raw→curated do KRA-30 seria
cópia de arquivo com outro nome.

Há ainda uma confusão de vocabulário que o 0003 carregava sem perceber:
**fonte não tem star schema**. Star, snowflake, Inmon e Data Vault são formas
do *warehouse*, escolhidas depois — decisão do KRA-30, não desta. Com apenas
três tabelas, a forma normalizada e a forma em estrela são visualmente
idênticas, e foi essa coincidência que escondeu a troca.

O [ADR 0004](0004-terceira-porta-do-criterio.md) abriu a porta que autoriza
esta decisão pelo motivo verdadeiro, em vez de por uma justificativa de degrau
montada depois.

## Decisão

**A fonte é um modelo transacional normalizado.** Seis tabelas no degrau 1.

```text
cities          (city_id, name, state)
neighborhoods   (neighborhood_id, city_id FK, name, centroid_lat, centroid_long)
cuisine_types   (cuisine_type_id, name)

merchants       (merchant_id, name, cuisine_type_id FK, neighborhood_id FK,
                 latitude, longitude, price_tier, created_at, status)
customers       (ver 0006)
orders          (ver KRA-27)
```

### Tabelas de referência

| Tabela | Linhas | Conteúdo |
| -- | -- | -- |
| `cities` | 1 | São Paulo |
| `neighborhoods` | 15 | nome, FK para cidade, centróide aproximado |
| `cuisine_types` | 8 | `brasileira`, `hamburguer`, `pizza`, `japonesa`, `italiana`, `saudavel`, `doces_sobremesas`, `arabe` |

**Chave de referência é int pequeno, não UUIDv5.** Dado de referência é lista
fixa, não população derivada de seed: `neighborhood_id = 1..15` é o mesmo em
qualquer `S`, qualquer seed e qualquer execução. Isso é uma garantia **mais
forte** que a propriedade 2 do [ADR 0002](0002-modelo-de-replay-e-geracao.md),
não mais fraca. É também o que sistema real faz, e a diferença de custo de
`JOIN` entre int e uuid é coisa que se sente no degrau 4.

**Os centróides viram dado.** No 0003 as 15 coordenadas existiam apenas como
tabela dentro do registro, o que obrigaria o gerador a embuti-las no código.
Em `neighborhoods.centroid_lat/centroid_long`, gerador e pipeline leem da
mesma origem.

### Vocabulário de `neighborhoods` e centróides

Uma cidade só — São Paulo. O [ADR 0001](0001-parametros-dataset-degrau-1.md)
já gera em `America/Sao_Paulo`, e as fontes de calibração do degrau 4
(Pesquisa OD do Metrô SP, API Olho Vivo da SPTrans) são SP-específicas.

Coordenadas são centróides **aproximados**, não pontos levantados — a precisão
que importa é a estrutura de distâncias relativas, não o endereço.

| Bairro | lat | long |
| -- | -- | -- |
| Pinheiros | -23,5665 | -46,7020 |
| Vila Madalena | -23,5540 | -46,6900 |
| Itaim Bibi | -23,5860 | -46,6790 |
| Moema | -23,6020 | -46,6650 |
| Jardim Paulista | -23,5680 | -46,6620 |
| Vila Mariana | -23,5890 | -46,6340 |
| Perdizes | -23,5370 | -46,6790 |
| Santana | -23,5020 | -46,6250 |
| Tatuapé | -23,5400 | -46,5760 |
| Lapa | -23,5230 | -46,7040 |
| Butantã | -23,5710 | -46,7200 |
| Bela Vista | -23,5590 | -46,6450 |
| Consolação | -23,5540 | -46,6600 |
| Saúde | -23,6180 | -46,6390 |
| Ipiranga | -23,5920 | -46,6100 |

### Esquema de `merchants`

**Grão**: 1 linha por merchant, estado atual, **sem versionamento**.
**Chave primária**: `merchant_id`.

| Coluna | Tipo | Nulo? | Papel |
| -- | -- | -- | -- |
| `merchant_id` | uuid | não | chave natural da fonte |
| `name` | text | não | rótulo humano — **não é único** |
| `cuisine_type_id` | int FK → `cuisine_types` | não | dimensão de agregação do degrau 1 |
| `neighborhood_id` | int FK → `neighborhoods` | não | localização declarada no cadastro |
| `latitude` | double | não | ponto do merchant — origem da corrida no degrau 4 |
| `longitude` | double | não | idem |
| `price_tier` | int (1–4) | não | condiciona o ticket no KRA-27; atributo SCD2 no degrau 3 |
| `created_at` | timestamp (UTC) | não | cadastro, sempre anterior a 2026-04-01 |
| `status` | text (`active`/`inactive`) | não | estado atual, sem histórico |

`city` **sai da tabela** — chega por `neighborhoods.city_id`. É exatamente a
anomalia que a normalização remove: a cidade do merchant era derivável do
bairro dele, então repeti-la era guardar o mesmo fato em dois lugares.

`latitude`/`longitude` **ficam**, mesmo com o centróide em `neighborhoods`.
Parece redundância e não é: o centróide é do bairro, o ponto é do merchant. O
ponto sai de um sorteio num raio de cerca de **800 m** do centróide do bairro,
seeded pelo índice. O objetivo é *clustering* — Pinheiros → Pinheiros é
entrega curta, Pinheiros → Tatuapé é longa. Coordenada uniforme sobre a cidade
não teria essa estrutura, e a lição de particionamento espacial do degrau 4
ficaria artificial.

### Derivação da chave de entidade

```text
merchant_id = uuid5(NAMESPACE_PROJETO, f"{seed}:merchant:{i}")
```

UUIDv5 é `SHA-1(namespace + texto)` — uma função da entrada, não um sorteio. O
índice `i` devolve o mesmo UUID em qualquer execução, janela, máquina ou
linguagem, sem tabela de-para persistida.

Isso satisfaz a propriedade 2 do ADR 0002 e fecha uma armadilha da regra de
escala: quando `merchants = 30·√S` vai de 30 a 388, os índices 0–29 continuam
sendo **os mesmos 30 merchants**. Um gerador que sorteasse a população inteira
de uma vez trocaria todos de lugar sem erro aparente.

A seed entra *dentro* do texto derivado: trocar a seed troca o dataset
inteiro, como o ADR 0001 promete.

### `price_tier` continua int, não vira lookup

| Tier | Share | Merchants em S=1 |
| -- | -- | -- |
| 1 | 20% | 6 |
| 2 | 40% | 12 |
| 3 | 30% | 9 |
| 4 | 10% | 3 |

É a única exceção à normalização, e o motivo é a diferença entre escalas:

| | `cuisine_type` | `price_tier` |
| -- | -- | -- |
| Escala | **nominal** — sem ordem | **ordinal** — 1 < 2 < 3 < 4 |
| Conjunto | cresce com o tempo | fechado, nunca cresce |
| O número significa algo? | não, é só um id | sim — comparável e agregável |

Para `cuisine_type` o lookup é claramente certo: adiciona-se linha sem mexer
no schema, e o rótulo tem onde morar. Para `price_tier` o lookup existiria só
para guardar um rótulo cosmético (`$` a `$$$$`) e cobraria um `JOIN` de quem
escreve `WHERE price_tier >= 3` sem precisar dele. Pelo teste operacional do
[0004](0004-terceira-porta-do-criterio.md), lookup de escala ordinal fechada
não exercita nada que `cuisine_types` já não exercite.

`price_tier` é **independente** da taxa-base lognormal do ADR 0001 — ver
"Simplificações assumidas".

### `created_at`

Uniforme entre 2023-01-01 e 2026-03-31. Nenhum merchant é cadastrado durante a
janela: o ADR 0002 separou os papéis, e merchants são a população estável
entre janelas, enquanto customers é a que cresce.

### Regras de borda — as duas andam juntas

**Os 2 merchants com zero pedidos ficam `active`.** O ADR 0001 pagou por eles
para ter o caso de `LEFT JOIN` sem correspondência e o de dimensão sem fato.
Marcá-los `inactive` faria o `WHERE status = 'active'` que qualquer analista
escreve apagar exatamente o caso de borda comprado.

**Os 2 desativados no meio da janela são sorteados fora do top 20% por
taxa-base.** Truncar a janela ativa de um merchant grande deslocaria os alvos
de skew do ADR 0001 — que são asserção de teste, não comentário.

Desativação: data em 2026-05, derivada de `seed + índice`. Os pedidos param
ali; a **data não é persistida** — o snapshot guarda só o `status` atual. A
perda é deliberada (ver Consequências).

Em S=1: 26 ativos a janela inteira + 2 truncados em maio + 2 com zero pedidos
= 30.

### Integridade referencial

A fonte garante que não existe FK órfã: nenhum merchant aponta para bairro
inexistente, nenhum pedido para cliente inexistente. Num banco transacional
isso é o `FOREIGN KEY` recusando o insert.

**É aqui que mora a lição que a fonte achatada não oferecia.** No momento em
que esse dado vira arquivo num lake, nada garante mais nada — e descobrir o
que a fonte assegurava e o formato de pouso deixou de assegurar é assunto
central de engenharia de dados, disponível já no degrau 1.

## Alternativas consideradas e rejeitadas

**Manter achatado e normalizar depois, como schema drift do degrau 5.**
Tentadora: zero retrabalho hoje, e "fonte que evolui de achatada para
normalizada" é evolução de schema realista — a definição de domínio já lista
"campo renomeado" naquele degrau. Rejeitada porque o degrau 1 ficaria sem
integridade referencial e sem denormalização na curated, que são as duas
lições que motivaram a mudança. *A evolução da fonte continua disponível como
exercício futuro* — só não como substituto deste.

**Normalizar só `customers`, deixando `merchants` achatado.** Evitaria mexer
no 0003 hoje, e produziria o pior dos dois mundos: duas entidades
representando bairro de formas diferentes, com o gerador mantendo as duas.

**Lookups com chave UUID, por uniformidade com as entidades.** Rejeitada:
dado de referência não é população derivada de seed, e um UUID para 15 bairros
encareceria todo `JOIN` da escada para comprar consistência cosmética.

**`price_tiers` como tabela.** Ver acima.

**Endereço textual (rua, número, CEP).** Terceira representação do mesmo
ponto. CEP é especialmente tentador no Brasil — é a chave de junção natural, e
o dataset Olist é geolocalizado por ele. Fica de fora pelo mesmo argumento que
tirou `h3_cell`: chave espacial é o que o degrau 4 existe para calcular, não
para receber pronta.

**Cardápio / `menu_items` no degrau 1.** Sem `order_items` — pergunta em
aberto do KRA-27 — o menu é dimensão que ninguém referencia. E o dilema não
tem saída boa: menu estático é mentira, menu versionado é o degrau 3. O que o
cardápio ancoraria — variação de ticket entre merchants — `price_tier` ancora
com uma coluna.

**`avg_prep_time_minutes`.** O caso limítrofe mais instrutivo: parece assento
reservado do degrau 2 (duração entre eventos de status), mas no degrau 1 **não
existe duração nenhuma para invalidar**. Retrofit barato ⇒ fora.

**`h3_cell` / `geohash` pré-computados.** Calcular a chave de partição
espacial *é* a lição do degrau 4.

**Chave surrogate `merchant_sk`.** No degrau 3 a PK vira
`(merchant_id, valid_from)` ou um surrogate de versão. Criá-lo agora
resolveria de graça uma migração cuja dor é a lição.

**`commission_rate` e economia de marketplace.** Nenhum degrau depende.
Realismo puro, reprovado também pela porta 3: não exercita propriedade
transacional nenhuma.

**Múltiplas cidades.** Quebraria a calibração SP-específica do degrau 4 e a
única lição extra seria um `GROUP BY` mais largo. `cities` existe com uma
linha para que a hierarquia seja explícita, não para ser povoada agora.

**IDs sequenciais ou `MER-0001` para entidades.** O inteiro convida a tratar
ordem como informação, e a largura do padding depende de `S` (30 agora, 388 no
degrau 4).

## Simplificações assumidas

Registradas porque são onde este modelo é atacável.

- **A integridade referencial é propriedade do dado, não do formato.** A fonte
  é entregue como arquivos (KRA-28), e arquivo não tem `FOREIGN KEY`. O
  gerador garante a integridade **por construção**; nada no pouso a impõe.
  Isso é honesto quanto ao que o degrau 1 realmente tem — e é exatamente por
  isso que o pipeline não pode confiar nela sem verificar.
- **`cities` tem uma linha só.** Vale a hierarquia explícita
  (cidade > bairro > ponto), que é o que vira conversa de particionamento
  depois. Mas é, hoje, uma tabela de um valor.
- **Centróides são aproximados.** Servem para distância relativa, não para
  geocodificação.
- **`price_tier` independe do volume.** No mundo real caro e barato
  correlacionam com volume. Introduzir a correlação perturbaria os alvos de
  skew do ADR 0001, que são gabarito verificável. Realismo não paga embaçar
  ground truth.
- **`created_at` uniforme, não crescente.** Um marketplace real cadastra mais
  merchants a cada ano. Nada no degrau 1 lê essa curva.
- **Sem constraint de unicidade em `merchants.name`.** Deliberado — ver
  Consequências.

## Consequências

- **O 0003 fica `supersedida por 0004 e 0005`.** Duas decisões distintas
  saíram dele: o critério (0004) e a forma da fonte (0005).
- **KRA-25 reabre** para receber a descrição correspondente a este registro.
- **O modelo não responde "qual era o `price_tier` quando o pedido X foi
  feito?"** — nem "o merchant estava ativo em 12 de abril?", embora o próprio
  dado mostre pedido naquela data. Não é contradição no dado, é limitação do
  modelo, e é a pressão que justifica SCD2 no degrau 3.
- **Nome não é chave.** `name` pode repetir. Qualquer junção por nome é bug, e
  o modelo é assim de propósito.
- **KRA-26 herda** o vocabulário de bairros via FK, a mesma representação de
  localização (ponto como ground truth, bairro como referência) e a derivação
  de chave por UUIDv5.
- **KRA-27 herda** `merchant_id` e `customer_id` como FKs, `price_tier` como
  variável que condiciona o valor do pedido, e `delivery_neighborhood_id` como
  FK para `neighborhoods`. `order_items` segue em aberto lá.
- **KRA-28 herda** seis tabelas com **cadências diferentes**: as de referência
  praticamente não mudam, `orders` muda todo dia. Full refresh por tabela
  deixa de ser decisão única e vira decisão por tabela no degrau 2. `merchants`
  continua tabela pequena (30 em S=1, 388 no degrau 4) e **não** precisa de
  particionamento por data.
- **KRA-29 herda** cinco obrigações: gerar as seis tabelas; derivar
  `merchant_id` por UUIDv5 sobre `seed + índice`; garantir em teste que o
  merchant do índice `i` é idêntico para qualquer `S`; ler os centróides de
  `neighborhoods` em vez de embuti-los; e **garantir integridade referencial
  por construção**, com teste que verifique ausência de FK órfã. Seguem
  valendo: renormalizar as taxas-base após truncar os desativados, e
  reverificar os alvos de skew do ADR 0001 **depois** do truncamento.
- **KRA-30 ganha conteúdo real**: resolver os lookups é a denormalização que
  produz o star schema. A escolha de forma do warehouse (star, snowflake,
  Inmon, Data Vault) é decisão daquela issue — esta decide só a fonte.
- **A issue de modelagem de `payments` nasce sob esta convenção** e traz sua
  própria tabela de referência (`payment_methods`), decidida lá e não aqui.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede. É o que vai acontecer no degrau 3, quando `merchants` virar
  SCD2.
