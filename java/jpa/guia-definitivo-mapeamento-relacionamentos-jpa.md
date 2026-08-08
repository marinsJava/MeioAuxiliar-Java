# Guia Definitivo: Mapeamento de Relacionamentos JPA/Hibernate

> Todas as anotações de relacionamento do JPA — `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany` — com o **modelo mental correto**, as decisões que você toma em cada uma e as armadilhas que todo desenvolvedor encontra (e geralmente descobre tarde demais). Stack de referência: **Spring Boot 4.1 + Hibernate 7 + PostgreSQL**.

O maior problema com relacionamentos JPA não é decorar anotações — é não entender o que acontece **por baixo**: quando o SQL é gerado, por que o N+1 aparece, o que `mappedBy` realmente faz e por que `CascadeType.ALL` em `@ManyToMany` costuma ser um erro. Esse guia resolve isso: primeiro o modelo mental, depois as anotações.

---

## Índice

- [O modelo mental: dono do relacionamento](#o-modelo-mental-dono-do-relacionamento)
- [Fetch: quando os dados são carregados](#fetch-quando-os-dados-são-carregados)
- [Cascade: propagando operações](#cascade-propagando-operações)
- [@ManyToOne — o mais simples e o mais usado](#manytoone--o-mais-simples-e-o-mais-usado)
- [@OneToMany — coleção no lado pai](#onetomany--coleção-no-lado-pai)
- [@OneToOne — um para um](#onetoone--um-para-um)
- [@ManyToMany — muitos para muitos](#manytomany--muitos-para-muitos)
- [Entidade de junção: quando a tabela associativa tem dados](#entidade-de-junção-quando-a-tabela-associativa-tem-dados)
- [O problema N+1 e como evitá-lo](#o-problema-n1-e-como-evitá-lo)
- [Equals, hashCode e coleções JPA](#equals-hashcode-e-coleções-jpa)
- [Tabela de decisão rápida](#tabela-de-decisão-rápida)
- [Armadilhas comuns](#armadilhas-comuns)
- [Checklist de mapeamento](#checklist-de-mapeamento)
- [Referências](#referências)

---

## O modelo mental: dono do relacionamento

Toda vez que você mapeia um relacionamento, o JPA precisa saber **quem é responsável por manter a chave estrangeira no banco**. Esse lado é chamado de **dono do relacionamento** (*owning side*).

A regra é simples:

> **O dono é o lado que tem a coluna (ou tabela) de junção no banco. É o único lado que o JPA usa para gravar o relacionamento.**

O outro lado — o *inverse side* — existe apenas para navegação em código. Mudanças feitas só nele **não são persistidas**.

Como identificar cada lado:

| Indicador | O que significa |
|---|---|
| Tem `@JoinColumn` | É o **dono** |
| Tem `mappedBy` | É o **inverso** (não é dono) |
| Em `@ManyToMany`: tem `@JoinTable` | É o **dono** |

```
Banco:
  tabela pedido_item
    ├── id
    ├── pedido_id   ← chave estrangeira — dono desse relacionamento é PedidoItem
    └── produto_id
```

```java
// Dono: tem a FK, tem @JoinColumn
@ManyToOne
@JoinColumn(name = "pedido_id")
private Pedido pedido;

// Inverso: mappedBy aponta para o campo do dono
@OneToMany(mappedBy = "pedido")
private List<PedidoItem> itens;
```

> 💡 **Regra de bolso:** `mappedBy` sempre aponta para o **nome do atributo no lado dono**, não para o nome da tabela nem da coluna. Se o atributo se chama `pedido`, `mappedBy = "pedido"`.

---

## Fetch: quando os dados são carregados

Fetch define **se** os dados associados são carregados junto com a entidade ou só quando acessados:

| Estratégia | Comportamento | Padrão em |
|---|---|---|
| `EAGER` | Carrega tudo na mesma query (ou queries extras) imediatamente | `@ManyToOne`, `@OneToOne` |
| `LAZY` | Carrega só quando o atributo é acessado no código | `@OneToMany`, `@ManyToMany` |

```java
@ManyToOne(fetch = FetchType.LAZY)   // muda o padrão de EAGER para LAZY
@JoinColumn(name = "usuario_id")
private Usuario usuario;

@OneToMany(mappedBy = "pedido", fetch = FetchType.EAGER)   // força carregamento imediato
private List<PedidoItem> itens;
```

**A recomendação geral: prefira `LAZY` em tudo.** `EAGER` carrega dados que você talvez nunca use, e em grafos de entidades com vários relacionamentos EAGER aninhados, uma query simples pode virar um JOIN catastrófico. Carregue explicitamente o que precisar com JPQL/fetch join quando necessário.

> ⚠️ `LAZY` em `@ManyToOne` e `@OneToOne` tem uma pegadinha: como esses relacionamentos são de um único objeto, o Hibernate precisa de um proxy para adiá-los. Se a entidade alvo for `final` ou usar Lombok's `@Builder` sem construtor padrão, o proxy não consegue ser criado e o carregamento cai de volta para EAGER silenciosamente.

---

## Cascade: propagando operações

Cascade define se operações feitas na entidade pai (salvar, deletar etc.) propagam automaticamente para as entidades filhas:

| Tipo | O que propaga |
|---|---|
| `PERSIST` | `entityManager.persist()` — salvar |
| `MERGE` | `entityManager.merge()` — atualizar |
| `REMOVE` | `entityManager.remove()` — deletar |
| `REFRESH` | `entityManager.refresh()` — recarregar do banco |
| `DETACH` | Detacha a entidade da sessão |
| `ALL` | Todos os acima de uma vez |

```java
// Com PERSIST: salvar Pedido já salva os itens automaticamente
@OneToMany(mappedBy = "pedido", cascade = CascadeType.PERSIST)
private List<PedidoItem> itens;

// REMOVE com cuidado: deletar Pedido apaga os itens também
@OneToMany(mappedBy = "pedido", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<PedidoItem> itens;
```

**Quando usar cada um:**

- `{PERSIST, MERGE}` — o combo mais seguro para a maioria dos `@OneToMany`. Salva e atualiza junto, sem o risco de deletar em cascata por acidente;
- `ALL` — adequado para composição real (ex.: `Pedido → PedidoItem`), quando o filho **não existe sem o pai** e não é referenciado em outro lugar;
- **Nunca use `ALL` em `@ManyToMany`** — você deletaria entidades que outros relacionamentos referenciam.

**`orphanRemoval`** — complementa o cascade:

```java
@OneToMany(mappedBy = "pedido", cascade = CascadeType.ALL, orphanRemoval = true)
private List<PedidoItem> itens;
```

Com `orphanRemoval = true`, ao remover um item da coleção (`pedido.getItens().remove(item)`), o JPA emite um `DELETE` automaticamente no banco. Sem isso, o item fica "órfão" — sem pai na memória mas ainda existindo no banco.

---

## @ManyToOne — o mais simples e o mais usado

Representa "muitos registros desta tabela apontam para um da outra". A chave estrangeira fica **nesta entidade** — por isso ela sempre é a dona do relacionamento.

```java
@Entity
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "usuario_id",         // nome da coluna FK no banco
        nullable = false             // garante NOT NULL no DDL
    )
    private Usuario usuario;
}
```

**`@JoinColumn` em detalhe:**

| Atributo | Para que serve | Padrão (se omitido) |
|---|---|---|
| `name` | Nome da coluna FK no banco | `{nome_atributo}_id` |
| `nullable` | Se permite NULL | `true` |
| `unique` | Se é UNIQUE (converte em @OneToOne efetivamente) | `false` |
| `foreignKey` | Customiza o nome da constraint FK | gerado pelo Hibernate |
| `referencedColumnName` | A coluna na tabela alvo (quando não é a PK) | PK da entidade alvo |

Exemplo com nome de constraint explícito (boa prática para produção):

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(
    name = "usuario_id",
    nullable = false,
    foreignKey = @ForeignKey(name = "fk_pedido_usuario")
)
private Usuario usuario;
```

> 💡 Nomear as constraints explicitamente evita nomes gerados como `FK4sd8f9sd2` que aparecem nas mensagens de erro do banco e são impossíveis de rastrear.

---

## @OneToMany — coleção no lado pai

Representa "um registro desta tabela tem muitos da outra". A FK fica **na tabela filha** — por isso `@OneToMany` quase sempre vem com `mappedBy`, indicando que o dono real está do outro lado.

```java
@Entity
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(
        mappedBy = "usuario",                              // campo em Pedido que aponta para cá
        cascade = {CascadeType.PERSIST, CascadeType.MERGE},
        fetch = FetchType.LAZY,                            // padrão; explicitado por clareza
        orphanRemoval = true
    )
    private List<Pedido> pedidos = new ArrayList<>();      // inicializar evita NullPointerException
}
```

**A variante sem `mappedBy` — com tabela de junção:**

Quando `@OneToMany` é usado **sem** `mappedBy`, o Hibernate cria uma tabela associativa intermediária — o mesmo comportamento de um `@ManyToMany`. Isso raramente é o que você quer. Use `mappedBy` e deixe a FK na tabela filha, que é o design correto para 1:N.

**Métodos auxiliares de bidirecionalidade:**

Em relacionamentos bidirecionais, você precisa manter **ambos os lados sincronizados em memória** — o JPA persiste pelo dono, mas a consistência em código é sua responsabilidade:

```java
// Em Usuario:
public void adicionarPedido(Pedido pedido) {
    this.pedidos.add(pedido);
    pedido.setUsuario(this);       // mantém o lado dono sincronizado
}

public void removerPedido(Pedido pedido) {
    this.pedidos.remove(pedido);
    pedido.setUsuario(null);
}
```

> 💡 Sem esses métodos auxiliares, é fácil fazer `usuario.getPedidos().add(pedido)` e esquecer de setar `pedido.setUsuario(usuario)` — o registro entra na lista em memória mas a FK não é gravada no banco (porque o dono, `Pedido`, não foi atualizado).

---

## @OneToOne — um para um

Representa "um registro desta tabela corresponde a exatamente um da outra". Há duas formas de mapear:

### Forma 1 — FK na entidade atual (dono)

O lado que tem `@JoinColumn` é o dono e carrega a FK:

```java
@Entity
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(
        cascade = CascadeType.ALL,
        fetch = FetchType.LAZY,     // LAZY aqui exige configuração extra — veja a nota
        orphanRemoval = true
    )
    @JoinColumn(
        name = "endereco_id",
        unique = true,              // garante a unicidade no banco
        nullable = false
    )
    private Endereco endereco;
}
```

### Forma 2 — Compartilhando a PK (mais rara, mais elegante)

A entidade filha usa a mesma PK da entidade pai — sem coluna FK extra:

```java
@Entity
public class Endereco {

    @Id
    private Long id;                // mesma PK de Usuario

    @OneToOne(mappedBy = "endereco")
    @MapsId                         // indica que este @Id vem do relacionamento
    private Usuario usuario;
}
```

`@MapsId` é a anotação correta para compartilhar PK. O banco fica mais limpo (sem coluna extra) e a integridade é natural.

> ⚠️ **`LAZY` em `@OneToOne` é enganoso:** por design do JPA, o proxy só funciona se o relacionamento for `optional = false` (não pode ser nulo). Caso contrário, o Hibernate precisa verificar se o valor existe — o que exige uma query — e aí o lazy não funciona de verdade. Para relacionamentos opcionais, considere aceitar EAGER ou buscar via query específica quando necessário.

---

## @ManyToMany — muitos para muitos

Representa "muitos registros desta tabela associam-se com muitos da outra". O JPA cria (ou mapeia) uma **tabela de junção** para armazenar os pares.

```java
@Entity
public class Marmita {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "marmita_tag",                     // nome da tabela de junção
        joinColumns = @JoinColumn(
            name = "marmita_id",                  // FK para esta entidade
            foreignKey = @ForeignKey(name = "fk_marmita_tag_marmita")
        ),
        inverseJoinColumns = @JoinColumn(
            name = "tag_id",                      // FK para a outra entidade
            foreignKey = @ForeignKey(name = "fk_marmita_tag_tag")
        )
    )
    private Set<Tag> tags = new HashSet<>();       // Set evita duplicatas
}
```

O lado inverso (sem `@JoinTable`, com `mappedBy`):

```java
@Entity
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(mappedBy = "tags")
    private Set<Marmita> marmitas = new HashSet<>();
}
```

**`Set` vs `List` em `@ManyToMany`:**

Prefira `Set`. O Hibernate tem um comportamento problemático com `List` em `@ManyToMany`: ao remover qualquer elemento, ele apaga **todos** os registros da tabela de junção e reinsere os que sobraram — um `DELETE` seguido de vários `INSERT`s desnecessários. `Set` não tem esse problema.

> ⚠️ **Nunca use `CascadeType.REMOVE` ou `CascadeType.ALL` em `@ManyToMany`.** Se você deletar uma `Marmita` e o cascade se propagar, todas as `Tag`s associadas seriam deletadas — mesmo que outras marmitas as usem. O `@ManyToMany` não tem dono de ciclo de vida claro; cada entidade existe independentemente.

---

## Entidade de junção: quando a tabela associativa tem dados

A tabela de junção de um `@ManyToMany` suporta apenas as duas FKs. Se você precisar armazenar dados extras na associação (quantidade, data, observação), a solução é transformar a tabela de junção numa **entidade própria** e substituir o `@ManyToMany` por dois `@ManyToOne`:

**Antes — `@ManyToMany` simples (tabela só com FKs):**
```
pedido_item (pedido_id, produto_id)
```

**Depois — entidade de junção com dados extras:**
```
pedido_item (id, pedido_id, produto_id, quantidade, preco_unitario)
```

```java
@Entity
@Table(name = "pedido_item")
public class PedidoItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "pedido_id", nullable = false,
                foreignKey = @ForeignKey(name = "fk_pedido_item_pedido"))
    private Pedido pedido;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "produto_id", nullable = false,
                foreignKey = @ForeignKey(name = "fk_pedido_item_produto"))
    private Produto produto;

    @Column(nullable = false)
    private Integer quantidade;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal precoUnitario;
}
```

E os lados pai passam a usar `@OneToMany`:

```java
// Em Pedido:
@OneToMany(mappedBy = "pedido", cascade = CascadeType.ALL, orphanRemoval = true)
private List<PedidoItem> itens = new ArrayList<>();

// Em Produto:
@OneToMany(mappedBy = "produto")
private List<PedidoItem> itensDePedido = new ArrayList<>();
```

> 💡 **Na prática, esta é a forma mais comum de lidar com N:M em projetos reais.** A tabela de junção "pura" raramente sobrevive — quase sempre aparecem dados extras que precisam ser armazenados na associação. Começar com entidade de junção é a escolha mais robusta.

---

## O problema N+1 e como evitá-lo

N+1 é a causa mais comum de lentidão silenciosa em aplicações JPA. Acontece quando o Hibernate executa **1 query para buscar a lista** e depois **N queries extras** para carregar os relacionamentos de cada item:

```java
// Parece inocente:
List<Pedido> pedidos = pedidoRepository.findAll();
pedidos.forEach(p -> System.out.println(p.getUsuario().getNome())); // ← dispara 1 query por pedido
```

Se `findAll()` retornar 100 pedidos, isso gera **101 queries** no banco.

### Solução 1 — JPQL com `JOIN FETCH`

Instrui o Hibernate a buscar o relacionamento na mesma query:

```java
@Query("SELECT p FROM Pedido p JOIN FETCH p.usuario WHERE p.status = :status")
List<Pedido> findByStatusComUsuario(@Param("status") StatusPedido status);
```

SQL gerado: um único `SELECT ... JOIN` — eficiente.

### Solução 2 — `@EntityGraph`

Alternativa declarativa, sem escrever JPQL:

```java
@EntityGraph(attributePaths = {"usuario", "itens"})
List<Pedido> findByStatus(StatusPedido status);
```

Útil quando você quer variar o que é carregado por método, sem duplicar JPQL.

### Solução 3 — Projeções / DTOs diretos

Quando você não precisa da entidade completa — só de alguns campos — projete direto para um DTO via JPQL ou interface de projeção:

```java
@Query("SELECT new br.com.exemplo.dto.PedidoResumo(p.id, u.nome, p.total) " +
       "FROM Pedido p JOIN p.usuario u WHERE p.status = :status")
List<PedidoResumo> findResumoByStatus(@Param("status") StatusPedido status);
```

Sem entidade carregada, sem proxy, sem N+1.

**Como detectar:** ative o log de SQL no `application.yml` durante o desenvolvimento:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # mostra os parâmetros
```

Quando você ver um bloco de queries repetitivas com parâmetros crescentes, é N+1.

---

## Equals, hashCode e coleções JPA

Entidades JPA em coleções (`Set`, `List` como `HashSet`) exigem cuidado especial com `equals` e `hashCode`. O problema: o `id` gerado pelo banco (via `@GeneratedValue`) é `null` antes do primeiro `persist`. Se você usar o id no `hashCode`, a entidade muda de "posição" no Set depois do persist — e coleções como `HashSet` perdem a referência.

**A solução recomendada** — usar a PK mas tratar o caso de null:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Pedido other)) return false;
    return id != null && id.equals(other.id);
}

@Override
public int hashCode() {
    return getClass().hashCode();   // constante — seguro antes e depois do persist
}
```

`hashCode` constante por classe não é ideal para performance em Sets grandes, mas é **correto** para entidades JPA — o requisito do contrato `equals/hashCode` é que o valor não mude durante o ciclo de vida do objeto na coleção.

> ⚠️ **Cuidado com Lombok:** `@Data` e `@EqualsAndHashCode` gerados pelo Lombok em entidades JPA incluem todos os campos por padrão — incluindo os relacionamentos, o que pode causar `StackOverflowError` em bidirecionais e os problemas de hashCode descritos acima. Prefira `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` com `@EqualsAndHashCode.Include` na PK, ou escreva `equals`/`hashCode` manualmente.

---

## Tabela de decisão rápida

Consulta rápida para escolher a anotação e as configurações certas:

| Situação | Anotação | Dono | Fetch padrão | Cascade recomendado |
|---|---|---|---|---|
| Muitos X têm um Y (FK em X) | `@ManyToOne` | X (tem `@JoinColumn`) | EAGER | nenhum |
| Um X tem muitos Y | `@OneToMany(mappedBy)` | Y (o `@ManyToOne`) | LAZY | `{PERSIST, MERGE}` ou `ALL + orphanRemoval` |
| Um X tem um Y (FK em X) | `@OneToOne` + `@JoinColumn` | X | EAGER | `ALL + orphanRemoval` |
| Um X tem um Y (PK compartilhada) | `@OneToOne` + `@MapsId` | filho | EAGER | `ALL` |
| Muitos X têm muitos Y (só FKs) | `@ManyToMany` + `@JoinTable` | quem tem `@JoinTable` | LAZY | `{PERSIST, MERGE}` |
| Muitos X têm muitos Y (com dados extras) | Entidade de junção + dois `@ManyToOne` | a entidade de junção | LAZY | `ALL + orphanRemoval` no pai principal |

---

## Armadilhas comuns

1. **`mappedBy` errado** — `mappedBy` aponta para o **nome do atributo Java** no lado dono, não para o nome da coluna ou da tabela. `mappedBy = "usuario"` funciona se o campo em `Pedido` se chama `private Usuario usuario`. Se chamar `private Usuario user`, tem que ser `mappedBy = "user"`.

2. **`CascadeType.ALL` em `@ManyToMany`** — ao deletar uma entidade, você deleta as entidades do outro lado que outros registros referenciam. Cascata de remoção em N:M é quase sempre errado.

3. **Bidirecional não sincronizado** — você adiciona no lado inverso e esquece o dono. O JPA grava pelo dono; se o dono não foi atualizado, a FK não é salva. Use métodos auxiliares para manter os dois lados sincronizados.

4. **`LazyInitializationException`** — você acessa um relacionamento LAZY fora de uma sessão Hibernate ativa (fora de `@Transactional`). Soluções: buscar com JOIN FETCH, usar `@Transactional` no service, ou usar projeções DTO.

5. **`StackOverflowError` em `toString`/`equals`** — Lombok's `@Data` em relacionamento bidirecional gera `toString` que navega de A para B e de B para A infinitamente. Use `@ToString(exclude = "pedidos")` ou `@JsonManagedReference`/`@JsonBackReference` para serialização JSON.

6. **`List` em `@ManyToMany` causando deletes extras** — o Hibernate apaga tudo e reinsere ao modificar qualquer elemento. Prefira `Set`.

7. **`@JoinColumn` nos dois lados** — em bidirecional, só o dono tem `@JoinColumn`. O inverso tem `mappedBy`. Colocar `@JoinColumn` nos dois cria duas FKs no banco ou comportamentos imprevisíveis.

8. **`orphanRemoval` sem `CascadeType.PERSIST`** — se você usar `orphanRemoval = true` mas não cascatear o `PERSIST`, ao adicionar um novo filho na coleção e salvar o pai, o filho não é persistido automaticamente. Use `CASCADE = {PERSIST, MERGE}` ou `ALL` junto com `orphanRemoval`.

9. **`FetchType.EAGER` em várias coleções** — duas coleções EAGER na mesma entidade geram um `MultipleBagFetchException` (com `List`) ou um produto cartesiano explosivo (com `Set`). Use LAZY e carregue explicitamente o que precisar.

10. **`@GeneratedValue` com `@OneToOne @MapsId` ignorado** — ao usar `@MapsId`, a PK do filho é determinada pelo pai. Se você também puser `@GeneratedValue` no filho, ele é ignorado. Não coloque os dois juntos.

---

## Checklist de mapeamento

Para revisar quando mapear um relacionamento novo:

- [ ] Identifiquei quem é o **dono** (quem tem a FK/tabela de junção no banco)?
- [ ] O dono tem `@JoinColumn` (ou `@JoinTable` em N:M)?
- [ ] O inverso tem `mappedBy` apontando para o **nome do atributo** no dono?
- [ ] O `fetch` está explícito e é `LAZY` por padrão (exceto quando há necessidade real de EAGER)?
- [ ] O `cascade` está calibrado (sem `ALL` em `@ManyToMany`, sem `REMOVE` acidental)?
- [ ] Usei `orphanRemoval = true` onde o filho não existe sem o pai?
- [ ] Inicializei as coleções (`= new ArrayList<>()` ou `= new HashSet<>()`)?
- [ ] Em `@ManyToMany`, usei `Set` em vez de `List`?
- [ ] Se o relacionamento bidirecional, implementei métodos auxiliares para manter os dois lados sincronizados?
- [ ] `equals`/`hashCode` está correto para JPA (sem incluir coleções, sem quebrar antes do persist)?
- [ ] Nomeei as constraints de FK explicitamente com `@ForeignKey(name = "...")`?
- [ ] Testei o N+1: ativei o log SQL e confirmei que as queries são as esperadas?

---

## Referências

1. **Jakarta Persistence 3.2 — Specification** — https://jakarta.ee/specifications/persistence/3.2/
2. **Hibernate ORM — Association Mappings** — https://docs.jboss.org/hibernate/orm/7.0/userguide/html_single/Hibernate_User_Guide.html#associations
3. **Vlad Mihalcea — High-Performance Java Persistence** — a referência mais completa sobre N+1, fetch strategies e otimização de relacionamentos JPA
4. **Spring Data JPA — @EntityGraph** — https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.entity-graph
5. **Guia de Padrões de Projeto em Java (com ORM)** — [../design-patterns/guia-padroes-projeto-java-orm.md](../design-patterns/guia-padroes-projeto-java-orm.md)
6. **Guia Definitivo de Flyway** — [../../bancos-de-dados/flyway/guia-definitivo-flyway.md](../../bancos-de-dados/flyway/guia-definitivo-flyway.md)

> Regra de bolso para lembrar de tudo: **o dono grava, o inverso navega; LAZY por padrão; cascade só onde faz sentido de ciclo de vida; N:M com dados extras vira entidade de junção.**
