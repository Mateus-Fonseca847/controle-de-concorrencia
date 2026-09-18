# Controle de concorrência — guia para iniciantes

Material de estudo do projeto de Banco de Dados — tema: controle de concorrência.

## Como usar este guia

Este guia ensina controle de concorrência do zero, supondo que você nunca viu o assunto. Todo termo técnico é explicado na primeira vez que aparece, e o termo em inglês vem entre parênteses, porque é assim que ele aparece nos livros e na documentação.

Leia na ordem. Os temas 1 e 2 são a base; o tema 3 é o critério que define o que é "certo"; os temas 4, 5 e 6 são as soluções. Cada tema termina com uma frase que resume o que você precisa levar para o próximo.

O alvo final é o seu artigo no padrão SBC sobre controle de concorrência. Os temas 1 a 3 alimentam a introdução, os temas 4 a 6 o desenvolvimento, e o roteiro prático no fim gera os exemplos.

## Tema 1 — Transação e as quatro propriedades ACID

Uma transação é um conjunto de comandos que o banco de dados trata como uma unidade indivisível: ou todos acontecem, ou nenhum acontece.

O exemplo clássico é uma transferência de R$ 100 entre duas contas. São dois comandos: tirar 100 da conta A e somar 100 na conta B. Se a energia cair entre um e outro, o dinheiro simplesmente evapora. Agrupar os dois numa transação é o que impede isso.

Dois comandos marcam o fim de uma transação:

- `COMMIT` — confirma. O banco promete que tudo o que a transação fez está salvo de verdade.
- `ROLLBACK` — desfaz. O banco volta tudo ao estado anterior, como se a transação nunca tivesse existido.

### As quatro propriedades (ACID)

ACID é uma sigla em inglês para quatro garantias que uma transação oferece.

| Letra | Nome | O que significa, em uma frase |
| --- | --- | --- |
| A | Atomicidade (atomicity) | Tudo ou nada: a transação não fica pela metade. |
| C | Consistência (consistency) | O banco sai de um estado válido e chega em outro estado válido, respeitando as regras definidas (chaves, restrições). |
| I | Isolamento (isolation) | Uma transação não enxerga o meio do trabalho da outra; é como se cada uma estivesse sozinha no banco. |
| D | Durabilidade (durability) | Depois do `COMMIT`, o dado sobrevive a queda de energia ou travamento do servidor. |

### Por que só o "I" interessa aqui

O seu tema, controle de concorrência, é o conjunto de técnicas que o banco usa para entregar o **I** de isolamento. As outras três letras são entregues por outro mecanismo, chamado recuperação (recovery), que usa um arquivo de registro chamado log.

Vale escrever isso no artigo logo na introdução: concorrência e recuperação são problemas vizinhos e frequentemente confundidos.

**Leve para o próximo tema:** sem isolamento, transações simultâneas se atrapalham. O tema 2 mostra exatamente como.

## Tema 2 — Os quatro problemas que a concorrência causa

Quando duas transações mexem no mesmo dado ao mesmo tempo sem nenhum controle, quatro erros podem aparecer. Eles têm nome próprio e são citados em qualquer artigo da área, então vale decorar os quatro.

Nas tabelas abaixo, T1 e T2 são duas transações rodando ao mesmo tempo. O tempo corre de cima para baixo.

### 1. Atualização perdida (lost update)

Duas transações leem o mesmo valor, cada uma calcula um novo valor a partir dele, e a segunda gravação apaga a primeira.

| Tempo | T1 | T2 | Saldo no banco |
| --- | --- | --- | --- |
| 1 | lê saldo = 100 |  | 100 |
| 2 |  | lê saldo = 100 | 100 |
| 3 | grava 100 − 30 = 70 |  | 70 |
| 4 |  | grava 100 − 50 = 50 | 50 |

O saque de R$ 30 desapareceu. O saldo correto seria 20.

### 2. Leitura suja (dirty read)

Uma transação lê um valor que a outra ainda não confirmou, e que pode ser desfeito depois.

