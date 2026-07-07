# Guia Definitivo de Swagger / OpenAPI

> Documentação de API que se gera sozinha a partir do código, com springdoc-openapi. Stack de referência: **Spring Boot 4.1 + springdoc-openapi 3.x** (a linha 3.x é a compatível com Spring Boot 4; a 2.8.x é para Spring Boot 3).

A melhor documentação de API é a que **nasce do código** e nunca fica desatualizada. É exatamente isso que o Swagger UI + OpenAPI entregam: você anota (pouco) o código, e ganha uma página interativa onde qualquer pessoa explora e testa a API sem sair do navegador.

---

## Índice

- [Swagger vs. OpenAPI vs. springdoc: quem é quem](#quem-é-quem)
- [Setup para Spring Boot 4](#setup-para-spring-boot-4)
- [O que você ganha de graça](#o-que-você-ganha-de-graça)
- [Enriquecendo a documentação com anotações](#enriquecendo-a-documentação)
- [Metadados globais da API](#metadados-globais-da-api)
- [Botão Authorize: documentando o JWT](#botão-authorize)
- [Validação vira documentação](#validação-vira-documentação)
- [Agrupando e escondendo endpoints](#agrupando-e-escondendo-endpoints)
- [Cuidados em produção](#cuidados-em-produção)
- [Referências](#referências)

---

## Quem é quem

Três nomes que se confundem, mas têm papéis distintos:

- **OpenAPI** — a *especificação*: um formato padrão (JSON/YAML) que descreve a API — endpoints, parâmetros, schemas, respostas. É o "contrato" legível por máquina.
- **Swagger** — o conjunto de *ferramentas* em torno dessa spec. O mais famoso é o **Swagger UI**, a página interativa que renderiza a spec.
- **springdoc-openapi** — a *biblioteca* que, num projeto Spring Boot, **gera a spec OpenAPI automaticamente** a partir dos seus controllers e serve o Swagger UI. É a cola entre o seu código e a documentação.

> ⚠️ Esqueça o **SpringFox** — a biblioteca antiga está abandonada desde 2020 e é incompatível com Spring Boot 3+. Em projetos modernos, springdoc-openapi é o caminho único.

---

## Setup para Spring Boot 4

Uma única dependência (o *starter* já traz o Swagger UI junto):

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.0.3</version>
</dependency>
```

> Atenção à compatibilidade: **springdoc 3.x → Spring Boot 4.x** (Spring Framework 7); **springdoc 2.8.x → Spring Boot 3.x**. Usar a linha errada gera erros de inicialização difíceis de diagnosticar. Para gerenciar a versão de forma centralizada, existe o `springdoc-openapi-bom`.

Configuração opcional no `application.yml` (para ajustar caminhos e ordenação):

```yaml
springdoc:
  swagger-ui:
    path: /swagger-ui.html          # caminho da página interativa
    operations-sorter: alpha        # ordena endpoints alfabeticamente
    tags-sorter: alpha
  api-docs:
    path: /v3/api-docs              # caminho da spec JSON
```

---

## O que você ganha de graça

Só de adicionar a dependência e subir a aplicação, você já tem, **sem escrever nada**:

- A spec OpenAPI em JSON: `http://localhost:8080/v3/api-docs`
- A página interativa Swagger UI: `http://localhost:8080/swagger-ui.html`

O springdoc varre seus `@RestController`, lê os métodos, parâmetros, DTOs e códigos de status, e monta tudo. Na página, cada endpoint tem um botão **Try it out** que dispara a requisição real e mostra a resposta — é testar a API sem Postman, direto do navegador.

> No springdoc 3.x há também suporte ao **Scalar**, uma interface alternativa e mais moderna ao Swagger UI clássico. Você pode escolher qual usar; o Swagger UI segue sendo o padrão mais conhecido.

---

## Enriquecendo a documentação

A geração automática é boa, mas anotações tornam a doc realmente útil. As principais, todas do pacote `io.swagger.v3.oas.annotations`:

```java
@RestController
@RequestMapping("/medicamentos")
@Tag(name = "Medicamentos", description = "Gestão do catálogo de medicamentos")
public class MedicamentoController {

    @Operation(
        summary = "Busca um medicamento por id",
        description = "Retorna os dados completos de um medicamento específico."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Encontrado"),
        @ApiResponse(responseCode = "404", description = "Medicamento não existe",
                     content = @Content)   // sem corpo de exemplo para o erro
    })
    @GetMapping("/{id}")
    public MedicamentoDTO buscar(
            @Parameter(description = "ID do medicamento", example = "1")
            @PathVariable Long id) {
        return service.buscar(id);
    }
}
```

O que cada uma faz: `@Tag` agrupa e descreve o controller; `@Operation` documenta o método; `@ApiResponses`/`@ApiResponse` descrevem os possíveis retornos; `@Parameter` explica um parâmetro; e `@Schema` (no DTO) descreve campos:

```java
public record MedicamentoDTO(
    @Schema(description = "Nome comercial", example = "Losartana")
    String nome,

    @Schema(description = "Dosagem em miligramas", example = "50")
    Integer dosagemMg
) {}
```

> A régua: anote o que **não é óbvio pelo nome**. Descrições, exemplos e o significado dos códigos de erro agregam muito; repetir o óbvio ("campo nome: o nome") só polui.

---

## Metadados globais da API

Para o cabeçalho da documentação (título, versão, contato), defina um `@OpenAPIDefinition` ou um bean `OpenAPI`:

```java
@Configuration
@OpenAPIDefinition(
    info = @Info(
        title = "API FitMarmita",
        version = "1.0",
        description = "API de gestão de marmitas saudáveis e assinaturas",
        contact = @Contact(name = "Matheus", email = "contato@fitmarmita.com")
    )
)
public class OpenApiConfig { }
```

---

## Botão Authorize

Numa API protegida por JWT, por padrão o Swagger UI não sabe enviar o token — e você não consegue testar endpoints protegidos. A configuração abaixo adiciona o botão **Authorize**, onde você cola o token uma vez e ele passa a ir em todas as chamadas:

```java
@Configuration
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT"
)
@OpenAPIDefinition(
    info = @Info(title = "API FitMarmita", version = "1.0"),
    security = @SecurityRequirement(name = "bearerAuth")   // exige o esquema globalmente
)
public class OpenApiConfig { }
```

O fluxo na página: você chama `/auth/login`, copia o token retornado, clica em **Authorize**, cola, e a partir daí o Swagger envia `Authorization: Bearer ...` automaticamente. Lembre de **liberar as rotas do Swagger** na configuração de segurança, senão a própria página fica bloqueada:

```java
.requestMatchers("/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**").permitAll()
```

(Esse mesmo passo aparece no [guia de JWT](../../java/spring-security/guia-jwt-na-pratica.md), fechando o ciclo entre documentação e segurança.)

---

## Validação vira documentação

Um bônus que poucos exploram: o springdoc **lê as anotações de Bean Validation** dos seus DTOs e reflete na documentação. Se um campo tem `@NotBlank`, `@Size` ou `@Positive`, isso aparece no schema do Swagger como restrição (obrigatório, tamanho, mínimo), sem você escrever nada a mais:

```java
public record MarmitaDTO(
    @NotBlank @Size(max = 100)
    String nome,                    // Swagger mostra: required, maxLength 100

    @Positive
    Integer calorias                // Swagger mostra: minimum > 0
) {}
```

Ou seja, a mesma anotação que **valida** a entrada também **documenta** a regra. Uma fonte de verdade só.

---

## Agrupando e escondendo endpoints

Em APIs grandes, você pode separar a documentação em grupos (ex.: "público" e "admin") com `GroupedOpenApi`:

```java
@Bean
public GroupedOpenApi apiPublica() {
    return GroupedOpenApi.builder()
        .group("publico")
        .pathsToMatch("/medicamentos/**", "/marmitas/**")
        .build();
}
```

E esconder o que não deve aparecer (endpoints internos, controllers técnicos) com `@Hidden`:

```java
@Hidden                          // some da documentação
@GetMapping("/internal/health-detail")
public Status detalheInterno() { ... }
```

---

## Cuidados em produção

Documentação interativa aberta ao mundo pode ser um risco — ela expõe toda a superfície da sua API a quem quiser sondar. As abordagens comuns:

- **Desabilitar em produção**, habilitando só em dev/homologação, via propriedade por perfil:
  ```yaml
  # application-prod.yml
  springdoc:
    api-docs:
      enabled: false
    swagger-ui:
      enabled: false
  ```
- **Ou proteger com autenticação** — deixar o Swagger acessível apenas a usuários autenticados/admin, em vez de `permitAll()`.

> A decisão depende do contexto: API interna de empresa pode manter a doc protegida; API pública de parceiros muitas vezes **quer** a doc acessível. O que você não quer é expor sem intenção a estrutura completa de uma API sensível.

---

## Referências

1. **springdoc-openapi (site oficial)** — https://springdoc.org/
2. **Especificação OpenAPI** — https://spec.openapis.org/
3. **Anotações Swagger Core** — https://github.com/swagger-api/swagger-core/wiki/Swagger-2.X---Annotations
4. **springdoc para Spring Boot 4 (v3)** — https://springdoc.org/v4/

> Ciclo completo: o Swagger **documenta** a API; o [Postman](../postman/guia-definitivo-postman.md) importa essa spec e a **testa**; o [JWT](../../java/spring-security/guia-jwt-na-pratica.md) é a segurança que o botão Authorize exercita.
