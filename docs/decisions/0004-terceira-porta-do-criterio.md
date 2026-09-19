# 0004 — Terceira porta do critério de inclusão

- **Status**: aceita
- **Data**: 2026-09-19
- **Issue**: [KRA-26](https://linear.app/krasinski-projects/issue/KRA-26/modelar-customers) (onde a questão apareceu)
- **Supersede**: [0003](0003-modelo-de-merchants.md), junto com [0005](0005-fonte-normalizada.md)
- **Restringe**: [KRA-25](https://linear.app/krasinski-projects/issue/KRA-25/modelar-merchants), [KRA-26](https://linear.app/krasinski-projects/issue/KRA-26/modelar-customers), [KRA-27](https://linear.app/krasinski-projects/issue/KRA-27/modelar-orders), e toda modelagem seguinte
- **Raciocínio completo**: [Modelo de Dados — Degrau 1](https://linear.app/krasinski-projects/document/modelo-de-dados-degrau-1-37cb769afc21)
- **Convenção de ADR**: ver [0001](0001-parametros-dataset-degrau-1.md)

## Contexto

O [ADR 0003](0003-modelo-de-merchants.md) fixou um critério de duas portas
para decidir o que entra num modelo, e bastava uma:

1. alguma pergunta do degrau atual precisa da coluna; ou
2. a ausência hoje **corrompe** o que já existe, e não apenas adia trabalho.

Fechando com a frase que fazia o trabalho pesado: *realismo sozinho não é
porta de entrada.*

Esse critério foi escrito supondo um propósito único — aprender arquitetura
de dados pela pressão que o volume cria. Durante o KRA-26 apareceu um
segundo, declarado explicitamente: **aprender modelagem transacional**. É
território que normalmente pertence ao time de aplicação, e o dono do
repositório não trabalhou com ele; o projeto passa a servir também para isso.

Com duas portas, toda estrutura transacional teria que ser espremida numa
justificativa de degrau — "o KRA-30 precisa resolver o lookup" — mesmo quando
o motivo honesto é outro. Justificativa construída a posteriori é a forma
educada de mentir no registro, e um registro assim não sobrevive a leitura
atenta daqui a seis meses.

## Decisão

Abre-se uma terceira porta, e continua bastando uma:

3. **estrutura que exercita uma propriedade do modelo transacional** — chave
   estrangeira, integridade referencial, forma normal, anomalia de
   atualização, ou grão de entidade com ciclo de vida próprio.

### O limite, que é o que torna a porta utilizável

A porta 3 autoriza **estrutura**, não **atributo**.

| Passa | Não passa |
| -- | -- |
| `cuisine_type` virando tabela com FK | `loyalty_tier` |
| `payments` como entidade 1—N, não coluna em `orders` | `birth_date`, telefone |
| KYC como tabela de eventos de verificação | canal de aquisição |

**Teste operacional**: se a justificativa da inclusão puder ser escrita sem
citar uma propriedade do modelo transacional, a porta 3 não se aplica.

Sem esse limite a porta engole as outras duas — realismo passaria a justificar
qualquer coisa, e o princípio central do projeto (*cada ferramenta conquistada
pela pressão que o dado cria*) viraria letra morta.

## Alternativas consideradas e rejeitadas

**Manter duas portas e justificar a normalização pela porta 1.** Funciona para
este caso: o KRA-30 realmente precisa de lookups para resolver. Falha no
próximo, e no seguinte — cada estrutura transacional futura precisaria de uma
justificativa de degrau montada depois do fato. Rejeitada por corroer
justamente o que o critério existe para proteger.

**Abrir a porta sem limite**, adotando realismo como justificativa suficiente,
sob o argumento de que o dataset também é ativo reutilizável para análises e
modelos futuros. Rejeitada: sem limite entram `birth_date`, telefone,
`loyalty_tier` e o resto, e o projeto vira tutorial com dado bonito.

## Ressalva honesta

**A porta 3 é mais frouxa que as outras duas, e isso não tem conserto.**
Portas 1 e 2 são verificáveis contra o dado — ou existe pergunta que precisa
da coluna, ou existe algo que a ausência corrompe. A porta 3 depende de julgar
se algo "exercita uma propriedade", e essa fronteira tem casos difíceis.

O caso concreto já encontrado é `price_tier`: é valor categórico como
`cuisine_type`, e pela leitura preguiçosa da porta 3 viraria lookup também.
Não vira — escala ordinal fechada não exercita nada que `cuisine_types` já não
exercite (ver [0005](0005-fonte-normalizada.md)). Esse julgamento **não é
dedutível da regra**; é aplicação dela, e vai precisar ser refeito a cada
caso. Quem vier depois deve atacar por aqui.

## O que esta decisão não faz

O segundo propósito completo — um dataset realista reutilizável para análises
e modelos **fora** da escada — **não** está adotado. A porta 3 cobre apenas a
parte estrutural. Atributos puramente realistas seguem fora, e adotar aquele
propósito seria decisão própria, com ADR próprio, porque mudaria o critério de
todas as modelagens seguintes.

## Consequências

- **KRA-25 reabre.** `merchants` foi modelado sob duas portas; o
  [0005](0005-fonte-normalizada.md) o restabelece sob três.
- O 0005 é a primeira decisão tomada sob a porta 3, e serve de precedente
  de como ela se aplica.
- **CPF/documento continua fora por motivo independente da porta**: seria a
  chave de deduplicação perfeita, e mataria no nascimento a lição que as
  duplicatas de cadastro do [0006](0006-modelo-de-customers.md) existem para
  criar.
- **KYC, quando entrar, entra por aqui** — como tabelas com grão de evento
  (uma verificação é tentativa datada, não atributo do cliente), nunca como
  coluna em `customers`. Adicionar tabela é barato em qualquer degrau;
  adicionar coluna a fato já gerado não é.
- Mudar este critério num degrau novo **não** edita este arquivo: escreve um
  ADR que o supersede.
