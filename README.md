# 📚 Catálogo de Meios Auxiliares para Programação

Coletânea curada de guias, referências e materiais de apoio para desenvolvimento de software, com foco em **Java**, Spring Boot e boas práticas de engenharia. Cada material é escrito para ser direto ao ponto, com exemplos reais e conexões com o que se usa no dia a dia.

> Use o índice abaixo para navegar. Cada link leva ao guia completo.

---

## ☕ Java

### Padrões de Projeto
| Guia | Descrição |
|---|---|
| [Padrões de Projeto em Java (com ORM)](java/design-patterns/guia-padroes-projeto-java-orm.md) | Os 23 padrões GoF + padrões de persistência (Fowler/PoEAA), cada um conectado a onde aparece no JPA/Hibernate/Spring Data. |

### Spring Security & Autenticação
| Guia | Descrição |
|---|---|
| [Configuração do Spring Security](java/spring-security/guia-configuracao-spring-security.md) | A fundação: cadeia de filtros, SecurityFilterChain com Lambda DSL (SS7), autorização por requisição e por método, codificação de senhas, UserDetailsService, CSRF, CORS e testes. |
| [Login e Cadastro com OAuth 2.0 / OpenID Connect](java/spring-security/guia-login-cadastro-oauth2-oidc.md) | Autenticação moderna com Spring Boot 4.1 + Spring Security 7: fluxo Authorization Code + PKCE, login social, provisionamento de usuário (JIT) e proteção de API com JWT. |
| [Autenticação JWT Stateless (prático)](java/spring-security/guia-jwt-na-pratica.md) | Implementação completa de login com JWT feito à mão (jjwt 0.12.6): os dois fluxos, JwtService, JwtAuthenticationFilter, AuthenticationManager, CORS/preflight e Swagger com Authorize. |

### Arquitetura & Organização
| Guia | Descrição |
|---|---|
| [Estrutura de Arquivos em Projetos Java](java/arquitetura/guia-estrutura-de-arquivos.md) | Convenções Maven/Gradle, Package by Layer vs. Package by Feature, arquitetura Hexagonal/Clean e nomenclatura. |

### Spring Boot
| Guia | Descrição |
|---|---|
| [Tratamento de Exceptions](java/spring-boot/guia-tratamento-de-exceptions.md) | Prático: exceptions customizadas enxutas, GlobalExceptionHandler com @RestControllerAdvice + ProblemDetail, validação, erros de segurança, tabela de referência (exception → status) e checklist. |

### Testes
| Guia | Descrição |
|---|---|
| [JUnit (testes em Java)](java/testes/guia-definitivo-junit.md) | Prático e completo: JUnit 6, AssertJ, testes parametrizados, Mockito, a pirâmide de testes, slices (@WebMvcTest/@DataJpaTest), @MockitoBean, testes de segurança, Testcontainers e JaCoCo. |

### Clean Code
*(em construção)*

---

## 🔧 Controle de Versão

| Guia | Descrição |
|---|---|
| [Git (guia completo)](ferramentas/git/git-completo.md) | Dos conceitos ao fluxo profissional: as três áreas, branches, merge/rebase, remotos e GitHub, desfazer (reset/reflog), stash, tags, .gitignore, Conventional Commits, GitHub Flow/Git Flow, Pull Requests, troubleshooting e cheat sheet. |

## 🛠️ Ferramentas & Infraestrutura

| Guia | Descrição |
|---|---|
| [Docker](ferramentas/docker/guia-definitivo-docker.md) | Containers do zero ao deploy: imagens, Dockerfile multi-stage para Spring Boot, volumes, redes e Docker Compose com PostgreSQL. |
| [Kubernetes](ferramentas/kubernetes/guia-definitivo-kubernetes.md) | Orquestração: arquitetura do cluster, objetos fundamentais, kubectl, deploy de Spring Boot, probes com Actuator, HPA e Helm. |
| [Postman](ferramentas/postman/guia-definitivo-postman.md) | Testando APIs: coleções, variáveis/environments, captura automática de token JWT, testes com `pm.test`, Collection Runner, import de OpenAPI e Newman no CI. |
| [Swagger / OpenAPI](ferramentas/swagger/guia-definitivo-swagger.md) | Documentação que se gera do código com springdoc 3.x (Spring Boot 4): anotações, metadados, botão Authorize (JWT), validação como doc e cuidados em produção. |

## 🗄️ Bancos de Dados

| Guia | Descrição |
|---|---|
| [PostgreSQL](bancos-de-dados/postgresql/guia-definitivo-postgresql.md) | Do básico ao profissional: tipos (JSONB, UUID), índices, EXPLAIN, integração com Spring Boot/JPA, Flyway e backup. |
| [Flyway (migrations)](bancos-de-dados/flyway/guia-definitivo-flyway.md) | Versionamento de schema: como funciona (histórico + checksum), seed por chave natural, migrations repetíveis, o erro de checksum e como recuperar em dev vs. produção. |

## 🐍 Python
*(em construção)*

## 💡 Conceitos (agnósticos de linguagem)
*(SOLID, algoritmos, estruturas de dados — em construção)*

---

## Como este catálogo é organizado

- **Por linguagem** no primeiro nível (`java/`, `python/`), porque é como a maioria busca material.
- **Por subtópico** dentro de cada linguagem (`design-patterns/`, `spring-security/`...).
- **Tópicos transversais** (ferramentas, conceitos) têm pastas próprias para não duplicar conteúdo.
- Arquivos em **kebab-case minúsculo, sem acentos** — gera URLs limpas no GitHub.

Cada guia segue um template comum: **título → resumo de uma linha → índice → conteúdo → referências.**

---

## Contribuindo

Sugestões, correções e novos materiais são bem-vindos. Veja [CONTRIBUTING.md](CONTRIBUTING.md).

## Licença

Conteúdo textual sob [Creative Commons Attribution 4.0 (CC BY 4.0)](LICENSE). Os trechos de código nos guias podem ser usados livremente. Veja o arquivo [LICENSE](LICENSE) para detalhes.