| Tempo | T1 | T2 | Observação |
| --- | --- | --- | --- |
| 1 | grava saldo = 500 |  | ainda sem `COMMIT` |
| 2 |  | lê saldo = 500 | leu algo provisório |
| 3 | `ROLLBACK` |  | o 500 nunca existiu |
| 4 |  | decide com base em 500 | decisão baseada em um dado falso |

O nome "suja" vem daí: o dado lido estava sujo, ainda não confirmado.

### 3. Leitura não repetível (non-repeatable read)

A mesma transação lê o mesmo registro duas vezes e recebe valores diferentes, porque outra transação alterou o registro no intervalo.

| Tempo | T1 | T2 |
| --- | --- | --- |
| 1 | lê preço = 10 |  |
| 2 |  | grava preço = 15 e confirma |
| 3 | lê preço = 15 |  |

T1 fez a mesma pergunta duas vezes e ouviu duas respostas. Um relatório calculado assim fica internamente inconsistente.

### 4. Fantasma (phantom read)

Parecido com o anterior, mas o que muda não é um registro: é a **quantidade** de registros que satisfazem uma busca.

| Tempo | T1 | T2 |
| --- | --- | --- |
| 1 | conta clientes de SP → 40 |  |
| 2 |  | insere um cliente de SP e confirma |
| 3 | conta clientes de SP → 41 |  |

A linha nova "apareceu do nada" no meio da transação, como um fantasma. É o caso mais difícil de evitar, porque não basta trancar as linhas existentes — é preciso trancar também as linhas que ainda não existem.

**Leve para o próximo tema:** os quatro erros têm uma causa comum — as operações se intercalaram numa ordem que nenhuma execução em fila produziria. O tema 3 transforma essa ideia em um critério formal.

## Tema 3 — Serialidade: o que conta como "certo"

Uma execução simultânea é considerada correta quando produz o mesmo resultado que alguma execução em fila das mesmas transações. Esse critério se chama serialidade (serializability).

Repare na palavra "alguma": não importa qual ordem. Se T1 e T2 rodam juntas e o resultado é igual ao de T1 depois T2, está correto. Se for igual ao de T2 depois T1, também está.

### Três palavras que você vai usar no artigo

- **Escalonamento (schedule)** — a ordem em que as operações das várias transações realmente se intercalaram no tempo. É a "partitura" da execução.
- **Escalonamento serial** — aquele em que as transações não se misturam: uma termina antes da outra começar. É sempre correto, mas é lento, porque ninguém aproveita o tempo ocioso das outras.
- **Conflito** — duas operações conflitam quando são de transações diferentes, tocam o mesmo dado, e pelo menos uma delas é uma escrita. Leitura com leitura nunca conflita; a ordem entre duas leituras não muda nada.

### O grafo de precedência

Há um teste visual simples para saber se um escalonamento é correto. Ele se chama grafo de precedência (precedence graph) e funciona assim:

1. Desenhe um círculo para cada transação.
2. Sempre que uma operação de Ti conflitar com uma operação posterior de Tj, desenhe uma seta de Ti para Tj.
3. Olhe o desenho: se as setas formarem um ciclo (um caminho que volta ao ponto de partida), o escalonamento é incorreto. Sem ciclo, é correto.

A intuição é direta: uma seta de T1 para T2 significa "T1 teve que vir antes de T2". Um ciclo significa que T1 precisa vir antes de T2 e T2 precisa vir antes de T1 ao mesmo tempo — impossível de conciliar com qualquer fila.

Apliquem isso à atualização perdida do tema 2: T1 lê o saldo antes de T2 escrever (seta T1 → T2) e T2 lê o saldo antes de T1 escrever (seta T2 → T1). Ciclo. Está provado que aquele escalonamento é incorreto.

**Leve para o próximo tema:** o critério está definido, mas ninguém desenha grafos em tempo real num banco em produção. Os temas 4 e 5 mostram as duas estratégias práticas para garantir serialidade na marra.

## Tema 4 — Abordagem pessimista: cadeados

A estratégia pessimista parte do princípio de que conflito vai acontecer, então previne antes. A ferramenta é o cadeado (lock): antes de tocar num dado, a transação pede o cadeado dele; enquanto não conseguir, espera parada.

