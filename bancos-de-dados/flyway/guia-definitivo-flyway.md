# Guia Definitivo de Flyway

> Versionamento de banco de dados na prática, do primeiro migration à recuperação de erros de checksum. Stack de referência: **Flyway 12.x + Spring Boot 4.1 + PostgreSQL 18**.

Flyway é o "Git do seu schema": ele versiona a evolução do banco em arquivos SQL numerados, aplicados na ordem certa e **uma única vez cada**. Sem ele, "rodou a alteração no banco de produção?" vira uma pergunta assustadora. Com ele, o estado do banco é reproduzível e rastreável em qualquer ambiente.

---

## Índice

- [O problema que o Flyway resolve](#o-problema-que-o-flyway-resolve)
- [Como funciona: a tabela de histórico e o checksum](#como-funciona)
- [Setup com Spring Boot + PostgreSQL](#setup-com-spring-boot--postgresql)
- [Convenção de nomes: V, R e U](#convenção-de-nomes)
- [Escrevendo migrations: schema e dados](#escrevendo-migrations)
- [Boas práticas de seed: referencie por chave natural, não por ID](#boas-práticas-de-seed)
- [Migrations repetíveis (R__)](#migrations-repetíveis)
- [O erro de checksum (FlywayValidateException)](#o-erro-de-checksum)
- [Como corrigir: dev vs. produção](#como-corrigir)
- [Comandos e conceitos de referência](#comandos-e-conceitos-de-referência)
- [Armadilhas comuns](#armadilhas-comuns)
- [Referências](#referências)

---

## O problema que o Flyway resolve

Sem uma ferramenta de migração, mudanças no schema viram scripts soltos, aplicados à mão, em ordem incerta, e ninguém sabe qual ambiente está em qual estado. O Flyway resolve isso com uma ideia simples: **toda mudança de banco é um arquivo versionado, imutável, no controle de versão junto com o código.** O banco deixa de ser um estado misterioso e passa a ser a soma determinística das migrations aplicadas.

---

## Como funciona

O coração do Flyway é uma tabela que ele cria e mantém no seu banco: **`flyway_schema_history`**. Cada migration aplicada vira uma linha ali, registrando versão, descrição, quem aplicou, quando, e — o detalhe crucial — um **checksum** do conteúdo do arquivo.

O fluxo, toda vez que a aplicação sobe:

1. Flyway lê a `flyway_schema_history` para saber o que já foi aplicado.
2. Compara com os arquivos de migration disponíveis.
3. Aplica, em ordem de versão, **apenas as pendentes**.
4. Registra cada uma na tabela de histórico.

```
Arquivos no projeto          flyway_schema_history (no banco)
├── V001__cria_tabelas.sql   →  [✓ aplicada, checksum abc123]
├── V002__marmita_teste.sql  →  [✓ aplicada, checksum def456]
└── V003__nova_coluna.sql    →  [  pendente → aplica agora ]
```

Duas garantias que vêm disso:

- **Roda uma vez só por versão.** Depois que a V002 está registrada, ela nunca mais é reaplicada — mesmo que você reinicie a aplicação mil vezes. É o que torna seguro colocar `INSERT`s de dados de teste numa migration.
- **No Spring Boot, roda sozinho.** Basta a dependência estar no classpath: ao subir o app, o Flyway aplica as migrations pendentes automaticamente, **antes** de o Hibernate validar o schema. Você não chama nada manualmente.

---

## Setup com Spring Boot + PostgreSQL

Um ponto que **pega muita gente** desde o Flyway 10: o suporte a cada banco virou um módulo separado. Só o `flyway-core` não basta para PostgreSQL — sem o módulo específico, você toma um `Unsupported Database: PostgreSQL` na subida.

```xml
<!-- Núcleo do Flyway -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<!-- OBRIGATÓRIO para PostgreSQL (Flyway 10+) -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

> Com Spring Boot você **não** declara a versão do Flyway — o BOM do Spring Boot já gerencia isso, garantindo uma versão compatível. Deixe sem `<version>`.

Os arquivos ficam por convenção em `src/main/resources/db/migration/`. Configuração típica no `application.yml`:

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false     # true só ao adotar Flyway num banco já existente
  jpa:
    hibernate:
      ddl-auto: validate           # Flyway cuida do schema; Hibernate só CONFERE
```

A combinação de ouro: **`ddl-auto: validate`**. O Flyway é o dono do schema; o Hibernate apenas valida que as entidades batem com as tabelas. Nunca deixe os dois "criando" o schema ao mesmo tempo — é receita para conflito.

---

## Convenção de nomes

O nome do arquivo **é** a configuração. O padrão:

```
V002__marmita_teste.sql
│└┬┘  └──────┬──────┘
│ │          └── descrição (underscores viram espaços no histórico)
│ └── versão (002)
└── prefixo do tipo
```

Os três prefixos:

| Prefixo | Tipo | Quando roda |
|---|---|---|
| **`V`** | Versionada | Uma vez, em ordem de versão. O caso comum (DDL, seed). |
| **`R`** | Repetível | Sempre que seu **conteúdo muda** (checksum novo). Roda depois das versionadas. |
| **`U`** | Undo | Reverte uma versionada (recurso das edições pagas). |

Atenção ao **duplo underscore** (`__`) separando a versão da descrição — errar isso (um só underscore) faz o Flyway não reconhecer o arquivo.

---

## Escrevendo migrations

Uma migration é SQL puro. Pode conter DDL (estrutura) e DML (dados). Exemplo real de seed de dados — inserir uma marmita e vinculá-la a tags:

```sql
-- V002__marmita_teste.sql

-- 1) Insere a marmita
INSERT INTO marmita (nome, calorias, criado_em)
VALUES ('Frango Grelhado com Batata Doce', 520, now());

-- 2) Vincula tags à marmita recém-criada, na tabela de junção marmita_tag
INSERT INTO marmita_tag (marmita_id, tag_id)
SELECT m.id, t.id
FROM marmita m
JOIN tag t ON t.slug IN ('fit', 'proteico', 'sem-lactose')
WHERE m.nome = 'Frango Grelhado com Batata Doce';
```

Repare que o vínculo N–N não usa IDs fixos — ele **busca os IDs em tempo de execução** por meio de um `JOIN`. O porquê disso é importante o suficiente para uma seção própria.

---

## Boas práticas de seed

O erro clássico ao popular dados é escrever `INSERT INTO marmita_tag VALUES (1, 3)`, cravando IDs numéricos. Isso é **frágil**: se a ordem de inserção das tags mudar numa migration anterior, ou se alguém rodar as migrations em outra ordem num ambiente novo, o `id = 3` pode ser outra tag — e o vínculo silenciosamente aponta para o lugar errado.

A solução é **referenciar por chave natural** (um `slug`, um código, um nome único) em vez do ID sintético:

```sql
-- ❌ Frágil: depende da ordem de criação dos IDs
INSERT INTO marmita_tag (marmita_id, tag_id) VALUES (1, 3);

-- ✅ Robusto: resolve os IDs pelo slug, seja qual for o valor deles
INSERT INTO marmita_tag (marmita_id, tag_id)
SELECT m.id, t.id
FROM marmita m
JOIN tag t ON t.slug IN ('fit', 'proteico')
WHERE m.nome = 'Frango Grelhado com Batata Doce';
```

O `slug` é estável e legível; o `id` é um detalhe de implementação que pode variar. Ao ancorar o vínculo na chave natural, a migration funciona igual em qualquer ambiente, independentemente da ordem dos IDs.

> 💡 A mesma lógica vale para datas calculadas. Se um seed precisa "sempre a segunda-feira da semana atual" (padrão ISO), não crave uma data — calcule: `date_trunc('week', CURRENT_DATE)::date` retorna sempre a segunda-feira, casando com a regra que o seu service usa. Dados semânticos, não valores mágicos.

---

## Migrations repetíveis

Nem tudo se encaixa no modelo "roda uma vez". Views, funções e procedures você quer **redefinir** sempre que mudam, sem inventar uma versão nova a cada ajuste. Para isso existem as **repetíveis** (`R__`), que não têm número de versão e rodam sempre que o **checksum do arquivo muda**:

```sql
-- R__view_marmitas_fit.sql
CREATE OR REPLACE VIEW vw_marmitas_fit AS
SELECT m.* FROM marmita m
JOIN marmita_tag mt ON mt.marmita_id = m.id
JOIN tag t ON t.id = mt.tag_id
WHERE t.slug = 'fit';
```

Como usa `CREATE OR REPLACE`, editar o arquivo e subir o app reaplica a definição. Repetíveis rodam **depois** de todas as versionadas pendentes, em ordem alfabética.

---

## O erro de checksum

Esta é a dor mais comum — e a mais importante de entender. Quando o Flyway aplica a V002, ele grava o **checksum** do arquivo na `flyway_schema_history`. Toda vez que a aplicação sobe, ele **revalida**: recalcula o checksum do arquivo atual e compara com o gravado.

**Se você editar uma migration já aplicada, o checksum muda, deixa de bater com o registrado, e o Flyway lança `FlywayValidateException` — a aplicação não sobe.**

```
Migration checksum mismatch for migration version 002
-> Applied to database : def456
-> Resolved locally    : 999zzz   (você editou o arquivo!)
```

Isso **não é um bug — é a proteção funcionando.** O Flyway está te dizendo: "esta migration já rodou em algum banco com um conteúdo diferente; se eu deixasse passar, ambientes ficariam inconsistentes". A regra de ouro que decorre disso:

> **Nunca edite uma migration já aplicada. Crie uma nova.** Precisa mudar algo que a V002 fez? Escreva a V003 com a correção (um `ALTER`, um `UPDATE`). Migrations aplicadas são imutáveis, como um commit que já foi para o repositório compartilhado.

---

## Como corrigir

O tratamento depende do ambiente, e a diferença é séria.

### Em desenvolvimento (banco descartável)

Se o dado ainda está só na sua máquina, a correção manual é aceitável: apague o que a migration inseriu **e** remova a linha correspondente da `flyway_schema_history`, deixando o Flyway reaplicar o arquivo já corrigido na próxima subida.

```sql
-- 1) Desfaz o que a V002 fez (ajuste ao seu caso)
DELETE FROM marmita_tag WHERE marmita_id IN (SELECT id FROM marmita WHERE nome = 'Frango Grelhado com Batata Doce');
DELETE FROM marmita WHERE nome = 'Frango Grelhado com Batata Doce';

-- 2) Remove o registro da migration para o Flyway reaplicá-la
DELETE FROM flyway_schema_history WHERE version = '002';
```

Ao subir de novo, o Flyway vê a V002 como pendente e aplica o conteúdo corrigido. **Alternativa mais limpa em dev:** apagar o banco inteiro (ou `flyway clean`, que dropa tudo) e deixar recriar do zero — só faça isso onde perder os dados não importa.

### Em produção (jamais edite manualmente às cegas)

Nunca mexa na `flyway_schema_history` de produção na mão para "resolver" um mismatch. As opções corretas:

- **`flyway repair`** — o comando oficial que **recalcula e corrige os checksums** na tabela de histórico para bater com os arquivos atuais (útil quando a diferença é inofensiva, como uma mudança de formatação). Ele conserta a tabela sem reaplicar nada.
- **Nova migration** — se a mudança é de fato uma correção de schema/dado, escreva uma V nova. Essa é quase sempre a resposta certa.

> Em dev, apagar e reaplicar é rápido e sem consequências. Em produção, a mentalidade é oposta: o histórico é sagrado, e você avança com migrations novas, nunca reescrevendo o passado.

---

## Comandos e conceitos de referência

| Comando / conceito | O que faz |
|---|---|
| `flyway migrate` | Aplica as migrations pendentes (o Spring Boot faz isso sozinho na subida). |
| `flyway info` | Mostra o estado de cada migration (aplicada, pendente, falha). |
| `flyway validate` | Confere se os checksums batem (o que dispara o erro se você editou algo). |
| `flyway repair` | Corrige a `flyway_schema_history` (checksums, entradas de falha). |
| `flyway baseline` | Marca um banco já existente como ponto de partida (adoção do Flyway). |
| `flyway clean` | **Apaga tudo** do schema. Só em dev; desabilitado por padrão por segurança. |
| `flyway_schema_history` | A tabela onde tudo é registrado (versão, checksum, sucesso). |

---

## Armadilhas comuns

1. **Faltou o módulo do PostgreSQL** → `Unsupported Database`. Adicione `flyway-database-postgresql`.
2. **Editar migration aplicada** → `FlywayValidateException`. Nunca edite; crie uma nova.
3. **Um underscore só** no nome (`V002_x.sql`) → o Flyway ignora o arquivo. Use `__` (duplo).
4. **IDs fixos em seed** → vínculos frágeis. Referencie por chave natural (slug).
5. **`ddl-auto: update` junto com Flyway** → os dois disputam o schema. Use `validate`.
6. **Datas cravadas em seed** que deveriam ser calculadas → use expressões (`date_trunc(...)`).
7. **Rodar `flyway clean` sem pensar** → apaga o banco. Nunca perto de produção.
8. **Buracos de versão** (pular de V002 para V004) → funciona, mas evite; mantenha a sequência clara.

---

## Referências

1. **Documentação oficial** — https://documentation.red-gate.com/flyway
2. **Spring Boot — Database Initialization (Flyway)** — https://docs.spring.io/spring-boot/how-to/data-initialization.html
3. **Migrations (naming, versioned vs repeatable)** — https://documentation.red-gate.com/fd/migrations-184127470.html
4. **PostgreSQL Database module** — https://documentation.red-gate.com/fd/postgresql-database-277579325.html

> Este guia é o companheiro do [Guia Definitivo de PostgreSQL](../postgresql/guia-definitivo-postgresql.md). O Flyway versiona o schema; o Postgres é onde ele vive.
