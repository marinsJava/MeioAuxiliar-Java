# Guia Definitivo de Padrões de Projeto em Java (com exemplos de ORM)

> Um mapa completo dos 23 padrões GoF (Gang of Four) + os padrões de persistência que você usa todo dia no JPA/Hibernate/Spring Data — muitas vezes sem perceber.

A grande sacada deste guia: **um ORM como o Hibernate é praticamente um museu vivo de padrões de projeto**. Quase todo padrão clássico aparece em algum canto da camada de persistência. Em vez de decorar exemplos artificiais de "AnimalFactory", você vai reconhecer os padrões nas ferramentas que já usa.

---

## Índice

- [Como ler este guia](#como-ler-este-guia)
- [Parte 1 — Padrões Criacionais](#parte-1--padrões-criacionais)
  - [1. Singleton](#1-singleton)
  - [2. Factory Method](#2-factory-method)
  - [3. Abstract Factory](#3-abstract-factory)
  - [4. Builder](#4-builder)
  - [5. Prototype](#5-prototype)
- [Parte 2 — Padrões Estruturais](#parte-2--padrões-estruturais)
  - [6. Adapter](#6-adapter)
  - [7. Bridge](#7-bridge)
  - [8. Composite](#8-composite)
  - [9. Decorator](#9-decorator)
  - [10. Facade](#10-facade)
  - [11. Flyweight](#11-flyweight)
  - [12. Proxy](#12-proxy)
- [Parte 3 — Padrões Comportamentais](#parte-3--padrões-comportamentais)
  - [13. Chain of Responsibility](#13-chain-of-responsibility)
  - [14. Command](#14-command)
  - [15. Interpreter](#15-interpreter)
  - [16. Iterator](#16-iterator)
  - [17. Mediator](#17-mediator)
  - [18. Memento](#18-memento)
  - [19. Observer](#19-observer)
  - [20. State](#20-state)
  - [21. Strategy](#21-strategy)
  - [22. Template Method](#22-template-method)
  - [23. Visitor](#23-visitor)
- [Parte 4 — Padrões específicos de persistência (Fowler / PoEAA)](#parte-4--padrões-específicos-de-persistência)
- [Mapa rápido: onde cada padrão aparece no JPA/Hibernate](#mapa-rápido)
- [Leitura recomendada](#leitura-recomendada)

---

## Como ler este guia

Os 23 padrões GoF se dividem em três famílias:

| Família | Pergunta que responde | Exemplos |
|---|---|---|
| **Criacionais** | *Como* objetos são criados? | Singleton, Factory, Builder |
| **Estruturais** | Como objetos se *compõem*? | Adapter, Decorator, Proxy |
| **Comportamentais** | Como objetos *colaboram* e distribuem responsabilidades? | Strategy, Observer, Template Method |

Cada padrão abaixo segue a estrutura: **intenção → exemplo Java → conexão com ORM**.

Um aviso honesto: nem todo padrão tem uma analogia perfeita com ORM. Onde a conexão é forçada, eu digo isso claramente em vez de inventar. Padrão de projeto não é troféu — é vocabulário para resolver problemas recorrentes.

---

## Parte 1 — Padrões Criacionais

### 1. Singleton

**Intenção:** garantir que uma classe tenha apenas uma instância e fornecer um ponto global de acesso a ela.

```java
// Versão moderna e thread-safe usando enum (recomendada por Joshua Bloch)
public enum ConfigManager {
    INSTANCE;

    private final Properties props = new Properties();

    public String get(String chave) {
        return props.getProperty(chave);
    }
}

// Uso:
String url = ConfigManager.INSTANCE.get("db.url");
```

A versão clássica com `getInstance()` e *double-checked locking* ainda é comum em entrevistas:

```java
public class ConfigManager {
    private static volatile ConfigManager instancia;

    private ConfigManager() {}

    public static ConfigManager getInstance() {
        if (instancia == null) {                 // 1ª checagem (sem lock)
            synchronized (ConfigManager.class) {
                if (instancia == null) {          // 2ª checagem (com lock)
                    instancia = new ConfigManager();
                }
            }
        }
        return instancia;
    }
}
```

**Conexão com ORM:** a `SessionFactory` do Hibernate (ou a `EntityManagerFactory` do JPA) é o exemplo canônico. Ela é cara de construir (lê metadados, monta o pool de conexões, compila mapeamentos) e por isso é criada **uma única vez** por aplicação e compartilhada entre todas as threads. As `Session`/`EntityManager`, ao contrário, são baratas e descartáveis (uma por requisição/transação).

```java
public class HibernateUtil {
    private static final SessionFactory FACTORY = buildSessionFactory();

    private static SessionFactory buildSessionFactory() {
        return new Configuration().configure().buildSessionFactory();
    }

    public static SessionFactory getSessionFactory() {
        return FACTORY;
    }
}
```

> ⚠️ No Spring você quase nunca escreve isso à mão: o container gerencia o `EntityManagerFactory` como um *bean* singleton. Aliás, **todo bean Spring com escopo padrão é um Singleton** gerenciado pelo container — só que sem o acoplamento global do `getInstance()`.

---

### 2. Factory Method

**Intenção:** definir uma interface para criar um objeto, mas deixar as subclasses decidirem qual classe instanciar. Adia a instanciação para subclasses.

```java
public abstract class Dialogo {
    // Factory Method — subclasses decidem o produto concreto
    protected abstract Botao criarBotao();

    public void renderizar() {
        Botao botao = criarBotao();
        botao.desenhar();
    }
}

public class DialogoWindows extends Dialogo {
    @Override
    protected Botao criarBotao() {
        return new BotaoWindows();
    }
}

public class DialogoWeb extends Dialogo {
    @Override
    protected Botao criarBotao() {
        return new BotaoHtml();
    }
}
```

**Conexão com ORM:** `EntityManagerFactory.createEntityManager()` é literalmente um *factory method* — você pede um `EntityManager` sem saber (nem se importar com) a classe concreta retornada pelo provedor (Hibernate, EclipseLink etc.). O mesmo vale para `SessionFactory.openSession()`.

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("meuPU");
EntityManager em = emf.createEntityManager(); // factory method em ação
```

---

### 3. Abstract Factory

**Intenção:** fornecer uma interface para criar **famílias** de objetos relacionados, sem especificar suas classes concretas. É o Factory Method "elevado a uma família de produtos".

```java
public interface GUIFactory {
    Botao criarBotao();
    CheckBox criarCheckBox();
}

public class WindowsFactory implements GUIFactory {
    public Botao criarBotao()       { return new BotaoWindows(); }
    public CheckBox criarCheckBox() { return new CheckBoxWindows(); }
}

public class MacFactory implements GUIFactory {
    public Botao criarBotao()       { return new BotaoMac(); }
    public CheckBox criarCheckBox() { return new CheckBoxMac(); }
}

// Cliente trabalha só com a interface — a família inteira vem coesa
public class Aplicacao {
    private final Botao botao;
    private final CheckBox checkbox;

    public Aplicacao(GUIFactory factory) {
        this.botao = factory.criarBotao();
        this.checkbox = factory.criarCheckBox();
    }
}
```

**Conexão com ORM:** a especificação JPA é, na prática, uma *abstract factory*. Trocar a propriedade do provedor faz toda a "família" de implementações mudar de uma vez — `EntityManagerFactory`, `EntityManager`, `Query`, `CriteriaBuilder` passam a ser os do Hibernate **ou** os do EclipseLink, sempre coerentes entre si. Seu código continua programando contra as interfaces `jakarta.persistence.*`.

---

### 4. Builder

**Intenção:** separar a construção de um objeto complexo da sua representação, permitindo criar objetos passo a passo de forma legível e imutável.

```java
public class Pessoa {
    private final String nome;        // obrigatório
    private final String email;       // obrigatório
    private final String telefone;    // opcional
    private final LocalDate nascimento; // opcional

    private Pessoa(Builder b) {
        this.nome = b.nome;
        this.email = b.email;
        this.telefone = b.telefone;
        this.nascimento = b.nascimento;
    }

    public static Builder builder(String nome, String email) {
        return new Builder(nome, email);
    }

    public static class Builder {
        private final String nome;
        private final String email;
        private String telefone;
        private LocalDate nascimento;

        public Builder(String nome, String email) {
            this.nome = nome;
            this.email = email;
        }

        public Builder telefone(String t)        { this.telefone = t; return this; }
        public Builder nascimento(LocalDate d)    { this.nascimento = d; return this; }

        public Pessoa build() {
            return new Pessoa(this); // valide invariantes aqui se quiser
        }
    }
}

// Uso fluente:
Pessoa p = Pessoa.builder("Matheus", "matheus@exemplo.com")
                 .telefone("21999990000")
                 .nascimento(LocalDate.of(1995, 5, 20))
                 .build();
```

**Conexão com ORM (três frentes):**

1. **Lombok `@Builder`** — você já usa isso. Em entidades, evite gerar o builder por cima de campos `final`/JPA cegamente; combine com `@NoArgsConstructor` (exigido pelo JPA) e `@AllArgsConstructor`:

   ```java
   @Entity
   @Getter @Setter
   @NoArgsConstructor
   @AllArgsConstructor
   @Builder
   public class Medicamento {
       @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
       private Long id;
       private String nome;
       private String dosagem;
   }

   Medicamento m = Medicamento.builder()
       .nome("Losartana")
       .dosagem("50mg")
       .build();
   ```

2. **Criteria API** — o `CriteriaBuilder` é um builder de consultas SQL type-safe:

   ```java
   CriteriaBuilder cb = em.getCriteriaBuilder();
   CriteriaQuery<Medicamento> cq = cb.createQuery(Medicamento.class);
   Root<Medicamento> root = cq.from(Medicamento.class);
   cq.select(root).where(cb.like(root.get("nome"), "L%"));
   ```

3. A própria configuração do Hibernate é construída com um builder (`Configuration().configure().buildSessionFactory()`).

---

### 5. Prototype

**Intenção:** criar novos objetos copiando uma instância existente (o "protótipo"), em vez de instanciar do zero. Útil quando a criação é cara ou a configuração é complexa.

```java
public class Documento implements Cloneable {
    private String titulo;
    private List<String> paragrafos = new ArrayList<>();

    @Override
    public Documento clone() {
        try {
            Documento copia = (Documento) super.clone();
            // cópia profunda da lista para não compartilhar referência
            copia.paragrafos = new ArrayList<>(this.paragrafos);
            return copia;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

**Conexão com ORM:** aqui a analogia é mais sutil. Um uso comum é **"clonar para inserir"** — pegar uma entidade já carregada, copiar seus campos (menos o `@Id`) e persistir como um novo registro. Isso evita repreencher um objeto complexo manualmente. No Spring, `@Scope("prototype")` em um bean é o mesmo conceito de nomenclatura, mas com semântica diferente (cria nova instância a cada injeção; não clona uma existente).

> 💡 Cuidado: **nunca** use `clone()` ingênuo em entidades gerenciadas. Você arrasta o `@Id` e o estado da *persistence context*. O padrão correto é um construtor de cópia ou um *mapper* (MapStruct) que copia só os dados de negócio.


### Tabela de decisão — quando usar (e quando não usar) padrões criacionais
 
| Sintoma / problema concreto | Padrão indicado | Quando **NÃO** usar |
|---|---|---|
| O construtor tem muitos parâmetros, vários opcionais, e fica ilegível; ou você quer um objeto imutável montado passo a passo | **Builder** | Objeto com 2–3 campos simples → `new` direto é mais claro |
| A classe concreta a instanciar só é conhecida em runtime (depende de configuração, tipo de entrada, ambiente) | **Factory Method** | Você sempre instancia a mesma classe → a factory é cerimônia vazia |
| Precisa criar **famílias** de objetos relacionados que devem ser coerentes entre si (ex.: todos do mesmo provedor/tema) | **Abstract Factory** | Só existe uma família, ou os objetos não se relacionam → exagero |
| Ter mais de uma instância seria de fato um problema (cache compartilhado, pool, registro central) | **Singleton** — e, em apps Spring, **deixe o container fazer** (bean singleton) | Em projeto Spring, evite `getInstance()` manual; prefira um bean. Fora disso, raramente justifica o acoplamento global |
| Criar do zero é caro/complexo e copiar um molde pré-configurado sai mais barato | **Prototype** | Criação já é barata → copiar só adiciona risco (cópia rasa vs. profunda) |
| Nenhum dos acima — o objeto se cria de forma clara e direta | **Nenhum padrão: use `new`** | — |
 
> A linha mais importante é a última. Antes de aplicar qualquer padrão criacional, faça o teste mental: *"se eu usar `new` aqui, o que dá errado?"*. Se a resposta for "nada, fica claro e funciona", nenhum padrão é necessário.
 
---

## Parte 2 — Padrões Estruturais

### 6. Adapter

**Intenção:** converter a interface de uma classe em outra interface que o cliente espera. Permite que classes com interfaces incompatíveis trabalhem juntas.

```java
// Interface que meu sistema espera
public interface Notificador {
    void enviar(String destino, String mensagem);
}

// Biblioteca externa com API incompatível
public class ApiSmsLegada {
    public void dispatch(String numero, String texto, int prioridade) { /* ... */ }
}

// Adapter faz a ponte
public class SmsAdapter implements Notificador {
    private final ApiSmsLegada api = new ApiSmsLegada();

    @Override
    public void enviar(String destino, String mensagem) {
        api.dispatch(destino, mensagem, 1); // adapta a chamada
    }
}
```

**Conexão com ORM:** os *dialects* do Hibernate (`PostgreSQLDialect`, `MySQLDialect`, `OracleDialect`) são adapters: o Hibernate fala uma "linguagem" interna de geração de SQL, e cada *dialect* adapta isso para o SQL específico de cada banco. Você troca o banco trocando o adapter. Drivers JDBC também são adapters entre a API `java.sql` e o protocolo nativo de cada SGBD.

---

### 7. Bridge

**Intenção:** desacoplar uma abstração da sua implementação, para que as duas variem independentemente. Evita a explosão combinatória de subclasses.

```java
// Implementação (a "ponte")
public interface Renderizador {
    void renderizarCirculo(float raio);
}

public class RenderizadorVetorial implements Renderizador { /* ... */ }
public class RenderizadorRaster implements Renderizador { /* ... */ }

// Abstração — referencia a implementação, não herda dela
public abstract class Forma {
    protected final Renderizador renderizador;
    protected Forma(Renderizador r) { this.renderizador = r; }
    public abstract void desenhar();
}

public class Circulo extends Forma {
    private final float raio;
    public Circulo(Renderizador r, float raio) { super(r); this.raio = raio; }
    public void desenhar() { renderizador.renderizarCirculo(raio); }
}
```

Sem Bridge, você teria `CirculoVetorial`, `CirculoRaster`, `QuadradoVetorial`... (N formas × M renderizadores). Com Bridge, `N + M` classes.

**Conexão com ORM:** a separação **JPA (abstração) ↔ provedor (implementação)** é uma bridge em escala de arquitetura. Sua aplicação programa contra `EntityManager`; a implementação concreta (Hibernate) varia independentemente. Outro exemplo: a JPA é a abstração e o **JDBC** é a implementação por baixo — você pode trocar o driver/banco sem mudar a abstração.

---

### 8. Composite

**Intenção:** compor objetos em estruturas de árvore para representar hierarquias parte-todo. Permite tratar objetos individuais e composições de forma uniforme.

```java
public interface ComponenteSistemaArquivos {
    long tamanho();
}

public class Arquivo implements ComponenteSistemaArquivos {
    private final long bytes;
    public Arquivo(long bytes) { this.bytes = bytes; }
    public long tamanho() { return bytes; }
}

public class Pasta implements ComponenteSistemaArquivos {
    private final List<ComponenteSistemaArquivos> filhos = new ArrayList<>();
    public void adicionar(ComponenteSistemaArquivos c) { filhos.add(c); }

    public long tamanho() {
        // trata folha e composto da mesma forma, recursivamente
        return filhos.stream().mapToLong(ComponenteSistemaArquivos::tamanho).sum();
    }
}
```

**Conexão com ORM:** entidades **auto-relacionadas** modelam composites direto no banco. Categoria com subcategorias, comentários com respostas, organograma de funcionários:

```java
@Entity
public class Categoria {
    @Id @GeneratedValue private Long id;
    private String nome;

    @ManyToOne
    private Categoria pai;

    @OneToMany(mappedBy = "pai", cascade = CascadeType.ALL)
    private List<Categoria> filhas = new ArrayList<>();
}
```

A árvore de objetos que o Hibernate materializa **é** a estrutura Composite.

---

### 9. Decorator

**Intenção:** anexar responsabilidades adicionais a um objeto dinamicamente, envolvendo-o. Alternativa flexível à herança para estender comportamento.

```java
public interface FonteDados {
    void escrever(String dado);
    String ler();
}

// Decorator base
public abstract class DecoradorFonteDados implements FonteDados {
    protected final FonteDados envolvido;
    protected DecoradorFonteDados(FonteDados f) { this.envolvido = f; }
}

public class CriptografiaDecorator extends DecoradorFonteDados {
    public CriptografiaDecorator(FonteDados f) { super(f); }

    public void escrever(String dado) {
        envolvido.escrever(criptografar(dado)); // adiciona comportamento
    }
    public String ler() { return descriptografar(envolvido.ler()); }
    // criptografar/descriptografar omitidos
}

// Uso: empilha decorators
FonteDados fonte = new CompressaoDecorator(
                       new CriptografiaDecorator(
                           new ArquivoFonteDados("dados.txt")));
```

O `java.io` inteiro é Decorator: `new BufferedReader(new InputStreamReader(new FileInputStream(...)))`.

**Conexão com ORM:** o exemplo mais direto é o **`DataSource` decorado** — proxies de pool de conexão (HikariCP) e wrappers que adicionam logging/métricas de SQL (como `datasource-proxy` ou o `P6Spy`) envolvem o `DataSource` real sem que o Hibernate saiba. Você "decora" a fonte de dados com observabilidade.

---

### 10. Facade

**Intenção:** fornecer uma interface unificada e simples para um subsistema complexo. Esconde a complexidade interna atrás de uma fachada.

```java
// Subsistema complexo
class Estoque    { boolean reservar(Long produtoId, int qtd) { /*...*/ return true; } }
class Pagamento  { String cobrar(Long clienteId, BigDecimal valor) { /*...*/ return "OK"; } }
class Entrega    { void agendar(Long pedidoId) { /*...*/ } }

// Facade
public class PedidoFacade {
    private final Estoque estoque = new Estoque();
    private final Pagamento pagamento = new Pagamento();
    private final Entrega entrega = new Entrega();

    public void finalizarPedido(Long clienteId, Long produtoId, BigDecimal valor) {
        estoque.reservar(produtoId, 1);
        pagamento.cobrar(clienteId, valor);
        entrega.agendar(produtoId);
    }
}
```

**Conexão com ORM:** a **camada `@Service`** do Spring é uma facade clássica sobre o subsistema de persistência. Em vez de o controller orquestrar vários repositórios, transações e mapeamentos, ele chama um único método de serviço. A própria interface `Session`/`EntityManager` também é uma facade: por trás de um `em.persist(obj)` simples existem a *persistence context*, o *dirty checking*, a fila de SQL e o *flush* — tudo escondido.

```java
@Service
public class MedicamentoService {
    // facade: esconde repositório, validação, mapeamento e transação
    @Transactional
    public MedicamentoDTO cadastrar(MedicamentoDTO dto) { /* orquestra tudo */ }
}
```

---

### 11. Flyweight

**Intenção:** usar compartilhamento para suportar grandes quantidades de objetos pequenos de forma eficiente, separando estado **intrínseco** (compartilhável) de **extrínseco** (contextual).

```java
// Estado intrínseco compartilhado (ex.: tipo de partícula)
public final class TipoParticula {
    private final String sprite;
    private final String cor;
    public TipoParticula(String sprite, String cor) { this.sprite = sprite; this.cor = cor; }
}

public class FabricaTipos {
    private static final Map<String, TipoParticula> cache = new HashMap<>();

    public static TipoParticula get(String sprite, String cor) {
        return cache.computeIfAbsent(sprite + "|" + cor,
                                     k -> new TipoParticula(sprite, cor));
    }
}
// 1 milhão de partículas, mas só alguns poucos TipoParticula em memória.
```

No JDK: `Integer.valueOf()` cacheia os valores de -128 a 127 (flyweight), e o *String pool* é flyweight de strings literais.

**Conexão com ORM:** o **cache de segundo nível** (2nd-level cache) do Hibernate é flyweight em espírito: entidades imutáveis e muito reusadas (tabelas de domínio, países, tipos) ficam carregadas uma vez e compartilhadas entre sessões, em vez de virem do banco a cada consulta. Marcar uma entidade como `@Immutable` + `@Cache` é exatamente "isto é compartilhável".

> 💬 Honestamente, essa é uma analogia "de espírito", não estrutural. O cache otimiza reuso, mas não separa estado intrínseco/extrínseco como o Flyweight clássico.

---

### 12. Proxy

**Intenção:** fornecer um substituto/marcador para outro objeto, controlando o acesso a ele. Variantes: proxy virtual (cria sob demanda), de proteção (controla acesso), remoto (objeto em outra JVM).

```java
public interface Imagem {
    void exibir();
}

public class ImagemReal implements Imagem {
    private final String arquivo;
    public ImagemReal(String arquivo) {
        this.arquivo = arquivo;
        carregarDoDisco(); // operação cara!
    }
    public void exibir() { /* desenha */ }
    private void carregarDoDisco() { /* lê arquivo pesado */ }
}

// Proxy virtual: adia o carregamento caro até ser realmente necessário
public class ImagemProxy implements Imagem {
    private final String arquivo;
    private ImagemReal real;

    public ImagemProxy(String arquivo) { this.arquivo = arquivo; }

    public void exibir() {
        if (real == null) {
            real = new ImagemReal(arquivo); // carrega só agora (lazy)
        }
        real.exibir();
    }
}
```

**Conexão com ORM — este é o exemplo mais importante do guia.** O **lazy loading** do Hibernate é Proxy virtual puro. Quando você carrega uma entidade com associação `LAZY`, o Hibernate não busca os relacionados; ele coloca um **proxy** no lugar (gerado em runtime via CGLIB/ByteBuddy). O SQL só dispara quando você acessa o objeto pela primeira vez.

```java
@Entity
public class Receita {
    @Id @GeneratedValue private Long id;

    @ManyToOne(fetch = FetchType.LAZY) // <- Hibernate injeta um PROXY aqui
    private Paciente paciente;
}

Receita r = em.find(Receita.class, 1L);
// r.getPaciente() neste momento é um proxy "vazio" (só com o id)
String nome = r.getPaciente().getNome(); // <- AQUI o SELECT do paciente dispara
```

Isso também explica a temida `LazyInitializationException`: se você acessa o proxy **depois** da `Session` fechar, ele não consegue mais buscar os dados. É o controle de acesso do Proxy falhando porque o "objeto real" ficou inalcançável.

> 🔧 O `getReference()` do JPA também devolve um proxy explicitamente — útil para associar por id sem ir ao banco.

---

## Parte 3 — Padrões Comportamentais

### 13. Chain of Responsibility

**Intenção:** passar uma requisição por uma cadeia de handlers; cada um decide tratá-la ou repassá-la ao próximo. Desacopla emissor de receptor.

```java
public abstract class Handler {
    protected Handler proximo;
    public Handler encadear(Handler h) { this.proximo = h; return h; }
    public abstract void tratar(Requisicao req);

    protected void passarAdiante(Requisicao req) {
        if (proximo != null) proximo.tratar(req);
    }
}

public class AutenticacaoHandler extends Handler {
    public void tratar(Requisicao req) {
        if (!req.autenticado()) throw new SecurityException("401");
        passarAdiante(req);
    }
}
public class RateLimitHandler extends Handler {
    public void tratar(Requisicao req) {
        if (excedeuLimite(req)) throw new RuntimeException("429");
        passarAdiante(req);
    }
}
```

**Conexão com ORM:** menos direta, mas o **`EntityInterceptor` / cadeia de listeners** do Hibernate e, sobretudo, a **cadeia de filtros do Spring Security** (que protege seus endpoints de dados) são Chain of Responsibility. Cada filtro decide se trata e/ou repassa a requisição.

---

### 14. Command

**Intenção:** encapsular uma requisição como um objeto, permitindo parametrizar, enfileirar, registrar e desfazer operações.

```java
public interface Comando {
    void executar();
    void desfazer();
}

public class InserirTextoComando implements Comando {
    private final Documento doc;
    private final String texto;
    public InserirTextoComando(Documento doc, String texto) {
        this.doc = doc; this.texto = texto;
    }
    public void executar() { doc.inserir(texto); }
    public void desfazer() { doc.remover(texto.length()); }
}

// Invoker mantém histórico para undo
public class Editor {
    private final Deque<Comando> historico = new ArrayDeque<>();
    public void executar(Comando c) { c.executar(); historico.push(c); }
    public void desfazer() { if (!historico.isEmpty()) historico.pop().desfazer(); }
}
```

**Conexão com ORM:** o **`ActionQueue` do Hibernate** é Command em estado puro. Cada `persist`, `merge` ou `remove` que você chama **não** vira SQL imediatamente — vira uma *ação* (objeto-comando) enfileirada. No `flush()`, o Hibernate executa a fila na ordem correta. Por isso a ordem dos seus `persist` no código nem sempre é a ordem dos `INSERT` no banco: a fila de comandos reordena por dependência.

---

### 15. Interpreter

**Intenção:** dada uma linguagem, definir uma representação para sua gramática junto com um interpretador que usa essa representação para interpretar sentenças.

```java
public interface Expressao {
    boolean interpretar(Map<String, Boolean> contexto);
}

public class Variavel implements Expressao {
    private final String nome;
    public Variavel(String nome) { this.nome = nome; }
    public boolean interpretar(Map<String, Boolean> ctx) {
        return ctx.getOrDefault(nome, false);
    }
}
public class E implements Expressao { // operador AND
    private final Expressao esq, dir;
    public E(Expressao e, Expressao d) { this.esq = e; this.dir = d; }
    public boolean interpretar(Map<String, Boolean> ctx) {
        return esq.interpretar(ctx) && dir.interpretar(ctx);
    }
}
```

**Conexão com ORM:** o **HQL/JPQL** é uma linguagem com gramática própria, e o Hibernate tem um **interpretador** (parser + árvore de sintaxe + tradutor) que converte `"FROM Medicamento m WHERE m.dosagem = :d"` em SQL. A **Criteria API** representa essa mesma gramática como uma árvore de objetos `Expression`/`Predicate` — que é literalmente o padrão Interpreter materializado em objetos Java.

---

### 16. Iterator

**Intenção:** fornecer uma forma de acessar sequencialmente os elementos de uma coleção sem expor sua representação interna.

```java
// Java já dá isso de graça via Iterable/Iterator
public class Playlist implements Iterable<Musica> {
    private final List<Musica> musicas = new ArrayList<>();

    @Override
    public Iterator<Musica> iterator() {
        return musicas.iterator();
    }
}

// for-each usa o Iterator por baixo
for (Musica m : new Playlist()) { /* ... */ }
```

**Conexão com ORM:** o método `ScrollableResults` / `Query.getResultStream()` do Hibernate permite **iterar sobre resultados grandes** sem carregar tudo em memória de uma vez — um iterator que busca do banco sob demanda (página a página). É o que salva você de um `OutOfMemoryError` ao processar milhões de linhas.

```java
try (Stream<Medicamento> stream = em
        .createQuery("FROM Medicamento", Medicamento.class)
        .getResultStream()) {
    stream.forEach(this::processar); // itera lazy, sem carregar tudo
}
```

---

### 17. Mediator

**Intenção:** definir um objeto que encapsula como um conjunto de objetos interage, promovendo baixo acoplamento ao impedir que eles se refiram uns aos outros diretamente.

```java
public interface MediadorChat {
    void enviar(String msg, Usuario remetente);
}

public class SalaDeChat implements MediadorChat {
    private final List<Usuario> usuarios = new ArrayList<>();
    public void registrar(Usuario u) { usuarios.add(u); }

    public void enviar(String msg, Usuario remetente) {
        // usuários não se conhecem; o mediador roteia
        usuarios.stream()
                .filter(u -> u != remetente)
                .forEach(u -> u.receber(msg));
    }
}
```

**Conexão com ORM:** aqui a analogia é fraca, e prefiro dizer isso a forçar. O exemplo arquitetural mais próximo é o **`ApplicationEventPublisher` do Spring** (e os *domain events* publicados a partir de entidades): em vez de um serviço chamar outro diretamente, ele publica um evento e o mediador (o publisher) entrega aos interessados. Útil para desacoplar efeitos colaterais de persistência (ex.: "medicamento cadastrado" → cria evento no Google Calendar).

---

### 18. Memento

**Intenção:** capturar e externalizar o estado interno de um objeto (sem violar encapsulamento) para que ele possa ser restaurado depois.

```java
public class EditorTexto {
    private String conteudo = "";

    public void digitar(String t) { conteudo += t; }

    public Memento salvar()            { return new Memento(conteudo); }
    public void restaurar(Memento m)   { this.conteudo = m.estado(); }

    // Memento: estado imutável, opaco para o mundo externo
    public record Memento(String estado) {}
}

EditorTexto editor = new EditorTexto();
editor.digitar("Olá");
EditorTexto.Memento ponto = editor.salvar(); // snapshot
editor.digitar(" mundo");
editor.restaurar(ponto); // volta para "Olá"
```

**Conexão com ORM:** o Hibernate guarda um **snapshot do estado original** de cada entidade no momento em que ela é carregada na *persistence context*. No `flush`, ele compara o estado atual com esse snapshot (*dirty checking*) para gerar apenas os `UPDATE` necessários. Esse snapshot interno é, conceitualmente, um Memento. Em outro nível, frameworks de auditoria como **Hibernate Envers** materializam mementos como linhas de histórico que permitem "voltar no tempo" de uma entidade.

---

### 19. Observer

**Intenção:** definir uma dependência um-para-muitos entre objetos, de forma que quando um muda de estado, todos os dependentes são notificados automaticamente.

```java
public interface Observador {
    void atualizar(BigDecimal novoPreco);
}

public class Acao { // Subject/Observable
    private final List<Observador> observadores = new ArrayList<>();
    private BigDecimal preco;

    public void inscrever(Observador o) { observadores.add(o); }

    public void setPreco(BigDecimal p) {
        this.preco = p;
        notificar(); // avisa todo mundo
    }
    private void notificar() {
        observadores.forEach(o -> o.atualizar(preco));
    }
}
```

**Conexão com ORM:** os **callbacks de ciclo de vida do JPA** são Observer. Você "observa" eventos da entidade e reage:

```java
@Entity
public class Medicamento {
    @Id @GeneratedValue private Long id;
    private Instant criadoEm;
    private Instant atualizadoEm;

    @PrePersist
    void aoCriar()    { this.criadoEm = Instant.now(); }   // observa o INSERT

    @PreUpdate
    void aoAtualizar(){ this.atualizadoEm = Instant.now(); } // observa o UPDATE
}
```

Os `@EntityListeners` e os eventos do Hibernate (`PostInsertEventListener` etc.) são o mesmo padrão num nível mais baixo. Sua anotação favorita `@CreationTimestamp`/`@UpdateTimestamp` é açúcar sintático em cima desses observers.

---

### 20. State

**Intenção:** permitir que um objeto altere seu comportamento quando seu estado interno muda. O objeto parece mudar de classe.

```java
public interface EstadoPedido {
    EstadoPedido pagar(Pedido p);
    EstadoPedido enviar(Pedido p);
}

public class Novo implements EstadoPedido {
    public EstadoPedido pagar(Pedido p)  { return new Pago(); }    // transição válida
    public EstadoPedido enviar(Pedido p) {
        throw new IllegalStateException("Não pode enviar pedido não pago");
    }
}
public class Pago implements EstadoPedido {
    public EstadoPedido pagar(Pedido p)  { throw new IllegalStateException("Já pago"); }
    public EstadoPedido enviar(Pedido p) { return new Enviado(); }
}

public class Pedido {
    private EstadoPedido estado = new Novo();
    public void pagar()  { estado = estado.pagar(this); }
    public void enviar() { estado = estado.enviar(this); }
}
```

**Conexão com ORM:** internamente o JPA modela o **ciclo de vida da entidade** como uma máquina de estados — `transient` (nova), `managed` (gerenciada), `detached` (desconectada), `removed` (marcada para remoção). Operações como `persist`, `merge`, `detach`, `remove` são as transições. Compreender esses estados é, na prática, entender o State pattern da *persistence context*:

```
new Entidade()  --persist-->  MANAGED  --remove-->  REMOVED  --flush-->  (deletada)
                                 |
                              detach / close
                                 v
                              DETACHED  --merge-->  MANAGED
```

---

### 21. Strategy

**Intenção:** definir uma família de algoritmos, encapsular cada um e torná-los intercambiáveis. A estratégia varia independentemente dos clientes que a usam.

```java
public interface EstrategiaFrete {
    BigDecimal calcular(Pedido pedido);
}

public class FreteSedex implements EstrategiaFrete {
    public BigDecimal calcular(Pedido p) { /* cálculo Sedex */ return BigDecimal.TEN; }
}
public class FreteRetirada implements EstrategiaFrete {
    public BigDecimal calcular(Pedido p) { return BigDecimal.ZERO; }
}

public class CarrinhoService {
    public BigDecimal total(Pedido p, EstrategiaFrete frete) {
        return p.subtotal().add(frete.calcular(p)); // injeta a estratégia
    }
}
```

**Conexão com ORM (em todo lugar):**

- **Geração de id:** `GenerationType.IDENTITY`, `SEQUENCE`, `TABLE`, `AUTO` são estratégias de geração de chave primária intercambiáveis via uma única anotação.
  ```java
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE) // troque a estratégia aqui
  private Long id;
  ```
- **Fetch:** `FetchType.LAZY` vs `EAGER` é estratégia de carregamento.
- **Nomenclatura:** a `PhysicalNamingStrategy`/`ImplicitNamingStrategy` define como nomes de classe viram nomes de tabela/coluna.
- **Dialect:** já citado no Adapter, também tem cara de estratégia de geração de SQL.

No Spring você troca a estratégia injetando outra implementação do mesmo bean — Strategy + injeção de dependência andam de mãos dadas.

---

### 22. Template Method

**Intenção:** definir o esqueleto de um algoritmo em um método, deixando alguns passos para as subclasses. Subclasses redefinem passos sem mudar a estrutura geral.

```java
public abstract class ProcessadorRelatorio {
    // template method — define a ordem fixa
    public final void gerar() {
        abrirConexao();
        var dados = buscarDados();   // passo variável (abstrato)
        var doc = formatar(dados);   // passo variável (abstrato)
        salvar(doc);
        fecharConexao();
    }

    protected abstract List<?> buscarDados();
    protected abstract String formatar(List<?> dados);

    private void abrirConexao()  { /* comum a todos */ }
    private void fecharConexao() { /* comum a todos */ }
    private void salvar(String doc) { /* comum a todos */ }
}
```

**Conexão com ORM:** o **`JdbcTemplate`** do Spring é o exemplo de manual: ele faz o "esqueleto" (abrir conexão → preparar statement → executar → **fechar conexão e tratar exceções**) e você só fornece o passo variável (o SQL e o `RowMapper`). Toda a parte tediosa e propensa a erro (vazar conexão!) está no template.

```java
List<Medicamento> lista = jdbcTemplate.query(
    "SELECT * FROM medicamento WHERE dosagem = ?",
    (rs, rowNum) -> new Medicamento(rs.getLong("id"), rs.getString("nome")), // só o passo variável
    "50mg");
```

O `TransactionTemplate` segue a mesma ideia para transações. E quando você estende `JpaRepository`, o Spring Data fornece o esqueleto das operações CRUD e você só declara o que varia (a interface e queries derivadas).

---

### 23. Visitor

**Intenção:** representar uma operação a ser executada sobre os elementos de uma estrutura de objetos, permitindo definir novas operações sem alterar as classes dos elementos.

```java
public interface Visitante {
    void visitar(Livro livro);
    void visitar(Eletronico eletronico);
}

public interface ItemCarrinho {
    void aceitar(Visitante v);
}

public class Livro implements ItemCarrinho {
    public void aceitar(Visitante v) { v.visitar(this); } // double dispatch
}

// Nova operação sem tocar nas classes de item:
public class CalculadoraImposto implements Visitante {
    public void visitar(Livro l)       { /* livro: imposto reduzido */ }
    public void visitar(Eletronico e)  { /* eletrônico: imposto cheio */ }
}
```

**Conexão com ORM:** internamente, os tradutores de HQL/Criteria do Hibernate percorrem a **árvore de sintaxe (AST)** da consulta usando visitors para gerar SQL — cada tipo de nó (projeção, junção, predicado) é "visitado" por um tradutor. É um padrão de infraestrutura do framework, raramente algo que você escreve na camada de aplicação. Sendo honesto: você quase nunca implementa Visitor manualmente em uma app CRUD típica.

---

## Parte 4 — Padrões específicos de persistência

Os padrões abaixo **não** são GoF — vêm do *Patterns of Enterprise Application Architecture* (Martin Fowler). São os que mais aparecem no seu dia a dia com Spring Data e Hibernate.

### Repository

**Intenção:** mediar entre o domínio e a camada de mapeamento de dados, agindo como uma coleção em memória de objetos de domínio. Esconde o "como buscar" atrás de uma interface de coleção.

```java
public interface MedicamentoRepository extends JpaRepository<Medicamento, Long> {
    // query derivada: o Spring Data implementa para você
    List<Medicamento> findByDosagem(String dosagem);

    @Query("SELECT m FROM Medicamento m WHERE m.nome LIKE %:termo%")
    List<Medicamento> buscarPorNome(String termo);
}
```

> É o padrão mais visível do Spring Data. Você declara a interface; o Spring gera a implementação em runtime (sim — usando Proxy, fechando o ciclo do padrão #12).

### DAO (Data Access Object)

Primo mais antigo do Repository. O DAO é orientado a **tabela/fonte de dados** (`MedicamentoDao` com `inserir`, `atualizar`, `deletarPorId`), enquanto o Repository é orientado a **coleção de domínio**. Na prática, no mundo Spring, o Repository venceu — mas você verá DAO em código legado e em livros.

### Unit of Work

**Intenção:** manter uma lista de objetos afetados por uma transação de negócio e coordenar a escrita das mudanças e a resolução de problemas de concorrência. → **A `persistence context` / `EntityManager` É o Unit of Work do JPA.** Ela rastreia tudo que você alterou e, no commit/flush, persiste como uma unidade atômica. É por isso que basta alterar um campo de uma entidade gerenciada (sem chamar `save`) e o `UPDATE` acontece sozinho.

### Identity Map

**Intenção:** garantir que cada objeto seja carregado apenas uma vez, mantendo um mapa dos objetos já carregados. → **O cache de primeiro nível (1st-level cache) do Hibernate.** Dentro da mesma `Session`, dois `em.find(Medicamento.class, 1L)` retornam **a mesma instância** Java (`==` é `true`) e disparam **um** único SELECT. Isso garante consistência de identidade dentro da transação.

```java
Medicamento a = em.find(Medicamento.class, 1L); // SELECT no banco
Medicamento b = em.find(Medicamento.class, 1L); // vem do Identity Map, sem SQL
System.out.println(a == b); // true
```

### Lazy Load

Já coberto em profundidade no Proxy (#12) — é o mesmo mecanismo, e Fowler o lista como padrão de persistência por si só, com quatro variantes (lazy initialization, virtual proxy, value holder, ghost).

### Data Mapper vs Active Record

- **Data Mapper:** uma camada separa os objetos de domínio do banco; o domínio não sabe que existe persistência. → **É a filosofia do JPA/Hibernate.** Sua entidade `Medicamento` é (idealmente) um POJO; quem mapeia para o banco é o EntityManager.
- **Active Record:** o próprio objeto sabe se salvar (`medicamento.save()`). → Estilo do Rails/Eloquent; no Java aparece em frameworks como o ActiveJDBC, e é o oposto da abordagem JPA.

### Specification

**Intenção:** encapsular regras de negócio/critérios de consulta como objetos combináveis. → **Spring Data `Specification`** (sobre a Criteria API) permite montar filtros dinâmicos compostos:

```java
public class MedicamentoSpecs {
    public static Specification<Medicamento> comDosagem(String dosagem) {
        return (root, query, cb) -> cb.equal(root.get("dosagem"), dosagem);
    }
    public static Specification<Medicamento> nomeContem(String termo) {
        return (root, query, cb) -> cb.like(root.get("nome"), "%" + termo + "%");
    }
}

// Combina critérios de forma fluente (ótimo para filtros opcionais de tela):
repo.findAll(comDosagem("50mg").and(nomeContem("losartana")));
```

> Isso resolve elegantemente o problema do `@RequestParam` opcional que você já enfrentou: em vez de empilhar `if`s e queries variantes, você compõe specifications conforme os filtros que vierem preenchidos.

---

## Mapa rápido

Onde cada padrão te encontra no JPA/Hibernate/Spring:

| Padrão GoF | Onde aparece na persistência |
|---|---|
| Singleton | `EntityManagerFactory` / `SessionFactory`; beans Spring |
| Factory Method | `emf.createEntityManager()`, `sf.openSession()` |
| Abstract Factory | A especificação JPA gerando a família do provedor |
| Builder | `@Builder` (Lombok), `CriteriaBuilder`, `Configuration` |
| Prototype | Construtor de cópia "clonar-para-inserir"; `@Scope("prototype")` |
| Adapter | `Dialect`s do Hibernate; drivers JDBC |
| Bridge | JPA (abstração) ↔ provedor / JDBC (implementação) |
| Composite | Entidades auto-relacionadas (árvore de categorias) |
| Decorator | `DataSource` decorado (HikariCP, P6Spy, métricas) |
| Facade | Camada `@Service`; a própria interface `EntityManager` |
| Flyweight | Cache de 2º nível (entidades imutáveis compartilhadas) |
| **Proxy** | **Lazy loading** (o exemplo nº 1); `getReference()` |
| Chain of Responsibility | Filtros do Spring Security; interceptors |
| Command | `ActionQueue` (fila de ações antes do flush) |
| Interpreter | Parser de HQL/JPQL; árvore da Criteria API |
| Iterator | `getResultStream()`, `ScrollableResults` |
| Mediator | `ApplicationEventPublisher` / domain events |
| Memento | Snapshot de dirty checking; Hibernate Envers |
| Observer | `@PrePersist`/`@PostLoad`, `@EntityListeners` |
| State | Ciclo de vida da entidade (transient→managed→detached→removed) |
| Strategy | `GenerationType`, `FetchType`, naming strategy |
| Template Method | `JdbcTemplate`, `TransactionTemplate`, `JpaRepository` |
| Visitor | Tradução interna da AST de consultas (infraestrutura) |

| Padrão Fowler (PoEAA) | Onde aparece |
|---|---|
| Repository | `JpaRepository` (Spring Data) |
| DAO | DAOs em código legado |
| Unit of Work | `EntityManager` / persistence context |
| Identity Map | Cache de 1º nível |
| Lazy Load | Lazy loading (= Proxy) |
| Data Mapper | A filosofia do JPA |
| Specification | `Specification` do Spring Data |

---

## Leitura recomendada

1. **Design Patterns** — Gamma, Helm, Johnson, Vlissides (o livro "GoF" original; denso, mas é a fonte).
2. **Head First Design Patterns** — Freeman & Robson (a porta de entrada didática; exemplos em Java).
3. **Patterns of Enterprise Application Architecture** — Martin Fowler (a fonte dos padrões da Parte 4).
4. **Effective Java** — Joshua Bloch (itens sobre Builder, Singleton com enum, e quando *não* usar padrões).
5. **Refactoring.Guru** — referência online gratuita e visual, com exemplos em Java.

Uma última nota para o seu projeto de Clean Code/Design Patterns: o erro mais comum em trabalho acadêmico é **espalhar padrões para impressionar**. O bom uso é o oposto — aplicar um padrão só quando ele remove uma dor real (duplicação, acoplamento, rigidez). Os avaliadores experientes percebem na hora a diferença entre "usei Strategy porque o requisito tinha variação de algoritmo" e "usei Strategy porque a rubrica pedia um padrão comportamental".