### Dois tipos de cadeado

- **Compartilhado (shared, S)** — para ler. Vários leitores podem segurar o mesmo cadeado compartilhado ao mesmo tempo, porque ler junto não causa problema.
- **Exclusivo (exclusive, X)** — para escrever. Só um por vez, e ele não convive com nenhum outro cadeado no mesmo dado.

A regra de convivência costuma aparecer como esta tabela, chamada matriz de compatibilidade:

| Já concedido ↓ / Pedido → | Compartilhado (S) | Exclusivo (X) |
| --- | --- | --- |
| Compartilhado (S) | pode | espera |
| Exclusivo (X) | espera | espera |

### Bloqueio em duas fases (2PL)

Só usar cadeados não basta: é preciso usá-los na hora certa. O protocolo que garante serialidade se chama bloqueio em duas fases (two-phase locking, 2PL), e divide a vida da transação em duas fases:

1. **Fase de crescimento** — a transação só adquire cadeados, nunca libera.
2. **Fase de encolhimento** — depois de liberar o primeiro cadeado, ela não pode adquirir mais nenhum.

A imagem mental é uma montanha: sobe pegando chaves, e a partir do momento em que devolve a primeira, só desce. Está demonstrado que qualquer execução seguindo essa regra é serializável.

Na prática, os bancos usam a variante rigorosa (strict 2PL), em que todos os cadeados só são devolvidos no `COMMIT`. Isso elimina de quebra a leitura suja, porque ninguém consegue ler um dado antes de ele ser confirmado.

### Granularidade

Granularidade é o tamanho do que se tranca. Dá para trancar uma linha, uma página (um bloco de linhas no disco) ou a tabela inteira.

O dilema é direto: cadeados pequenos permitem mais paralelismo, mas o banco precisa administrar milhares deles, o que consome memória e tempo. Cadeados grandes são baratos de administrar e travam meio mundo. É um dos trade-offs mais citáveis do seu artigo.

### Impasse (deadlock)

O preço de fazer transações esperarem é que a espera pode virar circular:

| Tempo | T1 | T2 |
| --- | --- | --- |
| 1 | tranca a conta A |  |
| 2 |  | tranca a conta B |
| 3 | pede a conta B → espera T2 |  |
| 4 |  | pede a conta A → espera T1 |

Ninguém nunca mais anda. Isso é um impasse (deadlock). Os bancos lidam com ele de duas maneiras:

- **Detecção** — o banco monta um grafo de espera (wait-for graph), procura ciclos de tempos em tempos e, achando um, escolhe uma transação vítima e a desfaz com `ROLLBACK`. É o que o PostgreSQL faz.
- **Prevenção** — o banco impede que o ciclo se forme, usando a idade das transações para decidir quem espera e quem é abortada.

**Leve para o próximo tema:** cadeados são seguros, mas transformam concorrência em fila. O tema 5 mostra como fugir da espera.

## Tema 5 — Abordagem otimista e múltiplas versões

A estratégia otimista parte do princípio oposto: conflito é raro, então não vale a pena pagar o preço da espera. Deixa todo mundo trabalhar livremente e confere no final.

### Validação em três fases

1. **Leitura** — a transação lê o que quiser e faz suas alterações numa cópia particular, invisível para os outros.
2. **Validação** — na hora de confirmar, o banco pergunta: alguém alterou, nesse meio tempo, algum dado que eu li? Se sim, a transação é abortada e precisa ser refeita do zero.
3. **Escrita** — passou na validação, as alterações da cópia particular são efetivadas para todos.

O cálculo é simples: se poucas transações disputam os mesmos dados, quase todas passam na validação e ninguém esperou por nada. Se a disputa é alta, muitas são abortadas e o trabalho refeito custa mais caro do que a espera custaria. Escolher entre pessimista e otimista é escolher uma aposta sobre o nível de disputa.

### MVCC: controle de concorrência multiversão

MVCC é a sigla de multiversion concurrency control, controle de concorrência por múltiplas versões. É o mecanismo dominante hoje — PostgreSQL, Oracle e o motor InnoDB do MySQL usam alguma forma dele.

