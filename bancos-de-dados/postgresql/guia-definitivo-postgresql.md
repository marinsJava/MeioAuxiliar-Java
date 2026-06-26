# Guia Definitivo de PostgreSQL

> Do básico ao uso profissional, com foco em integração com Java/Spring Boot. Versão de referência: **PostgreSQL 18** (estável em junho/2026; a 19 está em beta).

PostgreSQL (ou "Postgres") é um banco de dados relacional open-source com mais de 35 anos de desenvolvimento, conhecido por confiabilidade, robustez e conformidade com padrões. É, hoje, o banco relacional padrão para a maioria dos projetos novos — e o companheiro natural de uma stack Spring Boot.

---

## Índice

- [Por que PostgreSQL](#por-que-postgresql)
- [Subindo um Postgres em segundos (Docker)](#subindo-um-postgres-em-segundos)
- [A hierarquia: cluster, database, schema, tabela](#a-hierarquia)
- [psql: o cliente de linha de comando](#psql-o-cliente-de-linha-de-comando)
- [Tipos de dados que importam](#tipos-de-dados-que-importam)
- [DDL e DML essenciais](#ddl-e-dml-essenciais)
- [Índices: o que separa rápido de lento](#índices)
- [EXPLAIN: lendo a mente do planejador](#explain)
- [Conectando com Spring Boot](#conectando-com-spring-boot)
- [Migrations com Flyway](#migrations-com-flyway)
- [Backup e restore](#backup-e-restore)
- [Recursos avançados que vale conhecer](#recursos-avançados)
- [Boas práticas](#boas-práticas)
- [Referências](#referências)

---

## Por que PostgreSQL

Além de gratuito e com licença permissiva, ele roda igual em Linux, macOS e Windows e traz um ecossistema enorme de extensões. Os diferenciais que pesam no dia a dia:

- **Conformidade com SQL** e transações ACID sólidas.
- **Tipos ricos:** `JSONB` (JSON binário indexável), arrays, `UUID`, tipos geométricos, ranges — você ganha flexibilidade quase de banco de documentos sem abrir mão do relacional.
- **Extensibilidade:** PostGIS (geoespacial), `pg_trgm` (busca fuzzy), `pgvector` (busca vetorial para IA), e muito mais.
- **Cadência previsível:** uma versão maior por ano, suportada por 5 anos.

---

## Subindo um Postgres em segundos

Para desenvolvimento, esqueça instalação manual. Docker é o caminho:

```bash
docker run -d \
  --name pg \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=app \
  -e POSTGRES_DB=meuapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:18
```

Em segundos você tem um banco pronto na porta 5432, com os dados persistidos no volume `pgdata`. Para conectar via terminal:

```bash
docker exec -it pg psql -U app -d meuapp
```

---

## A hierarquia

Entender a hierarquia evita confusão (especialmente a diferença entre *database* e *schema*, que pega muita gente):

```
Instância (cluster) PostgreSQL          ← um servidor rodando
├── Database: meuapp                     ← bancos isolados entre si
│   ├── Schema: public (padrão)          ← "namespace" de objetos
│   │   ├── Tabela: medicamento
│   │   ├── Tabela: paciente
│   │   └── View, Função, Índice...
│   └── Schema: auditoria                 ← você pode ter vários schemas
└── Database: outro_app
```

- **Cluster/instância** — o servidor PostgreSQL em si (não confundir com cluster do Kubernetes!).
- **Database** — bancos completamente isolados. Você não faz `JOIN` entre databases diferentes.
- **Schema** — um namespace dentro do database. O `public` é o padrão. Útil para organizar (ex.: separar tabelas de domínio das de auditoria) ou para multi-tenancy.
- **Roles** — usuários e grupos. Postgres unifica os dois conceitos em "roles".

---

## psql: o cliente de linha de comando

O `psql` tem metacomandos (começam com `\`) que agilizam muito:

```sql
\l                  -- lista os databases
\c meuapp           -- conecta a um database
\dt                 -- lista as tabelas
\d medicamento      -- descreve a estrutura de uma tabela
\dn                 -- lista schemas
\du                 -- lista roles (usuários)
\di                 -- lista índices
\x                  -- alterna saída expandida (ótimo para linhas largas)
\timing             -- liga/desliga a medição de tempo das queries
\q                  -- sai
```

---

## Tipos de dados que importam

Postgres tem dezenas de tipos. Os que você mais vai usar e alguns que valem conhecer:

| Categoria | Tipos | Observação |
|---|---|---|
| Numéricos | `INTEGER`, `BIGINT`, `NUMERIC(p,s)`, `REAL` | Use `NUMERIC` para dinheiro (nunca `float`!) |
| Texto | `VARCHAR(n)`, `TEXT` | `TEXT` não tem custo extra; use à vontade |
| Data/hora | `DATE`, `TIMESTAMP`, `TIMESTAMPTZ` | **Prefira `TIMESTAMPTZ`** (com fuso) quase sempre |
| Booleano | `BOOLEAN` | `true`/`false` |
| Identificador | `UUID` | Bom para chaves distribuídas; veja `gen_random_uuid()` |
| Serial | `GENERATED ALWAYS AS IDENTITY` | Forma moderna de auto-incremento (melhor que `SERIAL`) |
| Semiestruturado | `JSONB` | JSON binário, indexável e consultável |
| Coleção | arrays (`INTEGER[]`, `TEXT[]`) | Listas dentro de uma coluna |

> 💡 `TIMESTAMPTZ` vs `TIMESTAMP`: o primeiro guarda o instante normalizado em UTC e converte conforme o fuso da sessão. Em aplicações que cruzam fusos (ou só para evitar dor de cabeça futura), é a escolha segura. No Java, mapeia bem para `Instant`/`OffsetDateTime`.

---

## DDL e DML essenciais

```sql
-- DDL: estrutura
CREATE TABLE medicamento (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome        VARCHAR(200) NOT NULL,
    dosagem     VARCHAR(50),
    principio   TEXT,
    metadados   JSONB,                      -- dados flexíveis
    criado_em   TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_medicamento_nome UNIQUE (nome)
);

ALTER TABLE medicamento ADD COLUMN ativo BOOLEAN NOT NULL DEFAULT true;

-- DML: dados
INSERT INTO medicamento (nome, dosagem, principio)
VALUES ('Losartana', '50mg', 'Losartana potássica')
RETURNING id;                               -- retorna o id gerado

SELECT * FROM medicamento WHERE dosagem = '50mg';

UPDATE medicamento SET ativo = false WHERE id = 1;

DELETE FROM medicamento WHERE ativo = false;

-- Consultando JSONB
SELECT nome FROM medicamento WHERE metadados ->> 'fabricante' = 'EMS';
```

O `RETURNING` é uma joia do Postgres: ele devolve dados das linhas afetadas por `INSERT`/`UPDATE`/`DELETE` na mesma instrução — muito útil para pegar o id recém-gerado sem uma segunda consulta.

---

## Índices

Índice é o que transforma uma busca de segundos em milissegundos. A regra mental: **colunas usadas em `WHERE`, `JOIN` e `ORDER BY` são candidatas a índice.**

```sql
-- B-tree (padrão) — para igualdade e ordenação
CREATE INDEX idx_medicamento_dosagem ON medicamento (dosagem);

-- Índice composto — a ORDEM das colunas importa
CREATE INDEX idx_med_nome_dosagem ON medicamento (nome, dosagem);

-- GIN — para JSONB e arrays
CREATE INDEX idx_medicamento_metadados ON medicamento USING GIN (metadados);

-- Índice parcial — indexa só um subconjunto (economiza espaço)
CREATE INDEX idx_medicamento_ativos ON medicamento (nome) WHERE ativo = true;
```

Cuidados honestos sobre índices: eles **aceleram leitura mas desaceleram escrita** (todo `INSERT`/`UPDATE` precisa atualizar os índices) e **ocupam espaço**. Não saia indexando tudo — indexe com base em queries reais e meça o efeito. Índice que não é usado é só custo.

---

## EXPLAIN

Quando uma query está lenta, o `EXPLAIN ANALYZE` mostra exatamente o que o planejador faz e quanto custa:

```sql
EXPLAIN ANALYZE
SELECT * FROM medicamento WHERE dosagem = '50mg';
```

O que procurar na saída:
- **`Seq Scan`** (varredura sequencial) numa tabela grande com filtro = sinal de que falta um índice.
- **`Index Scan`** = o índice está sendo usado (bom).
- Compare o `cost` estimado com o `actual time` real para entender onde o tempo é gasto.

Aprender a ler o `EXPLAIN` é o que diferencia quem "acha" que otimizou de quem **sabe** que otimizou.

---

## Conectando com Spring Boot

A integração é direta. Dependências (`pom.xml`):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Configuração (`application.yml`):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/meuapp
    username: app
    password: ${DB_PASSWORD}
    hikari:                       # HikariCP é o pool de conexões padrão do Spring Boot
      maximum-pool-size: 10
      minimum-idle: 2
  jpa:
    hibernate:
      ddl-auto: validate          # em produção, NUNCA use create/update — veja abaixo
    properties:
      hibernate:
        format_sql: true
    open-in-view: false           # desligue: evita o anti-padrão Open Session in View
```

Dois ajustes que valem ouro:

- **`ddl-auto: validate`** em produção. Deixe o `update`/`create` apenas para protótipos rápidos. Quem cuida do schema é a ferramenta de migration (Flyway), não o Hibernate — assim você tem controle e histórico das mudanças.
- **`open-in-view: false`.** O padrão `true` mantém a sessão JPA aberta durante a renderização da resposta, o que mascara problemas de lazy loading e segura conexões por mais tempo. Desligar força um design mais limpo.

> O dialect não precisa mais ser configurado à mão nas versões atuais — o Hibernate detecta o PostgreSQL automaticamente pela conexão.

---

## Migrations com Flyway

Migrations versionam a evolução do schema, como o Git versiona o código. Cada mudança é um arquivo SQL numerado e imutável.

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

Os arquivos ficam em `src/main/resources/db/migration/` com a convenção `V<versão>__<descrição>.sql`:

```
db/migration/
├── V1__cria_tabela_medicamento.sql
├── V2__adiciona_coluna_ativo.sql
└── V3__cria_indice_dosagem.sql
```

```sql
-- V1__cria_tabela_medicamento.sql
CREATE TABLE medicamento (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome      VARCHAR(200) NOT NULL,
    dosagem   VARCHAR(50),
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

O Spring Boot roda as migrations pendentes automaticamente na inicialização. A regra de ouro: **nunca edite uma migration já aplicada** — crie uma nova. O Flyway guarda um checksum de cada uma e acusa se alguma foi alterada.

---

## Backup e restore

```bash
# Backup de um database (formato custom, comprimido e flexível — recomendado)
pg_dump -U app -Fc meuapp > backup.dump

# Backup em SQL puro (legível, portável)
pg_dump -U app meuapp > backup.sql

# Restore do formato custom
pg_restore -U app -d meuapp_novo backup.dump

# Restore de SQL puro
psql -U app -d meuapp_novo < backup.sql

# Backup de TODOS os databases do cluster (inclui roles)
pg_dumpall -U postgres > backup_completo.sql
```

Dentro de um container Docker, prefixe com `docker exec`:

```bash
docker exec -t pg pg_dump -U app -Fc meuapp > backup.dump
```

> O formato custom (`-Fc`) é o mais versátil: comprime, permite restaurar tabelas seletivamente e paraleliza. Para automação real, agende backups e **teste o restore periodicamente** — backup que nunca foi restaurado é só uma esperança.

---

## Recursos avançados

Vale saber que existem, mesmo que você não use agora:

- **JSONB com operadores ricos:** `->`, `->>`, `@>` (contém), mais índices GIN, dão a você consultas tipo banco de documentos dentro do relacional.
- **Busca textual (full-text search):** tipos `tsvector`/`tsquery` para busca eficiente em texto, com ranking.
- **`pg_trgm`:** busca por similaridade ("você quis dizer...?") e `LIKE` acelerado.
- **`pgvector`:** armazena embeddings e faz busca por similaridade vetorial — base de aplicações de IA/RAG.
- **CTEs e Window Functions:** `WITH ... AS (...)` e `OVER (PARTITION BY ...)` para consultas analíticas elegantes.
- **Particionamento de tabelas:** divide tabelas gigantes por faixa/lista para manter performance.

---

## Boas práticas

1. **Use connection pooling.** O HikariCP já vem no Spring Boot; em escala, considere o PgBouncer na frente do banco.
2. **Não use o superusuário da aplicação.** Crie um role com privilégios mínimos para a app.
3. **`NUMERIC` para dinheiro**, nunca `float`/`double` (erros de arredondamento são inaceitáveis em valores).
4. **`TIMESTAMPTZ` por padrão** para datas com hora.
5. **Schema pela migration, não pelo Hibernate** (`ddl-auto: validate`).
6. **Faça `VACUUM`/`ANALYZE`** — o autovacuum cuida disso, mas saiba que existe e monitore.
7. **Indexe com base em queries reais**, medindo com `EXPLAIN ANALYZE`.
8. **Senhas e URLs fora do código** — variáveis de ambiente ou cofre, nunca no Git.

---

## Referências

1. **Documentação oficial** — https://www.postgresql.org/docs/
2. **Tutorial oficial** — https://www.postgresql.org/docs/current/tutorial.html
3. **Spring Data JPA** — https://docs.spring.io/spring-data/jpa/reference/
4. **Flyway** — https://documentation.red-gate.com/flyway
5. **Use The Index, Luke!** (índices e performance, gratuito) — https://use-the-index-luke.com/
6. **PostgreSQL Exercises** (prática interativa) — https://pgexercises.com/

> Para rodar este Postgres em container, veja o [Guia Definitivo de Docker](../../ferramentas/docker/guia-definitivo-docker.md). Para orquestrá-lo em cluster, o [Guia Definitivo de Kubernetes](../../ferramentas/kubernetes/guia-definitivo-kubernetes.md) (com a ressalva sobre bancos gerenciados em produção).
