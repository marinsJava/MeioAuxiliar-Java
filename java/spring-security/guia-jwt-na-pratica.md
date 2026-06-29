# Guia Prático: Autenticação JWT Stateless no Spring Boot

> Implementação completa de login com JWT feito à mão, do zero ao Swagger. Stack: **Spring Boot 4.1 + Spring Security 7 + jjwt 0.12.6**. Este guia foca no **fluxo** e no **porquê** de cada peça — é o tipo de coisa que confunde no começo justamente porque são muitas partes pequenas colaborando.

Se o [Guia de Configuração do Spring Security](guia-configuracao-spring-security.md) é a fundação e o [Guia de OAuth/OIDC](guia-login-cadastro-oauth2-oidc.md) é a autenticação delegada, este aqui é o meio-termo: você mesmo emite e valida os tokens, sem provedor externo. É o cenário mais comum em APIs próprias.

---

## Índice

- [O quadro geral: por que stateless?](#o-quadro-geral)
- [Os dois fluxos que você precisa enxergar](#os-dois-fluxos)
- [Anatomia de um JWT](#anatomia-de-um-jwt)
- [Dependências](#dependências)
- [1. Codificação de senha (BCrypt)](#1-codificação-de-senha)
- [2. UserDetailsService: trazendo o usuário do banco](#2-userdetailsservice)
- [3. JwtService: gerar e validar tokens](#3-jwtservice)
- [4. JwtAuthenticationFilter: o porteiro de cada requisição](#4-jwtauthenticationfilter)
- [5. SecurityConfig: amarrando tudo](#5-securityconfig)
- [6. Endpoint de login](#6-endpoint-de-login)
- [7. CORS e o preflight (OPTIONS)](#7-cors-e-o-preflight)
- [8. Swagger com o botão Authorize](#8-swagger-com-o-botão-authorize)
- [Recapitulando o fluxo completo](#recapitulando-o-fluxo-completo)
- [Dificuldades comuns (e por que acontecem)](#dificuldades-comuns)
- [Referências](#referências)

---

## O quadro geral

No modelo tradicional (com sessão), depois que você loga, o servidor guarda quem você é numa **sessão** na memória dele e te devolve um cookie. A cada requisição, ele consulta essa sessão. Isso funciona, mas amarra o usuário a um servidor específico e dificulta escalar.

No modelo **stateless com JWT**, o servidor **não guarda nada**. Quando você loga, ele te entrega um **token assinado** que contém quem você é. A cada requisição, você envia esse token, e o servidor apenas **verifica a assinatura** — sem consultar sessão nenhuma. Qualquer servidor com a mesma chave consegue validar. Daí o nome: *stateless* (sem estado).

A troca: o servidor fica mais simples e escalável, mas você assume a responsabilidade de gerar, assinar e validar os tokens. É isso que vamos construir.

---

## Os dois fluxos

Toda a confusão se dissolve quando você separa os **dois momentos** distintos. Eles usam código diferente e acontecem em horas diferentes.

```
═══════════ FLUXO 1: LOGIN (acontece UMA vez) ═══════════

  Cliente                  AuthController            JwtService
    │  POST /login            │                          │
    │  {email, senha}         │                          │
    ├────────────────────────►│                          │
    │              AuthenticationManager valida           │
    │              (UserDetailsService + BCrypt)          │
    │                         ├─────────────────────────►│
    │                         │   gera token assinado     │
    │                         │◄─────────────────────────┤
    │   { token: "eyJ..." }   │                          │
    │◄────────────────────────┤                          │
    │  (cliente guarda o token)                           │

═══════ FLUXO 2: REQUISIÇÕES SEGUINTES (acontece SEMPRE) ═══════

  Cliente              JwtAuthenticationFilter      Controller
    │ GET /api/medicamentos    │                        │
    │ Authorization: Bearer eyJ│                        │
    ├─────────────────────────►│                        │
    │          valida assinatura do token                │
    │          popula o SecurityContext                  │
    │                          ├───────────────────────►│
    │                          │   executa (já sabe      │
    │                          │   quem é o usuário)     │
    │      resposta            │◄───────────────────────┤
    │◄─────────────────────────┤                        │
```

- **Fluxo 1 (login):** valida usuário e senha **uma vez** e devolve um token. Acontece no `AuthController`.
- **Fluxo 2 (todas as outras requisições):** o `JwtAuthenticationFilter` intercepta, valida o token e diz ao Spring "esse usuário está autenticado". Acontece em **toda** requisição protegida.

Guarde isso: **o login NÃO usa o filtro JWT; as requisições seguintes NÃO usam o AuthController.** São caminhos separados.

---

## Anatomia de um JWT

Um JWT é uma string com três partes separadas por ponto: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiJtYXRoZXVzIiwiZXhwIjoxNz...} . dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
└──── header ────┘   └──────────── payload ────────────┘   └──────────── signature ──────────┘
   (algoritmo)            (claims: subject, exp...)              (prova de autenticidade)
```

- **Header** — diz qual algoritmo assina (ex.: HS256).
- **Payload** — os *claims* (afirmações): quem é o usuário (`sub`/subject), quando expira (`exp`), etc. **Atenção: o payload é apenas codificado em Base64, não criptografado — qualquer um lê.** Nunca coloque senha ou dado sensível ali.
- **Signature** — o resultado de assinar header+payload com a sua chave secreta. É isto que torna o token confiável: se alguém alterar o payload, a assinatura não bate mais.

> A segurança não vem de esconder o conteúdo (ele é visível), e sim da **assinatura**: sem a chave secreta do servidor, ninguém consegue forjar um token válido nem alterar um existente sem invalidá-lo.

---

## Dependências

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- jjwt 0.12.6 — dividido em 3 artefatos -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>

<!-- Swagger / documentação OpenAPI -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.6</version>
</dependency>
```

E no `application.yml`, a chave secreta e a validade (nunca no código!):

```yaml
jwt:
  secret: ${JWT_SECRET}          # chave Base64 forte, vinda de variável de ambiente
  expiration: 3600000            # 1 hora em milissegundos
```

---

## 1. Codificação de senha

Senha **nunca** é guardada em texto claro. A recomendação é o `DelegatingPasswordEncoder`, que prefixa o hash com o algoritmo (`{bcrypt}...`), permitindo trocar de algoritmo no futuro sem quebrar as senhas já salvas.

```java
@Bean
PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder(); // BCrypt por padrão
}
```

No cadastro, codifique antes de salvar:

```java
@Service
@RequiredArgsConstructor
public class UsuarioService {

    private final UsuarioRepository repository;
    private final PasswordEncoder passwordEncoder;   // injete a INTERFACE, não BCryptPasswordEncoder

    public void cadastrar(String email, String senhaPura) {
        Usuario u = new Usuario();
        u.setEmail(email);
        u.setSenha(passwordEncoder.encode(senhaPura));   // gera {bcrypt}$2a$10$...
        repository.save(u);
    }
}
```

Dois detalhes que confundem:

- **Injete `PasswordEncoder` (a interface), não `BCryptPasswordEncoder`.** Assim você troca a implementação sem mexer no serviço — e é o `DelegatingPasswordEncoder` que está no contexto, não o BCrypt puro.
- O `matches(senhaPura, hashSalvo)` lê o prefixo `{bcrypt}` para saber como verificar. Você nunca compara senhas "na mão" — quem faz isso é o Spring, durante a autenticação.

---

## 2. UserDetailsService

O Spring Security não sabe nada sobre a sua tabela de usuários. O `UserDetailsService` é a ponte: ele ensina o Spring a **carregar um usuário pelo identificador** (e-mail, no nosso caso).

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UsuarioRepository repository;

    @Override
    public UserDetails loadUserByUsername(String email) {
        Usuario usuario = repository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("Usuário não encontrado"));

        return User.builder()
            .username(usuario.getEmail())
            .password(usuario.getSenha())             // hash BCrypt já salvo
            .authorities("ROLE_" + usuario.getRole()) // ex.: ROLE_USER
            .build();
    }
}
```

> 💡 **DaoAuthenticationProvider automático:** você não precisa configurar o "provedor de autenticação" manualmente. Quando o Spring encontra um `UserDetailsService` **e** um `PasswordEncoder` no contexto, ele monta um `DaoAuthenticationProvider` sozinho, conectando os dois. Esse provider é quem, no login, busca o usuário pelo `UserDetailsService` e compara a senha com o `PasswordEncoder`. É "mágica" que na verdade é só convenção.

---

## 3. JwtService

A peça que **gera** e **valida** tokens. Aqui está a API nova do jjwt 0.12.x (bem diferente das versões antigas — `subject()` em vez de `setSubject()`, `parseSignedClaims()` em vez de `parseClaimsJws()`).

```java
@Service
public class JwtService {

    private final SecretKey key;
    private final long expiration;

    public JwtService(@Value("${jwt.secret}") String secret,
                      @Value("${jwt.expiration}") long expiration) {
        // decodifica a chave Base64 e constrói a SecretKey HMAC
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
        this.expiration = expiration;
    }

    // ===== GERAR (usado no login) =====
    public String gerarToken(UserDetails usuario) {
        Instant agora = Instant.now();
        return Jwts.builder()
            .subject(usuario.getUsername())                       // quem é (o "sub")
            .issuedAt(Date.from(agora))                           // quando foi emitido
            .expiration(Date.from(agora.plusMillis(expiration)))  // quando expira
            .signWith(key)                                        // assina com a chave secreta
            .compact();                                           // produz a string final
    }

    // ===== VALIDAR / LER (usado no filtro) =====
    public String extrairSubject(String token) {
        return parseClaims(token).getSubject();
    }

    public boolean tokenValido(String token, UserDetails usuario) {
        try {
            Claims claims = parseClaims(token);
            boolean mesmoUsuario = claims.getSubject().equals(usuario.getUsername());
            boolean naoExpirou = claims.getExpiration().after(new Date());
            return mesmoUsuario && naoExpirou;
        } catch (JwtException e) {
            return false;   // assinatura inválida, token malformado ou expirado
        }
    }

    private Claims parseClaims(String token) {
        return Jwts.parser()
            .verifyWith(key)            // verifica a assinatura com a chave
            .build()
            .parseSignedClaims(token)   // lança JwtException se algo estiver errado
            .getPayload();
    }
}
```

O ponto-chave: `signWith(key)` na geração e `verifyWith(key)` na validação usam a **mesma chave secreta**. Quem não tem a chave não consegue gerar um token que passe no `verifyWith`. Se o token for adulterado, `parseSignedClaims` lança `JwtException`.

---

## 4. JwtAuthenticationFilter

Este é o "porteiro" do Fluxo 2. Ele intercepta **toda** requisição, procura o token no cabeçalho `Authorization`, valida e — se estiver tudo certo — avisa ao Spring que o usuário está autenticado.

Estende `OncePerRequestFilter` para garantir que roda **uma vez por requisição** (sem isso, poderia rodar várias vezes em encaminhamentos internos).

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final CustomUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        // Sem "Bearer ..."? Deixa passar — outra parte da cadeia decide (vai dar 401 se a rota for protegida)
        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);           // remove o prefixo "Bearer "
        String email = jwtService.extrairSubject(token);

        // Só autentica se ainda não houver autenticação no contexto
        if (email != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails usuario = userDetailsService.loadUserByUsername(email);

            if (jwtService.tokenValido(token, usuario)) {
                var auth = new UsernamePasswordAuthenticationToken(
                        usuario, null, usuario.getAuthorities());
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                // ESTA linha é o que "loga" o usuário nesta requisição:
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }

        filterChain.doFilter(request, response);   // sempre continua a cadeia
    }
}
```

O coração do filtro é `SecurityContextHolder.getContext().setAuthentication(auth)` — é essa linha que faz o Spring considerar o usuário autenticado **para esta requisição**. Sem ela, o token seria lido mas ignorado.

---

## 5. SecurityConfig

Agora juntamos tudo. Esta é a configuração stateless, e cada linha tem um motivo.

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(Customizer.withDefaults())                 // habilita CORS (libera o preflight)
            .csrf(csrf -> csrf.disable())                    // stateless não usa CSRF
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()      // login e cadastro: PÚBLICOS
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()                 // todo o resto exige token
            )
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))  // sem sessão no servidor
            // insere o filtro JWT ANTES do filtro padrão de usuário/senha:
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    // Expõe o AuthenticationManager para o AuthController usar no login
    @Bean
    AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

Decisões importantes, uma a uma:

- **`csrf.disable()`** — CSRF protege contra ataques que dependem de cookies de sessão. Como somos stateless (token no header, sem cookie de sessão), esse vetor não se aplica. **Só desabilite porque é stateless** — num app com sessão, isso seria falha de segurança.
- **`permitAll()` em `/auth/**`** — o problema do **ovo e a galinha**: para fazer login você ainda não tem token; logo, o endpoint de login **precisa** ser público. Se ele exigisse autenticação, ninguém nunca conseguiria logar.
- **`STATELESS`** — diz ao Spring para não criar nem usar sessão. Cada requisição se sustenta sozinha pelo token. Por isso também não usamos `formLogin`/`httpBasic`.
- **`addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)`** — posiciona o nosso filtro **antes** do filtro padrão de login por formulário na cadeia. Assim, quando uma requisição chega com token, ele já é processado e o usuário é autenticado antes de o Spring tentar outros mecanismos.
- **`AuthenticationManager` como bean** — precisamos dele exposto para chamar no endpoint de login (próximo passo). Por padrão ele não fica acessível; este bean o expõe.

---

## 6. Endpoint de login

Aqui o Fluxo 1 se completa: recebe credenciais, valida com o `AuthenticationManager`, e se passar, emite o token.

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final CustomUserDetailsService userDetailsService;
    private final JwtService jwtService;

    @PostMapping("/login")
    public TokenResponse login(@RequestBody LoginRequest req) {
        // valida e-mail + senha (dispara o DaoAuthenticationProvider por baixo)
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.email(), req.senha()));

        // se não lançou exceção, as credenciais são válidas → emite o token
        UserDetails usuario = userDetailsService.loadUserByUsername(req.email());
        String token = jwtService.gerarToken(usuario);
        return new TokenResponse(token);
    }

    public record LoginRequest(String email, String senha) {}
    public record TokenResponse(String token) {}
}
```

A chamada `authenticationManager.authenticate(...)` é onde a senha é de fato verificada: ela aciona o `DaoAuthenticationProvider`, que usa o `UserDetailsService` para buscar o usuário e o `PasswordEncoder` para comparar a senha. Se a senha estiver errada, ele lança `BadCredentialsException` (que vira 401) e o código nem chega a gerar token.

---

## 7. CORS e o preflight

CORS é uma regra **do navegador**: por segurança, ele bloqueia chamadas de um site (origem) para uma API em outra origem, a menos que a API **autorize** explicitamente. Isso aparece quando seu front (ex.: React em `localhost:5173`) chama a API em `localhost:8080`.

```java
@Configuration
public class CorsConfig {

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:5173", "https://meuapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

O detalhe que **mais confunde**: o **preflight**. Antes de um `POST`/`PUT`/`DELETE` "complexo", o navegador envia automaticamente uma requisição `OPTIONS` (o "preflight") para perguntar "ei API, posso fazer essa chamada?". Essa requisição vem **sem o token**. Se a sua segurança exigir autenticação para tudo, o preflight toma 401 e a chamada real nem acontece.

A solução é o `http.cors(...)` que colocamos no `SecurityConfig`: ele faz o Spring Security tratar o CORS (e liberar o preflight `OPTIONS`) **antes** da autenticação. Por isso o `cors()` na config não é decorativo — é o que destrava o front.

---

## 8. Swagger com o botão Authorize

O springdoc gera a documentação interativa em `/swagger-ui.html`. Mas, por padrão, ele não sabe que sua API usa JWT — você não consegue testar endpoints protegidos. A config abaixo adiciona o botão **Authorize** (onde você cola o token):

```java
@Configuration
@OpenAPIDefinition(
    info = @Info(title = "Minha API", version = "1.0"),
    security = @SecurityRequirement(name = "bearerAuth")
)
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT"
)
public class OpenApiConfig { }
```

Com isso, o Swagger mostra o cadeado nos endpoints e o botão **Authorize**. Você loga pelo `/auth/login`, copia o token, clica em Authorize, cola, e a partir daí o Swagger envia o `Authorization: Bearer ...` automaticamente em cada teste. Lembre de liberar as rotas do Swagger no `SecurityConfig` (já fizemos: `/swagger-ui/**` e `/v3/api-docs/**`).

---

## Recapitulando o fluxo completo

Juntando as peças na ordem em que elas colaboram:

1. **Cadastro:** `UsuarioService` codifica a senha com BCrypt e salva.
2. **Login (`POST /auth/login`):** `AuthController` chama o `AuthenticationManager`, que aciona o `DaoAuthenticationProvider` (montado automaticamente a partir do `UserDetailsService` + `PasswordEncoder`). Senha confere → `JwtService.gerarToken()` emite o JWT.
3. **Requisição protegida:** o cliente manda `Authorization: Bearer <token>`. O `JwtAuthenticationFilter` intercepta, `JwtService` valida a assinatura, e o usuário é colocado no `SecurityContext`.
4. **Autorização:** o `authorizeHttpRequests` decide se aquele usuário pode acessar a rota.
5. Tudo isso **sem sessão** no servidor — cada requisição se prova sozinha pelo token.

---

## Dificuldades comuns

Os tropeços mais frequentes (e por que acontecem):

1. **403/401 no front mas funciona no Postman** → quase sempre é **CORS/preflight**. O Postman não faz preflight; o navegador faz. Confira o `http.cors()` e a liberação do `OPTIONS`.
2. **"Por que minha role não funciona?"** → o `hasRole("ADMIN")` espera a authority `ROLE_ADMIN` (com prefixo). No `UserDetailsService`, gere `"ROLE_" + role`.
3. **`@PreAuthorize` ignorado** → faltou `@EnableMethodSecurity`. Sem ela, a anotação é silenciosamente ignorada — parece protegido, mas não está.
4. **Código jjwt que não compila** → você misturou a API antiga (`setSubject`, `parseClaimsJws`) com a 0.12.x (`subject`, `parseSignedClaims`). Use só a nova.
5. **Login retorna 401 com senha certa** → a senha no banco não está codificada com o mesmo encoder, ou foi salva sem prefixo. Sempre salve via `passwordEncoder.encode(...)`.
6. **`WeakKeyException` ao gerar o token** → a chave secreta é curta demais. Para HS256, use uma chave Base64 de pelo menos 256 bits (32 bytes).
7. **Esqueceu de liberar `/auth/**`** → cai no ovo-e-galinha: ninguém consegue logar porque o login exige login.

> ⚠️ Nota de segurança honesta: JWT stateless tem um custo — **você não consegue "deslogar" um token do lado do servidor** antes de ele expirar (não há sessão para invalidar). Por isso, use expiração curta e, se precisar de logout real ou revogação, combine com *refresh tokens* e uma lista de revogação. Para muitos casos, delegar a um provedor (veja o guia de OAuth/OIDC) é mais simples e seguro que manter isso à mão.

---

## Referências

1. **jjwt (repositório oficial)** — https://github.com/jwtk/jjwt
2. **Spring Security — Architecture** — https://docs.spring.io/spring-security/reference/servlet/architecture.html
3. **springdoc-openapi** — https://springdoc.org/
4. **JWT — introdução** — https://jwt.io/introduction
5. **Guia complementar:** [Configuração do Spring Security](guia-configuracao-spring-security.md) e [OAuth 2.0 / OIDC](guia-login-cadastro-oauth2-oidc.md)

> Reforço final: este guia mostra JWT "na mão" para fins de aprendizado e controle total. Em produção, pese sempre contra a opção de um provedor de identidade — menos código de segurança seu é menos superfície para errar.