A ideia central: quando uma transação altera uma linha, o banco não sobrescreve o valor antigo. Ele cria uma **nova versão** da linha e mantém a antiga por um tempo. Cada transação, ao começar, recebe uma espécie de fotografia do banco (snapshot) e enxerga apenas as versões que já estavam confirmadas naquele instante.

A consequência prática é a frase que você vai ver repetida na documentação do PostgreSQL: leitura nunca bloqueia escrita, e escrita nunca bloqueia leitura. Quem lê pega a versão antiga e segue a vida; quem escreve cria a versão nova em paralelo. Só escrita contra escrita na mesma linha ainda precisa de espera.

### O preço do MVCC

Nada é de graça. Guardar versões antigas significa que o banco acumula lixo — linhas que ninguém mais enxerga. Alguém precisa recolher esse lixo depois; no PostgreSQL esse processo se chama `VACUUM`. Um banco mal configurado incha, e esse inchaçao tem nome na comunidade: table bloat.

**Leve para o próximo tema:** essas técnicas não são um botão de liga e desliga. O tema 6 mostra o botão que o programador realmente controla.

## Tema 6 — Níveis de isolamento

Isolamento perfeito é caro. Por isso o padrão SQL deixa o programador escolher quanta segurança quer pagar, através de quatro níveis. Quanto mais alto o nível, menos erros acontecem e mais lento fica.

O comando é `SET TRANSACTION ISOLATION LEVEL <nível>`.

| Nível | Leitura suja | Leitura não repetível | Fantasma |
| --- | --- | --- | --- |
| Read uncommitted (lê não confirmado) | pode ocorrer | pode ocorrer | pode ocorrer |
| Read committed (lê confirmado) | evitada | pode ocorrer | pode ocorrer |
| Repeatable read (leitura repetível) | evitada | evitada | pode ocorrer |
| Serializable (serializável) | evitada | evitada | evitada |

Dois detalhes que rendem parágrafo no artigo:

- O padrão define os níveis pelos erros que cada um **permite**, não pela técnica usada. Cada fabricante implementa como quiser, e por isso o mesmo nome se comporta diferente em bancos diferentes.
- O padrão exige apenas que o nível seja pelo menos tão rigoroso quanto o pedido. O PostgreSQL, por exemplo, não tem read uncommitted de verdade: pedir esse nível entrega read committed.

### O furo que o padrão não previu

Em 1995, um artigo de Berenson e colegas mostrou que a tabela acima é incompleta. Existe um erro que ela não lista e que escapa até do repeatable read: o **write skew** (desvio de escrita).

O exemplo clássico é a escala de plantão de um hospital, com a regra "pelo menos um médico de plantão". Há dois de plantão, Ana e Bruno, e os dois pedem folga ao mesmo tempo:

| Tempo | T1 (Ana) | T2 (Bruno) |
| --- | --- | --- |
| 1 | conta plantonistas → 2, pode sair |  |
| 2 |  | conta plantonistas → 2, pode sair |
| 3 | remove Ana |  |
| 4 |  | remove Bruno |

Cada transação, sozinha, respeitou a regra. Juntas, deixaram o hospital sem nenhum médico. Note que as duas escreveram em **linhas diferentes** — por isso nenhum cadeado de linha e nenhuma checagem de versão pega o problema.

A solução é o nível serializable de verdade. O PostgreSQL implementa isso desde a versão 9.1 com uma técnica chamada SSI (serializable snapshot isolation), que detecta esses padrões perigosos e aborta uma das transações.

### O trade-off para a conclusão do artigo

Subir o nível de isolamento reduz erros e aumenta o custo, de duas formas: mais espera (no caso dos cadeados) ou mais transações abortadas e refeitas (no caso do MVCC e do SSI). Medir esse custo é exatamente o experimento sugerido no roteiro abaixo.

## Roteiro prático

A melhor forma de fixar o assunto é reproduzir cada erro com as próprias mãos. O experimento exige apenas um banco instalado e duas janelas de terminal abertas ao mesmo tempo, cada uma com uma conexão.

### O experimento base

