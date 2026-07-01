# Guia Prático: Tratamento de Exceptions no Spring Boot

> Do desenho das exceptions customizadas ao `GlobalExceptionHandler` centralizado, com código pronto para copiar e uma tabela de referência para consultar na hora de implementar. Stack: **Spring Boot 4.1 (Spring 6+)**, usando `ProblemDetail` (RFC 9457).

A ideia que orienta tudo: numa aplicação você **cria poucas** exceptions próprias e **trata muitas** que já vêm prontas do JDK e do Spring. O trabalho central não é inventar um catálogo enorme de classes — é **traduzir** cada exception para uma resposta HTTP coerente, num lugar só.

---

## Índice

- [Fundação rápida: checked vs. unchecked](#fundação-rápida)
- [Os dois grupos de exceptions](#os-dois-grupos-de-exceptions)
- [Parte 1 — Exceptions customizadas (o conjunto enxuto)](#parte-1--exceptions-customizadas)
- [Parte 2 — O GlobalExceptionHandler](#parte-2--o-globalexceptionhandler)
- [Parte 3 — Validação (Bean Validation)](#parte-3--validação)
- [Parte 4 — Exceptions de segurança (fora do advice)](#parte-4--exceptions-de-segurança)
- [📋 Tabela de referência: exception → status HTTP](#tabela-de-referência)
- [Armadilhas comuns](#armadilhas-comuns)
- [✅ Checklist de implementação](#checklist-de-implementação)
- [Referências](#referências)

---

## Fundação rápida

A hierarquia do Java: `Throwable` se divide em `Error` (falhas graves da JVM — você não trata) e `Exception`. Dentro de `Exception`, a divisão que importa:

- **Checked** (herda de `Exception`) — o compilador **obriga** a tratar ou declarar (`throws`). Ex.: `IOException`.
- **Unchecked** (herda de `RuntimeException`) — o compilador **não** obriga. Ex.: `IllegalArgumentException`, `NullPointerException`.

> **Por que o mundo Spring prefere unchecked:** exceptions checked poluem assinaturas de método e forçam `try/catch` em cascata que muitas vezes só re-lançam o erro. As suas exceptions de negócio devem herdar de **`RuntimeException`** — assim elas sobem naturalmente pelas camadas até o `GlobalExceptionHandler`, sem sujar cada método no caminho. Além disso, o Spring só faz rollback automático de transação para unchecked (voltamos nisso nas armadilhas).

---

## Os dois grupos de exceptions

| Grupo | Quem lança | O que você faz | Exemplos |
|---|---|---|---|
| **Você cria** (poucas) | Sua camada de serviço/domínio | Modela e lança | `ResourceNotFoundException`, `BusinessRuleException` |
| **Já existem** (muitas) | JDK, Spring MVC, JPA, Security | Só traduz para HTTP | `MethodArgumentNotValidException`, `DataIntegrityViolationException` |

O guia trata dos dois: a Parte 1 é o que você cria; as Partes 2 a 4 são a tradução de ambos para respostas HTTP.

---

## Parte 1 — Exceptions customizadas

O conjunto saudável é enxuto. A régua para criar uma nova: **só crie se ela levar a um status HTTP ou tratamento diferente.** Se duas situações resultam no mesmo `404` com a mesma estrutura, são a mesma exception com mensagens diferentes.

### A exception base

Ter uma base comum dá um ponto único de captura e separa "erro de negócio" de "erro técnico".

```java
package com.exemplo.exception;

public abstract class BusinessException extends RuntimeException {
    protected BusinessException(String mensagem) {
        super(mensagem);
    }
}
```

### As três que cobrem a maioria dos casos

```java
// 404 — recurso inexistente (a mais usada de todas)
public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String recurso, Object id) {
        super("%s não encontrado(a) com id %s".formatted(recurso, id));
    }
}

// 409 — conflito / duplicidade (viola unicidade)
public class DuplicateResourceException extends BusinessException {
    public DuplicateResourceException(String mensagem) {
        super(mensagem);
    }
}

// 422 — violação de regra de negócio
public class BusinessRuleException extends BusinessException {
    public BusinessRuleException(String mensagem) {
        super(mensagem);
    }
}
```

Uso na camada de serviço:

```java
public Medicamento buscar(Long id) {
    return repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Medicamento", id));
}

public void cadastrar(MedicamentoDTO dto) {
    if (repository.existsByNome(dto.nome())) {
        throw new DuplicateResourceException("Já existe medicamento com o nome " + dto.nome());
    }
    if (dto.dataInicio().isBefore(LocalDate.now())) {
        throw new BusinessRuleException("A data de início não pode estar no passado");
    }
    // ...
}
```

> Você **não** precisa criar exceptions para autenticação/autorização — o Spring Security já tem `AuthenticationException` (401) e `AccessDeniedException` (403). Veja a Parte 4.

---

## Parte 2 — O GlobalExceptionHandler

Aqui está o coração do guia: uma classe `@RestControllerAdvice` que centraliza toda a tradução exception → HTTP. `@RestControllerAdvice` = `@ControllerAdvice` (intercepta exceptions de todos os controllers) + `@ResponseBody` (serializa o retorno no corpo).

Usamos `ProblemDetail` (RFC 9457), o formato padronizado de erro do Spring 6+, com corpo contendo `type`, `title`, `status`, `detail` e `instance`. Estendemos `ResponseEntityExceptionHandler` para herdar o tratamento das exceptions do Spring MVC (validação, JSON malformado etc.) e customizar só o necessário.

**`GlobalExceptionHandler.java` — pronto para copiar:**

```java
package com.exemplo.exception;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
import java.time.Instant;

@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // ---- 404 ----
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return montar(HttpStatus.NOT_FOUND, "Recurso não encontrado", ex.getMessage());
    }

    // ---- 409 (duplicidade de negócio) ----
    @ExceptionHandler(DuplicateResourceException.class)
    public ProblemDetail handleDuplicate(DuplicateResourceException ex) {
        return montar(HttpStatus.CONFLICT, "Conflito", ex.getMessage());
    }

    // ---- 422 (regra de negócio) ----
    @ExceptionHandler(BusinessRuleException.class)
    public ProblemDetail handleBusiness(BusinessRuleException ex) {
        return montar(HttpStatus.UNPROCESSABLE_ENTITY, "Regra de negócio violada", ex.getMessage());
    }

    // ---- 409 (constraint do banco — última linha de defesa) ----
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ProblemDetail handleDataIntegrity(DataIntegrityViolationException ex) {
        // NÃO exponha ex.getMessage() aqui — vaza detalhes do schema do banco
        return montar(HttpStatus.CONFLICT, "Conflito de integridade",
                      "A operação viola uma restrição de dados.");
    }

    // ---- 400 (tipo errado na URL: /medicamentos/abc onde espera Long) ----
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ProblemDetail handleTypeMismatch(MethodArgumentTypeMismatchException ex) {
        String detalhe = "O parâmetro '%s' tem valor inválido: '%s'".formatted(ex.getName(), ex.getValue());
        return montar(HttpStatus.BAD_REQUEST, "Parâmetro inválido", detalhe);
    }

    // ---- 500 (rede de segurança para o não previsto) ----
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneric(Exception ex) {
        // Logue o stack trace INTERNAMENTE (aqui), mas nunca o exponha ao cliente
        logger.error("Erro não tratado", ex);
        return montar(HttpStatus.INTERNAL_SERVER_ERROR, "Erro interno",
                      "Ocorreu um erro inesperado. Tente novamente mais tarde.");
    }

    // Helper para padronizar a montagem do ProblemDetail
    private ProblemDetail montar(HttpStatus status, String titulo, String detalhe) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(status, detalhe);
        pd.setTitle(titulo);
        pd.setProperty("timestamp", Instant.now());
        return pd;
    }
}
```

Como o Spring escolhe o handler: sempre o **mais específico** que casa com a exception lançada. O `handleGeneric(Exception.class)` não "rouba" os específicos — ele só pega o que nenhum outro tratou. Por isso ele é a rede de segurança indispensável: sem ele, qualquer exception não prevista vira um `500` cru com stack trace.

---

## Parte 3 — Validação

Quando o `@Valid` falha, você quer devolver **quais campos** falharam, não só uma mensagem genérica. A exception é a `MethodArgumentNotValidException`, e como estendemos `ResponseEntityExceptionHandler`, basta sobrescrever o handler dela.

Primeiro, a validação no DTO (Bean Validation, namespace `jakarta.validation`):

```java
public record MedicamentoDTO(
    @NotBlank(message = "O nome é obrigatório")
    String nome,

    @NotNull(message = "A dosagem é obrigatória")
    @Positive(message = "A dosagem deve ser positiva")
    Integer dosagemMg,

    @Future(message = "A data de início deve ser futura")
    LocalDate dataInicio
) {}
```

E o controller aciona a validação com `@Valid`:

```java
@PostMapping
public ResponseEntity<Void> criar(@Valid @RequestBody MedicamentoDTO dto) {
    service.cadastrar(dto);
    return ResponseEntity.status(HttpStatus.CREATED).build();
}
```

Agora o handler que transforma os erros de campo num mapa legível. Adicione este método ao `GlobalExceptionHandler`:

```java
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid(
        MethodArgumentNotValidException ex, HttpHeaders headers,
        HttpStatusCode status, WebRequest request) {

    Map<String, String> erros = new HashMap<>();
    ex.getBindingResult().getFieldErrors()
      .forEach(erro -> erros.put(erro.getField(), erro.getDefaultMessage()));

    ProblemDetail pd = ProblemDetail.forStatusAndDetail(
        HttpStatus.BAD_REQUEST, "Um ou mais campos são inválidos");
    pd.setTitle("Erro de validação");
    pd.setProperty("timestamp", Instant.now());
    pd.setProperty("erros", erros);   // { "nome": "O nome é obrigatório", ... }

    return handleExceptionInternal(ex, pd, headers, HttpStatus.BAD_REQUEST, request);
}
```

Resposta resultante (formato limpo e previsível para o front consumir):

```json
{
  "type": "about:blank",
  "title": "Erro de validação",
  "status": 400,
  "detail": "Um ou mais campos são inválidos",
  "timestamp": "2026-06-30T14:22:10Z",
  "erros": {
    "nome": "O nome é obrigatório",
    "dosagemMg": "A dosagem deve ser positiva"
  }
}
```

> Para validação em parâmetros soltos (`@RequestParam`/`@PathVariable` com `@Validated` no controller), a exception é outra (`HandlerMethodValidationException` no Spring 6.1+). O `ResponseEntityExceptionHandler` já a trata por padrão; sobrescreva `handleHandlerMethodValidationException` só se quiser customizar o corpo.

---

## Parte 4 — Exceptions de segurança

Este é o mal-entendido clássico: exceptions do Spring Security **não passam pelo `@RestControllerAdvice`**. Elas são lançadas nos *filtros*, antes de a requisição chegar ao controller — então o advice, que intercepta exceptions **de controllers**, nunca as vê. Um `@ExceptionHandler(AccessDeniedException.class)` no advice simplesmente não dispara para erros vindos do filtro.

O tratamento correto é via dois componentes registrados na configuração de segurança:

- **`AuthenticationEntryPoint`** — dispara quando o usuário **não está autenticado** (401).
- **`AccessDeniedHandler`** — dispara quando está autenticado mas **não tem permissão** (403).

```java
@Component
public class JsonAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res,
                         AuthenticationException ex) throws IOException {
        escreverProblema(res, HttpStatus.UNAUTHORIZED, "Não autenticado",
                         "Credenciais ausentes ou inválidas.");
    }

    static void escreverProblema(HttpServletResponse res, HttpStatus status,
                                 String titulo, String detalhe) throws IOException {
        res.setStatus(status.value());
        res.setContentType("application/problem+json");
        res.getWriter().write("""
            {"title":"%s","status":%d,"detail":"%s"}
            """.formatted(titulo, status.value(), detalhe));
    }
}
```

E registra no `SecurityFilterChain`:

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint(new JsonAuthenticationEntryPoint())
    .accessDeniedHandler((req, res, e) ->
        JsonAuthenticationEntryPoint.escreverProblema(
            res, HttpStatus.FORBIDDEN, "Acesso negado", "Você não tem permissão para este recurso."))
);
```

Assim seus erros de segurança devolvem JSON no mesmo espírito do resto da API, em vez da página de erro padrão. (Sobre a cadeia de filtros e por que isso corre "por fora", veja o [Guia de Configuração do Spring Security](../spring-security/guia-configuracao-spring-security.md).)

---

## Tabela de referência

Consulta rápida na hora de implementar — qual exception vira qual status, e onde é tratada:

| Situação | Exception | Status | Onde tratar |
|---|---|---|---|
| Recurso não encontrado | `ResourceNotFoundException` (sua) | **404** | `@RestControllerAdvice` |
| Corpo JSON inválido / validação `@Valid` | `MethodArgumentNotValidException` | **400** | advice (override) |
| Parâmetro/path com tipo errado | `MethodArgumentTypeMismatchException` | **400** | advice |
| JSON malformado / ilegível | `HttpMessageNotReadableException` | **400** | advice (já tratado pela base) |
| Validação de `@RequestParam`/`@PathVariable` | `HandlerMethodValidationException` | **400** | advice (base) |
| Não autenticado | `AuthenticationException` | **401** | `AuthenticationEntryPoint` |
| Sem permissão | `AccessDeniedException` | **403** | `AccessDeniedHandler` |
| Método HTTP errado (POST onde só há GET) | `HttpRequestMethodNotSupportedException` | **405** | advice (base) |
| Content-Type não suportado | `HttpMediaTypeNotSupportedException` | **415** | advice (base) |
| Duplicidade (regra de negócio) | `DuplicateResourceException` (sua) | **409** | advice |
| Constraint do banco (unique, FK) | `DataIntegrityViolationException` | **409** | advice |
| Regra de negócio violada | `BusinessRuleException` (sua) | **422** | advice |
| Qualquer coisa não prevista | `Exception` | **500** | advice (rede de segurança) |

> "advice (base)" = já vem tratado por herdar de `ResponseEntityExceptionHandler`; você só sobrescreve se quiser mudar o corpo.

---

## Armadilhas comuns

1. **Exceptions de segurança no advice não disparam** — são lançadas nos filtros, antes do controller. Trate com `AuthenticationEntryPoint`/`AccessDeniedHandler` (Parte 4).
2. **Engolir exception** — `catch (Exception e) {}` vazio (ou que só loga e segue) esconde bugs. Se capturou, trate de verdade ou re-lance.
3. **Vazar detalhes internos** — nunca devolva stack trace nem `ex.getMessage()` de exceptions técnicas (como `DataIntegrityViolationException`) ao cliente. Logue internamente, responda com mensagem neutra.
4. **Rollback de transação é camada diferente** — capturar a exception no advice **não desfaz** o rollback. Se uma `RuntimeException` já disparou rollback no `@Transactional`, tratá-la no advice muda só a resposta HTTP; a transação já foi revertida. (E `@Transactional` **não** faz rollback para checked exceptions por padrão — outro motivo para preferir unchecked.)
5. **Handler genérico faltando** — sem `@ExceptionHandler(Exception.class)`, o imprevisto vira `500` cru com stack trace exposto.
6. **Ordem de captura** — não é preciso ordenar manualmente; o Spring escolhe o handler mais específico. Mas cuidado ao capturar tipos-pai (`RuntimeException`) que podem "cobrir" exceptions que você queria tratar de forma diferente.
7. **Criar exception demais** — se a nova classe leva ao mesmo status e corpo de uma existente, ela não se justifica. Prefira reusar com mensagem diferente.

---

## Checklist de implementação

Para consultar quando for montar isso num projeto novo:

- [ ] Criar a base `BusinessException extends RuntimeException`.
- [ ] Criar `ResourceNotFoundException` (404) — quase sempre necessária.
- [ ] Criar `DuplicateResourceException` (409) e `BusinessRuleException` (422) conforme o domínio pedir.
- [ ] Criar `GlobalExceptionHandler extends ResponseEntityExceptionHandler` com `@RestControllerAdvice`.
- [ ] Adicionar handlers para as suas exceptions + `DataIntegrityViolationException` + `Exception` (rede de segurança).
- [ ] Sobrescrever `handleMethodArgumentNotValid` para devolver os erros por campo.
- [ ] Garantir que o handler genérico **loga** internamente e **não vaza** stack trace.
- [ ] Registrar `AuthenticationEntryPoint` (401) e `AccessDeniedHandler` (403) na config de segurança.
- [ ] Anotar os DTOs com Bean Validation (`@NotBlank`, `@NotNull`, `@Positive`...) e os endpoints com `@Valid`.
- [ ] Testar cada caminho: 400, 404, 409, 422, 401, 403 e o 500 genérico.

---

## Referências

1. **Spring — Error Handling (ProblemDetail)** — https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html
2. **RFC 9457 (Problem Details for HTTP APIs)** — https://www.rfc-editor.org/rfc/rfc9457
3. **Bean Validation (Jakarta)** — https://beanvalidation.org/
4. **Spring Security — respostas de erro** — veja o [Guia de Configuração do Spring Security](../spring-security/guia-configuracao-spring-security.md)
5. **Effective Java (Bloch)** — capítulo de Exceptions (checked vs unchecked, quando criar, não engolir)

> Regra de bolso para lembrar de tudo: **crie exception só quando muda o status HTTP; trate tudo num `@RestControllerAdvice`; segurança corre por fora; nunca vaze o interno.**
