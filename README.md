# Controle de concorrência em bancos de dados relacionais

Estudo prático sobre como um SGBD impede que transações simultâneas corrompam os dados, e quanto custa cada nível de proteção. O repositório reúne o artigo acadêmico (padrão SBC), os scripts que reproduzem as anomalias clássicas e as medições de desempenho por nível de isolamento.

Trabalho da disciplina de Banco de Dados — Engenharia da Computação, CEFET/RJ.

## A pergunta investigada

Quando duas transações mexem no mesmo dado ao mesmo tempo, o banco pode produzir resultados que nenhuma execução sequencial produziria. As técnicas que evitam isso não são gratuitas: cobram espera ou transações abortadas. Este trabalho reproduz as anomalias na prática e mede esse custo.

## Estrutura

| Pasta | Conteúdo |
| --- | --- |
| `artigo/` | Fonte LaTeX do artigo no padrão SBC e a bibliografia |
| `estudo/` | Guia de estudo do tema, escrito para quem está começando |
| `experimentos/` | Scripts SQL que reproduzem cada anomalia, e o ambiente Docker |
| `resultados/` | Medições brutas e gráficos gerados a partir delas |

## Como reproduzir

Pré-requisitos: Docker e Docker Compose instalados.

```bash
git clone https://github.com/Mateus-Fonseca847/controle-de-concorrencia.git
cd controle-de-concorrencia/experimentos
docker compose up -d
```

Cada anomalia precisa de **duas conexões simultâneas** ao banco — é a intercalação entre elas que revela o comportamento. Abra dois terminais e, em cada um:

```bash
docker compose exec db psql -U postgres -d concorrencia
```

Os arquivos em `experimentos/` indicarão, em comentários, o que rodar na sessão 1 e o que rodar na sessão 2, e em que ordem.

## Experimentos

| Arquivo | Anomalia | Nível em que ocorre |
| --- | --- | --- |
| `01-atualizacao-perdida.sql` | Atualização perdida | Read committed |
| `02-leitura-suja.sql` | Leitura suja | Read uncommitted |
| `03-leitura-nao-repetivel.sql` | Leitura não repetível | Read committed |
| `04-fantasma.sql` | Fantasma | Repeatable read |
| `05-write-skew.sql` | Write skew | Repeatable read |

## Resultados

<!-- Aqui vamos adicionar resultados da pesquisa -->

## Artigo


Em construção. Fonte LaTeX em `artigo/main.tex`.

## Referências principais



## Autores

Mateus Fonseca — [LinkedIn](https://www.linkedin.com/in/mateus-souza-fonseca/), [Email](mateusfonseca847@gmail.com)
Fernando Correia — [LinkedIn](https://www.linkedin.com/in/fernando-grillo-83a304412/)[Email](fcorreiagrillo@gmail.com)