Em cada janela, comece com `BEGIN;` (que abre uma transação) e execute os comandos alternadamente, seguindo a ordem das tabelas do tema 2. A regra é não dar `COMMIT` até o momento indicado — é a espera que revela o comportamento.

Sugestão de tabela para os testes:

```sql
CREATE TABLE conta (id int PRIMARY KEY, saldo numeric);
INSERT INTO conta VALUES (1, 100), (2, 100);
```

### Ordem sugerida de estudo

| Etapa | O que fazer | Resultado esperado |
| --- | --- | --- |
| 1 | Ler os temas 1 a 3 e refazer o grafo de precedência no papel | entender o critério de correção |
| 2 | Reproduzir as quatro anomalias em dois terminais, no nível read committed | ver quais aparecem e quais não |
| 3 | Repetir tudo em repeatable read e em serializable | ver o nível mudando o comportamento |
| 4 | Reproduzir o write skew do hospital e observar o erro de serialização | entender o limite do padrão SQL |
| 5 | Medir desempenho por nível de isolamento com a ferramenta `pgbench` | gerar os gráficos do artigo |

### Leituras para o artigo

O artigo no padrão SBC exige referências bibliográficas de verdade. Comece por estas:

- Capítulos de transações e controle de concorrência de Elmasri e Navathe, ou de Silberschatz, Korth e Sudarshan — provavelmente já na bibliografia da sua disciplina.
- Berenson et al., "A critique of ANSI SQL isolation levels" (1995) — a origem do write skew, e a referência que dá densidade à sua discussão.
- A documentação oficial do PostgreSQL sobre controle de concorrência — fonte primária, atual, e fácil de citar.

Como essas referências vêm da minha memória, confira ano, edição e capítulo antes de colocar no artigo. Citação errada custa nota.

## Glossário

| Termo | Em inglês | Definição em uma linha |
| --- | --- | --- |
| ACID | ACID | As quatro garantias de uma transação: atomicidade, consistência, isolamento e durabilidade. |
| Atualização perdida | Lost update | Duas transações leem o mesmo valor e a segunda gravação apaga o efeito da primeira. |
| Bloqueio em duas fases | Two-phase locking (2PL) | Protocolo em que a transação primeiro só adquire cadeados e depois só os libera. |
| Cadeado | Lock | Marca que reserva um dado para uma transação e faz as outras esperarem. |
| Commit | Commit | Comando que confirma em definitivo tudo o que a transação fez. |
| Conflito | Conflict | Duas operações de transações diferentes sobre o mesmo dado, sendo ao menos uma delas escrita. |
| Escalonamento | Schedule | A ordem real em que as operações das várias transações se intercalaram. |
| Fantasma | Phantom | Linhas novas que passam a satisfazer uma busca já feita dentro da mesma transação. |
| Granularidade | Granularity | O tamanho do que se tranca: linha, página ou tabela. |
| Grafo de precedência | Precedence graph | Desenho dos conflitos entre transações; um ciclo nele indica execução incorreta. |
| Impasse | Deadlock | Espera circular em que duas transações travam uma à outra para sempre. |
| Leitura não repetível | Non-repeatable read | A mesma linha lida duas vezes na mesma transação devolve valores diferentes. |
| Leitura suja | Dirty read | Leitura de um valor ainda não confirmado, que pode ser desfeito depois. |
| MVCC | Multiversion concurrency control | Técnica que guarda várias versões de cada linha para que leitura e escrita não se bloqueiem. |
| Nível de isolamento | Isolation level | Ajuste que define quais anomalias o banco aceita permitir em troca de desempenho. |
| Rollback | Rollback | Comando que desfaz tudo o que a transação fez. |
| Serialidade | Serializability | Critério de correção: o resultado precisa ser igual ao de alguma execução em fila. |
| Snapshot | Snapshot | A fotografia do banco que uma transação enxerga, congelada no instante em que ela começou. |
| Transação | Transaction | Conjunto de comandos tratado como unidade indivisível. |
| Write skew | Write skew | Duas transações escrevem em linhas diferentes e juntas quebram uma regra que cada uma respeitou. |
