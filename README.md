# Carga de CSV no Solr com Airflow

Desafio técnico de julho de 2025: pegar um CSV de alunos, limpar com pandas e indexar no Apache Solr,
com o Airflow orquestrando e tudo rodando em Docker Compose. Os alunos do CSV são fictícios.

## A DAG

`solr_pipeline` (em `airflow/dags/`) roda uma vez por semana e tem duas tarefas, uma depois da outra:

1. `format_csv` lê `data/alunos.csv`, descarta as linhas sem nome ou sem data de nascimento, padroniza
   os nomes, converte as datas (as que não convertem também saem) e calcula a idade pela data de
   nascimento. O resultado vai para `data/alunos_clean.csv`.
2. `load_to_solr` lê o CSV limpo, monta um documento por aluno e envia todos para a coleção
   `alunos_collection` com o pysolr. O `id` do documento é o nome junto com a data de nascimento, então
   rodar de novo sobrescreve os mesmos documentos em vez de duplicar.

As duas funções ficam em `scripts/` e também rodam fora do Airflow, com pandas e pysolr instalados:

```bash
python scripts/format_csv.py data/alunos.csv data/alunos_clean.csv
python scripts/load_to_solr.py http://localhost:8983/solr/alunos_collection data/alunos_clean.csv
```

A idade calculada fica no CSV limpo (`Idade_Calc`), mas o documento do Solr usa a coluna `Idade` que
veio no arquivo. O filtro que comparava as duas está comentado no código, porque descartava todos os
registros.

## Infraestrutura

`docker/docker-compose-airflow.yml` sobe o Airflow 2.5.1 com CeleryExecutor: webserver e scheduler num
contêiner, o worker em outro, Postgres para os metadados e Redis como broker. O Solr 8.11 sobe com a
coleção já criada. As pastas `airflow/dags`, `scripts` e `data` são montadas nos contêineres, e o
pysolr é instalado quando eles sobem. Usuário, senhas e a chave Fernet que estão no compose servem só
para rodar localmente.

## Rodando

Precisa de Docker com o Compose.

```bash
git clone https://github.com/BrunoMaia23/carga-solr-airflow.git
cd carga-solr-airflow/docker
docker compose -f docker-compose-solr.yml up -d
docker compose -f docker-compose-airflow.yml up --build -d
```

Em uns dois minutos o Airflow responde em http://localhost:8080 (usuário `admin`, senha `admin`). A
DAG já nasce ativa e roda uma vez sozinha; para rodar de novo, use o botão de trigger. No Solr, em
http://localhost:8983/solr, a coleção `alunos_collection` mostra os documentos com a consulta `q=*:*`.

Para desligar: `docker compose -f docker-compose-airflow.yml down`, e o mesmo com o arquivo do Solr.
