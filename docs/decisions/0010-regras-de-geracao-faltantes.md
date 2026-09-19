# 0010 — Regras de geração que faltavam: merchants, referência e duplicatas

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: nenhuma — a questão apareceu na revisão do modelo v0, como no
  [0009](0009-garantias-declaradas-da-fonte.md) e no [0002](0002-modelo-de-replay-e-geracao.md)
- **Restringe**: [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1) sobretudo, e também [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto) e [KRA-30](https://linear.app/krasinski-projects/issue/KRA-30/primeira-transformacao-raw-para-curated)
- **Completa**: [0005](0005-fonte-normalizada.md) — sem supersedê-lo (ver ao final)
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

## Contexto

A revisão do modelo v0 percorreu as oito tabelas coluna a coluna perguntando
uma coisa só: **existe regra que diga como produzir este campo?** `customers`,
`orders` e `payments` passam inteiros. `merchants` tem quatro buracos, as
tabelas de referência têm um, e o [0006](0006-modelo-de-customers.md) tem uma
regra escrita com verbo modal.

O padrão dos buracos explica por que eles existem: **estão quase todos em
`merchants`**, a primeira entidade modelada — no 0003, sob duas portas, antes
de o [0004](0004-terceira-porta-do-criterio.md) abrir a terceira e de o 0005
reconstruir a tabela. O que o 0005 refez foi o *esquema*. As regras de
povoamento que o 0003 nunca teve continuaram não existindo, e o 0006 nasceu
depois já com todas as suas.

O que falta, em lista:

1. como as 8 culinárias se distribuem entre os 30 merchants;
2. como os 30 merchants se distribuem entre os 15 bairros;
3. **quais** 2 merchants ficam com zero pedidos;
4. como `merchants.name` é composto;
5. que id cada bairro e cada culinária recebe, e qual é o `state` de `cities`;
6. o que exatamente varia entre os dois cadastros de uma duplicata.

## Decisão

### Uniforme nos dois eixos, e por que não "plausível"

| Eixo | Regra |
| -- | -- |
| `cuisine_type_id` | uniforme sobre 8 culinárias: 6 com 4 merchants, 2 com 3 |
| `neighborhood_id` | uniforme sobre 15 bairros: **2 por bairro**, exatos |

A alternativa era um mix plausível — pizza e hambúrguer maiores, japonesa e
árabe menores. Foi a recomendação inicial desta revisão e **foi revertida**,
por dois motivos:

**Realismo sozinho não é porta de entrada.** É o critério do 0004, e o 0005 já
o aplicou no caso vizinho quando recusou correlacionar `price_tier` com volume.
Um share plausível seria um parâmetro novo que nenhuma pergunta do degrau 1 lê.

**Uniforme garante cobertura por construção.** Com 30 merchants e shares
desiguais, alguma culinária termina com 1 ou 2 — e, no limite, com zero. Uma
culinária sem merchant seria um terceiro caso de *dimensão sem fato* entrando
**por acidente**, quando os dois que existem (2 merchants e 500 clientes sem
pedido) foram comprados de propósito pelo ADR 0001 e são asserção de teste.
Caso de borda acidental é indistinguível de bug.

### A armadilha de implementação que a uniformidade cria

Culinária, bairro e `price_tier` são três atribuições por índice sobre a mesma
população de 30. **Elas têm que sair de derivações independentes de
`seed + índice` — nunca do mesmo `i mod k`, nem de blocos contíguos de índice.**

O risco é concreto: se a culinária for `i mod 8` e o `price_tier` sair de
blocos contíguos (índices 0–5 tier 1, 6–17 tier 2, …), os dois eixos passam a
correlacionar **por construção**, sem que nada acuse. O 0005 decidiu
explicitamente que `price_tier` é independente da taxa-base, e o mesmo vale
entre os três eixos categóricos.

A forma correta é uma permutação *seeded* por eixo: as contagens ficam exatas,
a atribuição fica determinística, e os eixos ficam independentes entre si.

### `merchants.name`

Composto de duas listas curtas indexadas por `seed + índice`, mesmo mecanismo
que o 0006 já usa para `customers` — um termo de estabelecimento e um núcleo,
sem dependência nova.

**E um par é forçado a colidir.** Dois merchants recebem exatamente o mesmo
`name`, escolhidos por `seed` entre os 30, sem restrição de tamanho: o par
cair num merchant grande ou pequeno muda o *tamanho* do erro que uma junção por
nome comete, não a existência dele.

Forçar é o ponto. O 0005 diz que `name` repete de propósito e que "qualquer
junção por nome é bug"; o [0009](0009-garantias-declaradas-da-fonte.md) declara
`merchants.name` explicitamente não-único. Com 30 merchants sorteados de duas
listas curtas, a colisão **provavelmente não acontece** — e a armadilha ficaria
decorativa, com uma não-unicidade declarada e nenhum caso que a exercite. É o
mesmo movimento dos 2 merchants sem pedido: caso de borda comprado, não
esperado.

**Qual é o par não é persistido.** Mesma decisão que o 0005 tomou para a data de
desativação: é derivável de `seed + índice`, então o teste recalcula. Persistir
seria entregar a resposta dentro da prova, como uma coluna `same_person_as`
faria com as duplicatas de cliente.

### Quais 2 merchants ficam com zero pedidos

**Sorteados fora do top 20% por taxa-base** — a mesma regra que o 0005 escreveu
para os 2 desativados, e que ele não estendeu a estes.

O argumento é *mais* forte aqui, não menos: truncar a janela de um merchant
grande remove parte da massa dele; zerá-lo remove toda. Se um dos dois sem
pedido saísse do top 20%, os alvos de skew do ADR 0001 — top 20% em 55–60%,
maior merchant ≤ 15% — sairiam do lugar antes mesmo da renormalização.

**Os quatro merchants são distintos**: 2 com zero pedidos + 2 desativados em
maio + 26 ativos a janela inteira = 30, como o 0005 já contava.

A ordem das operações sobre a taxa-base fica explícita, porque é onde o erro
mora:

```text
taxa-base lognormal por merchant (sigma ~ 1,0)
  -> zerar a taxa dos 2 sem pedido        (fora do top 20%)
  -> truncar a janela dos 2 desativados   (fora do top 20%)
  -> renormalizar para o total fechar em 10.000·S
  -> reverificar os alvos de skew DEPOIS das tres operacoes, nunca antes
```

### Ids de referência e `cities.state`

| Tabela | Ids |
| -- | -- |
| `cities` | `1` = São Paulo, `state = "SP"` |
| `neighborhoods` | `1..15`, na ordem da tabela do 0005 — Pinheiros = 1 … Ipiranga = 15 |
| `cuisine_types` | `1..8`, na ordem listada no 0005 — `brasileira` = 1 … `arabe` = 8 |
| `payment_methods` | já explícitos no [0008](0008-modelo-de-payments.md) |

O 0005 promete que `neighborhood_id = 1..15` é o mesmo em qualquer `S`, qualquer
seed, qualquer execução e qualquer linguagem. **Essa promessa não pode depender
da ordem em que alguém digitou uma tabela em Markdown.** Ela vale porque a
config é a origem, e é a config que este registro fixa como normativa — o
arquivo de configuração do KRA-29 carrega o mapeamento id → nome, e o ADR é o
que ele materializa.

### Nomenclatura de coordenada

`centroid_lat` e `centroid_long` passam a `centroid_latitude` e
`centroid_longitude`. A regra é **não abreviar**; as outras seis colunas de
coordenada já estão certas.

Custo de adiar: o nome da coluna entra no arquivo no KRA-28 e passa a ser lido
pelo gerador, pelo pipeline e por toda consulta escrita depois. Renomear hoje é
uma linha de config; renomear depois do pouso é migração.

### O que varia entre os dois cadastros de uma duplicata

O 0006 descreve as variações qualitativamente e usa um modal — "localização
**pode** diferir" —, a única regra do modelo inteiro escrita assim. Vira
determinística, indexada pelo par `p` (0..59):

| Campo | Regra |
| -- | -- |
| `name` | variação `p mod 3`: acento removido / sobrenome abreviado / caixa trocada |
| `email` | variação `p mod 3`: caixa trocada / ponto inserido antes do `@` / provedor diferente |
| `created_at` | o segundo cadastro é **posterior** ao primeiro, em ponto válido da janela do 0006 |
| localização | difere nos 30 pares com `p` par; nos outros 30, o segundo cadastro copia bairro e ponto do primeiro |

Os 30 pares que mudam de endereço e os 30 que não mudam dão ao dedup do degrau
3 os dois casos que ele precisa distinguir — mesma pessoa no mesmo lugar, e
mesma pessoa que mudou — sem que localização vire chave de nada.

## Alternativas consideradas e rejeitadas

**Shares plausíveis de culinária.** Ver acima. Continua disponível como
calibração no degrau 4, junto com as outras distribuições, se alguma pergunta
passar a lê-la — é exatamente o que "distribuição é parâmetro injetável"
compra.

**Merchants concentrados em bairro comercial.** O realista: Itaim e Pinheiros
com mais restaurantes que Saúde. Rejeitada pelo mesmo critério, com um
agravante — mexeria na estrutura de distâncias que o degrau 4 vai calibrar
contra a Pesquisa OD do Metrô SP, e essa calibração ainda não existe para
comparar. Distorcer antes de ter contra o que conferir é distorcer às cegas.

**Deixar o nome repetido ao acaso das listas.** Estatisticamente mais honesto,
nada plantado. Rejeitada porque com 30 merchants a colisão provavelmente não
ocorre, e a não-unicidade que o 0009 declara ficaria sem caso de teste no
degrau 1 inteiro.

**Persistir qual é o par de nomes colididos.** Derivável de `seed + índice`, e
persistir entregaria a resposta dentro da prova.

**Ids de referência em ordem alfabética.** Igualmente estável, e obrigaria a
reescrever a tabela de um registro congelado para que documento e config
concordem. A ordem do 0005 já está publicada.

**Manter `centroid_lat`/`centroid_long`.** Duas letras a menos numa coluna que
ninguém digita à mão, contra três grafias diferentes para a mesma coisa em oito
tabelas.

**Sortear a distribuição de bairro em vez de fixar 2 por bairro.** Mais
realista na variância e abre a porta para um bairro ficar sem merchant nenhum —
o caso acidental que a uniformidade existe para impedir.

## Simplificações assumidas

- **Uniformidade perfeita nos dois eixos** é menos realista que qualquer
  sorteio. É o preço de garantir cobertura por construção, e está na mesma
  família da distribuição uniforme de clientes entre bairros do 0006.
- **As listas de nome são curtas, então a colisão deixa de ser exceção quando
  `S` cresce.** No degrau 4 são 388 merchants sobre as mesmas duas listas: o que
  aqui é um par forçado, lá vira ruído de fundo. É aceitável porque `name` nunca
  é chave — mas o par forçado deixa de ser o caso interessante.
- **A variação de grafia das duplicatas é categórica e balanceada** — 20 pares
  de cada tipo. Dedup real enfrenta variação contínua e enviesada, então o
  exercício do degrau 3 é mais fácil que a realidade.
- **`cities.state` é uma coluna de um valor**, como a própria tabela. Existe
  pela hierarquia explícita que o 0005 decidiu, não por ser lida.

## Consequências

- **KRA-29 herda sete regras concretas**: uniformidade nos dois eixos com
  permutação *seeded* independente por eixo; a ordem zerar → truncar →
  renormalizar → reverificar; os 2 sem pedido fora do top 20%; os quatro
  merchants especiais distintos entre si; a composição de `name` com um par
  forçado a colidir, não persistido; o mapeamento id → nome das quatro tabelas
  de referência na config; e a tabela determinística de variação das 60
  duplicatas.
- **KRA-28 herda uma troca de nome de coluna** (`centroid_latitude`,
  `centroid_longitude`) e o motivo de ela vir antes daquela issue: o nome entra
  no arquivo e renomear depois vira migração.
- **KRA-30 herda dois fatos de agregação**: cada culinária tem 3 ou 4 merchants
  e cada bairro tem 2, então nenhuma agregação categórica do degrau 1 encontra
  grupo vazio por acidente; e existe **um par de merchants com o mesmo nome**,
  que é o caso concreto que torna testável o "junção por nome é bug" do 0005.
- **O degrau 4 herda um botão**: trocar uniforme por calibrado, nos dois eixos,
  é mudança de config mais *rebuild* — nunca reescrita do gerador.
- **Este registro completa o 0005 sem supersedê-lo.** O 0005 decide o esquema e
  os vocabulários; este decide como a população é povoada. São perguntas
  diferentes, como o 0006 e o 0007 também são em relação a ele. Se o degrau 3
  mudar o esquema de `merchants`, supersede o 0005; se mudar as regras de
  povoamento, supersede este.
- Mudar estas regras num degrau novo **não** edita este arquivo: escreve um ADR
  que o supersede.
