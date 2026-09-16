# 0002 — Modelo de replay e frequência de geração

- **Status**: aceita
- **Data**: 2026-09-16
- **Issue**: [KRA-24](https://linear.app/krasinski-projects/issue/KRA-24/definir-parametros-do-dataset-inicial) (onde a questão apareceu)
- **Restringe**: [KRA-28](https://linear.app/krasinski-projects/issue/KRA-28/decidir-formato-e-local-de-pouso-do-dado-bruto), [KRA-29](https://linear.app/krasinski-projects/issue/KRA-29/construir-o-gerador-do-degrau-1)
- **Raciocínio completo**: [Parâmetros do Dataset — Degrau 1](https://linear.app/krasinski-projects/document/parametros-do-dataset-degrau-1-c5e2974a425d)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

O [ADR 0001](0001-parametros-dataset-degrau-1.md) decidiu *quais são os
números* do dataset. Este decide outra coisa: **com que frequência o
gerador roda, e como o dado "chega" no pipeline**. São perguntas
separadas, então são registros separados.

## Contexto

O 0001 fixou uma janela estática (2026-04-01 a 2026-06-30) e uma seed
fixa. Isso deixa uma pergunta em aberto que ninguém tinha feito: o
gerador roda uma vez e produz um dump morto, ou roda periodicamente
produzindo dado novo?

A pergunta importa porque a escada inteira depende de dado *chegando*:
carga incremental e idempotência no degrau 2, backfill e correção
retroativa no degrau 3, watermark e dado fora de ordem no degrau 6.
Nenhuma dessas lições existe sobre um arquivo parado.

E ela traz uma preocupação legítima junto: uma janela estática tem fim,
então a camada analítica "acabaria" em 30 de junho.

## Decisão

### Três relógios, independentes

|                                                          | degrau 1        | degrau 2+ |
| -------------------------------------------------------- | --------------- | --------- |
| **Tempo do evento** — quando o pedido aconteceu          | 01/04 a 30/06   | o mesmo   |
| **Tempo de geração** — quando o gerador roda             | uma vez         | uma vez   |
| **Tempo de chegada** — quando o dado aparece no pipeline | tudo de uma vez | em fatias |

A confusão que este ADR resolve é tratar os três como se fossem um só.

### O gerador roda uma vez

Janela fixa e seed fixa: re-executar produz exatamente o mesmo dado.
Re-executar é *rebuild*, não dado novo — só faz sentido quando um
parâmetro muda ou um bug do gerador é corrigido.

### O dado "novo" vem de replay, não de geração

A partir do degrau 2, dado novo **não** vem de rodar o gerador contra a
data de hoje. Vem de reproduzir a janela fixa em fatias, recortando por
data de evento um dataset que já existe inteiro.

**Cursor de replay** é a variável que guarda até que dia do dataset o
pipeline já processou. Não é tecnologia, é um marcador de página:

```text
execução 1  →  lê 2026-04-01  →  cursor = 2026-04-01
execução 2  →  lê 2026-04-02  →  cursor = 2026-04-02
execução 3  →  lê 2026-04-03  →  cursor = 2026-04-03
```

O cursor não tem relação com a data real. Avançá-lo dez vezes numa tarde
ou uma vez por dia durante três meses é indistinguível para o pipeline:
"hoje", para ele, é onde o cursor está.

Isso não é invenção de laboratório — é o que todo orquestrador já faz. A
*logical date* de uma DAG do Airflow (`{{ ds }}`) é o cursor, e backfill
é mandar rodar datas lógicas passadas. Adotar o modelo agora antecipa o
que a ferramenta vai exigir no degrau 2, em vez de contradizê-la.

No degrau 6, streaming contínuo sai de **replay acelerado**: 1 dia de
tempo de evento por minuto de relógio de parede, e os 91 dias viram ~1h30
de stream, sem gerador ao vivo.

### O gerador nasce extensível por janela

Gerar uma janela posterior (`2026-07-01..2026-09-30`) tem que ser troca de
parâmetro, não backfill. Três propriedades, baratas agora e caras de
retrofitar:

1. **Janela como parâmetro** (início e fim), não constante embutida.
2. **Entidades estáveis entre janelas** — merchants e clientes derivam de
   `seed + índice`, não de sorteio novo a cada execução. Uma janela
   posterior devolve *os mesmos* 30 merchants, mais os clientes que
   entraram no período, nunca uma população nova.
3. **Continuidade de estado** — a tendência de +5%/mês ancorada numa data
   absoluta, não no início da janela, ou cada execução reinicia o
   crescimento e o volume dá um degrau artificial na emenda.

É isso que mantém a janela fixa reversível. Sem as três, ela vira decisão
irreversível por acidente.

## Alternativa considerada e rejeitada

**Gerador contínuo acoplado ao relógio de parede** — um cronjob que gera
dado para "ontem real", todo dia.

Parece mais realista e custaria as três coisas que sustentam a escada:

- **Ground truth** — sem a história gerada de antemão não existe gabarito
  para comparar com a saída do pipeline.
- **Reprodutibilidade** — duas execuções nunca seriam comparáveis, e todo
  diff viraria ruído em vez de sinal.
- **Controle do atraso** — "evento de terça chegando na quinta", o
  problema central dos degraus 2 e 6, só é encenável de propósito se a
  terça já está na mão quando a quinta é encenada.

Rejeitada por isso. O que ela resolveria — fluxo contínuo — o replay
acelerado já resolve.

## Consequências

- **`CURRENT_DATE` fica proibido em transformação.** A data de referência
  é parâmetro da execução, sempre. É o custo direto do modelo, e é o tipo
  de disciplina que se quer ter: transformação idempotente se parametriza
  pela data de execução, nunca pelo relógio — é o que permite reprocessar
  março em julho e obter o mesmo resultado.
- **A camada analítica não acaba.** O "hoje" do warehouse é onde o cursor
  está; "últimos 7 dias" sempre existe, em dias de dataset.
- **KRA-28 herda uma restrição**: seja qual for o formato do dado bruto,
  precisa permitir recorte por data de evento de forma barata. Isso não
  antecipa Parquet nem lakehouse — um diretório com um arquivo por dia
  resolve. Só exclui o dump monolítico único, que começaria o degrau 2
  com retrabalho.
- **KRA-29 herda as três propriedades** de extensibilidade, mais um teste
  que gere duas janelas adjacentes e verifique que as entidades batem e
  que o volume não salta na emenda.
- **O gap deixa de ser risco e vira exercício.** "A fonte parou por seis
  semanas, agora precisa de backfill e retomada incremental" é cenário
  real — melhor encenado de propósito, com gabarito, no degrau 3.
- **Fica em aberto para o degrau 5**: alinhar o tempo do dataset ao
  relógio de parede. Dataset em abril + API de clima real exige endpoint
  histórico (limitado, às vezes pago) em vez do "clima agora" (grátis). É
  a única tensão concreta identificada contra esta decisão, e a
  extensibilidade é o que mantém a porta aberta.
