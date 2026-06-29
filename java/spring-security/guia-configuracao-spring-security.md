# Guia Definitivo de Configuração do Spring Security

> Como o Spring Security funciona por dentro e como configurá-lo de forma correta e moderna. Stack de referência: **Spring Boot 4.1 + Spring Security 7** (a Lambda DSL é obrigatória nesta versão).

A maioria das frustrações com Spring Security vem de tentar configurá-lo sem entender o que ele faz por baixo. Este guia inverte a ordem: primeiro o **modelo mental** (a cadeia de filtros), depois a configuração. Quando você "vê" a cadeia, a configuração deixa de ser mágica e vira lógica.

> Este guia cobre a **fundação** da configuração. Para o aprofundamento em login federado, veja o guia complementar de [Login e Cadastro com OAuth 2.0 / OIDC](guia-login-cadastro-oauth2-oidc.md).

---

## Índice

- [O modelo mental: a cadeia de filtros](#o-modelo-mental-a-cadeia-de-filtros)
- [Autenticação vs. Autorização](#autenticação-vs-autorização)
- [A configuração moderna (Spring Security 7)](#a-configuração-moderna)
- [Autorização por requisição](#autorização-por-requisição)
- [Mecanismos de autenticação](#mecanismos-de-autenticação)
- [Codificação de senhas](#codificação-de-senhas)
- [UserDetailsService: usuários do seu banco](#userdetailsservice-usuários-do-seu-banco)
- [O SecurityContext: quem está logado](#o-securitycontext-quem-está-logado)
- [Segurança em nível de método](#segurança-em-nível-de-método)
- [CSRF: quando manter e quando desabilitar](#csrf)
- [CORS](#cors)
- [Múltiplas SecurityFilterChain](#múltiplas-securityfilterchain)
- [Testando a segurança](#testando-a-segurança)
- [Armadilhas comuns](#armadilhas-comuns)
- [Referências](#referências)

---

## O modelo mental: a cadeia de filtros

Spring Security é, na essência, **uma cadeia de filtros (servlet filters) que interceptam toda requisição antes de ela chegar ao seu controller.** Cada filtro tem uma única responsabilidade, e eles agem em ordem.

```
Requisição HTTP
      │
      ▼
┌─────────────────────────────────────────────────────┐
│         DelegatingFilterProxy (ponte Servlet→Spring) │
│                       │                               │
│                       ▼                               │
│              FilterChainProxy                         │
│                       │                               │
│   escolhe a SecurityFilterChain que casa com a URL    │
│                       ▼                               │
│  ┌─────────────── SecurityFilterChain ─────────────┐ │
│  │  1. CsrfFilter                                   │ │
│  │  2. (filtros de autenticação:                    │ │
│  │      UsernamePasswordAuthenticationFilter,       │ │
│  │      BasicAuthenticationFilter,                  │ │
│  │      BearerTokenAuthenticationFilter, OAuth2...) │ │
│  │  3. ExceptionTranslationFilter                   │ │
│  │  4. AuthorizationFilter  (decide: pode ou não?)  │ │
│  └──────────────────────┬───────────────────────────┘ │
└─────────────────────────┼─────────────────────────────┘
                          ▼
                  Seu @Controller   ← só chega aqui se passar por tudo
```

Três peças para guardar:

- **DelegatingFilterProxy** — a ponte entre o mundo dos servlets (Tomcat) e o mundo do Spring. É o que injeta o Spring Security no fluxo de requisições.
- **FilterChainProxy** — recebe a requisição e escolhe **qual** cadeia de segurança aplicar (você pode ter várias, como veremos).
- **SecurityFilterChain** — a cadeia em si, uma sequência ordenada de filtros. Configurá-la é o seu trabalho principal.

Quando você escreve um `SecurityFilterChain` bean, está **montando essa lista de filtros**. Cada método do DSL (`csrf`, `formLogin`, `authorizeHttpRequests`) liga, desliga ou ajusta um filtro da cadeia.

---

## Autenticação vs. Autorização

Os dois conceitos centrais, que mexem com filtros diferentes:

- **Autenticação (AuthN)** — "*quem* é você?". Validar credenciais e estabelecer a identidade. Filtros de autenticação cuidam disso e populam o `SecurityContext`.
- **Autorização (AuthZ)** — "o que você *pode* fazer?". Dado que sabemos quem é, decidir se tem permissão para o recurso. O `AuthorizationFilter` (perto do fim da cadeia) cuida disso.

A ordem importa: você sempre autentica **antes** de autorizar. Faz sentido — não dá para decidir o que alguém pode fazer sem saber quem é.

---

## A configuração moderna

Esqueça tutoriais antigos com `WebSecurityConfigurerAdapter` — essa classe **foi removida** e não existe mais. A configuração atual é baseada em **beans**, e no Spring Security 7 a **Lambda DSL é obrigatória** (o antigo encadeamento com `.and()` e o `authorizeRequests()` foram removidos de vez).

A configuração mínima funcional:

```java
package com.exemplo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())   // tela de login padrão
            .httpBasic(Customizer.withDefaults());   // autenticação HTTP Basic
        return http.build();
    }
}
```

Pontos da nova sintaxe:

- Cada aspecto (`authorizeHttpRequests`, `formLogin`, `httpBasic`) recebe um **lambda** que o configura. Não há mais `.and()` para emendar — o `HttpSecurity` é retornado automaticamente.
- `Customizer.withDefaults()` é o atalho para "ative isto com a configuração padrão" (equivale ao lambda vazio `it -> {}`).
- Sempre termine com `return http.build();`.

> ⚠️ Migrando código antigo? As trocas mecânicas são: `authorizeRequests` → `authorizeHttpRequests`; `antMatchers`/`mvcMatchers` → `requestMatchers`; `@EnableGlobalMethodSecurity` → `@EnableMethodSecurity`; e DSLs customizadas com `.apply(...)` → `.with(...)`.

---

## Autorização por requisição

O `authorizeHttpRequests` define regras por URL. **A ordem é decisiva**: as regras são avaliadas de cima para baixo, e a primeira que casar vence. Por isso, coloque as regras específicas **antes** das genéricas.

```java
.authorizeHttpRequests(auth -> auth
    // específicas primeiro
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/relatorios/**").hasAnyRole("ADMIN", "GERENTE")
    .requestMatchers(HttpMethod.POST, "/api/medicamentos").hasAuthority("SCOPE_write")
    .requestMatchers("/login", "/css/**").permitAll()
    // genérica por último — "todo o resto exige autenticação"
    .anyRequest().authenticated()
)
```

Métodos de regra principais: `permitAll()` (livre), `authenticated()` (precisa estar logado), `hasRole("X")`, `hasAnyRole(...)`, `hasAuthority(...)`, `denyAll()`.

> 🔑 `hasRole("ADMIN")` vs `hasAuthority("ADMIN")`: o `hasRole` adiciona automaticamente o prefixo `ROLE_`, esperando uma authority chamada `ROLE_ADMIN`. O `hasAuthority` usa o nome exato. Essa diferença é fonte clássica de bugs de "por que minha role não funciona" — geralmente é o prefixo faltando ou sobrando.

---

## Mecanismos de autenticação

O Spring Security suporta vários mecanismos, cada um ligado por um método do DSL:

- **`formLogin`** — formulário de login com sessão. Para aplicações web tradicionais (server-side, Thymeleaf).
- **`httpBasic`** — usuário/senha no cabeçalho `Authorization`. Simples, útil para APIs internas e testes.
- **`oauth2Login`** — login via provedor externo (Google, Keycloak). Veja o [guia de OAuth/OIDC](guia-login-cadastro-oauth2-oidc.md).
- **`oauth2ResourceServer`** — valida tokens JWT em APIs stateless. Também no guia de OAuth.

Você pode combinar mais de um. A escolha depende do tipo de cliente: navegador com sessão → `formLogin`; SPA/mobile consumindo API → token (resource server).

---

## Codificação de senhas

Regra absoluta e inegociável: **senha nunca é armazenada em texto claro.** O Spring Security força isso — sem um `PasswordEncoder`, ele recusa autenticar.

A recomendação atual é o **`DelegatingPasswordEncoder`**, que prefixa o hash com o algoritmo usado (`{bcrypt}`, `{argon2}`...). Isso permite migrar de algoritmo no futuro sem quebrar as senhas já gravadas.

```java
@Bean
PasswordEncoder passwordEncoder() {
    // cria um DelegatingPasswordEncoder com BCrypt como padrão
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

Uma senha gravada fica assim no banco:

```
{bcrypt}$2a$10$N9qo8uLOickgx2ZMRZoMy...
```

O prefixo diz ao Spring qual algoritmo usar para verificar. Hashes antigos com prefixo diferente continuam validando; novos usam o padrão atual.

> ❌ **Nunca** use `{noop}` (sem codificação) fora de um exemplo descartável. Ele guarda a senha em texto puro e existe só para protótipos. Em qualquer código que vá para o GitHub, use BCrypt ou Argon2.

---

## UserDetailsService: usuários do seu banco

Para autenticar contra usuários reais (não em memória), você implementa o `UserDetailsService` — a ponte entre o Spring Security e a sua tabela de usuários.

```java
@Service
@RequiredArgsConstructor
public class JpaUserDetailsService implements UserDetailsService {

    private final UsuarioRepository repository;

    @Override
    public UserDetails loadUserByUsername(String email) {
        Usuario usuario = repository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("Usuário não encontrado: " + email));

        return User.builder()
            .username(usuario.getEmail())
            .password(usuario.getSenha())           // já deve estar codificada (BCrypt)
            .authorities(mapearAuthorities(usuario)) // ex.: ROLE_USER, ROLE_ADMIN
            .build();
    }

    private Collection<? extends GrantedAuthority> mapearAuthorities(Usuario u) {
        return List.of(new SimpleGrantedAuthority("ROLE_" + u.getRole().name()));
    }
}
```

Com esse bean no contexto, o Spring Security o usa automaticamente no fluxo de `formLogin`/`httpBasic`. Você não precisa amarrar nada à mão — o framework descobre o `UserDetailsService` e o `PasswordEncoder` por injeção.

> A entidade `Usuario` aqui **tem** senha (autenticação local). Se você for usar login social/OIDC, a senha vive no provedor e a abordagem muda — é o cenário do guia de OAuth.

---

## O SecurityContext: quem está logado

Depois da autenticação, a identidade do usuário fica guardada no **`SecurityContextHolder`** — acessível de qualquer lugar durante a requisição. É de lá que você recupera o usuário atual.

```java
// Forma direta
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();

// Forma preferida em controllers — injeção pelo Spring
@GetMapping("/perfil")
public PerfilDTO meuPerfil(@AuthenticationPrincipal UserDetails usuario) {
    return service.buscarPerfil(usuario.getUsername());
}
```

> Por padrão, o `SecurityContext` é armazenado na sessão (apps com `formLogin`) ou reconstruído a cada requisição a partir do token (APIs stateless com JWT). Em APIs stateless você dirá explicitamente para **não** usar sessão (veja a config stateless no guia de OAuth).

---

## Segurança em nível de método

Além de proteger URLs, você pode proteger **métodos** — útil para regras finas na camada de serviço. Habilite com `@EnableMethodSecurity`:

```java
@Configuration
@EnableMethodSecurity   // substitui o antigo @EnableGlobalMethodSecurity
public class MethodSecurityConfig { }
```

E anote os métodos:

```java
@Service
public class MedicamentoService {

    @PreAuthorize("hasRole('ADMIN')")               // verifica ANTES de executar
    public void deletar(Long id) { /* ... */ }

    @PreAuthorize("#email == authentication.name")   // só o próprio dono
    public PerfilDTO verPerfil(String email) { /* ... */ }

    @PostAuthorize("returnObject.dono == authentication.name")  // verifica o retorno
    public Documento buscar(Long id) { /* ... */ }
}
```

A diferença: `@PreAuthorize` decide **antes** de rodar o método (mais comum e mais seguro); `@PostAuthorize` decide **depois**, podendo inspecionar o objeto retornado. As expressões usam SpEL e têm acesso a `authentication` e aos argumentos do método (via `#nome`).

---

## CSRF

CSRF (*Cross-Site Request Forgery*) é um ataque onde um site malicioso faz o navegador da vítima enviar requisições autenticadas ao seu sistema. A proteção do Spring exige um token em requisições que mudam estado (`POST`, `PUT`, `DELETE`).

A decisão de manter ou desabilitar depende de como você autentica:

- **App web com sessão e cookies (formLogin)** → **MANTENHA o CSRF habilitado** (é o padrão). Desabilitar abre um buraco de segurança real.
- **API stateless com token (JWT no header `Authorization`)** → **pode desabilitar**. Sem cookies de sessão, o vetor de CSRF não se aplica; o token no header já protege.

```java
// API stateless — desabilitar é aceitável
.csrf(csrf -> csrf.disable())
```

> ⚠️ Não copie `csrf.disable()` de tutoriais cegamente. Para apps de navegador com sessão, isso é uma vulnerabilidade. Para SPAs que leem o cookie `XSRF-TOKEN` e o reenviam no header, o Spring Security usa um manipulador específico (`XorCsrfTokenRequestAttributeHandler`, com proteção contra BREACH) — uma fonte famosa de `403` misteriosos após upgrade. Configure-o conforme a documentação oficial em vez de simplesmente desligar.

---

## CORS

CORS controla quais origens (domínios) podem chamar sua API pelo navegador. Essencial quando seu front (ex.: React em `localhost:5173`) consome a API em outra porta/domínio.

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .cors(Customizer.withDefaults())   // ativa o CORS usando o bean abaixo
        // ... resto da config
    return http.build();
}

@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://meuapp.com", "http://localhost:5173"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

> Não confunda CORS com CSRF — são coisas diferentes que costumam aparecer juntas. CORS libera o navegador a fazer a chamada; CSRF protege contra chamadas forjadas. E evite `setAllowedOrigins(List.of("*"))` combinado com `allowCredentials(true)` — além de inseguro, o navegador rejeita.

---

## Múltiplas SecurityFilterChain

Aplicações reais costumam ter regras diferentes para áreas diferentes — por exemplo, uma API stateless com token **e** um painel web com sessão. A solução é ter **várias** `SecurityFilterChain`, cada uma com seu escopo via `securityMatcher` e prioridade via `@Order`.

```java
@Bean
@Order(1)   // ordem menor = avaliada primeiro
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")              // esta cadeia só vale para /api/**
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(csrf -> csrf.disable());
    return http.build();
}

@Bean
@Order(2)   // pega tudo que não casou com /api/**
SecurityFilterChain webChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(a -> a
            .requestMatchers("/login", "/css/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());   // sessão + CSRF (padrão)
    return http.build();
}
```

A regra de ouro: **a primeira cadeia cujo `securityMatcher` casa vence.** Por isso a mais específica (`/api/**`) recebe o `@Order` menor. Errar essa ordem faz uma cadeia "engolir" requisições destinadas a outra — bug sutil e comum.

---

## Testando a segurança

A dependência `spring-security-test` permite testar as regras sem subir um servidor real:

```java
@WebMvcTest(MedicamentoController.class)
class MedicamentoControllerTest {

    @Autowired MockMvc mvc;

    @Test
    void anonimo_recebe_401() throws Exception {
        mvc.perform(get("/api/medicamentos"))
           .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void usuario_comum_nao_deleta() throws Exception {
        mvc.perform(delete("/api/medicamentos/1").with(csrf()))
           .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void admin_deleta() throws Exception {
        mvc.perform(delete("/api/medicamentos/1").with(csrf()))
           .andExpect(status().isNoContent());
    }
}
```

`@WithMockUser` simula um usuário autenticado com as roles indicadas. O `.with(csrf())` injeta um token CSRF válido nos testes de endpoints que mudam estado. Testar os três caminhos — anônimo, role errada, role certa — é a melhor rede de segurança contra regressões de segurança em upgrades.

---

## Armadilhas comuns

1. **Ordem das regras de autorização** — específica antes da genérica. `anyRequest()` sempre por último.
2. **Confundir `hasRole` com `hasAuthority`** — o prefixo `ROLE_` é a pegadinha.
3. **Desabilitar CSRF em app de navegador** — vulnerabilidade. Só desabilite em API stateless por token.
4. **Usar `{noop}` ou senha em texto claro** — sempre BCrypt/Argon2.
5. **Ordem errada entre múltiplas chains** — `@Order` menor para a mais específica.
6. **Esquecer `@EnableMethodSecurity`** — sem ela, as anotações `@PreAuthorize` são silenciosamente ignoradas (o pior tipo de falha: parece protegido, mas não está).
7. **Segredos no código** — chaves, senhas e URLs vão para variáveis de ambiente, nunca para o Git.

---

## Referências

1. **Spring Security — Reference** — https://docs.spring.io/spring-security/reference/
2. **Architecture (a cadeia de filtros)** — https://docs.spring.io/spring-security/reference/servlet/architecture.html
3. **Authorization** — https://docs.spring.io/spring-security/reference/servlet/authorization/index.html
4. **Migração para Spring Security 7** — https://docs.spring.io/spring-security/reference/migration-7/configuration.html
5. **Spring Security Test** — https://docs.spring.io/spring-security/reference/servlet/test/index.html

> Companheiro deste guia: [Login e Cadastro com OAuth 2.0 / OIDC](guia-login-cadastro-oauth2-oidc.md), que aprofunda autenticação federada, provisionamento de usuários e proteção de API com JWT.
