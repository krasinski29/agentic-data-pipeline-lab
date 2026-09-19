# 0003 — Modelo de `merchants`

- **Status**: supersedida por [0004](0004-terceira-porta-do-criterio.md) e [0005](0005-fonte-normalizada.md)
- **Data**: 2026-09-16
- **Issue**: [KRA-25](https://linear.app/krasinski-projects/issue/KRA-25/modelar-merchants)
- **Restringe**: [KRA-26](https://linear.app/krasinski-projects/issue/KRA-26/modelar-customers), [KRA-27](https://linear.app/krasinski-projects/issue/KRA-27/modelar-orders), [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

O [ADR 0001](0001-parametros-dataset-degrau-1.md) fixou *quantos* merchants
existem (30 em S=1, `30·√S` em geral) e como os pedidos se distribuem entre
eles. Este fixa **o que cada merchant é**: colunas, tipos, chave e
relacionamentos. Aqui ficam os valores normativos que o gerador (KRA-29)
tem que respeitar; o documento no Linear guarda o porquê de cada um.

## Contexto

Modelar uma dimensão tem uma tentação óbvia: adicionar coluna porque um
marketplace real teria. Na modelagem isso é o equivalente a instalar
lakehouse no degrau 1 — parece maduro e não ensina nada.

O critério adotado tem duas portas, e basta uma:

1. alguma pergunta do degrau 1 precisa da coluna; ou
2. é assento reservado cuja **ausência hoje corrompe** o que já existe,
   e não apenas adia trabalho.

Realismo sozinho não é porta de entrada.

Há ainda uma restrição herdada que molda a chave: o
[ADR 0002](0002-modelo-de-replay-e-geracao.md) exige entidades derivadas de
`seed + índice`, estáveis entre janelas.

## Decisão

**Grão**: 1 linha por merchant, estado atual, **sem versionamento**.
**Chave primária**: `merchant_id`.

### Esquema

| Coluna          | Tipo                  | Nulo? | Papel                                                |
| --------------- | --------------------- | ----- | ---------------------------------------------------- |
| `merchant_id`   | uuid                  | não   | chave natural da fonte                               |
| `name`          | text                  | não   | rótulo humano — **não é único**                      |
| `cuisine_type`  | text (enum, 8)        | não   | dimensão de agregação do degrau 1                    |
| `price_tier`    | int (1–4)             | não   | condiciona o ticket em KRA-27; atributo SCD2 no degrau 3 |
| `city`          | text                  | não   | constante hoje: `São Paulo`                          |
| `neighborhood`  | text (enum, 15)       | não   | chave de agregação legível                           |
| `latitude`      | double                | não   | origem da corrida — degrau 4                         |
| `longitude`     | double                | não   | idem                                                 |
| `created_at`    | timestamp (UTC)       | não   | cadastro, sempre anterior a 2026-04-01               |
| `status`        | text (`active`/`inactive`) | não | estado atual, sem histórico                      |

**Relacionamento**: `merchants 1 —— N orders` via `orders.merchant_id`.
Nenhum vínculo com `customers`.

### Derivação da chave

```text
merchant_id = uuid5(NAMESPACE_PROJETO, f"{seed}:merchant:{i}")
```

UUIDv5 é `SHA-1(namespace + texto)` — uma função da entrada, não um
sorteio. O índice `i` devolve o mesmo UUID em qualquer execução, janela,
máquina ou linguagem, sem tabela de-para persistida.

Isso satisfaz a propriedade 2 do ADR 0002 e fecha uma armadilha da regra
de escala: quando `merchants = 30·√S` vai de 30 a 388, os índices 0–29
continuam sendo **os mesmos 30 merchants**. Um gerador que sorteasse a
população inteira de uma vez trocaria todos de lugar sem erro aparente.

A seed entra *dentro* do texto derivado: trocar a seed troca o dataset
inteiro, como o ADR 0001 promete.

### Vocabulário de `cuisine_type`

`brasileira`, `hamburguer`, `pizza`, `japonesa`, `italiana`, `saudavel`,
`doces_sobremesas`, `arabe`.

### Vocabulário de `neighborhood` e centróides

Uma cidade só — São Paulo. O ADR 0001 já gera em `America/Sao_Paulo`, e
as fontes de calibração do degrau 4 (Pesquisa OD do Metrô SP, API Olho
Vivo da SPTrans) são SP-específicas.

Coordenadas são centróides **aproximados**, não pontos levantados — a
precisão que importa é a estrutura de distâncias relativas, não o
endereço.

| Bairro           | lat       | long      |
| ---------------- | --------- | --------- |
| Pinheiros        | -23,5665  | -46,7020  |
| Vila Madalena    | -23,5540  | -46,6900  |
| Itaim Bibi       | -23,5860  | -46,6790  |
| Moema            | -23,6020  | -46,6650  |
| Jardim Paulista  | -23,5680  | -46,6620  |
| Vila Mariana     | -23,5890  | -46,6340  |
| Perdizes         | -23,5370  | -46,6790  |
| Santana          | -23,5020  | -46,6250  |
| Tatuapé          | -23,5400  | -46,5760  |
| Lapa             | -23,5230  | -46,7040  |
| Butantã          | -23,5710  | -46,7200  |
| Bela Vista       | -23,5590  | -46,6450  |
| Consolação       | -23,5540  | -46,6600  |
| Saúde            | -23,6180  | -46,6390  |
| Ipiranga         | -23,5920  | -46,6100  |

O ponto de cada merchant sai de um sorteio num raio de **cerca de 800 m**
do centróide do seu bairro, seeded pelo índice. O objetivo é *clustering*:
Pinheiros → Pinheiros é entrega curta, Pinheiros → Tatuapé é longa.
Coordenada uniforme sobre a cidade não teria essa estrutura, e a lição de
particionamento espacial do degrau 4 ficaria artificial.

### Distribuição de `price_tier`

| Tier | Share | Merchants em S=1 |
| ---- | ----- | ---------------- |
| 1    | 20%   | 6                |
| 2    | 40%   | 12               |
| 3    | 30%   | 9                |
| 4    | 10%   | 3                |

`price_tier` é **independente** da taxa-base lognormal do ADR 0001 — ver
"Simplificações assumidas".

### `created_at`

Uniforme entre 2023-01-01 e 2026-03-31. Nenhum merchant é cadastrado
durante a janela: o ADR 0002 separou os papéis, e merchants são a
população estável entre janelas, enquanto customers é a que cresce.

### Regras de borda — as duas andam juntas

**Os 2 merchants com zero pedidos ficam `active`.** O ADR 0001 pagou por
eles para ter o caso de `LEFT JOIN` sem correspondência e o de dimensão
sem fato. Marcá-los `inactive` faria o `WHERE status = 'active'` que
qualquer analista escreve apagar exatamente o caso de borda comprado.

**Os 2 desativados no meio da janela são sorteados fora do top 20% por
taxa-base.** Truncar a janela ativa de um merchant grande deslocaria os
alvos de skew do ADR 0001 — que são asserção de teste, não comentário.

Desativação: data em 2026-05, derivada de `seed + índice`. Os pedidos
param ali; a **data não é persistida** — o snapshot guarda só o `status`
atual. A perda é deliberada (ver Consequências).

Em S=1: 26 ativos a janela inteira + 2 truncados em maio + 2 com zero
pedidos = 30.

## Alternativas consideradas e rejeitadas

**Cardápio / `menu_items` no degrau 1.** Sem `order_items` — pergunta em
aberto do KRA-27, não decidida aqui — o menu é dimensão que ninguém
referencia. E o dilema não tem saída boa: menu estático é mentira, menu
versionado é o que o degrau 3 existe para ensinar. O que o cardápio
ancoraria — variação de ticket entre merchants — `price_tier` ancora com
uma coluna.

**`avg_prep_time_minutes`.** O caso limítrofe mais instrutivo: parece
assento reservado do degrau 2 (duração entre eventos de status), mas no
degrau 1 **não existe duração nenhuma para invalidar**. Adicioná-lo no
degrau 2 não contradiz dado anterior. Retrofit barato ⇒ fora. O teste não
é "vai ser usado depois?" (quase tudo vai), é "a ausência hoje corrompe o
que já existe?".

**`h3_cell` / `geohash` pré-computados.** Calcular a chave de partição
espacial *é* a lição do degrau 4. Pré-computar é instalar a ferramenta
antes da pressão — o que o princípio central do projeto proíbe.

**Chave surrogate `merchant_sk`.** No degrau 3 a PK vira
`(merchant_id, valid_from)` ou um surrogate de versão. Criá-lo agora
resolveria de graça uma migração cuja dor é a lição.

**`commission_rate` e economia de marketplace.** Nenhum degrau da escada
depende. Realismo puro.

**Múltiplas cidades.** Quebraria a calibração SP-específica do degrau 4 e
a única lição extra seria um `GROUP BY` mais largo.

**IDs sequenciais ou `MER-0001`.** Legíveis, mas o inteiro convida a
tratar ordem como informação, e a largura do padding de `MER-000N` depende
de `S` (30 agora, 388 no degrau 4) — ou sobra zero desde já, ou o id muda
de forma ao escalar.

## Simplificações assumidas

Registradas porque são onde este modelo é atacável, e quem vier depois
merece saber por onde.

- **`price_tier` independe do volume.** No mundo real caro e barato
  correlacionam com volume. Introduzir a correlação perturbaria os alvos
  de skew do ADR 0001, que são gabarito verificável. Realismo não paga
  embaçar ground truth.
- **`created_at` uniforme, não crescente.** Um marketplace real cadastra
  mais merchants a cada ano. Nada no degrau 1 lê essa curva.
- **`city` é coluna constante.** Vale a hierarquia explícita
  (cidade > bairro > ponto), que é o que vira conversa de particionamento
  depois. Mas é, hoje, uma coluna de um valor só.
- **Centróides são aproximados.** Servem para distância relativa, não para
  geocodificação.

## Consequências

- **O modelo não responde "qual era o `price_tier` quando o pedido X foi
  feito?"** — nem "o merchant estava ativo em 12 de abril?", embora o
  próprio dado mostre pedido naquela data. Não é contradição no dado, é
  limitação do modelo, e é a pressão que justifica SCD2 no degrau 3.
- **KRA-26 herda** o vocabulário de 15 bairros e a mesma representação de
  localização (ponto como ground truth, bairro denormalizado como rótulo).
  Cliente e merchant precisam viver no mesmo espaço, ou distância de
  entrega não existe.
- **KRA-27 herda** `merchant_id` como FK e `price_tier` como variável que
  condiciona o valor do pedido. `order_items` segue em aberto lá.
- **KRA-28 herda** que `merchants` é tabela pequena (30 em S=1, 388 no
  degrau 4) e **não** precisa de particionamento por data, diferente de
  `orders`. Formato único para as duas é escolha a justificar, não default.
- **KRA-29 herda** quatro obrigações: derivar `merchant_id` por UUIDv5
  sobre `seed + índice`; garantir em teste que o merchant do índice `i` é
  idêntico para qualquer `S`; renormalizar as taxas-base após truncar os
  desativados, para o total fechar em 10.000; e **reverificar os alvos de
  skew do ADR 0001 depois do truncamento**, não antes.
- **Nome não é chave.** `name` pode repetir. Qualquer junção por nome é
  bug, e o modelo é assim de propósito.
- Mudar este modelo num degrau novo **não** edita este arquivo: escreve um
  ADR que o supersede. É o que vai acontecer no degrau 3, quando
  `merchants` virar SCD2.
