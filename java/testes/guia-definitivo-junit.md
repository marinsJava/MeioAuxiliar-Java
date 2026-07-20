# Guia Definitivo de JUnit (Testes em Java)

> Do primeiro `@Test` a testes de integração com Testcontainers, com código pronto para copiar. Stack de referência: **JUnit 6 + Spring Boot 4.1 + Mockito + AssertJ** (Java 21).

Teste automatizado não é burocracia — é a rede que te deixa refatorar sem medo. Este guia cobre o ciclo completo: a mecânica do JUnit, mocks com Mockito, os *test slices* do Spring, e testes de integração com banco real.

> ⚠️ **Atenção à versão:** o Spring Boot 4 adota **JUnit 6** como base e **removeu o suporte a JUnit 4** por completo. Além disso, `@MockBean` e `@SpyBean` **foram removidos** na 4.0 — o correto agora é `@MockitoBean`/`@MockitoSpyBean`. Boa parte dos tutoriais na internet ainda mostra o código antigo, que simplesmente não compila. Este guia usa a API atual.

---

## Índice

- [JUnit 6: o que mudou](#junit-6-o-que-mudou)
- [Setup](#setup)
- [Anatomia de um teste (padrão AAA)](#anatomia-de-um-teste)
- [Ciclo de vida: @BeforeEach e amigos](#ciclo-de-vida)
- [Assertions: JUnit vs. AssertJ](#assertions)
- [Testando exceptions](#testando-exceptions)
- [Organização: @DisplayName, @Nested, @Disabled](#organização)
- [Testes parametrizados](#testes-parametrizados)
- [Mockito: isolando dependências](#mockito-isolando-dependências)
- [A pirâmide de testes no Spring](#a-pirâmide-de-testes-no-spring)
- [Teste de unidade da camada de serviço](#teste-de-unidade-da-camada-de-serviço)
- [@WebMvcTest — a camada web](#webmvctest--a-camada-web)
- [@DataJpaTest — a camada de persistência](#datajpatest--a-camada-de-persistência)
- [@SpringBootTest — integração completa](#springboottest--integração-completa)
- [Testando segurança](#testando-segurança)
- [Testcontainers: banco real nos testes](#testcontainers-banco-real-nos-testes)
- [Cobertura com JaCoCo](#cobertura-com-jacoco)
- [📋 Tabela de referência rápida](#tabela-de-referência-rápida)
- [Armadilhas comuns](#armadilhas-comuns)
- [✅ Checklist](#checklist)
- [Referências](#referências)

---

## JUnit 6: o que mudou

O JUnit 6 chegou em setembro de 2025 e é a base do Spring Boot 4. A boa notícia: migrar do JUnit 5.x para o 6 é **muito** mais simples que foi do 4 para o 5 — na prática, para quem já está no 5.14 com Java 17+, é quase só um *bump* de versão.

O que efetivamente mudou:

- **Baseline Java 17+** (o Spring Boot 4 pede 17, mas 21 LTS é a recomendação).
- **Versionamento unificado** — Platform, Jupiter e Vintage agora compartilham o mesmo número de versão (antes o Platform seguia outro esquema, fonte clássica de confusão no `pom.xml`).
- **Anotações de nulidade (JSpecify)** em toda a API, melhorando o suporte da IDE.
- **JUnit Vintage depreciado** — o motor que rodava testes JUnit 4 agora é só uma ponte temporária de migração.
- Suporte melhor a `@Nested` e um modelo de cancelamento para pipelines de CI.

A arquitetura permanece a mesma três-em-um: **Platform** (lança os testes), **Jupiter** (a API que você escreve) e **Vintage** (compatibilidade com JUnit 3/4). Quando você escreve `@Test`, está usando o Jupiter.

---

## Setup

Com Spring Boot, é uma dependência só — o starter já traz **JUnit Jupiter, AssertJ, Mockito, Hamcrest** e o suporte de teste do Spring:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Não declare versão — o BOM do Spring Boot gerencia. Os testes ficam em `src/test/java`, **espelhando** a estrutura de `src/main/java`:

```
src/
├── main/java/com/exemplo/medicamento/MedicamentoService.java
└── test/java/com/exemplo/medicamento/MedicamentoServiceTest.java
```

Rodar: `mvn test` (unitários) ou `mvn verify` (inclui os de integração, sufixo `*IT`).

---

## Anatomia de um teste

O padrão universal é **AAA: Arrange, Act, Assert** (preparar, agir, verificar). Manter essa estrutura visível torna qualquer teste legível em segundos:

```java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CalculadoraTest {

    @Test
    void deveSomarDoisNumerosPositivos() {
        // Arrange — prepara o cenário
        Calculadora calc = new Calculadora();

        // Act — executa a ação sob teste
        int resultado = calc.somar(2, 3);

        // Assert — verifica o resultado
        assertThat(resultado).isEqualTo(5);
    }
}
```

Três convenções que valem adotar:

- **A classe de teste não precisa ser `public`** no JUnit Jupiter (nem os métodos). Deixe *package-private*.
- **Nomeie o método descrevendo o comportamento**, não o método testado: `deveLancarExcecaoQuandoMedicamentoNaoExiste` diz muito mais que `testBuscar`.
- **Um comportamento por teste.** Se você precisa de "e também verifica que...", provavelmente são dois testes.

---

## Ciclo de vida

Quatro anotações controlam o que roda antes/depois:

```java
class MedicamentoServiceTest {

    @BeforeAll                       // uma vez, ANTES de todos (precisa ser static)
    static void setupGeral() { }

    @BeforeEach                      // antes de CADA teste
    void setup() { }

    @AfterEach                       // depois de CADA teste
    void limpar() { }

    @AfterAll                        // uma vez, DEPOIS de todos (static)
    static void teardownGeral() { }
}
```

O ponto crucial: **o JUnit cria uma nova instância da classe de teste para cada método**. Isso garante isolamento — um teste não contamina o outro por meio de campos de instância. É por isso que `@BeforeAll`/`@AfterAll` precisam ser `static` (rodam fora de qualquer instância).

Use `@BeforeEach` para montar o estado comum; evite lógica pesada ali, ou seus testes ficam lentos e acoplados.

---

## Assertions

O JUnit traz assertions próprias, mas o `spring-boot-starter-test` inclui o **AssertJ**, cuja API fluente é bem mais legível e é a preferida na prática:

```java
import static org.assertj.core.api.Assertions.*;

// Básico
assertThat(resultado).isEqualTo(5);
assertThat(nome).isEqualTo("Losartana").startsWith("Los").hasSize(9);  // encadeável!
assertThat(ativo).isTrue();
assertThat(objeto).isNotNull();

// Coleções — onde o AssertJ brilha
assertThat(medicamentos)
    .hasSize(3)
    .extracting(Medicamento::getNome)
    .containsExactly("Losartana", "Dipirona", "Omeprazol");

assertThat(lista).isNotEmpty().doesNotContainNull();

// Optional
assertThat(repository.findById(1L)).isPresent();
assertThat(repository.findById(99L)).isEmpty();

// Objetos por campo (sem precisar de equals)
assertThat(medicamento)
    .extracting("nome", "dosagemMg")
    .containsExactly("Losartana", 50);
```

Compare a legibilidade com o equivalente em JUnit puro (`assertEquals(5, resultado)`, `assertTrue(...)`) — o AssertJ ganha em expressividade e nas mensagens de erro, que dizem exatamente o que diferiu.

> 💡 Para vários assertions relacionados, use `assertAll` (JUnit) ou `SoftAssertions` (AssertJ): todos são avaliados e você vê **todas** as falhas de uma vez, em vez de parar na primeira.

---

## Testando exceptions

Verificar que algo falha corretamente é tão importante quanto verificar o caminho feliz:

```java
// Com JUnit
@Test
void deveLancarExcecaoQuandoMedicamentoNaoExiste() {
    var ex = assertThrows(ResourceNotFoundException.class,
                          () -> service.buscar(999L));
    assertThat(ex.getMessage()).contains("não encontrado");
}

// Com AssertJ (mais fluente)
@Test
void deveRejeitarDataNoPassado() {
    assertThatThrownBy(() -> service.cadastrar(dtoComDataPassada))
        .isInstanceOf(BusinessRuleException.class)
        .hasMessageContaining("não pode estar no passado");
}

// Verificar que NÃO lança
assertThatCode(() -> service.cadastrar(dtoValido)).doesNotThrowAnyException();
```

Isso conecta direto com o [guia de tratamento de exceptions](../spring-boot/guia-tratamento-de-exceptions.md): cada exception customizada que você cria merece um teste provando que é lançada na condição certa.

---

## Organização

```java
@DisplayName("Serviço de Medicamentos")
class MedicamentoServiceTest {

    @Nested
    @DisplayName("Ao buscar por id")
    class Buscar {
        @Test
        @DisplayName("retorna o medicamento quando existe")
        void retornaQuandoExiste() { }

        @Test
        @DisplayName("lança exceção quando não existe")
        void lancaQuandoNaoExiste() { }
    }

    @Nested
    @DisplayName("Ao cadastrar")
    class Cadastrar {
        @Test
        void salvaComDadosValidos() { }

        @Test
        @Disabled("Aguardando definição da regra de estoque")
        void validaEstoque() { }
    }
}
```

`@Nested` agrupa cenários relacionados e deixa o relatório de testes legível como uma especificação. `@DisplayName` permite frases com acentos e espaços. `@Disabled` desliga um teste — sempre **com justificativa**, senão vira lixo esquecido.

---

## Testes parametrizados

Quando o mesmo teste vale para várias entradas, não copie e cole — parametrize:

```java
@ParameterizedTest
@ValueSource(strings = {"", "  ", "\t"})
void deveRejeitarNomeEmBranco(String nomeInvalido) {
    assertThatThrownBy(() -> service.cadastrar(new MedicamentoDTO(nomeInvalido, 50)))
        .isInstanceOf(BusinessRuleException.class);
}

@ParameterizedTest(name = "{0} mg deve ser válido: {1}")
@CsvSource({
    "50,  true",
    "0,   false",
    "-10, false"
})
void deveValidarDosagem(int dosagem, boolean esperado) {
    assertThat(validador.dosagemValida(dosagem)).isEqualTo(esperado);
}

@ParameterizedTest
@EnumSource(StatusPedido.class)
void todoStatusTemDescricao(StatusPedido status) {
    assertThat(status.getDescricao()).isNotBlank();
}

// Para casos complexos: uma fábrica de argumentos
@ParameterizedTest
@MethodSource("cenariosDeDesconto")
void deveCalcularDesconto(BigDecimal valor, int qtd, BigDecimal esperado) {
    assertThat(service.calcular(valor, qtd)).isEqualByComparingTo(esperado);
}

static Stream<Arguments> cenariosDeDesconto() {
    return Stream.of(
        Arguments.of(new BigDecimal("100"), 1, new BigDecimal("100")),
        Arguments.of(new BigDecimal("100"), 10, new BigDecimal("90"))
    );
}
```

Fontes de dados: `@ValueSource` (valores simples), `@CsvSource` (linhas inline), `@CsvFileSource` (arquivo CSV), `@EnumSource` (todos os valores de um enum), `@MethodSource` (um método que devolve `Stream<Arguments>` — o mais flexível).

---

## Mockito: isolando dependências

Para testar um serviço **em isolamento**, você substitui suas dependências por **mocks** — objetos falsos que você programa. Assim o teste não depende de banco, rede ou de outro componente.

```java
@ExtendWith(MockitoExtension.class)     // ativa o Mockito no JUnit
class MedicamentoServiceTest {

    @Mock                                // dependência falsa
    MedicamentoRepository repository;

    @InjectMocks                         // a classe sob teste, com os mocks injetados
    MedicamentoService service;

    @Test
    void deveRetornarMedicamentoQuandoExiste() {
        // Arrange — programa o comportamento do mock
        var medicamento = new Medicamento(1L, "Losartana", 50);
        when(repository.findById(1L)).thenReturn(Optional.of(medicamento));

        // Act
        var resultado = service.buscar(1L);

        // Assert
        assertThat(resultado.nome()).isEqualTo("Losartana");
        verify(repository).findById(1L);          // confirma que foi chamado
    }

    @Test
    void deveLancarExcecaoQuandoNaoExiste() {
        when(repository.findById(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.buscar(999L))
            .isInstanceOf(ResourceNotFoundException.class);

        verify(repository, never()).save(any());   // garante que NÃO salvou
    }
}
```

Os verbos essenciais do Mockito:

| Comando | Para quê |
|---|---|
| `when(x).thenReturn(y)` | Programa o retorno de uma chamada |
| `when(x).thenThrow(ex)` | Faz o mock lançar exceção |
| `verify(mock).metodo()` | Confirma que foi chamado |
| `verify(mock, times(2))` / `never()` | Confirma a quantidade de chamadas |
| `any()`, `eq()`, `anyLong()` | *Matchers* para argumentos |
| `ArgumentCaptor` | Captura o argumento recebido para inspecionar |

O `ArgumentCaptor` resolve um caso muito comum — verificar **o que** foi salvo:

```java
@Test
void deveSalvarComDataDeCriacao() {
    service.cadastrar(new MedicamentoDTO("Dipirona", 500));

    var captor = ArgumentCaptor.forClass(Medicamento.class);
    verify(repository).save(captor.capture());

    assertThat(captor.getValue().getCriadoEm()).isNotNull();
}
```

> **Mock vs. Spy:** `@Mock` cria um objeto totalmente falso (todos os métodos retornam vazio até você programar). `@Spy` embrulha um objeto real, executando o comportamento verdadeiro exceto onde você sobrescreve. Prefira mocks; spies costumam sinalizar um design que poderia ser mais simples.

---

## A pirâmide de testes no Spring

Aqui está o modelo mental que organiza tudo. Você tem três níveis, e a proporção importa:

```
        ╱╲          Poucos: @SpringBootTest
       ╱  ╲         (sobe a app inteira — lentos, mas realistas)
      ╱────╲
     ╱      ╲       Alguns: test slices
    ╱        ╲      (@WebMvcTest, @DataJpaTest — sobem só uma fatia)
   ╱──────────╲
  ╱            ╲    Muitos: testes de unidade
 ╱______________╲   (JUnit + Mockito puro — milissegundos)
```

A regra: **suba o mínimo de contexto necessário.** Teste de unidade com Mockito roda em milissegundos e não precisa de Spring nenhum. Só suba contexto quando precisar testar a integração entre camadas. Uma suíte cheia de `@SpringBootTest` fica insuportavelmente lenta.

---

## Teste de unidade da camada de serviço

É onde mora a lógica de negócio e onde você deve concentrar a maior parte dos testes. **Sem Spring, sem banco** — só JUnit + Mockito, como no exemplo do Mockito acima. Rápido e focado.

---

## @WebMvcTest — a camada web

Sobe **apenas** a camada MVC (controllers, serialização JSON, validação, filtros de segurança) — sem serviços reais, sem banco. As dependências do controller são substituídas por `@MockitoBean`:

```java
@WebMvcTest(MedicamentoController.class)
class MedicamentoControllerTest {

    @Autowired MockMvc mvc;

    @MockitoBean                    // ⚠️ NÃO use @MockBean — removido no Spring Boot 4
    MedicamentoService service;

    @Test
    void deveRetornar200ComALista() throws Exception {
        when(service.listar()).thenReturn(List.of(new MedicamentoDTO("Losartana", 50)));

        mvc.perform(get("/medicamentos"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$[0].nome").value("Losartana"));
    }

    @Test
    void deveRetornar404QuandoNaoExiste() throws Exception {
        when(service.buscar(999L)).thenThrow(new ResourceNotFoundException("Medicamento", 999L));

        mvc.perform(get("/medicamentos/999"))
           .andExpect(status().isNotFound());
    }

    @Test
    void deveRetornar400ComCorpoInvalido() throws Exception {
        mvc.perform(post("/medicamentos")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"nome": "", "dosagemMg": -5}
                    """))
           .andExpect(status().isBadRequest());
    }
}
```

Esse último teste é valioso: ele valida de uma vez o Bean Validation **e** o seu `@RestControllerAdvice`, provando que entrada inválida vira `400` com o corpo certo.

> 🆕 O Spring Boot 4 introduziu o **`RestTestClient`**, uma API unificada para testar endpoints que substitui a escolha entre `MockMvc`, `WebTestClient` e `TestRestTemplate`. O `MockMvc` segue funcionando e é o mais difundido; vale conhecer o novo como caminho de futuro.

---

## @DataJpaTest — a camada de persistência

Sobe apenas o JPA (entidades, repositórios, `EntityManager`), com transação revertida ao fim de cada teste:

```java
@DataJpaTest
class MedicamentoRepositoryTest {

    @Autowired MedicamentoRepository repository;
    @Autowired TestEntityManager em;

    @Test
    void deveEncontrarPorDosagem() {
        em.persist(new Medicamento("Losartana", 50));
        em.persist(new Medicamento("Dipirona", 500));
        em.flush();

        var resultado = repository.findByDosagemMg(50);

        assertThat(resultado).hasSize(1)
            .extracting(Medicamento::getNome).containsExactly("Losartana");
    }
}
```

Por padrão o `@DataJpaTest` tenta usar um banco em memória (H2). **Isso é uma armadilha**: H2 não é PostgreSQL, e queries nativas, tipos como `JSONB` ou comportamentos específicos passam no teste e quebram em produção. Para testar contra o banco de verdade, use Testcontainers (adiante) e desligue a substituição:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class MedicamentoRepositoryIT { }
```

---

## @SpringBootTest — integração completa

Sobe o contexto inteiro da aplicação. Use com parcimônia, para validar fluxos ponta a ponta:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class MedicamentoIntegrationTest {

    @Autowired MockMvc mvc;

    @Test
    void deveCriarEBuscarMedicamento() throws Exception {
        mvc.perform(post("/medicamentos")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"nome": "Losartana", "dosagemMg": 50}
                    """))
           .andExpect(status().isCreated());

        mvc.perform(get("/medicamentos"))
           .andExpect(jsonPath("$[0].nome").value("Losartana"));
    }
}
```

> Dica de performance: o Spring **cacheia o contexto** entre classes de teste que tenham a mesma configuração. Cada variação (um `@MockitoBean` diferente, uma property diferente) cria um contexto novo e lento. Padronize suas configurações de teste para reaproveitar o cache.

---

## Testando segurança

Com o `spring-security-test` (já incluso no starter), você simula usuários autenticados:

```java
@WebMvcTest(MedicamentoController.class)
class MedicamentoSecurityTest {

    @Autowired MockMvc mvc;
    @MockitoBean MedicamentoService service;

    @Test
    void anonimoRecebe401() throws Exception {
        mvc.perform(get("/medicamentos")).andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void usuarioComumNaoDeleta() throws Exception {
        mvc.perform(delete("/medicamentos/1").with(csrf()))
           .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminDeleta() throws Exception {
        mvc.perform(delete("/medicamentos/1").with(csrf()))
           .andExpect(status().isNoContent());
    }
}
```

Testar os três caminhos — **anônimo, role errada, role certa** — é a melhor rede contra regressões de segurança em upgrades. Detalhes das regras no [guia de Configuração do Spring Security](../spring-security/guia-configuracao-spring-security.md).

---

## Testcontainers: banco real nos testes

Testcontainers sobe um **PostgreSQL de verdade em Docker**, só para o teste, e derruba ao final. Fim do "passou com H2, quebrou em produção".

> ⚠️ O **Testcontainers 2.0** trouxe mudanças que quebram código antigo: todos os módulos agora usam o prefixo `testcontainers-` e as classes de container mudaram de pacote. Código copiado de tutoriais antigos não compila.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
class MedicamentoIT {

    @Container
    @ServiceConnection            // ✨ configura datasource automaticamente
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:18");

    @Autowired MedicamentoRepository repository;

    @Test
    void devePersistirNoPostgresReal() {
        repository.save(new Medicamento("Losartana", 50));
        assertThat(repository.findAll()).hasSize(1);
    }
}
```

O `@ServiceConnection` é o grande facilitador: ele injeta URL, usuário e senha do container no `DataSource` sozinho — sem `@DynamicPropertySource` manual. E como o container é `static`, ele é reaproveitado por todos os testes da classe.

> Bônus: suas migrations Flyway rodam nesse banco real, testando também o schema versionado. Veja o [guia de Flyway](../../bancos-de-dados/flyway/guia-definitivo-flyway.md).

---

## Cobertura com JaCoCo

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

Rode `mvn test` e abra `target/site/jacoco/index.html`.

Uma palavra honesta sobre cobertura: **ela mede o que foi executado, não o que foi verificado.** Um teste sem nenhum `assert` que só chama métodos gera 100% de cobertura e zero valor. Use a métrica para achar **áreas não testadas** (especialmente regras de negócio), não como meta a perseguir. Perseguir 100% leva a testes inúteis de getters e setters.

---

## Tabela de referência rápida

| Preciso testar... | Use | Sobe contexto? |
|---|---|---|
| Lógica de negócio isolada | JUnit + `@ExtendWith(MockitoExtension.class)` | Não (rápido) |
| Controller, JSON, validação, status HTTP | `@WebMvcTest` + `@MockitoBean` | Só a camada web |
| Query de repositório, mapeamento JPA | `@DataJpaTest` | Só o JPA |
| Fluxo completo ponta a ponta | `@SpringBootTest` | Tudo (lento) |
| Comportamento com banco real | Testcontainers + `@ServiceConnection` | Tudo + Docker |
| Regras de autorização | `@WithMockUser` + `MockMvc` | Camada web |

| Anotação | Para quê |
|---|---|
| `@Test` | Marca um método de teste |
| `@BeforeEach` / `@AfterEach` | Roda antes/depois de cada teste |
| `@BeforeAll` / `@AfterAll` | Roda uma vez (métodos `static`) |
| `@DisplayName` | Nome legível no relatório |
| `@Nested` | Agrupa cenários relacionados |
| `@Disabled` | Desliga um teste (sempre com motivo) |
| `@ParameterizedTest` | Mesmo teste, várias entradas |
| `@Mock` / `@InjectMocks` | Mock e classe sob teste (Mockito puro) |
| `@MockitoBean` | Mock **no contexto Spring** (substitui `@MockBean`) |
| `@MockitoSpyBean` | Spy no contexto Spring (substitui `@SpyBean`) |

---

## Armadilhas comuns

1. **Usar `@MockBean`** → não existe mais no Spring Boot 4. Use `@MockitoBean` (pacote `org.springframework.test.context.bean.override.mockito`).
2. **Copiar exemplos de JUnit 4** (`@RunWith`, `org.junit.Test`, `@Before`) → não funciona. No Jupiter é `@ExtendWith`, `org.junit.jupiter.api.Test`, `@BeforeEach`.
3. **`@SpringBootTest` para tudo** → suíte lenta. Prefira unidade e *slices*.
4. **Testar com H2 e rodar PostgreSQL em produção** → falso positivo. Use Testcontainers para o que depende do banco.
5. **Teste sem assertion** → cobre linhas, não verifica nada. Todo teste precisa de pelo menos um `assertThat`.
6. **Testes dependentes de ordem** → cada teste deve montar seu próprio cenário e passar sozinho.
7. **Esquecer `.with(csrf())`** em `POST`/`DELETE` de testes com segurança stateful → `403` inesperado.
8. **Muitos `@MockitoBean` diferentes** → cada combinação cria um contexto novo, lentificando a suíte.
9. **Mockar o que você não possui** (bibliotecas de terceiros) → prefira testar contra a implementação real ou um *fake*.

---

## Checklist

Para consultar ao montar a suíte de um projeto:

- [ ] `spring-boot-starter-test` no `pom.xml` com `<scope>test</scope>`.
- [ ] Estrutura de `src/test/java` espelhando `src/main/java`.
- [ ] Testes de unidade (Mockito) para toda regra de negócio no serviço.
- [ ] `@WebMvcTest` para cada controller: caminho feliz, 404 e 400 (validação).
- [ ] `@DataJpaTest` para queries customizadas do repositório.
- [ ] Testes de segurança: anônimo, role errada, role certa.
- [ ] Testcontainers para o que depende de comportamento real do PostgreSQL.
- [ ] Um `@SpringBootTest` de fumaça cobrindo o fluxo principal.
- [ ] JaCoCo configurado — usado para achar buracos, não como meta.
- [ ] Nomes descritivos (`deveXQuandoY`) e um comportamento por teste.

---

## Referências

1. **JUnit — User Guide** — https://docs.junit.org/current/user-guide/
2. **Spring Boot — Testing** — https://docs.spring.io/spring-boot/reference/testing/index.html
3. **AssertJ** — https://assertj.github.io/doc/
4. **Mockito** — https://site.mockito.org/
5. **Testcontainers** — https://testcontainers.com/
6. **Novidades de teste no Spring Boot 4** — https://rieckpil.de/whats-new-for-testing-in-spring-boot-4-0-and-spring-framework-7/

> Regra de bolso: **teste comportamento, não implementação.** Se refatorar o interior de um método quebra o teste sem mudar o que o sistema faz, o teste estava acoplado demais. Bons testes sobrevivem à refatoração e falham só quando o comportamento muda de verdade.
